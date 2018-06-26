## ES6 指南

### 一. 简介

​	ECMAScript 6.0（以下简称 ES6）是 JavaScript 语言的下一代标准，已经在 2015 年 6 月正式发布了。它的目标，是使得 JavaScript 语言可以用来编写复杂的大型应用程序，成为企业级开发语言。

​	Javascript 严格模式，即在严格的条件下运行，不允许使用未声明的变量。例如：

```javascript
'use strict'
function hello() {}

// 所有的限制
- 变量必须声明后再使用
- 函数的参数不能有同名属性，否则报错
- 不能使用with语句
- 不能对只读属性赋值，否则报错
- 不能使用前缀 0 表示八进制数，否则报错
- 不能删除不可删除的属性，否则报错
- 不能删除变量delete prop，会报错，只能删除属性delete global[prop]
- eval不会在它的外层作用域引入变量
- eval和arguments不能被重新赋值
- arguments不会自动反映函数参数的变化
- 不能使用arguments.callee
- 不能使用arguments.caller
- 禁止this指向全局对象
- 不能使用fn.caller和fn.arguments获取函数调用的堆栈
- 增加了保留字（比如protected、static和interface）
```

### 二. let 和 const

#### 1. let

- let 和 var 都是声明变量的，作用域不同。let 是代码块作用域，var是全局作用域。

```javascript
for(let i = 0; i < 5; i++) {}
// console.log(i) // 报错

for(var j = 0; j < 5; j++) {}
console.log(j) // => 5
```

- 不可重复定义，不能未定义就使用。
- 尽量用 let 替换 var 去声明变量。

#### 2. const

	- 声明一个**只读**的常量。
- 对于对象常量，引用地址不能修改，对象的属性可以修改。若要让对象的属性也不能修改，则需要将对象进行冻结。`const user = Object.freeze({name: '但丁', age: 33})` 。

```javascript
// const 声明的对象常量，引用不能改变，属性可以改变
const user = {name: '但丁', age: 33}
console.log(user)
user.name = `${user.name}_x`
console.log(user)

user = {a: 11}	// 报错，TypeError: Assignment to constant variable.
console.log(user)
```

### 三. 变量的解构赋值

​	解构赋值：按照指定的语法（模式匹配），从**数组或对象**中抽取值，然后对相应的变量进行赋值。尽量不要在模式中使用圆括号。

#### 1. 数组

按照位置顺序进行匹配。

```javascript
let [a, b, c, d = 7] = [1, [2, 5], 3, 9]
console.log(a, b, c, d);	// => 1 [ 2, 5 ] 3 7 9
```

#### 2. 对象

​	按照对象的属性进行匹配。对象的解构赋值的内部机制，是先找到同名属性，然后再赋给对应的变量。真正被赋值的是后者，而不是前者。即当 **属姓名 === 变量名** 时，可以简写成 **属姓名**。

```javascript
// 下面的对象解构赋值是  let {x: x, y: y, z: zV} = {y: 200, x: 100, z: 99}
let {x, y, z: zV} = {y: 200, x: 100, z: 99}
console.log({x, y, z: zV})	// { x: 100, y: 200, z: 99 }
console.log(x, y, zV)	    // 100 200 99
```

#### 3. 字符串

​	字符串被转化为一个类似数组的对象。

```javascript
const [x1, x2, x3, x4, x5] = 'hello'
console.log(x1, x2, x3, x4, x5)		  // h e l l o
const {length: len} = 'hello' 
console.log(len)					// 5
```

#### 4. 函数的解构赋值

```javascript
[[1, 2], [3, 4]].map(([a, b]) => a + b)
// [3, 7]
```

#### 5. 使用场景

- 函数返回多个值

```javascript
function ex() {
    return [1, 3, 4]
}
let [a, b, c] = ex()
```

- 处理 JSON 对象

```javascript
let jsonData = {
  id: 42,
  status: "OK",
  data: [867, 5309]
}

let {id, status, data: number} = jsonData
```

- 遍历Iterator 接口的对象，for…of

```javascript
const map = new Map();
map.set('first', 'hello');
map.set('second', 'world');

for(let[key, value] of map) {}
for(let[key] of map) {}
for(let[, value] of map) {}
```

- 输入模块的指定方法

```javascript
const {add, decrement} = require('some-module')
```

#### 6. 字面量中动态计算属性名

在ES6中，把属性名用[ ]括起来，则括号中就可以引用提前定义的变量。

```javascript
var attName = 'name';
var p = {
    attName : '李四',  // 这里的attName是属性名，相当于各级p定义了属性名叫 attName的属性。
    age : 20
}
console.log(p[attName])  // 李四
```

