### **1. 原始 `.vue` 文件解析**

#### **1.1 文件内容读取**
- Webpack 通过入口文件或依赖分析找到 `.vue` 文件，将其内容作为字符串传递给 `vue-loader`。

#### **1.2 使用 `@vue/compiler-sfc` 解析**

- **生成 AST 和 Descriptor**：  

  `@vue/compiler-sfc` 的 `parse` 方法将 `.vue` 文件解析为一个 **Descriptor 对象**，结构如下：

  ```js
  {
    template: { content: '<div>...</div>', attrs: { lang: 'html' }, ... },
    script: { content: 'export default {...}', attrs: { lang: 'ts' }, ... },
    styles: [ { content: '.class{...}', attrs: { scoped: true }, ... } ],
    customBlocks: [ ... ] // 如 <docs> 块
  }
  ```

- **块分割**：  
  每个块（`template`、`script`、`style`、自定义块）被独立提取，并记录其属性（如 `lang`、`scoped`）。

---

### **2. 生成带虚拟模块引入的js代码**

#### **2.1 将vue文件转成普通js模块**

解析得到**Descriptor 对象**后，将vue文件转换成带虚拟模块引入的js代码并返回，内部实现类似如下：

```js
// vue-loader/lib/index.js
module.exports = function (source) {
  // 1. 解析 .vue 文件
  const { descriptor } = compiler.parse(source);
  
  // 2. 生成各块的虚拟模块路径
  const templateRequest = `${this.resourcePath}?vue&type=template`;
  const scriptRequest = `${this.resourcePath}?vue&type=script`;
  
  // 3. 生成导入语句
  let code = '';
  code += `import script from ${JSON.stringify(scriptRequest)};\n`;
  code += `import { render } from ${JSON.stringify(templateRequest)};\n`;
  code += `script.render = render;\n`;
  code += `export default script;\n`;
  
  return code;
};
```

`vue-loader`处理`vue`文件后，返回了类似下面的js模块代码提供给webpack：

```js
import script from './component.vue?vue&type=script'
import { render } from './component.vue?vue&type=template'
import './component.vue?vue&type=style&index=0'

script.render = render

export default script
```

#### **2.2. 生成了带查询参数的虚拟模块路径**

上文看到，`vue-loader` 处理 `.vue` 文件时，会通过 `@vue/compiler-sfc` 解析单文件组件，拆分成多个块（如 `<template>`、`<script>`、`<style>`）。针对每个块，`vue-loader` 生成了一个 **虚拟模块的导入请求**，格式为：  
`原始文件路径 + 查询参数（标识块类型）`。

例如，对于一个 `component.vue` 文件，生成的虚拟模块路径可能是：
```js
// 模板块
import { render } from 'component.vue?vue&type=template'

// 脚本块
import script from 'component.vue?vue&type=script'

// 样式块
import 'component.vue?vue&type=style&index=0'
```

这里的查询参数 `?vue&type=xxx` 是关键，它用于告诉 `Webpack` 如何重新处理这些路径。

#### **2.3 Webpack 解析虚拟模块**

当 Webpack 发现返回的js代码里有 `import script from 'component.vue?vue&type=script'` 时：

1. 将其视为新的模块请求，并继续触发对应的 Loader 处理。

2. 根据路径 `component.vue` 和查询参数 `?vue&type=script`，还是匹配到 `vue-loader`继续处理。

3. `vue-loader` 根据参数带的 `type=script` 专门处理 `<script>` 块的内容。

4. 处理后的脚本内容会被其他自定义的 Loader（如 `babel-loader`）进一步处理。

#### **2.4 克隆 Loader 规则**

我们通常会对js代码和css代码应用一些自定义的loader进行处理，比如`babel-loader`、`style-loader`等，`vue`文件中也包含了js代码和css代码，因此我们期望也进行处理，但默认情况下这些loader无法识别vue里面对应的js块和css块。

`VueLoaderPlugin` 就可以帮助我们实现，它的作用是 **克隆应用在普通js和css的loader规则**，并添加 `resourceQuery` 条件，确保它们可以作用于带有特定查询参数的import请求。

例如，我们在webpack配置了原始规则：
```js
// 原始配置
{
  test: /\.js$/,
  loader: 'babel-loader'
}
```

经过 `VueLoaderPlugin` 克隆后，新增以下规则：
```js
{
  resourceQuery: /type=script/, // 仅匹配 ?vue&type=script 的请求
  loader: 'babel-loader'
}
```

这样，`vue`文件中 `<script>` 块的内容先由`vue-loader`提取成js模块后，会再由`babel-loader` 进行处理。


---

### **3. 分块处理（Per-Block Processing）**

