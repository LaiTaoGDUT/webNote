

### **一、Vue2 的响应式实现**

#### **1. 核心机制：`Object.defineProperty`**
- **实现步骤**：
  1. **递归劫持**：遍历对象所有属性，递归调用 `defineReactive`。

  2. **依赖收集**：在 `getter` 中通过 `Dep` 收集依赖（`Watcher` 实例）。

  3. **触发更新**：在 `setter` 中通过 `Dep` 通知所有关联的 `Watcher` 更新。

- **关键代码片段**：

  ```javascript
  function defineReactive(obj, key, val) {
    const dep = new Dep(); // 每个属性对应一个 Dep
    Object.defineProperty(obj, key, {
      get() {
        if (Dep.target) { // Dep.target 是当前 Watcher
          dep.depend();   // 将 Watcher 添加到 Dep 的订阅列表
        }
        return val;
      },
      set(newVal) {
        if (newVal === val) return;
        val = newVal;
        dep.notify(); // 通知所有 Watcher 执行更新
      }
    });
  }
  ```

- **局限性**：

  - **动态属性**：无法检测新增属性（需使用 `Vue.set`），因其初始化时未定义 `getter/setter`。

  - **数组监听**：需重写数组方法（如 `push`, `pop`），通过原型链拦截方法调用：

    ```javascript
    const arrayProto = Array.prototype;
    const arrayMethods = Object.create(arrayProto);
    ['push', 'pop'].forEach(method => {
      const original = arrayProto[method];
      def(arrayMethods, method, function mutator(...args) {
        const result = original.apply(this, args);
        this.__ob__.dep.notify(); // 手动触发更新
        return result;
      });
    });
    ```

  - **性能问题**：递归劫持深层对象时，初始化时间呈指数级增长。

---

### **二、Vue3 的响应式实现**

#### **1. 核心机制：`Proxy` 与 `Reflect`**
- **实现步骤**：
  1. **创建代理**：通过 `Proxy` 包装目标对象，拦截 13 种操作（如 `get`, `set`, `deleteProperty`）。
  2. **惰性代理**：仅在访问属性时递归代理子对象，避免初始化性能损耗。
  3. **依赖追踪**：使用 `WeakMap → Map → Set` 三层结构存储依赖关系。

- **关键代码片段**：
  ```javascript
  const targetMap = new WeakMap(); // 存储目标对象到依赖的映射

  function reactive(obj) {
    return new Proxy(obj, {
      get(target, key, receiver) {
        track(target, key); // 收集依赖
        const res = Reflect.get(target, key, receiver);
        // 惰性代理：若值为对象，递归转换为响应式
        return typeof res === 'object' ? reactive(res) : res;
      },
      set(target, key, value, receiver) {
        const oldValue = target[key];
        const success = Reflect.set(target, key, value, receiver);
        if (success && oldValue !== value) {
          trigger(target, key); // 触发更新
        }
        return success;
      },
      deleteProperty(target, key) {
        const hadKey = Object.prototype.hasOwnProperty.call(target, key);
        const success = Reflect.deleteProperty(target, key);
        if (success && hadKey) {
          trigger(target, key); // 触发删除操作更新
        }
        return success;
      }
    });
  }
  ```

#### **2. 依赖收集与触发**
- **`track` 函数**：
  ```javascript
  function track(target, key) {
    if (!activeEffect) return; // activeEffect 是当前运行的副作用
    let depsMap = targetMap.get(target);
    if (!depsMap) {
      targetMap.set(target, (depsMap = new Map()));
    }
    let dep = depsMap.get(key);
    if (!dep) {
      depsMap.set(key, (dep = new Set()));
    }
    dep.add(activeEffect); // 将副作用添加到依赖集合
  }
  ```

- **`trigger` 函数**：
  ```javascript
  function trigger(target, key) {
    const depsMap = targetMap.get(target);
    if (!depsMap) return;
    const effects = depsMap.get(key);
    effects && effects.forEach(effect => {
      if (effect.options.scheduler) {
        effect.options.scheduler(effect); // 支持自定义调度
      } else {
        effect(); // 直接执行副作用
      }
    });
  }
  ```

#### **3. 对数组操作的额外拦截机制**

- **关键拦截点**

    Vue3 对数组代理使用额外逻辑处理，使响应式可以覆盖以下场景：

    - **直接修改索引**：`arr[0] = 1`
    - **修改 `length`**：`arr.length = 0`
    - **调用方法修改数组**：`arr.push()`, `arr.splice()` 等