### 四. Iterator

​	一个接口，为所有的数据结构提供一种统一的访问机制，即 **for…of**。一个数据结构只要具有 **Symbol.iterator** 属性，就是 iterable。原生的 iterable 数据结构有：

 - Array
 - Map
 - Set
 - String
- 函数的 arguments 对象

```javascript
// 原理
const obj = {
  [Symbol.iterator] : function () {
    return {
      next: function () {
        return {
          value: 1,
          done: true
        };
      }
    };
  }
};
```

### 五. 语法扩展

#### 1. 字符串

- 可以使用 for…of 遍历。

```javascript
for(let s of 'dante') {
	console.log(s)
}
```

- 新方法：
  - includes()
  - startsWith()
  - endsWith()   参数 (str, [n]) ，n 表示开始搜索的位置。
  - repeat(n)   返回一个新字符串，n 表示重复几次
  -  padStart(n, str)  头部补全
  - padEnd(n, str)     尾部补全，n 表示字符串的最小长度

```javascript
let s = 'hello world!'
s.includes('o')	// true
s.startsWith('hello') // true
s.endsWith('!')	// true
s.repeat(2)	// hello world!hello world!
'x'.padStart(4, '00') // 000x
'x'.padEnd(4, '00')   // x000
```

- 插值字符串（模板字符串），使用反引号 ``

```javascript
let arg1 = 'xyz'
let arg2 = 'orz'
let result = `变量arg1：${arg1}，变量arg2：${arg2}`
```

#### 2. 数值

- Number(obj)  — 将对象转化为数值
- Number.isFinite() — 对于非数值一律返回`false`
- Number.isNaN() — 只有对于`NaN`才返回`true`，非`NaN`一律返回`false`
- Number.parseInt()、Number.parseFloat() — 添加第二个参数，Number.parseInt(122, 10)

#### 3. 函数

- 函数可以指定默认值。`function(x, y=10){}` 。
- 函数的参数默认值位置，只有尾参数设置默认值，在调用式才能省略。

```javascript
function ox(x, y=1) {
	console.log(x, y)
}

ox(8)  // => 8 1，若函数定义为 function ox(x=1, y){}，则 y 是 undefined
```

- 参数必填函数

```javascript
function throwIfMissing() {
    throw new Error('Missing Parameter')
}
function foo(mbp = throwIfMissing()) {
    return mbp
}
// throwIfMissing(), 圆括号指运行时执行。即，参数已经赋值，则默认函数不会执行
```

##### rest 参数

​	（...参数名），该参数是一个数组。rest 参数之后不能再有其他参数（即只能是最后一个参数）。

```javascript
function push(arr, ...items) {
    items.forEach(item => arr.push(item))
    return arr
}
var a = []
console.log(push(a, 1, 2, 3, 4)) // => [1, 2, 3, 4]
```

##### 箭头函数

​	语法，( **=>** )	来定义函数。

```javascript
let f = v => v 			// let f = function(v) { return v }
let f = () => 5 		// let f = function() { return 5 }
let sum = (n1, n2) => n1 + n2	// let sum = function(n1, n2) { return n1 + n2 }

// 只有一行语句，且无返回值
let sayMsg = msg => void console.log(msg)

// 箭头函数返回对象，需要加圆括号
let creatUser = id => ({id: id, name: `名字[${id + 1}]`})

// 结合解构赋值, full({f: 'Michael', l: 'Dante'}) , 返回 Michale - Dante
let full = ({f, l}) => `${f} - ${l}` // let full = function(usr) {return usr.f + ' - ' + usr.l}

// 排序, 想想 lambda 表达式
let values = [4, 12, 22, 1, 2, 28]
let sortArr = values.sort((a, b) => a - b) // [ 1, 2, 4, 12, 22, 28 ]

// 结合 rest 参数
let headAndTail = (head, ...tail) => [head, tail] // headAndTail(1, 2, 3) => [1, [2, 3]]

// pipeline, 前一个函数的输出是后一个函数的输入
const pipeline = (...funcs) => val => funcs.reduce((a, b) => b(a), val)
const plus = a => a + 1
const mult = a => a * 2
const plusThenMult = pipeline(plus, mult)
plusThenMult(5) // => 12, plus 输出 6，mult(6) 输出 6 * 2

// 上面的写法相当于
mult(plus(5)) // => 12
```

⚠️注意：