生成了虚拟模块的import语句后，`vue-loader`继续针对不同type类型的import，将vue文件中不同的块处理成相应的js模块。

#### **3.1 `<template>` 块处理**
- **编译为渲染函数**：  
  使用 `@vue/compiler-dom` 将模板编译为 JavaScript 渲染函数。例如：
  ```js
  // 输入模板: <div>{{ msg }}</div>
  // 输出渲染函数:
  export function render(_ctx, _cache) {
    return _openBlock(), _createBlock("div", null, _toDisplayString(_ctx.msg), 1 /* TEXT */)
  }
  ```
- **处理 Source Map**：  
  将编译后的代码与原始模板的映射信息合并，生成最终的 Source Map。

#### **3.2 `<script>` 块处理**

- **语言转换**：  
  若 `<script>` 块使用 `lang="ts"`，Webpack 会调用 `ts-loader` 或 `babel-loader` 处理 TypeScript。

- **直接返回**
  直接返回`<script>`中的js代码，因为它们是可以被webpack直接处理的东西，如果对js代码用了其他loader，还会先经过其他loader处理后再返回。

#### **3.3 `<style>` 块处理**

- **作用域隔离（Scoped CSS）**：  
  通过 `postcss-modules` 插件为每个选择器添加唯一的 `data-v-xxxx` 属性。例如：
  ```css
  /* 原始样式 */
  .container { color: red; }
  
  /* 编译后 */
  .container[data-v-5f7d8e] { color: red; }
  ```
- **CSS Modules**：  
  若使用 `<style module>`，生成类名映射对象，例如：
  ```js
  // 注入到组件中的 $style
  export default {
    mounted() {
      console.log(this.$style.container) // 输出哈希后的类名
    }
  }
  ```

- **样式注入方式**：  
  - 开发环境：通过 `vue-style-loader` 动态注入 `<style>` 标签到 DOM。
  - 生产环境：使用 `mini-css-extract-plugin` 将样式提取到独立 CSS 文件。

---

### **4. 组装最终组件模块**

- 经过对三个虚拟模块import的解析返回后，再回看一开始生成的js模块代码，所有的import都解析成了普通的js代码，此时vue文件已经完全转化为了一个普通的js模块可被webpack正常打包，`vue-loader`的处理工作完成。
  ```js
  // 导入编译后的脚本和模板
  import script from './component.vue?vue&type=script'
  import { render } from './component.vue?vue&type=template'
  
  // 注入样式
  import './component.vue?vue&type=style&index=0'
  
  // 将渲染函数挂载到组件选项
  script.render = render
  
  // 导出 Vue 组件
  export default script
  ```

---

### **5. 源码流程示意图**
```text
1. Webpack 读取 .vue 文件
   │
2. vue-loader 调用 @vue/compiler-sfc 解析为 Descriptor
   │
3. 生成普通js模块代码，内部含有带查询参数的虚拟模块import语句（type=script/template/style）
   │
4. Webpack 针对生成的import语句，重新解析虚拟模块路径，匹配对应 Loader
   │
5. 各块处理：
   ├─ <template> → @vue/compiler-dom → 渲染函数
   ├─ <script> → babel-loader/ts-loader → 组件选项
   └─ <style> → css-loader + vue-style-loader → 样式注入
   │
6. 各模块均已解析为js模块，vue文件最终转化为普通的js模块代码
   │
7. 交给webpack打包处理js模块
```

---

### **6. 关键源码片段解析**

#### **`vue-loader/lib/index.js` 核心逻辑**

```js
module.exports = function (source) {
  const { descriptor } = compiler.parse(source)
  // 生成各块的 import 代码
  const templateImport = `import { render } from '${request}?vue&type=template'`
  const scriptImport = `import script from '${request}?vue&type=script'`
  // 返回组合后的代码
  return `
    ${templateImport}
    ${scriptImport}
    script.render = render
    export default script
  `
}
```

#### **`VueLoaderPlugin` 克隆规则**
```js
class VueLoaderPlugin {
  apply(compiler) {
    // 克隆 Webpack 的 module.rules，附加资源查询条件
    const rawRules = compiler.options.module.rules
    const clonedRules = cloneRules(rawRules, refs)
  }
}
```

---

### **总结**
`vue-loader` 的核心在于将 SFC 拆解为独立的虚拟模块，利用 Webpack 的模块化系统和 Loader 链分别处理每个块，最终合并为完整的 Vue 组件。通过 `@vue/compiler-sfc` 的深度集成和 `VueLoaderPlugin` 的规则管理，实现了对模板、脚本、样式的高效编译，同时支持了 HMR、Scoped CSS 等高级特性。理解这一流程有助于调试复杂构建问题，优化 Vue 项目的构建配置。