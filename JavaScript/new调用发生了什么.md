在 JavaScript 中，使用 `new` 调用函数（构造函数）时，会触发以下过程：

---

### **0. 构造函数的 `prototype` 属性是如何生成的？**
在 JavaScript 中，**函数在创建时**（非调用时），引擎会自动为它生成一个 `prototype` 属性，规则如下：

1. **默认生成的 `prototype` 对象**  

   每个函数（构造函数）被定义时，都会自动生成一个 `prototype` 对象，包含以下内容：

   ```javascript
   function Person() {}
   Person.prototype = Object.create(Object.prototype);
   Person.prototype.constructor = Person; // 指向构造函数本身
   ```

2. **`constructor` 属性的意义**  

   - `constructor` 用于标识实例的构造函数。  

     **示例：**
     ```javascript
     const p = new Person();
     console.log(p.constructor === Person); // true（通过原型链访问）
     ```

3. **手动修改 `prototype` 的影响**  

   - 如果覆盖构造函数的 `prototype`（如 `Person.prototype = {}`），新对象的 `constructor` 会丢失，需手动修复：

     ```javascript
     Person.prototype = {
       method() {},
       // 需显式指定 constructor
       constructor: Person 
     };
     ```

---

### 1. **创建一个新对象**

   - 引擎会先创建一个空的普通对象（即 `{}`）。

   - 这个对象的原型（`[[Prototype]]`，即 `__proto__`）会被设置为构造函数的 `prototype` 属性。  
   
     **示例：**  
     ```javascript
     const obj = Object.create(Persion.prototype);
     ```

---

### 2. **绑定 `this` 到新对象**
   - 构造函数内部的 `this` 会指向这个新创建的对象。  

     **示例：**  
     ```javascript
     function Person(name) {
       this.name = name;
     }
     // 相当于
     // const obj = Object.create(Persion.prototype);
     // const person = Person.apply(obj)
     ```

---

### 3. **执行构造函数**

   - 执行构造函数内部的代码，通常会通过 `this` 给新对象添加属性或方法。  

     **示例：**  
     ```javascript
     const p = new Person("John");
     console.log(p.name); // "John"（通过 this 赋值）
     ```

---

### 4. **处理返回值**

   - 如果构造函数显式返回一个**对象**（如 `return { age: 30 }`），则 `new` 会丢弃自己创建的对象，直接返回这个对象。

   - 如果返回的是**原始值**（如 `return 123`），则忽略返回值，依然返回 `new` 创建的对象。  

     **示例：**  
     ```javascript
     function Person() {
       return { age: 30 }; // 返回对象，自动创建的obj被丢弃
     }
     const p = new Person();
     console.log(p); // { age: 30 }（而非 Person 的实例）
     ```

---

### 手写`new`调用：
```javascript
function _new(constructorFunc, ...args) {
  // 1. 创建新对象，并绑定原型
  const obj = Object.create(constructorFunc.prototype);
  
  // 2. 执行构造函数，将 this 指向新对象
  const result = constructorFunc.apply(obj, args);
  
  // 3. 处理返回值
  return result instanceof Object ? result : obj;
}
```

---

### 注意事项
- **`new`调用绑定与其他`this`绑定的优先级**：

    `new`调用的优先级是最高的，超过其他`this`绑定。

- **忘记 `new` 的后果**：  

  如果直接调用构造函数（如 `Person()`），`this` 会指向全局对象（浏览器中是 `window`），可能导致意外错误。  
  解决方法：在构造函数内部检查 `this`（如 `if (!(this instanceof Person)) return new Person(...)`）。

- **以下函数不能作为构造函数**：  
  ```javaScript
    // 生成器函数
    function* generatorFunction() {}
    // 作为对象属性声明的函数
    const method = { foo() {} }.foo;
    // 箭头函数
    const arrowFunction = () => {};
    // 异步函数
    async function asyncFunction() {}
    // Symbol
    Symbol()
    // BigInt
    BigInt()
  ```