- **核心代码逻辑**

    在 Vue3 的响应式模块中，`set`陷阱对数组进行额外的处理，包含以下关键逻辑：

    ```javascript
    // Proxy 的 set 陷阱处理数组
    function set(target: Array<any>, key: string | symbol, value: any, receiver: object): boolean {
    const oldLength = target.length;
    const hadKey = Array.isArray(target) 
        ? (Number(key) < target.length) // 判断是修改已有索引还是新增元素
        : hasOwn(target, key);
    
    const result = Reflect.set(target, key, value, receiver);

    // 触发更新（分两种情况）
    if (!hadKey) {
        // 新增属性（如通过索引添加元素）
        trigger(target, TriggerOpTypes.ADD, key, value);
    } else if (hasChanged(value, oldValue)) {
        // 修改已有属性
        trigger(target, TriggerOpTypes.SET, key, value, oldValue);
    }

    // 处理 length 变化
    if (Array.isArray(target) && key === 'length') {
        // 直接修改 length（如 arr.length = 2）
        const newLength = Number(value);
        for (let i = newLength; i < oldLength; i++) {
        trigger(target, TriggerOpTypes.DELETE, i.toString());
        }
    } else if (Array.isArray(target)) {
        // 隐式修改 length（如 arr[10] = 1）
        const lengthKey = 'length';
        const newLength = target.length;
        if (newLength !== oldLength && hasOwn(target, lengthKey)) {
        trigger(target, TriggerOpTypes.SET, lengthKey);
        }
    }

    return result;
    }
    ```



- **核心处理流程**

    1. **直接修改 `length`**

        当执行 `arr.length = 2` 时：
        - **Proxy 触发 `set` 陷阱**，`key` 为 `'length'`。
        - **对比新旧长度**：若新长度 < 旧长度，遍历被删除的索引（如索引 2 到旧长度-1），触发 `DELETE` 操作。
        - **触发 `length` 的更新**：标记 `length` 属性变化。

    2. **通过索引隐式修改 `length`**

        当执行 `arr[5] = 10` 且原数组长度 < 5 时：

        - **Proxy 触发 `set` 陷阱**，`key` 为 `'5'`。
        - **检测到索引超过原长度**，若数组长度被隐形增加，比如执行 `arr[5] = 10`后数组的 `length` 自动变为 6。
        - **对比新旧 `length`**，新旧长度不一致，触发 `length` 属性的 `SET` 操作。

    3. **调用数组方法**
        当执行 `arr.push(1)` 时：
        - **Proxy 拦截方法调用**：`push` 方法会先读取 `length`（触发 `get` 陷阱），再修改 `length` 和索引`1`（触发多次 `set` 陷阱）。
        - **自动处理依赖**：无需重写方法，所有操作会通过 Proxy 拦截后触发更新。

#### **4. 性能优化细节**
- **惰性代理（Lazy Proxy）**：
  - 仅在访问对象属性时创建子对象的 Proxy，避免初始化时深度遍历。
  - 示例：访问 `obj.a.b` 时，`obj.a` 返回后才会代理 `obj.a.b`。

- **依赖粒度优化**：
  - 每个属性对应独立的 `Set` 集合存储副作用，更新时仅触发相关依赖。
  - 对比 Vue2 的组件级 `Watcher`，Vue3 的更新触发路径更短。

- **内存优化**：
  - 使用 `WeakMap` 自动清理未被引用的对象依赖，防止内存泄漏。
  - 避免 Vue2 中为每个属性创建 `Dep` 实例的内存开销。

---

### **三、关键技术差异对比**

#### **1. 数组处理机制**
| **操作类型**       | **Vue2**                              | **Vue3**                              |
|--------------------|---------------------------------------|---------------------------------------|
| **索引修改**       | `arr[0] = 1` 无法触发更新             | 直接触发 `set` 拦截                   |
| **length 修改**    | `arr.length = 0` 无法触发更新         | 触发 `set` 拦截并检测 `length` 变化   |
| **方法调用**       | 依赖重写的 7 个方法                   | 原生方法直接触发 Proxy 拦截           |

#### **2. 嵌套对象处理**
- **Vue2**：
  - **递归劫持**：初始化时深度遍历所有属性，立即创建 `getter/setter`。
  - **内存消耗**：深层对象导致大量 `Dep` 实例被创建。

- **Vue3**：
  - **按需代理**：仅在访问属性时递归创建 Proxy。
  - **代码示例**：
    ```javascript
    const obj = reactive({ a: { b: { c: 1 } } });
    // 访问 obj.a 时，才会代理 a 对象
    // 访问 obj.a.b 时，才会代理 b 对象
    ```

#### **3. 集合类型（Map/Set）支持**
- **Vue2**：  
  无法响应式监听 `Map` 和 `Set` 的操作（如 `map.set()`, `set.add()`）。

