### 一、Webpack 基础概念

#### 1. Webpack 是什么？
**本质**：  
Webpack 是一个现代 JavaScript 应用程序的 **静态模块打包器**（Static Module Bundler）。它将项目中的所有文件（JS、CSS、图片、字体等）视为一个依赖关系图（Dependency Graph），通过入口文件递归构建依赖关系，最终生成优化后的静态资源。

**类比理解**：  
想象一个工厂流水线：
- **原料**：项目中的各种文件（JS/CSS/图片）
- **流水线**：Webpack 的打包流程
- **加工工具**：Loader（处理特定文件）和 Plugin（扩展功能）
- **成品**：优化后的 `bundle.js` 和静态资源

---

#### 2. 核心概念

##### (1) Entry（入口）
**定义**：  
指定 Webpack 从哪个文件开始构建依赖图（默认 `./src/index.js`）

**关键细节**：
- **单入口**（SPA 应用）：
  ```javascript
  entry: './src/index.js'
  ```
- **多入口**（MPA 应用）：
  ```javascript
  entry: {
    main: './src/index.js',
    admin: './src/admin.js'
  }
  ```
- **动态入口**（通过函数生成）：
  ```javascript
  entry: () => new Promise(resolve => {
    resolve('./src/dynamic-entry.js')
  })
  ```

    **常见误区**：  
    入口文件必须是 **JavaScript 文件**（Webpack 的核心是处理 JS 模块依赖）

---

##### (2) Output（输出）
**定义**：  
指定打包后的文件存放位置和命名规则

**配置详解**：
```javascript
output: {
  filename: '[name].[contenthash].js', // 输出文件名规则
  path: path.resolve(__dirname, 'dist'), // 绝对路径
  publicPath: '/assets/', //  HTML 文件或者其他资源中引用这些打包后资源的路径前缀
  clean: true // Webpack 5 自带清理功能（替代 clean-webpack-plugin）
}
```
- **占位符说明**：
  - `[name]`：入口名称（如多入口的 `main` 或 `admin`）
  - `[contenthash]`：根据文件内容生成的哈希（用于缓存控制）
  - `[id]`：模块标识符
  - `[chunkhash]`：chunk 级别的哈希

- **路径问题重点**：  
必须使用 `path.resolve()` 处理路径，避免操作系统差异导致的路径错误

---

##### (3) Loader（加载器）
**核心作用**：  
让 Webpack 能够处理非 JavaScript 文件（如 CSS、图片）

**工作原理**：  
按从右到左的顺序链式调用 Loader（类似函数式编程的 compose）：
```javascript
{
  test: /\.scss$/,
  use: [
    'style-loader',  // 最后执行：将 CSS 注入 DOM
    'css-loader',    // 其次：处理 CSS 的 @import 和 url()
    'sass-loader'    // 最先执行：将 SCSS 编译为 CSS
  ]
}
```

**常见 Loader 分类**：
| Loader 类型       | 示例                   | 作用                     |
|-------------------|------------------------|--------------------------|
| 编译转换类        | `babel-loader`         | 转换 ES6+ 代码           |
| 文件处理类        | `file-loader`          | 处理图片/字体            |
| 样式处理类        | `sass-loader`          | 编译 SCSS/SASS           |
| 代码检查类        | `eslint-loader`        | 代码规范检查（已弃用）   |

**注意事项**：
- 每个 Loader 只做单一功能
- 使用 `exclude: /node_modules/` 避免处理第三方库文件
- Webpack 5 开始，部分资源可直接用 `type: 'asset/resource'` 替代 Loader

---

##### (4) Plugin（插件）
**与 Loader 的区别**：
| 特性         | Loader                  | Plugin                     |
|--------------|-------------------------|----------------------------|
| 作用阶段     | 模块加载阶段            | 整个构建流程               |
| 功能范围     | 处理单个文件            | 全局操作（如打包优化）     |
| 使用方式     | 在 `module.rules` 配置  | 在 `plugins` 数组实例化    |

**常用插件原理**：
- **HtmlWebpackPlugin**：  
  自动生成 HTML 文件，并自动注入打包后的 JS/CSS 资源
  ```javascript
  new HtmlWebpackPlugin({
    template: './src/index.html', // 自定义模板
    favicon: './src/favicon.ico', // 自动添加 favicon
    minify: true // 生产环境压缩 HTML
  })
  ```

- **MiniCssExtractPlugin**：  
  将 CSS 提取为独立文件（替代 `style-loader`）
  ```javascript
  new MiniCssExtractPlugin({
    filename: '[name].[contenthash].css'
  })
  ```

- **DefinePlugin**（Webpack 内置）：  
  定义全局常量（常用于环境变量）
  ```javascript
  new webpack.DefinePlugin({
    API_URL: JSON.stringify('https://api.example.com')
  })
  ```

