---
title: ES6 学习
date: 2017-03-20 19:40:59
tags: [es6, js]
---
> from: http://es6.ruanyifeng.com

## let命令
  - let取代var, let仅在代码块中有效
  - for循环计数器适合用let
  - let不存在变量提升
  - 暂时性死区: 
    ES6 明确规定，如果区块中存在let和const命令，这个区块对这些命令声明的变量，从一开始就形成了封闭作用域。**凡是在声明之前就使用这些变量，就会报错**。
    总之，在代码块内，使用let命令声明变量之前，该变量都是不可用的。这在语法上，称为“暂时性死区”（temporal dead zone，简称 TDZ）。
  ``` js
  if (true) {
    console.log(typeof a);

    // TDZ开始
    tmp = 'abc'; // ReferenceError
    console.log(tmp); // ReferenceError
    // Uncaught ReferenceError: Cannot access 'tmp' before initialization
    // at <anonymous>:3:7
    // (anonymous) @ VM211:3

    let tmp; // TDZ结束
    console.log(tmp); // undefined  
  }
  ```
  - 不允许重复声明
  - 块级作用域（全局作用域、函数作用域）

## const命令
  - 声明常量

## 顶层对象，在浏览器中指的是window对象
  - es6的var命令和function命令声明的全局变量依然是顶层对象的属性
  - let、const、class命令声明的全局变量不属于顶层对象的属性

<!-- more -->

## 数组的解构赋值
  - 只要某种数据结构有Iterator接口，都可以采用数组形式的解构赋值
  - 解构可以设置默认值
  - 对象也可以解构，数组的解构是按次序排列，对象的解构要求变量名必须和对象的属性名相同
  - 数值和布尔值的解构赋值比较特别，转为了对象
  - 函数参数也可以解构赋值
  - 用处：
      - 交换变量的值
      - 从函数返回多个值
      - 函数参数的定义
      - 提取JSON对象中的数据
      - 函数参数的默认值
      - 遍历map解构 ：任何部署了Iterator接口的对象，都可以用for…of循环遍历（Array,String,Set,Map）
      - 输入模块的指定方法

## Unicode表示字符，可以使用{}
  - 模板字符串${}
  - includes, startsWith, endsWith, repeat, padStart, padEnd

## 数值的扩展
  - isFinite()  isNaN()
  - 将parseInt()和parseFloat()方法移植到Number对象上，目的是减少全局性方法，使语言逐步模块化
  - isInteger()
  - 新增了极小的常量Number.EPSILON
  - 最大值Number.MAX_SAFE_INTEGER， 最小值Number.MAX_SAFE_INTEGER Number.isSafeInteger()

## 数组的扩展
  - from方法将类似数组的对象和可遍历的对象转为真正的数组
``` javascript
let arrayLike = {
    '0': 'a',
    '1': 'b',
    '2': 'c',
    length: 3
};

// ES5的写法
var arr1 = [].slice.call(arrayLike); // ['a', 'b', 'c']

// ES6的写法
let arr2 = Array.from(arrayLike); // ['a', 'b', 'c']
```
实际应用中，常见的类似数组的对象是DOM操作返回的NodeList集合，以及函数内部的arguments对象。Array.from都可以将它们转为真正的数组。
``` javascript
// NodeList对象
let ps = document.querySelectorAll('p');
Array.from(ps).forEach(function (p) {
  console.log(p);
});

// arguments对象
function foo() {
  var args = Array.from(arguments);
  // ...
}
```
  - of方法将一组值转换为数组

## 函数的扩展
  - 可以设置参数的默认值
  - 函数的length属性返回没有指定默认值的参数个数
  - reset参数：用于获取函数的多余参数，形式为…变量名，代表一个数组
``` javascript
// arguments变量的写法
function sortNumbers() {
  return Array.prototype.slice.call(arguments).sort();
}

// rest参数的写法
const sortNumbers = (...numbers) => numbers.sort();
```
  - 扩展运算符…：好比reset参数的逆运算，将一个数组转为用逗号分隔的参数序列
    - 替代数组的apply方法
    ``` javascript
    // ES5的写法
    function f(x, y, z) {
      // ...
    }
    var args = [0, 1, 2];
    f.apply(null, args);
    
    // ES6的写法
    function f(x, y, z) {
      // ...
    }
    var args = [0, 1, 2];
    f(...args);
    ```
    - 合并数组
    - 与解构赋值结合
    - 函数的返回值
    - 任何Iterator接口的对象，都可以用扩展运算符转为真正的数组。
    ``` javascript
    var obj = {a: 1, b: 2};
    let arr = [...obj]; // TypeError: Cannot spread non-iterable object
    ```
    - 对于没有iterator接口的对象，使用...语法会报错
    - 与React中的JSX扩展语法不同
  - 函数的name属性
  - 箭头函数=>
  - 函数绑定运算符是并排的两个双冒号（::），左边是对象，右边是一个函数，返回的还是原对象，可以采用链式写法
  - “尾调用优化”（Tail call optimization），即只保留内层函数的调用帧。如果所有函数都是尾调用，那么完全可以做到每次执行时，调用帧只有一项，这将大大节省内存

