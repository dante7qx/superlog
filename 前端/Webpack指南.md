## Webpack 指南

### 一. 介绍

​	webpack是现代前端应用的模块打包器。它会分析你的项目结构，找到 JS 模块和一些浏览器无法直接运行的拓展语言（.vue、TypeScript等），并将其转换和打包为合适的格式供浏览器使用。

​	常用命令

- `webpack` – building for development
- `webpack -p` – building for production (minification)
- `webpack --watch` – for continuous incremental building
- `webpack -d` – including source maps
- `webpack --colors` – making building output pretty

### 二. 核心概念

在项目根目录创建 webpack.config.js 进行配置，webpack 通过配置进行工作。项目的基本目录结构如下：

```properties
项目名
	-- package.json
	-- index.html
	-- index.js
	-- webpack.config.js
```

#### 1. entry

​	webpack开始构建的入口模块。webpack会找出入口模块中的依赖关系进行打包（bundling），可以指定多个。

```json
entry: "./app/entry", // string | object | array
```

#### 2. output

​	webpack打包后产生的文件，需要告诉 webpack 文件产生的位置和名字。

```json
output: {
    "filename": "bundle.js",
    "path": __dirname+"/build",	// 必须是绝对路径，编译目录（/build/js/），不能用于html中的js引用。
    "publicPath": "/asserts/" // 虚拟目录，自动指向path编译目录（/assets/ => /build/js/）。html中引用js文件时，必须引用此虚拟路径（但实际上引用的是内存中的文件，既不是/build/js/也不是/assets/）。
}
```

#### 3. loader

​	将所有类型的文件（非 JS 文件）转换为可以被浏览器直接使用的模块。所有的 loader 都定义在 module.rules 中，每个 rule 都是一个对象。例：

```json
module: {
  rules: [
      {
          "test": /\.txt&/,
          "use": "raw-loader"
      }
  ]
}
```

​	其中，test 表示配置 loader 要去处理的文件的匹配规则，use 表示对于匹配规则的文件要用 raw-loader 去进行转换。

#### 4. plugins

​	插件用来拓展**webpack**功能，在整个构建过程中生效，在于解决 **loader** 无法实现的其他事。使用方法

	1. 安装需要的插件

```shell
npm install -D webpack webpack-cli webpack-dev-server html-webpack-plugin
```

2. 配置plugin

```javascript
// 参考：http://www.cnblogs.com/ghostwu/p/7500289.html
const HtmlWebpackPlugin = require('html-webpack-plugin'); //通过 npm 安装
plugins: [
    new HtmlWebpackPlugin({
        template: './build/app.html'
    })
]
```

### 三. 使用积累

#### 1. source map

​	source map文件是 js 文件压缩后，文件的变量名替换对应变量所在位置等元信息数据文件。般这种文件和min.js主文件放在同一个目录下。方便开发人员调试。

### 六. 参考资料

- https://webpack.js.org
- https://github.com/webpack/webpack
- https://www.jianshu.com/p/42e11515c10f
- https://www.cnblogs.com/parry/p/7062251.html