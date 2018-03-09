## NPM  指南

### 一. 介绍

​	NPM是随NodeJS一起安装的包管理工具，由Node.js编写。JavaScript开发人员将JS模块发布到NPM中央注册表中。NPM由三部分组成。

	- 网站：开发者查找包（package）、设置参数以及管理 npm 使用体验的主要途径。https://www.npmjs.com/。
- 注册表：模块（package）存储数据库，保存了每个包（package）的信息。
- CLI：通过终端命令，开发者与npm进行交互。

常见的使用场景有以下几种：

- 允许用户从NPM服务器下载别人编写的第三方包到本地使用。
- 允许用户从NPM服务器下载并安装别人编写的命令行程序到本地使用。
- 允许用户将自己编写的包或命令行程序上传到NPM服务器供别人使用。

### 二. 安装NPM

1. 下载Node.js，安装后会自动的安装NPM。通过 `npm -v` 进行验证。

   ```shell
   bogon:Desktop dante$ npm -v
   5.6.0
   ```

2. 更新NPM

   ```shell
   npm install npm@latest -g .
   ```

### 三. 使用NPM

#### 1. 项目描述文件 

- package.json，类似于 Maven 中的 pom.xml 文件。通过命令可以自动创建

```nginx
npm init 
# 或者
npm init -y 

```

​	举例：

```shell
mkdir aa
cd aa
npm init -y

##  package.json 如下
{
  "name": "aa",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

- 版本：NPM使用语义版本号来管理代码，语义版本号分为X.Y.Z三位，分别代表主版本号、次版本号和补丁版本号
  - 补丁版本号：修复bug。
  - 次版本号：新增了功能，但是向下兼容。
  - 主版本号：大变动，不向下兼容。

#### 2. 模块安装

- 本地安装，只在某一个项目中使用。

  - 将安装包放在 ./node_modules 下（运行 npm 命令时所在的目录），如果没有 node_modules 目录，会在当前执行 npm 命令的目录下生成 node_modules 目录。
  - 可以通过 require() 来引入本地安装的包。

  ```sh
  ## 方式一，相当于 npm install <pkg> --save，
  ## 开发时依赖 npm install <pkg> --save-dev 或者 npm install -D <pkg>
  npm install <pkg> 或 npm install <pkg>@<version>

  ## 方式二，在 package.json 中编写依赖，然后执行 npm install
  "devDependencies": {
      "lodash": "^4.17.5"
  },
  "dependencies": {
      "lodash": "^4.17.5"
  }

  npm install
  ```

  - 更新，修改 package.json 中的版本号，然后在终端执行 `npm update` 。（ `npm outdated`  无输出。）
  - 卸载，终端执行 `npm uninstall <pkg>` 。（可选参数：`--save`，`--save-dev`）

- 全局

  - 将安装包放在 /usr/local 下或者你 node 的安装目录（/usr/local/lib/node_modules）。
  - 可以直接在命令行里使用。

  ```shell
  ## 安装
  npm install -g <pkg> 或 npm install -g <pkg>@<version>
  ## 更新
  npm update -g <pkg>
  ## 更新所有
  npm update -g
  ## 卸载
  npm uninstall -g <pkg>
  ```

- 详解

  ```shell
  npm install (with no args, in package dir)
  npm install [<@scope>/]<name>
  npm install [<@scope>/]<name>@<tag>
  npm install [<@scope>/]<name>@<version>
  npm install [<@scope>/]<name>@<version range>
  npm install <git-host>:<git-user>/<repo-name>
  npm install <git repo url>
  npm install <tarball file>
  npm install <tarball url>
  npm install <folder>

  alias: npm i
  common options: [-P|--save-prod|-D|--save-dev|-O|--save-optional] [-E|--save-exact] [-B|--save-bundle] [--no-save] [--dry-run]
  ```

#### 3. 常用命令

```sh
#查看当前项目下的包列表
npm ls
#查看全局包列表
npm ls －g
#清理缓存
npm cache clean
```

#### 4. NPM配置文件

读取配置文件：用户配置文件：`npm config ls`，全局配置文件:`npm config ls -l`

用户配置文件目录：`~/.npmrc`

#### 5.设置加速

方式一：

- `npm config set registry https://registry.npm.taobao.org `
- ~/.npmrc 下，添加 `registry=https://registry.npm.taobao.org`

方式二：

使用淘宝定制的 cnpm (gzip 压缩支持) 命令行工具代替默认的 npm:

```sh
$ npm install -g cnpm --registry=https://registry.npm.taobao.org
## 然后
$ cnpm install -D lodash
```

### 四. NPM 脚本

​	定义在 package.json 中的脚本，称为 npm脚本。通过 `npm run 脚本名称` 来执行，通过 `npm run` 可以查看项目的所有脚本命令。

优点：

	1. 集中管理项目的脚本。
	2. 重用对外的接口，即脚本段的重用。

- 原理

  ​	每次执行 npm run，会新建一个shell，在这个shell中执行指定的脚本，并且这个脚本会将当前项目下 **node_modules/.bin** 加入到 **PATH** 变量。这样**node_modules/.bin**里的所有脚本就可以直接用脚本名来进行调用。例：`"dev": "webpack-dev-server"`。通过 `--` 进行传参 `"dev": "webpack-dev-server —open"` 。

  - 执行顺序

    & 表示同时执行：	  npm run script1.js & npm run script2.js

    && 表示继发执行：npm run script1.js && npm run script2.js

  - 变量

    ​

### 五. 参考资料

- https://www.jianshu.com/p/a1bb4c317c3e
- https://www.npmjs.com.cn/
- http://www.runoob.com/nodejs/nodejs-npm.html
- https://www.jianshu.com/p/701229149d23