## 对象的扩展

## Proxy和Refelct
``` javascript
var proxy = new Proxy(target, handler);
```
new Proxy()表示生成一个Proxy实例，target参数表示所要拦截的目标对象，handler也是一个对象，用来定制拦截行为
``` javascript
var proxy = new Proxy({}, {
  get: function(target, property) {
    return 35;
  }
});

proxy.time // 35
proxy.name // 35
proxy.title // 35
```
### 用处
1. 属性拦截
2. 拦截过滤各种操作，如new/defineProperty/delete/getPrototypeOf等
3. 私有属性模拟
has方法用来拦截HasProperty操作，即判断对象是否具有某个属性时，这个方法会生效。典型的操作就是in运算符。
``` javascript
var handler = {
  has (target, key) {
    if (key[0] === '_') {
      return false;
    }
    return key in target;
  }
};
var target = { _prop: 'foo', prop: 'foo' };
var proxy = new Proxy(target, handler);
'_prop' in proxy // false
```

## Promise对象
``` javascript
var promise = new Promise(function(resolve, reject) {
  // ... some code
  if (/* 异步操作成功 */){
    resolve(value);
  } else {
    reject(error);
  }
});
```
resolve将Promise对象从Pending变成Resolved
reject将Promise对象从Pending变成Rejected
``` javascript
promise.then(function(value) {
  // success
}, function(error) {
  // failure
});
```
resolve和reject可以将参数传递给then方法里面的回调函数

## Symbol对象属性
``` javascript
var isMoving = Symbol("isMoving");
...
if (element[isMoving]) {
  smoothAnimations(element);
}
element[isMoving] = true;
```
- symbol-keyed属性不能通过.操作符来访问，必须使用方括号的方式
- 判断：if (isMoving in element)
- 删除：delete element[isMoving]
- for…in、Object.keys(obj) 和 Object.getOwnPropertyNames(obj)只会遍历到以字符串作为键的属性
- Object.getOwnPropertySymbols(obj)只会遍历所有的Symbol键
- Reflect.ownKeys(obj)会返回对象的所有字符串和Symbol键

## Class
``` javascript
//定义类
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }

  toString() {
    return '(' + this.x + ', ' + this.y + ')';
  }
}
```
this关键字代表实例对象。类的方法都是定义在prototype上，与传统的prototype实现的类一致。

``` javascript
class Point {
  // ...
}

typeof Point // "function"
Point === Point.prototype.constructor // true
```
上面代码表明，类的数据类型就是function，类本身就指向构造函数。

``` javascript
class B {}
let b = new B();

b.constructor === B.prototype.constructor // true
B.prototype.constructor === B //true
```
在类的实例上调用方法，其实就是调用原型上的方法。所以b实例的constructor方法就是B类原型的constructor方法。
prototype对象的constructor属性，直接指向“类”的本身。

``` javascript
class Point {
  
}

Object.assign(Point.prototype, {
  toString(){},
  toValue(){}
});
```
通过Object的assign方法可以一次向Point添加多个方法。

``` javascript
class Point {
  constructor(x, y) {
    // ...
  }

  toString() {
    // ...
  }
}

Object.keys(Point.prototype)
// []
Object.getOwnPropertyNames(Point.prototype)
// ["constructor","toString"]
```
上面代码中，toString方法是Point类内部定义的方法，它是不可枚举的。这一点与ES5的行为不一致。Object.keys这儿用来判断是否可枚举。
### 注意点
- 类和模块的内部，默认就是严格模式，所以不需要使用use strict指定运行模式。
- 不存在变量提升。
- static关键字，就表示该方法不会被实例继承，而是直接通过类来调用，但可以被子类继承。

## Set集合
用最简洁的代码实现数组去重：
ES6实现：
``` javascript
[...new Set([1,2,3,1,'a',1,'a'])]
```
ES5实现：
``` javascript
[1,2,3,1,'a',1,'a'].filter(function(ele,index,array){
    return index===array.indexOf(ele)
})
```