- **Vue3**：  
  通过 `collectionHandlers` 代理集合方法：
  ```javascript
  const mutableInstrumentations = {
    get(key) {
      const target = toRaw(this); // 获取原始对象
      track(target, 'get', key);  // 追踪依赖
      return target.get(key);
    },
    set(key, value) {
      const target = toRaw(this);
      const oldValue = target.get(key);
      target.set(key, value);
      if (oldValue !== value) {
        trigger(target, 'set', key); // 触发更新
      }
    }
  };
  ```

---

### **四、底层数据结构与算法**

#### **1. Vue2 的依赖管理**
- **数据结构**：
  ```javascript
  // Dep 类
  class Dep {
    constructor() {
      this.subs = []; // 存储 Watcher 实例
    }
    depend() {
      if (Dep.target) {
        this.subs.push(Dep.target); // Watcher 订阅 Dep
      }
    }
    notify() {
      this.subs.forEach(watcher => watcher.update());
    }
  }

  // Watcher 类
  class Watcher {
    constructor(vm, expOrFn, cb) {
      this.vm = vm;
      this.getter = parsePath(expOrFn);
      this.cb = cb;
      this.value = this.get();
    }
    get() {
      Dep.target = this; // 设置当前 Watcher
      const value = this.getter.call(this.vm, this.vm);
      Dep.target = null; // 重置
      return value;
    }
    update() {
      this.run();
    }
    run() {
      const value = this.get();
      if (value !== this.value) {
        this.cb.call(this.vm, value, this.value);
      }
    }
  }
  ```

#### **2. Vue3 的依赖管理**
- **数据结构**：
  ```javascript
  // 全局依赖存储
  const targetMap = new WeakMap();

  // 副作用函数（effect）
  let activeEffect;
  class ReactiveEffect {
    constructor(fn, scheduler) {
      this.fn = fn;         // 用户定义的副作用函数
      this.scheduler = scheduler; // 调度器（可选）
    }
    run() {
      activeEffect = this;
      return this.fn();
    }
  }

  // 创建副作用
  function effect(fn, options) {
    const _effect = new ReactiveEffect(fn, options?.scheduler);
    _effect.run();
    return _effect;
  }
  ```

---

### **五、极端场景处理**

#### **1. 循环引用**
- **Vue2**：  
  若对象存在循环引用（如 `obj.self = obj`），递归劫持会导致栈溢出。

- **Vue3**：  
  使用 `WeakMap` 缓存已代理对象，避免重复代理：
  ```javascript
  const reactiveMap = new WeakMap();
  function reactive(obj) {
    if (reactiveMap.has(obj)) {
      return reactiveMap.get(obj);
    }
    const proxy = new Proxy(obj, handlers);
    reactiveMap.set(obj, proxy);
    return proxy;
  }
  ```

#### **2. 大规模数据性能**
- **Vue2**：  
  初始化 10,000 个属性的对象需要约 200ms（递归劫持耗时）。

- **Vue3**：  
  初始化同样规模对象仅需约 50ms（惰性代理 + Proxy 高效拦截）。

---

### **总结**
Vue3 的响应式系统通过 `Proxy` 的动态拦截、`WeakMap` 的依赖存储和惰性代理机制，实现了 **更细粒度的依赖追踪**、**更低的内存消耗** 和 **更高的运行性能**，同时解决了 Vue2 在动态属性、数组操作和循环引用等场景下的固有问题。其技术细节体现了现代 JavaScript 语言特性（如 Proxy/Reflect）与高效数据结构的深度结合。

Vue3 通过 **Proxy 代理数组**并配合 **特殊处理逻辑**，能够精准捕获包括 `length` 变化在内的所有数组操作。以下是详细实现原理：











---

### **一、Proxy 对数组操作的拦截机制**

#### **1. 关键拦截点**
Vue3 的数组代理主要依赖 Proxy 的 **`set` 陷阱**和 **`deleteProperty` 陷阱**，覆盖以下场景：
- **直接修改索引**：`arr[0] = 1`
- **修改 `length`**：`arr.length = 0`
- **调用方法修改数组**：`arr.push()`, `arr.splice()` 等

#### **2. 核心代码逻辑**
在 Vue3 的响应式模块中，数组的代理处理器（`baseHandlers.ts`）包含以下关键逻辑：
```javascript
// 伪代码：Proxy 的 set 陷阱处理数组
function set(target: Array<any>, key: string | symbol, value: any, receiver: object): boolean {
  const oldLength = target.length;
  const hadKey = Array.isArray(target) 
    ? (Number(key) < target.length) // 判断是修改已有索引还是新增元素
    : hasOwn(target, key);
  
  const result = Reflect.set(target, key, value, receiver);

  // 触发更新（分两种情况）
  if (!hadKey) {
    // 新增属性（如通过索引添加元素）
    trigger(target, TriggerOpTypes.ADD, key, value);
  } else if (hasChanged(value, oldValue)) {
    // 修改已有属性
    trigger(target, TriggerOpTypes.SET, key, value, oldValue);
  }

  // 处理 length 变化
  if (Array.isArray(target) && key === 'length') {
    // 直接修改 length（如 arr.length = 2）
    const newLength = Number(value);
    for (let i = newLength; i < oldLength; i++) {
      trigger(target, TriggerOpTypes.DELETE, i.toString());
    }
  } else if (Array.isArray(target)) {
    // 隐式修改 length（如 arr[10] = 1）
    const lengthKey = 'length';
    const newLength = target.length;
    if (newLength !== oldLength && hasOwn(target, lengthKey)) {
      trigger(target, TriggerOpTypes.SET, lengthKey);
    }
  }

  return result;
}
```