1. 函数体内的 this 对象，是定义是所在的对象，不是使用时所在的对象。

   因为箭头函数本身没有自己的 **this**，导致内部的 **this** 就是外层代码块的 **this**。

   ```java
   // ES6
   function foo() {
     setTimeout(() => {
       console.log('id:', this.id);
     }, 100);
   }

   // ES5
   function foo() {
     var _this = this;
     setTimeout(function () {
       console.log('id:', _this.id);
     }, 100);
   }
   ```

2. 不能作构造函数，因为 this 的原因。

3. 不可以使用 `arguments ` 对象，该对象在函数体内不存在。如果要用，可以用 rest 参数代替。

4. 不可以使用 `yield` 命令，因此箭头函数不能用作 Generator 函数。

##### 双冒号运算符

​	“函数绑定”（function bind）运算符，用来取代`call`、`apply`、`bind`调用。语法 `obj::func` 。该运算符会自动将左边的对象，作为上下文环境（即`this`对象），绑定到右边的函数上面。

######    call、apply、bind

​	`call`、`apply`和`bind`是`Function`对象自带的三个方法，这三个方法的主要作用是改变函数中的`this`指向。

 -  call 

    `call([thisObj[,arg1[, arg2[, [,.argN]]]`，调用一个对象的一个方法，用另一个对象替换当前对象。

```javascript
let a = {name: '但丁'}
let b = {name: '无涯'}
function x(msg) {
	console.log(this.name, msg)
}
x.call(a, 123)	// 但丁 123
x.call(b, 456)  // 无涯 456
```

- apply

  `apply([thisObj[,argArray]])`，应用某一对象的一个方法，用另一个对象替换当前对象。同 call ，参数需要封装成一个数组。

```javascript
let a = {name: '但丁'}
let b = {name: '无涯'}
function x(msg) {
	console.log(this.name, msg)
}
x.apply(a, [123])	// 但丁 123
x.apply(b, [456])	// 无涯 456
```

- bind

  `bind()` 方法会创建一个新函数，称为绑定函数（返回值是函数），当调用这个绑定函数时，绑定函数会以创建它时传入 `bind()` 方法的第一个参数作为 `this`，传入 `bind()` 方法的第二个以及以后的参数加上绑定函数运行时本身的参数按照顺序作为原函数的参数来调用原函数。

```javascript
x.bind(a, 123)()	// 但丁 123 
x.bind(b, 456)()	// 无涯 456
```

###### 双冒号

```javascript 
obj::func;  		// func.bind(obj)
obj::func(5)		// func.call(obj, 5)
obj::func([5])	// func.apply(obj, [5])
```

##### 尾调用优化

​	在函数的最后一步调用另一个函数叫做**尾调用**。函数最后一步调用自身，称为 **尾递归**。对于尾递归来说，由于只存在一个调用帧，所以永远不会发生“栈溢出”错误。ES6 的尾调用优化只在严格模式下开启，正常模式是无效的。

​	优化原理：

​	函数调用会在内存形成一个“调用记录”，又称“调用帧”（call frame），保存调用位置和内部变量等信息。如果在函数`A`的内部调用函数`B`，那么在`A`的调用帧上方，还会形成一个`B`的调用帧。等到`B`运行结束，将结果返回到`A`，`B`的调用帧才会消失。如果函数`B`内部还调用函数`C`，那就还有一个`C`的调用帧，以此类推。所有的调用帧，就形成一个“调用栈”（call stack）。

​	尾调用只保留内层函数的调用帧。

```javascript
'use strict'
function fibonacci(n, ac1=1, ac2=1) {
    if(n <= 1) return ac2;
    return fibonacci(n - 1, ac2, ac1 + ac2)
}
```

##### 尾逗号

​	最后一个参数添加逗号，推荐**函数参数**与**数组**和**对象**的尾逗号规则。

```javascript
function func(param1, param2,) {}
let arr = [1, 2, 3,]
let obj = {name: '但丁', age: 33, gender: '男',}
```

#### 4. 数组

##### 展开运算符

也叫扩展运算符，将一个数组转为用逗号分隔的参数序列。即，…[1, 2, 3] 相当于 1, 2, 3。

```javascript
function push(arr, ...arg) {
	arr.push(...arg)
	return arr
}
const numbers = [4, 5, 5,]
push([], ...numbers)	// [4, 5, 5] , push([], 4, 5, 5,)
push([], numbers)		// [ [4, 5, 5] ]

// 合并数组
var arr1 = ['a', 'b'];
var arr2 = ['c'];
var arr3 = ['d', 'e'];
[...arr1, ...arr2, ...arr3]

// 字符串转数组
let str = 'hello!'
const arrStr = [...str]	// [ 'h', 'e', 'l', 'l', 'o', '!' ]
```

