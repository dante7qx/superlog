## Less 指南

### 一. 介绍

​	css（层叠样式表）是一门标记性语言，不是一门编程语言，无法使用变量、函数等概念。因此，css代码中会存在大量的冗余。

​	less（Leaner Style Sheets）是 css 的预编译语言，可以在 css 语法的基础上，使用变量，Mixin，运算以及函数等功能。

### 二. 安装

```bash
sudo npm install less -g
lessc styles.less > styles.css

==> 运行 npm install 结果
Downloading less to /usr/local/lib/node_modules/less_tmp
Copying /usr/local/lib/node_modules/less_tmp/_less@3.0.1@less to /usr/local/lib/node_modules/less
Installing less's dependencies to /usr/local/lib/node_modules/less/node_modules
[1/8] mime@^1.4.1 installed at node_modules/_mime@1.6.0@mime
[2/8] graceful-fs@^4.1.2 installed at node_modules/_graceful-fs@4.1.11@graceful-fs
[3/8] mkdirp@^0.5.0 installed at node_modules/_mkdirp@0.5.1@mkdirp
[4/8] source-map@^0.5.3 installed at node_modules/_source-map@0.5.7@source-map
[5/8] promise@^7.1.1 installed at node_modules/_promise@7.3.1@promise
[6/8] image-size@~0.5.0 installed at node_modules/_image-size@0.5.5@image-size
[7/8] errno@^0.1.1 installed at node_modules/_errno@0.1.7@errno
[8/8] request@2.81.0 installed at node_modules/_request@2.81.0@request
All packages installed (62 packages installed from npm registry, used 6s, speed 416.96kB/s, json 60(836.38kB), tarball 1.51MB)
[less@3.0.1] link /usr/local/bin/lessc@ -> /usr/local/lib/node_modules/less/bin/lessc
```

### 三. 使用

#### 1. 变量

​	**Less**  的变量根据范围接受它们的值。如果在指定范围内没有关于变量值的声明， **Less** 会一直往上查找，直至找到离它最近的声明。

```less
/* styles.less */
@color: #890809;
p {
    font-size: 14px;
    font-color: @color;
}

/**
 * styles.css
 */
p {
    font-size: 14px;
    font-color: #890809;
}
```

#### 2. Mixins

​	将已有的 class 和 id 样式应用到另一个不同的选择器上。Mixin 不出现在编译后的 css 中，则添加括号，并且支持传参。`#circle(@size 25px) 或者 @windows-size: 50%; #circle(@size: @windows-size)`

```less
/* styles.less */
@windows-size: 50%;

.pu {
    cursor: pointer;
}

#circle {
    background-color: #898989;
    border-radius: 100%;
}

#small-circle {
    width: 50%;
    height: 50%;
    #circle;
    .pu
}

#bb(@size: @windows-size) {
    width: @size;
    height: @size;
    background-color: #676767;
}

#small-bb {
    cursor: pointer;
    #bb
}

/* styles.css */
.pu {
  cursor: pointer;
}
#circle {
  background-color: #898989;
  border-radius: 100%;
}
#small-circle {
  width: 50%;
  height: 50%;
  background-color: #898989;
  border-radius: 100%;
  cursor: pointer;
}
#small-bb {
  cursor: pointer;
  width: 50%;
  height: 50%;
  background-color: #676767;
}
```

#### 3. 嵌套

​	按照 Dom 树的结构设置样式。

```less
/* styles.less */
@text-color: #000;
ul {
    @text-color: #fff;
    background-color: #0a3434;
    padding: 10px;
    list-style: none;
    li {
        background-color: @text-color; /* 变量向上查找，直到找到最近的 */	
        border-radius: 3px;
        margin: 10px 0;
    }
}

/* styles.css */
ul {
  background-color: #0a3434;
  padding: 10px;
  list-style: none;
}
ul li {
  background-color: #fff;
  border-radius: 3px;
  margin: 10px 0;
}
```

```less
/**
 * styles.less
 *
 * :after 选择器：在被选元素的内容后面插入内容，content 属性来指定要插入的内容。
 **/
.clearfix {
    display: none;
    zoom: 1;
    /* & 表示当前选择器的父元素 */
    &:after {
        content: "";
        display: block;
        font-size: 0;
        height: 0;
        clear: both;
        visibility: hidden;    
    }
}

/** styles.css **/
.clearfix {
  display: block;
  zoom: 1;
}
.clearfix:after {
  content: " ";
  display: block;
  font-size: 0;
  height: 0;
  clear: both;
  visibility: hidden;
}
```

#### 4. 运算

​	可以对数值和颜色进行基本（+ - * /）运算，也可使用 calc() 在嵌套函数中会计算变量和数学公式的值。

```less
/** styles.less **/
@div-width: 100px;
@div-color: #03A9F4;

div {
    height: 50px;
    display: inline-block;
}
#left {
    width: @div-width;
    background-color: @div-color - 100
}
#right {
    width: @div-width * 2;
    background-color: @div-color
}
#middle {
    width: calc(23 + (@div-width * 2))
}

/** styles.css **/
div {
  height: 50px;
  display: inline-block;
}
#left {
  width: 100px;
  background-color: #004590;
}
#right {
  width: 200px;
  background-color: #03A9F4;
}
#middle {
  width: calc(23 + (100px * 2));
}
```

#### 5. 函数

​	Less 内置了很多函数用于转换颜色、处理字符串、算术运算等。

```less
/** styles.less **/
@var: #004590;
div {
    width: 50px;
    height: 50px;
    background-color: @var;

    &:hover {
        background-color: fadeout(@var, 50%)
    }
}

/** styles.css **/
div {
  width: 50px;
  height: 50px;
  background-color: #004590;
}
div:hover {
  background-color: rgba(0, 69, 144, 0.5);
}
```

#### 6. 导入

​	**@import **伪指令用于在代码中导入文件。 它将LESS代码分布在不同的文件上，并允许轻松地维护代码的结构。您可以将* @import *语句放在代码中的任何位置。

### 四. 参考文档

- http://lesscss.org/
- https://less.bootcss.com/
- https://www.jianshu.com/p/c676041f387e
- https://www.jianshu.com/p/95d47de3c63e?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation