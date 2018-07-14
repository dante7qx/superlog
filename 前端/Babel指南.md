## Babel 指南

### 一. 介绍

​	Babel是一个 JS 编译 - 转换器。它可以将最新标准编写的 Javascript 向下编译成各个浏览器都支持的版本。比如将ES6 的代码转为 ES5。

### 二. 安装

#### 1. 全局安装

安装命令：`npm install --global babel-cli`

使用方式

```shell
babel hello.js ## 编译后的结果直接输出至终端
babel hello.js -o _hello.js ## 编译后的结构写入 _hello.js
babel src -d lib ## 将src目录整个编译到lib目录
```

备注：不推荐使用，原因如下

1. 在同一台机器上的不同项目或许会依赖不同版本的 Babel， 或需要有选择的更新。
2. 项目要运行，全局环境必须有Babel，也就是说项目产生了对环境的依赖。

#### 2. 项目内安装

在项目根目录执行 `npm install -D babel-cli` 。

```sh
mkdir cli-demo
cd cli-demo
npm init -y
mkdir src
cd src
vi index.js  ## console.log('hello babel-cli')
cd ../
npm install -D babel-cli
## 编辑 package.json, 修改 scripts  "build": "babel src -d lib"
## 终端运行
npm run build
```

运行结果

```powershell
bogon:cli-demo dante$ ll
total 16
drwxr-xr-x    7 dante  staff   224  2 28 17:56 ./
drwxr-xr-x    4 dante  staff   128  2 28 17:52 ../
-rw-r--r--    1 dante  staff    27  2 28 17:54 index.js
drwxr-xr-x    3 dante  staff    96  2 28 17:56 lib/
drwxr-xr-x  240 dante  staff  7680  2 28 17:55 node_modules/
-rw-r--r--    1 dante  staff   251  2 28 17:56 package.json
drwxr-xr-x    3 dante  staff    96  2 28 17:54 src/
```

#### 3. babel-register

​	将 **babel-register** 模块引入到项目文件中。不适用于正式产品，直接部署用此方式编译的代码不是好的做法。 在部署之前预先编译会更好。（不推荐） 

```shell
mkdir register-demo
cd register-demo
npm init -y
## 安装 babel-register 
npm install -D babel-register
## 编写index.js, console.log('hello, babel-register');
## node index.js, 不会使用 Babel 来编译
vi index.js  
node index.js

## 编写register.js，将 babel 注册到Node模块中
## 通过 babel 编译其中含 require('babel-register') 的所有文件
## 注意：require('babel-register') 不能在多个文件中注册
vi register.js	# require('babel-register'); require('./index.js');
node register.js
```

#### 4. babel-node

- 全局安装的，直接在终端输入 babel-node ，可以运行 ES6 代码。

```shell
bogon:cli-demo dante$ babel-node 
> (x => x * 2) (2)
4
```

- 项目内安装

```shell
npm install -D babel-cli

## 修改 package.json，使用babel-node替代node
"scripts": {
    "script-name": "babel-node script.js"
}
```

#### 5. babel-core

​	通过 [Babel API](https://babeljs.io/docs/usage/api/) 进行转码，需要安装 `npm install -D babel-core` 。API 示例如下：

```javascript
var babel = require('babel-core');

// 字符串转码
babel.transform('code();', options);
// => { code, map, ast }

// 文件转码（异步）
babel.transformFile('filename.js', options, function(err, result) {
  result; // => { code, map, ast }
});

// 文件转码（同步）
babel.transformFileSync('filename.js', options);
// => { code, map, ast }

// Babel AST转码
babel.transformFromAst(ast, code, options);
// => { code, map, ast }
```

### 三. 配置

​	在项目根目录下创建文件 **.babelrc**，通过 plugins 和 presets（一组plugins） 来指示 Babel 的工作。其中 plugins 和 presets都是一些 JS 模块，需要通过 npm 进行安装。

```json
{
    "plugins": [],
    "presets": []
}
```

##### 1. presets

- babel-preset-es2015：可以将ES6（ES2015）编译成 ES5

```shell
npm install -D babel-preset-es2015

# .babelrc 中
"presets": [ "es2015" ]
```

- babel-preset-stage-x：ES7不同阶段语法提案的转码规则，前者依赖后者，即 stage-1 依赖 stage-2，stage2依赖 stage-3，根据需要进行安装。

```shell
npm install --save-dev babel-preset-stage-0
npm install --save-dev babel-preset-stage-1
npm install --save-dev babel-preset-stage-2
npm install --save-dev babel-preset-stage-3
```

- babel-preset-env（重要）：包含了所有的 babel-preset-es 和 babel-preset-stage，所以要进行未来JS的兼容编译，只需要安装此 preset 就可以了。

```shell
npm install -D babel-preset-env

# .babelrc 中
"presets": [ "env" ]
```

##### 2. babel-polyfill

​	babel默认只转换语法，而不转换新的 API，若使用新的API就需要使用对应的转换插件或 polyfill。

- 方式一

  ```shell
  npm install --save babel-polyfill	# 不能做为 devDependency

  ## 在文件顶部导入 polyfill，require('babel-polyfill') 或者 import babel-polyfill
  ```

- 方式二

  ```markdown
  1. 安装 babel-preset-env 依赖
  	npm install -D babel-preset-env
  2. 在 .babelrc 中使用  
  	"presets": ["env"]
  3. 指定 useBuiltins 选项为true
  4. 指定浏览器环境或node环境
  5. 在webpack入口文件中使用import/require引入polyfill, 如import 'babel-polyfill'
  {
    "presets": [
      ["env", {
        "modules": false,
        "targets": {
          "browsers": ["ie >=6"]
        },
        "useBuiltIns": true,
        "debug": true
      }]
    ]
  }
  ```

##### 3. 基于环境自定义 Babel

​	Babel 将根据当前环境来开启 `env` 下的配置。当前环境可以使用 `process.env.BABEL_ENV` 来获得。 如果 `BABEL_ENV` 不可用，将会替换成 `NODE_ENV`，并且如果后者也没有设置，那么缺省值是`"development"` 。

```json
{
    "presets": ["es2015"],
    "plugins": [],
    "env": {
         "development": {
         	"plugins": [...]
     	},
     	"production": {
         	"plugins": [...]
      	 }
    }
}
```

### 四. 参考资料

- https://babeljs.io/
- http://blog.csdn.net/sinat_34056695/article/details/74452558
- http://www.ruanyifeng.com/blog/2016/01/babel.html
- https://www.jianshu.com/p/3b27dfc6785c
- https://babeljs.io/docs/plugins/
- https://www.npmjs.com/search?q=babel-pluginLess指南