对象转数组，只有 Iterator 接口的对象，才可以使用展开运算符进行转换。

- Array.from 和 Array.of

  - Array.from：将类似数组的对象（array-like object）和可遍历（iterable）的对象转为数组。

    - 类似数组对象：必须有`length`属性。
    - 第二个参数，可对元素进行处理的函数。

    ```javascript
    let arrayLike = {
        '0': 'a',
        '1': 'b',
        '2': 'c',
        length: 3
    };
    Array.from(arrayLike, x => x * x); // Array.from(arrayLike).map(x => x * *);
    // 将字符串转为数组，然后返回字符串的长度
    Array.from(string).length;
    ```

  - Array.of：将一组值转换为数组。等同于 `[].slice.call(arguments);`

- find() 和 findIndex()

  - find：参数回调，用于找出第一个符合条件的数组成员。回调函数参数 function(value, index, arr) {}。
  - findIndex：参数回调，返回第一个符合条件的数组成员的位置，若所有成员都不符合条件，则返回`-1`。

  ```javascript
  [1, 4, -5, 10].find((n) => n < 0)	// -5
  [1, 4, -5, 10].findIndex((n) => n < 0)	// 2
  ```

- entries()，keys()，values() 和 includes()

  ```javascript
  let arr = [1, 2, 3,]
  arr.keys();			// 0, 1, 2
  arr.values();		// 1, 2, 3
  arr.entries();		
  arr.includes(3);	// true
  arr.includes(3, 1)	// false, arr.includes(3, 2) true
  ```

#### 5. 对象

	- 简写

```javascript
// 属性简写
function f(x, y) {
  return {x, y};	// {x: x, y: y}
}
f(1,2) // {x: 1, y: 2}
// 方法简写
let name = '但丁';
const methods = {
    name,		// name: name
    add() {}	// add: function(){}
}
```

- 赋值器（setter）和取值器（getter）

```javascript
const cart = {
	_wheels: 4,

	get wheels () {
		return this._wheels;
	},

	set wheels (value) {
		if (value < this._wheels) {
			throw new Error('数值太小了！');
		}
		this._wheels = value;
	}
}
cart.wheels;		// 4
cart.wheels = 10;
cart.wheels;		// 10
```

### 六. Promise 对象

​	代表一个一步操作，有三种状态 `pending`（进行中）、`fulfilled`（已成功）和`rejected`（已失败）。只有异步操作的结果，可以决定当前是哪一种状态，任何其他操作都无法改变这个状态。

​	`Promise` 对象的状态改变，只有两种可能：从`pending`变为`fulfilled`（resolved）和从`pending`变为`rejected`。

​	一旦新建 Promise 对象就会立即执行。如果不设置回调函数，`Promise`内部抛出的错误，不会反应到外部。

```javascript
// resolve: 从 pending 变为 resolved
// reject:  从 pending 变为 rejected
const promise = new Promise((resolve, reject) => {
  // ... some code
  if (/* 异步操作成功 */){
    resolve(value);
  } else {
    reject(error);
  }
});

// then 的作用是为 Promise 实例添加状态改变时的回调函数。
// then方法的第一个参数是resolved状态的回调函数，第二个参数（可选）是rejected状态的回调函数
promise.then(
    () => {}, 
    err => {}
);

// 或者, 推荐使用，finally 是ES8的规范，目前不建议使用
promise
    .then(result => {···})
    .catch(error => {···})
    .finally(() => {···});
```

- 其他方法

  - Promise.all()

    ​

  - Promise.race()

    ​

  - Promise.resolve()

    ​

  - Promise.reject()

    ​

### 七. 模块

​	ES6的模块是静态的，在编译时就能确定模块间的依赖关系。模块不是对象，是通过 **export** 输出的指定代码，在通过 **import** 输入。

ES6 的模块自动采用严格模式，不管你有没有在模块头部加上`"use strict";`

一个模块就是一个独立的文件。该文件内部的所有变量，外部无法获取。如果你希望外部能够读取模块内部的某个变量，就必须使用`export`关键字输出该变量。

- export：定义模块对外接口。

```javascript
// profile.js

var name = 'dante';
function v1() { ... }
function v2() { ... }

export {
	name,
    v1 as func1,
    v2
}
// 只能使用一次。指定默认输出，加载时不用指定 {}
// 尽量使用 export default {}
export default { }   
```

- import：加载被 export 的模块。

```javascript
// main.js

import {name, func1, v2} from './profile.js';
import Vue from 'vue';
```



### … 待续



### 十. 参考资料

- http://es6.ruanyifeng.com/
- https://www.cnblogs.com/libin-1/p/6069031.html