**插件开发本质**：  
通过 Webpack 提供的生命周期钩子（hooks）介入打包过程，例如：
```javascript
class MyPlugin {
  apply(compiler) {
    compiler.hooks.done.tap('MyPlugin', stats => {
      console.log('打包完成！')
    })
  }
}
```

---

##### (5) Mode（模式）
**三种模式**：
```javascript
mode: 'development' | 'production' | 'none'
```

**不同模式的差异**：
| 功能               | development          | production           |
|--------------------|----------------------|----------------------|
| 代码压缩           | 否                   | 是                   |
| Source Map         | 完整（eval-cheap）   | 最小化（source-map） |
| 热更新（HMR）      | 支持                 | 不支持               |
| 打包速度           | 快                   | 慢（需优化）         |
| 错误提示           | 详细                 | 简洁                 |

**底层实现**：  
Webpack 会根据 `mode` 自动设置 `process.env.NODE_ENV`，并启用对应内置优化：
```javascript
// development 模式相当于：
new webpack.DefinePlugin({
  'process.env.NODE_ENV': JSON.stringify('development')
})
devtool: 'eval-cheap-source-map'

// production 模式相当于：
new webpack.DefinePlugin({
  'process.env.NODE_ENV': JSON.stringify('production')
})
devtool: 'source-map'
plugins: [new TerserPlugin()]
```

---

##### (6) Module（模块）
**Webpack 中的模块定义**：  
所有文件（JS、CSS、图片等）都是模块，通过 `import`/`require` 建立依赖关系

**模块类型**：
- **ES Modules**（ES6 的 `import/export`）
- **CommonJS**（Node.js 的 `require/module.exports`）
- **AMD**（异步模块定义）
- **CSS/Asset Modules**（通过 Loader 转换）

**模块解析规则**：  
Webpack 通过 `resolve` 配置定位模块：
```javascript
resolve: {
  extensions: ['.js', '.json'], // 自动补全文件扩展名
  alias: {
    '@': path.resolve(__dirname, 'src') // 路径别名
  }
}
```

---

#### 3. Webpack 的工作流程（简化版）
1. **初始化参数**：合并命令行参数和配置文件
2. **创建编译器**：实例化 `Compiler` 对象
3. **开始编译**：执行 `Compiler.run()`
4. **确定入口**：根据 `entry` 找到所有入口文件
5. **编译模块**：调用 Loader 转换文件，生成 AST 分析依赖
6. **完成编译**：得到模块依赖关系和转译后的内容
7. **输出资源**：根据依赖关系组装 Chunk，写入文件系统
8. **结束**：触发 `done` 钩子，完成打包

---

#### 4. 理解 Chunk 和 Bundle
| 术语    | 描述                                                                 |
|---------|----------------------------------------------------------------------|
| Chunk   | Webpack 内部构建过程中的代码块（如通过代码分割生成的块）             |
| Bundle  | 最终生成的物理文件（一个 Chunk 对应一个或多个 Bundle）               |

**示例**：  
- 入口文件 `index.js` → 生成一个 Chunk → 输出为 `main.bundle.js`
- 使用 `import()` 动态导入 → 生成新 Chunk → 输出为 `chunk-vendors.bundle.js`

--- 

#### 5. 模块热替换（HMR）原理
**实现步骤**：
1. Webpack Dev Server 启动时创建 WebSocket 连接
2. 文件修改后，Webpack 重新编译生成补丁文件（patch）
3. 通过 WebSocket 通知客户端（浏览器）
4. 客户端收到消息后，拉取最新补丁并局部更新模块

**配置要求**：
```javascript
devServer: {
  hot: true // 启用 HMR
},
target: 'web' // 明确指定目标环境
``` 

**支持 HMR 的 Loader**：  
部分 Loader（如 `style-loader`）已实现 HMR 接口，CSS 修改时无需刷新页面。

---

### 二、环境准备
1. 安装 Node.js（建议 v16+）
2. 初始化项目：
   ```bash
   mkdir webpack-demo && cd webpack-demo
   npm init -y
   ```
3. 安装 webpack：
   ```bash
   npm install webpack webpack-cli --save-dev
   ```

---

### 三、基础配置
1. **目录结构**：
   ```
   webpack-demo/
   ├── src/
   │   └── index.js
   ├── public/
   │   └── index.html
   ├── package.json
   └── webpack.config.js
   ```

2. **基础配置文件**（`webpack.config.js`）：
   ```javascript
   const path = require('path');

   module.exports = {
     entry: './src/index.js',
     output: {
       filename: 'bundle.js',
       path: path.resolve(__dirname, 'dist'),
     },
     mode: 'development'
   };
   ```

