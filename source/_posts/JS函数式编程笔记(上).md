---
title: JS函数式编程笔记(上)
date: 2018-07-02 19:20:49
tags: [函数式编程, js]
---
# 一等公民的函数
### 为啥说函数是一等公民？
- 函数和其他对象一样，可以像对待任何其他数据类型一样对待它们——把它们存在数组里，当作参数传递，赋值给变量...等等。
关键点在于，调用的时候没必要再去包裹一层多余的函数，因为二者是等价的。
``` js
\\包裹了多余的函数
var greeting = function(name) {return hi(name);}
\\ 一等公民式的调用
var greeting = hi;
```

### 一等公民函数调用的好处
``` js
//最初的函数
httpGet('/post/2', function(json){
  return renderPost(json);
});

// 需要增加对err异常的处理，要处理的地方比较多
httpGet('/post/2', function(json, err){
  return renderPost(json, err);
});

// 如果使用一等公民函数的形式，则其他的改动会少很多
httpGet('/post/2', renderPost);
```

<!-- more -->

# 纯函数的好处
> 纯函数是这样一种函数，即相同的输入，永远会得到相同的输出，而且没有任何可观察的**副作用**。
### 函数的副作用
slice和splice相比，slice是一种纯函数，splice会修改自身数据，在函数式编程中比较讨厌这种笨函数。
另外，看这个例子：

``` js
// 不纯的
var minimum = 21;

var checkAge = function(age) {
  return age >= minimum;
};
```

`checkAge`依赖外部变量`minimum`，`minimum`发生变化的时候，会影响`checkAge`函数的返回结果，增加认知负荷。
也可以调用`Object.freeze({minimum: 21})`将minimum变成一个不可变对象。
> 副作用是在计算过程中，系统状态的一种变化，或者是与外部世界进行的可观察的交互。也是滋生bug的温床。

### 追求“纯”的理由
- 可缓存性（Cacheable）
可以通过延迟执行的方式把不纯的函数转换为纯函数：
``` js
var pureHttpCall = memoize(function(url, params){
  return function() { return $.getJSON(url, params); }
});
```
之所以是纯函数，因为总是根据相同的输入返回相同的输出：同一个发送http请求的函数。
- 可移植性／自文档化（Portable / Self-Documenting）
与环境无关；注入依赖或者通过参数传递。
>  Erlang 语言的作者 Joe Armstrong 说的这句话：“面向对象语言的问题是，它们永远都要随身携带那些隐式的环境。你只需要一个香蕉，但却得到一个拿着香蕉的大猩猩...以及整个丛林”。
- 可测试性（Testable）
[Quickcheck](http://hackage.haskell.org/package/QuickCheck)——一个为函数式环境量身定制的测试工具
- 合理性（Reasonable）
引用透明性:如果一段代码可以替换成它执行所得的结果，而且是在不改变整个程序行为的前提下替换的，那么我们就说这段代码是引用透明的。
- 并行代码
可以并行运行任意纯函数。因为纯函数根本不需要访问共享的内存，而且根据其定义，纯函数也不会因副作用而进入竞争态（race condition）。


# 柯里化
> 在数学和计算机科学中，柯里化是一种将使用多个参数的一个函数转换成一系列使用一个参数的函数的技术。
另一种理解是：只传递给函数一部分参数来调用它，让它返回一个函数去处理剩下的参数。
### 例子
``` js
function add(a, b) {
    return a + b;
}

// 执行 add 函数，一次传入两个参数即可
add(1, 2) // 3

// 函数curry
var add = function(a) {
    return function(b) {
        return a + b;
    }
}
// 每次传入一个参数
add(1)(2) // 3
```
### 核心思想
可以借助Lodash 中的 curry 方法帮我们实现函数柯里化，核心思想是————比较多次接受的参数总数与函数定义时的入参数量，当接受参数的数量大于或等于被 Currying 函数的传入参数数量时，就返回计算结果，否则返回一个继续接受参数的函数。
``` js
function trueCurrying(fn, ...args) {

    if (args.length >= fn.length) {

        return fn(...args)

    }

    return function (...args2) {

        return trueCurrying(fn, ...args, ...args2)

    }
}
```
### 用处
curry的用处十分广泛，给函数传入一些参数后，可以得到一些新的函数。例如下面的`getChildren`函数，传给柯里化后的map函数，会返回一个接收参数类型为数组的新函数。
```
var curry = require('lodash').curry;
var map = curry(function(f, ary) {
  return ary.map(f);
});

var getChildren = function(x) {
  return x.childNodes;
};
var allTheChildren = map(getChildren);

// 返回所有div的子节点
allTheChildren(document.getElementsByTagName('div'))
```

# 代码组合
### 简介
``` js
var compose = function(f,g) {
  return function(x) {
    return f(g(x));
  };
};
```
`f`和`g`是两个函数，通过组合方式返回一个新的函数，`x`就是在两个管道之间传输的值。
举个例子，如果希望给`send in the clowns`加上`!`，并且转成大写，可以使用函数组合的方法来实现这个功能。
``` js
var toUpperCase = function(x) { return x.toUpperCase(); };
var exclaim = function(x) { return x + '!'; };
var shout = compose(exclaim, toUpperCase);

shout("send in the clowns");
//=> "SEND IN THE CLOWNS!"
```
`compose`使用主要特点有：
    - 代码的运行顺序是从右向左，创建了一个从右向左的数据流，初始函数一定放到参数的`最右面`；
    - `compose`的参数是函数，返回的也是一个函数；
    - 除了第一个函数的接受参数，其他函数的接受参数都是上一个函数的返回值，所以初始函数的参数是多元的，而其他函数的接受值是一元的；

### 结合律
组合的概念来源于数学课本，满足组合的特性——结合律
``` js
// 结合律（associativity）
var associative = compose(f, compose(g, h)) == compose(compose(f, g), h);
// true
```
结合律的一大好处是任何一个函数分组都可以被拆开来，然后再以它们自己的组合方式打包在一起。例如可以进行下面的重构：
``` js
var loudLastUpper = compose(exclaim, toUpperCase, head, reverse);

// 或
var last = compose(head, reverse);
var loudLastUpper = compose(exclaim, toUpperCase, last);

// 或
var last = compose(head, reverse);
var angry = compose(exclaim, toUpperCase);
var loudLastUpper = compose(angry, last);

// 更多变种...
```

### pointfree
> Pointfree style means never having to say your data.
通过管道把数据在接受单个参数的函数间传递，不需要去声明中间的变量。
``` js
// 非 pointfree，因为提到了数据：name
var initials = function (name) {
  return name.split(' ').map(compose(toUpperCase, head)).join('. ');
};

// pointfree
var initials = compose(join('. '), map(compose(toUpperCase, head)), split(' '));

initials("hunter stockton thompson");
// 'H. S. T'
```
pointfree 模式能够帮助我们减少不必要的命名，让代码保持简洁和通用。

**组合像一系列管道那样把不同的函数联系在一起，数据就可以也必须在其中流动**——毕竟纯函数就是输入对输出，所以打破这个链条就是不尊重输出，就会让我们的应用一无是处。

PS: **[Ramda](https://ramdajs.com/)
- 数据一律放在最后一个参数，理念是"function first，data last";
- 库所有的函数都支持柯里化**，可以很好地实践FP。

REFS:
[使用JavaScript实现“真·函数式编程”](http://jimliu.net/2015/10/21/real-functional-programming-in-javascript-1/)
[Ramda 函数库参考教程](http://www.ruanyifeng.com/blog/2017/03/ramda.html)