---

### **二、具体场景分析**

#### **1. 直接修改 `length`**
当执行 `arr.length = 2` 时：
- **Proxy 触发 `set` 陷阱**，`key` 为 `'length'`。
- **对比新旧长度**：若新长度 < 旧长度，遍历被删除的索引（如索引 2 到旧长度-1），触发 `DELETE` 操作。
- **触发 `length` 的更新**：标记 `length` 属性变化。

#### **2. 通过索引隐式修改 `length`**
当执行 `arr[5] = 10` 且原数组长度 < 5 时：
- **Proxy 触发 `set` 陷阱**，`key` 为 `'5'`。
- **检测到索引超过原长度**，数组的 `length` 自动变为 6。
- **对比新旧 `length`**，触发 `length` 属性的 `SET` 操作。

#### **3. 调用数组方法**
当执行 `arr.push(1)` 时：
- **Proxy 拦截方法调用**：`push` 方法会先读取 `length`（触发 `get` 陷阱），再修改 `length` 和索引（触发多次 `set` 陷阱）。
- **自动处理依赖**：无需重写方法，所有操作通过 Proxy 拦截后触发更新。

---

### **三、依赖触发流程**

#### **1. 依赖存储结构**
Vue3 使用三层结构存储依赖关系：
```javascript
targetMap: WeakMap<Target, Map<Key, Dep>> = {
  [array]: {
    '0': Set([effect1, effect2]), // 索引 0 的依赖
    'length': Set([effect3])      // length 的依赖
  }
}
```

#### **2. 触发更新逻辑**
- **修改索引**：触发对应索引的依赖（如 `'0'`）。
- **修改 `length`**：触发 `'length'` 的依赖，并根据长度变化触发被删除索引的 `DELETE` 操作。

---

### **四、与 Vue2 的对比**

| **特性**               | **Vue2**                          | **Vue3**                          |
|------------------------|-----------------------------------|-----------------------------------|
| **length 修改检测**    | 无法直接检测                     | 直接通过 Proxy 拦截               |
| **索引越界赋值**       | 无法触发更新                     | 自动触发 `length` 和索引更新      |
| **数组方法处理**       | 需重写 7 个方法                  | 原生方法直接触发 Proxy 拦截       |
| **性能开销**           | 每个数组需重写方法               | 无额外开销，依赖 Proxy 原生能力   |

---

### **五、源码关键实现**

#### **1. 数组的 `set` 陷阱处理（`baseHandlers.ts`）**
```typescript
// 实际源码节选
function createArrayInstrumentations() {
  const instrumentations: Record<string, Function> = {};
  ['push', 'pop', 'shift', 'unshift', 'splice'].forEach(key => {
    instrumentations[key] = function (this: unknown[], ...args: unknown[]) {
      pauseTracking(); // 暂停依赖收集
      const res = (Array.prototype as any)[key].apply(this, args);
      resetTracking(); // 恢复依赖收集
      return res;
    };
  });
  return instrumentations;
}

// 代理 get 陷阱处理数组方法
function get(target: Target, key: string | symbol, receiver: object) {
  if (Array.isArray(target) && hasOwn(arrayInstrumentations, key)) {
    return Reflect.get(arrayInstrumentations, key, receiver);
  }
  // 其他逻辑...
}
```

#### **2. `length` 变化的触发逻辑**
```typescript
// 触发 length 更新
if (key === 'length' && Array.isArray(target)) {
  const oldLength = oldValue as number;
  const newLength = value as number;
  if (newLength < oldLength) {
    // 触发被删除索引的清理
    for (let i = newLength; i < oldLength; i++) {
      trigger(target, TriggerOpTypes.DELETE, i.toString());
    }
  }
}
```

---

### **总结**
Vue3 通过 **Proxy 的动态拦截能力** 和 **精细的依赖触发逻辑**，实现了对数组 `length` 属性变化的精准监听。无论是直接修改 `length`，还是通过索引越界赋值隐式修改 `length`，甚至是调用原生数组方法，均能触发响应式更新。这一机制彻底解决了 Vue2 中数组监听不完整的痛点，且无需重写数组方法，性能和代码维护性显著提升。