3. **执行打包**：
   ```bash
   npx webpack
   ```

---

### 四、处理不同类型资源
1. **处理 CSS**：
   ```bash
   npm install style-loader css-loader --save-dev
   ```
   ```javascript
   // webpack.config.js
   module: {
     rules: [
       {
         test: /\.css$/i,
         use: ['style-loader', 'css-loader'],
       },
     ],
   },
   ```

2. **处理 SCSS**：
   ```bash
   npm install sass-loader sass --save-dev
   ```
   ```javascript
   {
     test: /\.s[ac]ss$/i,
     use: ['style-loader', 'css-loader', 'sass-loader']
   }
   ```

3. **处理图片**：
   ```bash
   npm install file-loader --save-dev
   ```
   ```javascript
   {
     test: /\.(png|svg|jpg|jpeg|gif)$/i,
     type: 'asset/resource'
   }
   ```

4. **处理字体**：
   ```javascript
   {
     test: /\.(woff|woff2|eot|ttf|otf)$/i,
     type: 'asset/resource'
   }
   ```

---

### 五、常用插件配置
1. **自动生成 HTML**：
   ```bash
   npm install html-webpack-plugin --save-dev
   ```
   ```javascript
   const HtmlWebpackPlugin = require('html-webpack-plugin');

   plugins: [
     new HtmlWebpackPlugin({
       template: './public/index.html'
     })
   ]
   ```

2. **清理构建目录**：
   ```bash
   npm install clean-webpack-plugin --save-dev
   ```
   ```javascript
   const { CleanWebpackPlugin } = require('clean-webpack-plugin');
   plugins: [new CleanWebpackPlugin()]
   ```

3. **开发服务器**：
   ```bash
   npm install webpack-dev-server --save-dev
   ```
   ```javascript
   devServer: {
     static: './dist',
     hot: true,
     port: 8080,
     open: true
   }
   ```

---

### 六、生产环境优化
```javascript
module.exports = {
  mode: 'production',
  optimization: {
    splitChunks: {
      chunks: 'all' // 代码分割
    },
    minimizer: [
      new TerserPlugin(), // JS 压缩（webpack5 内置）
      new CssMinimizerPlugin() // CSS 压缩
    ]
  },
  performance: {
    hints: false // 关闭体积警告
  }
}
```

---

### 七、完整配置示例
```javascript
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = {
  entry: './src/index.js',
  output: {
    filename: '[name].[contenthash].js',
    path: path.resolve(__dirname, 'dist')
  },
  mode: 'production',
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: 'babel-loader'
      },
      {
        test: /\.scss$/,
        use: [MiniCssExtractPlugin.loader, 'css-loader', 'sass-loader']
      },
      {
        test: /\.(png|svg|jpg)$/,
        type: 'asset/resource'
      }
    ]
  },
  plugins: [
    new CleanWebpackPlugin(),
    new HtmlWebpackPlugin({ template: './public/index.html' }),
    new MiniCssExtractPlugin({ filename: '[name].[contenthash].css' })
  ],
  devServer: {
    static: {
      directory: path.join(__dirname, 'dist'),
    },
    compress: true,
    port: 9000,
  }
};
```

---

### 八、高级配置
1. **环境变量**：
   ```javascript
   const isProduction = process.env.NODE_ENV === 'production';
   ```

2. **Source Map**：
   ```javascript
   devtool: isProduction ? 'source-map' : 'eval-cheap-module-source-map'
   ```

3. **Babel 集成**：
   ```bash
   npm install @babel/core @babel/preset-env babel-loader --save-dev
   ```
   ```javascript
   {
     test: /\.js$/,
     exclude: /node_modules/,
     use: {
       loader: 'babel-loader',
       options: {
         presets: ['@babel/preset-env']
       }
     }
   }
   ```

4. **ESLint 集成**：
   ```bash
   npm install eslint eslint-webpack-plugin --save-dev
   ```
   ```javascript
   const ESLintPlugin = require('eslint-webpack-plugin');
   plugins: [new ESLintPlugin()]
   ```

---

### 九、常见问题解决
1. **路径问题**：使用 `path.resolve(__dirname, '相对路径')`
2. **缓存问题**：输出文件名添加 `[contenthash]`
3. **加载器顺序**：loader 执行顺序从右到左
4. **版本兼容**：注意 webpack 5 与旧插件的兼容性

---

### 十、工作流程总结
1. 通过 `entry` 定位入口文件
2. 根据 `module.rules` 处理不同文件类型
3. 解析模块依赖关系形成依赖图
4. 使用 `plugins` 执行额外构建任务
5. 输出打包结果到 `output` 指定目录

建议结合官方文档（https://webpack.js.org/）进行更深入的学习，根据项目需求调整配置。