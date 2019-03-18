---
title: JS函数式编程笔记(下)
date: 2018-07-13 11:20:08
tags:
---
# 示例应用
### 声明式编程
- 命令式编程（imperative）：喜欢大量使用可变对象和指令，我们总是习惯于创建对象或者变量，并且修改它们的状态或者值，或者喜欢提供一系列指令，要求程序执行。
- 声明式编程（Declarative）：对于声明式的编程范式，你不在需要提供明确的指令操作，所有的细节指令将会更好的被程序库所封装，你要做的只是提出你要的要求，声明你的用意即可。
与命令式不同，声明式意味着我们要写表达式，而不是一步一步的指示。`SQL`是典型的声明式编程，明确指出想要什么(what)，而不是如何实现(how)。
``` js
// 命令式
var makes = [];
for (i = 0; i < cars.length; i++) {
  makes.push(cars[i].make);
}


// 声明式
var makes = cars.map(function(car){ return car.make; });
```
上述命令式编程要求先声明一个数组，再去遍历，然后执行循环中具体的方法。
使用`map`的版本是一个表达式，它对执行顺序没有要求,它指明的是做什么，不是怎么做。



REFS:
[声明式编程和命令式编程的比较](http://www.aqee.net/post/imperative-vs-declarative.html)

# Hindley-Milner 类型签名
> 类型签名在写纯函数时所起的作用非常大，短短一行，就能暴露函数的行为和目的。因为类型是可以推断的，所以明确的类型签名并不是必要的；不过你完全可以写精确度很高的类型签名，也可以让它们保持通用、抽象。
JavaScript 也有一些类型检查工具，比如 [Flow](http://flowtype.org/)，或者它的静态类型方言 [TypeScript](http://www.typescriptlang.org/) 。

### 举个栗子
``` js
//  capitalize :: String -> String
var capitalize = function(s){
  return toUpperCase(head(s)) + toLowerCase(tail(s));
}
capitalize("smurf");
//=> "Smurf"
```
从类型签名来看，`capitalize`方法接收了一个`String`，最终也返回了一个`String`。

对于柯里化后的函数，对类型的签名可以有不同的理解：
``` js
//  match :: Regex -> String -> [String]
var match = curry(function(reg, s){
  return s.match(reg);
});
```
- 一种思路是接收`Regex`和`String`两种类型参数，返回一个`[String]`
- 另一种思路是`match :: Regex -> (String -> [String])`，先接收了一个`Regex`参数，返回一个新的函数，该函数接收`String`并返回一个`[String]`。

### 再举个栗子
``` js
//  reduce :: (b -> a -> b) -> b -> [a] -> b
var reduce = curry(function(f, x, xs){
  return xs.reduce(f, x);
});
```
约定俗成，a、b可以是任一种类型，但相同的字母`a`代表同一类型。其中`(b -> a -> b)`约定了函数的类型签名：
- `b`是传入f的累加器（初始值是括号后面的`-> b`）;
- `a`是遍历`[a]`得到的`currentValue`;
- `f`返回的类型是`b`，最终`reduce`方法返回的类型也就是`f`的返回。
**类型签名的美妙之处在于明确告诉我们函数做了什么**

### 缩小可能性范围
大多数语言都有范型(也成为参数多态性)，其中函数是通过一个或多个抽象类型定义的。
``` js
// head :: [a] -> a
```
引入了类型签名后，可以缩小`head`函数的可能范围，a可以是任意类型的参数，即多态性(polymorphism)，对于任意类型的参数都要支持`[a] -> a`的映射，可以帮助我们缩小函数可能性的范围。
我们甚至可以用`[a] -> a`去到类型签名搜索引擎里面搜索我们需要的函数，详细移步到[Hoogle](https://www.haskell.org/hoogle/)去尝试，很有意思。

### 自由定理
```js
// head :: [a] -> a
compose(f, head) == compose(head, map(f));
```
等式左边说的是，先获取数组的头部，然后对它调用函数 f；等式右边说的是，先对数组中的每一个元素调用 f，然后再取其返回结果的头部。这两个表达式的作用是相等的，但是前者要快得多。
可以得到一个普适的道理：如果你映射某个函数到列表上，然后对其应用 `f`，其等同于对映射应用 `f`。
数学提供的这种形式化方法，可以帮助计算机去进行类似的代码优化。

### 类型约束
``` js
// sort :: Ord a => [a] -> [a]
```
签名也可以把类型约束为一个特定的接口（interface），a必须是`Ord`对象，在强类型语言中，可以是一个自定义的接口。

# 特百惠
### 容器
为什么叫特百惠呢？因为特百惠是一个家居品品牌，代表产品是容器。
``` js
var Container = function(x) {
  this.__value = x;
}

Container.of = function(x) { return new Container(x); };
```
使用`Container.of`作为构造器，暂且认为它是把值放到容器里的一种方式。

### 第一个 functor
容器中有了值之后，我们需要一种方式来操作容器中的值：
``` js
// (a -> b) -> Container a -> Container b
Container.prototype.map = function(f){
  return Container.of(f(this.__value))
}
```
这个`map`跟数组那个著名的`map`一样，除了前者的参数是`Container a`而后者是`[a]`。它们的使用方式也几乎一致：
``` js
Container.of(2).map(function(two){ return two + 2 })
//=> Container(4)


Container.of("flamethrowers").map(function(s){ return s.toUpperCase() })
//=> Container("FLAMETHROWERS")


Container.of("bombs").map(concat(' away')).map(_.prop('length'))
//=> Container(10)
```
Container将值传给map后，通过`f`方法，我们可以对值进行任意操作，操作结束后再放入`Container`中并返回，这样就可以连续对容器中的值进行操作。这样一直调用`map`的形式，不就是前面提到的组合么。
这里面起作用的**数学魔法**就是`functor`(函子):
> functor 是实现了`map`函数并遵守一些特定规则的容器类型。

为什么要使用`functor`这种方式来处理呢？
> 即让容器自己去运用函数能给我们带来什么好处？书中给出的答案是抽象——对于函数运用的抽象。

### 薛定谔的 Maybe
定义另外一个`functor`，同样实现了`map`函数的、类似容器的数据类型`Maybe'。
``` js
var Maybe = function(x) {
  this.__value = x;
}

Maybe.of = function(x) {
  return new Maybe(x);
}

Maybe.prototype.isNothing = function() {
  return (this.__value === null || this.__value === undefined);
}

Maybe.prototype.map = function(f) {
  return this.isNothing() ? Maybe.of(null) : Maybe.of(f(this.__value));
}
```
`Maybe`和`Container`的不同点就是它新增了一个`isNothing`，在每次调用函数前，先检查它自己的值是否为空。

``` js
Maybe.of("Malkovich Malkovich").map(match(/a/ig));
//=> this._value = match(/a/ig)(""Malkovich Malkovich"")
//=> Maybe(['a', 'a'])

Maybe.of(null).map(match(/a/ig));
//=> Maybe(null)
```
点记法（dot notation syntax）已经足够函数式了，但我们更想保持一种 pointfree 的风格。碰巧的是，`map`完全有能力以`curry`函数的方式来“代理”任何 functor，柯里化后就可以方便地使用`compose`了：
``` js
//  map :: Functor f => (a -> b) -> f a -> f b
var map = curry(function(f, any_functor_at_all) {
  return any_functor_at_all.map(f);
});
```
结合前面提到的类型签名，首先约束了f是Functor类型，去掉类型约束后签名是`(a -> b) -> f a -> f b`，可以看出传入的参数是一个函数`f`: (a -> b)，一个`functor`: f a，最后返回一个`functor`: f b。
这个方法可以帮助我们取出容器中的值，可以参考下面的例子：
``` js
//  safeHead :: [a] -> Maybe(a)
var safeHead = function(xs) {
  return Maybe.of(xs[0]); // this.__value = xs[0]
};

var streetName = compose(map(_.prop('street')), safeHead, _.prop('addresses'));

streetName({addresses: []});
// Maybe(null)

streetName({addresses: [{street: "Shady Ln.", number: 4201}]});
// Maybe("Shady Ln.")
```
safeHead返回了一个Maybe对象（eg2：Maybe({street: "Shady Ln.", number: 4201})），要想对隐藏在Maybe容器中的值进行操作，需要借助`map`函数来操作，通过调用该对象的map函数，传入`_.prop('street')`函数，对`this.__value`进行操作，返回一个新的Maybe对象（eg2: Maybe("Shady Ln.")）。

### “纯”错误处理
`Maybe`实现了空和非空两种类型的分开处理，利用这种思想，我们可以对错误进行更友好、健壮地处理。
用一个简单的例子示意一下：
``` js
var Left = function(x) {
  this.__value = x;
}

Left.of = function(x) {
  return new Left(x);
}

Left.prototype.map = function(f) {
  return this;
}

var Right = function(x) {
  this.__value = x;
}

Right.of = function(x) {
  return new Right(x);
}

Right.prototype.map = function(f) {
  return Right.of(f(this.__value));
}
```

``` js
var moment = require('moment');

//  getAge :: Date -> User -> Either(String, Number)
var getAge = curry(function(now, user) {
  var birthdate = moment(user.birthdate, 'YYYY-MM-DD');
  if(!birthdate.isValid()) return Left.of("Birth date could not be parsed");
  return Right.of(now.diff(birthdate, 'years'));
});

getAge(moment(), {birthdate: '2005-12-12'});
// Right(9)

getAge(moment(), {birthdate: '20010704'});
// Left("Birth date could not be parsed")
```
其中Left用来处理错误状态，Right用来处理正常情况。

### Old McDonald had Effects...
在纯函数那一章，通过将不纯的函数包裹在另一个函数中，使得它看起来像个纯函数。类似的例子：
``` js
//  getFromStorage :: String -> (_ -> String)
var getFromStorage = function(key) {
  return function() {
    return localStorage[key];
  }
}
```
这儿将`getFromStorage`改造成，相同的输入key总会对应相同的输出：一个从`localStorage`里取出某个特定元素的函数。（然而，感觉并没什么用啊= =）

``` js
var IO = function(f) {
  this.__value = f; //f总是一个函数
}

IO.of = function(x) {
  return new IO(function() {
    return x;
  });
}

IO.prototype.map = function(f) {
  //这儿和Maybe.of(f(this.__value))实现效果一致
  return new IO(_.compose(f, this.__value));
}
```
`IO(function(){ return x })`仅仅是为了延迟执行，其实我们得到的是 IO(x)。

实际使用的时候：
``` js
////// 纯代码库: lib/params.js ///////

//  url :: IO String
var url = new IO(function() { return window.location.href; });

//  toPairs =  String -> [[String]]
var toPairs = compose(map(split('=')), split('&'));

//  params :: String -> [[String]]
var params = compose(toPairs, last, split('?'));

//  findParam :: String -> IO Maybe [String]
var findParam = function(key) {
  return map(compose(Maybe.of, filter(compose(eq(key), head)), params), url);
};

////// 非纯调用代码: main.js ///////

// 调用 __value() 来运行它！
findParam("searchTerm").__value();
// Maybe(['searchTerm', 'wafflehouse'])
```
本质上是将逻辑分成了**纯代码库**和**非纯调用代码**两部分，纯代码库最后生成的函数是唯一的，最后的风险都放在了调用者身上。
`__value`的命名并不合理，`__value`的调用会触发前面已压栈的所有操作，替换为`unsafePerformIO`更能提醒用户它的变化无常。

functor 的概念来自于范畴学，并满足一些定律。
``` js
// identity
map(id) === id;

// composition
compose(map(f), map(g)) === map(compose(f, g));
```

# Monad
### pointed functor
`of`方法并不是用来避免使用`new`关键字的，而是用来把值放到默认最小化上下文（default minimal context）中的。`pointed functor`就是实现了`of`方法的Functor。
**默认最小化上下文是什么？**
> TODO
> 
### 混合比喻
``` js
// Support
// ===========================
var fs = require('fs');

//  readFile :: String -> IO String
var readFile = function(filename) {
  return new IO(function() {
    return fs.readFileSync(filename, 'utf-8');
  });
};

//  print :: String -> IO String
var print = function(x) {
  return new IO(function() {
    console.log(x);
    return x;
  });
}

// Example
// ===========================
//  cat :: IO (IO String)
var cat = compose(map(print), readFile);

cat(".git/config")
// IO(IO("[core]\nrepositoryformatversion = 0\n"))
```
最后返回了一个嵌套两层的`IO`对象，如果想要再次调用，对其中的值进行处理，则需要`map(map(f))`；像是穿着两套防护服在工作，很奇怪。
``` js
Maybe.prototype.join = function() {
  return this.isNothing() ? Maybe.of(null) : this.__value;
}

var mmo = Maybe.of(Maybe.of("nunchucks"));
// Maybe(Maybe("nunchucks"))

mmo.join();
// Maybe("nunchucks")

```
定义一个`join`方法，可以帮助我们简单地移除一层嵌套，在使用的时候，我们可以在每个map后面，都调用一次join方法，但是我们期望的不止如此。
```js
//  join :: Monad m => m (m a) -> m a
var join = function(mma){ return mma.join(); }

//  firstAddressStreet :: User -> Maybe Street
var firstAddressStreet = compose(
  join, map(safeProp('street')), join, map(safeHead), safeProp('addresses')
);
```

### chain函数
把`map`和`join`封装成chain函数，则`firstAddressStreet`方法可以改写成：
``` js
//  chain :: Monad m => (a -> m b) -> m a -> m b
var chain = curry(function(f, m){
  return m.map(f).join(); // 或者 compose(join, map(f))(m)
});

//  firstAddressStreet :: User -> Maybe Street
var firstAddressStreet = compose(
  chain(safeProp('street')), chain(safeHead), safeProp('addresses')
);
```
> 简单说，Monad就是一种设计模式，表示将一个运算过程，通过函数拆解成互相连接的多个步骤。你只要提供下一步运算所需的函数，整个运算就会自动进行下去。

REFS:
[图解 Monad](http://www.ruanyifeng.com/blog/2015/07/monad.html)

