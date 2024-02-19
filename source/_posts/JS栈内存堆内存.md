---
title: JS栈内存堆内存
date: 2019-03-20 11:20:08
tags:
---

## JS栈内存堆内存
栈存放简单变量，堆存放复杂对象，池存放常量(常量池，一般也归为栈)。

## 变量类型与内存的关系
### 基本数据类型
基本数据类型共有7种：
- String
- Number
- Boolean
- null
- undefined
- Symbol
- BigInt
基本数据类型保存在**栈内存**中，因为基本数据类型占用空间小、大小固定，通过按值来访问，属于被频繁使用的数据。
**PS: 需要注意的是闭包中的基本数据类型变量不保存在栈内存中，而是保存在堆内存中。这个问题，我们后文再说。**

### 引用数据类型
Array,Function,Object...可以认为除了上文提到的基本数据类型以外，所有类型都是引用数据类型。
引用数据类型存储在**堆内存**中，因为引用数据类型占据空间大、大小不固定。如果存储在栈中，将会影响程序运行的性能；引用数据类型在栈中存储了指针，该指针指向堆中该实体的起始地址。当解释器寻找引用值时，会首先检索其在栈中的地址，取得地址后从堆中获得实体。

## 栈内存和堆内存的优缺点
- 基本数据类型变量大小固定，并且操作简单容易，所以把它们放入栈中存储。 
- 引用类型变量大小不固定，所以把它们分配给堆中，让他们申请空间的时候自己确定大小，这样把它们分开存储能够使得程序运行起来占用的内存最小。
- 栈内存由于它的特点，所以它的系统效率较高。
- 堆内存需要分配空间和地址，还要把地址存到栈中，所以效率低于栈。

## 栈内存和堆内存的垃圾回收
- 栈内存中变量一般在它的当前执行环境结束就会被销毁被垃圾回收制回收
- 堆内存中的变量只有在所有对它的引用都结束的时候才会被回收

## 闭包和堆内存
闭包中的变量并不保存中栈内存中，而是保存在堆内存中。 这也就解释了函数调用之后之后为什么闭包还能引用到函数内的变量。
``` js
function A() {
  let a = 1;
  function B() {
      console.log(a);
  }
  return B;
}
let res = A();
```
函数 A 返回了一个函数 B，并且函数 B 中使用了函数 A 的变量，函数 B 就被称为闭包。函数 A 弹出调用栈后，函数 A 中的变量这时候是存储在堆上的，所以函数B依旧能引用到函数A中的变量。
现在的 JS 引擎可以通过**逃逸分析**辨别出哪些变量需要存储在堆上，哪些需要存储在栈上。

## 内存回收
### 标记清除算法
标记清除算法将“不再使用的对象”定义为“无法达到的对象”。 简单来说，就是从根部（在JS中就是全局对象）出发定时扫描内存中的对象。凡是能从根部到达的对象，都是还需要使用的。那些无法由根部出发触及到的对象被标记为不再使用，稍后进行回收。
工作流程：
- 垃圾收集器会在运行的时候会给存储在内存中的所有变量都加上标记。
- 从根部出发将能触及到的对象的标记清除。
- 那些还存在标记的变量被视为准备删除的变量。
- 最后垃圾收集器会执行最后一步内存清除的工作，销毁那些带标记的值并回收它们所占用的内存空间。

## 内存泄露排查
> 内存泄露：应用程序不再需要占用内存的时候，由于某些原因，内存没有被操作系统或可用内存池回收。一些编程语言提供了语言特性，可以帮助开发者做此类事情。另一些则寄希望于开发者对内存是否需要清晰明了。
> JavaScript 是一种垃圾回收语言。垃圾回收语言通过周期性地检查先前分配的内存是否可达，帮助开发者管理内存。

### JavaScript内存泄露
垃圾回收语言的内存泄漏主因是不需要的引用。大部分垃圾回收语言用的算法称之为 Mark-and-sweep 。算法由以下几步组成：
- 垃圾回收器创建了一个“roots”列表。Roots 通常是代码中全局变量的引用。JavaScript 中，“window” 对象是一个全局变量，被当作 root 。window 对象总是存在，因此垃圾回收器可以检查它和它的所有子对象是否存在（即不是垃圾）；
- 所有的 roots 被检查和标记为激活（即不是垃圾）。所有的子对象也被递归地检查。从 root 开始的所有对象如果是可达的，它就不被当作垃圾。
- 所有未被标记的内存会被当做垃圾，收集器现在可以释放内存，归还给操作系统了。
现代的垃圾回收器改良了算法，但是本质是相同的：可达内存被标记，其余的被当作垃圾回收。

### 四种常见的JavaScript内存泄漏类型：
- 意外的window全局变量
- 被遗忘的计时器或回调函数
- 脱离 DOM 的引用
<img src="/image/memory_leak_dom.png" width="900px">
id为main的DOM节点虽然已经从文档中删除，但elements仍持有对该节点的引用，导致这块内存无法被回收。
- 闭包

### 实例分析
#### 浏览器performance
切换到该tab下，勾选memory，并点击记录按钮，不断点击阅读下一章，持续一段时间。
<img src="/image/memory_0.png" width="900px">
三种迹象显示出现了内存泄漏，图中的 Nodes（绿线）、Listeners（黄线）和 JS heap（蓝线）。Nodes、Listeners稳定增长，并未下降，这是个显著的信号。

#### 浏览器 heap profile
切换到memory tab下，等待页面刷新完成，点击take heap snapshot保存当前快照。切换到下一章节，重复之前操作，保存快照。
选择Comparison，将快照与之前的进行对比，可以发现有的对象如content只增不减。
<img src="/image/heap_1.png" width="800px">
仔细分析代码，发现Content组件中有注册事件代理，监听`fontChange`事件变化，作为一个SPA，`EventProxy`一直存在于内存中，每次创建新的content时候，都会给`fontChange`事件新注册一个回调方法，之前的回调方法一直保存在回调数组中，没有被回收，导致content组件也不能被正确回收：
``` js
const eventProxy = {
  //...
  off: function(key) {
    this.onObj[key] = [];
    this.oneObj[key] = [];
  },
  //...
}

componentDidMount() {
    EventProxy.on('fontChange', diff => this.handleFontChange(diff));
}

componentWillUnmount() {
    EventProxy.off('fontChange'); //Release memory
    this.listener && window.removeEventListener('scroll', this.listener, false); //Release memory
}
```
调用`EventProxy.off('fontChange')`解除eventProxy对象对`this.handleFontChange`的引用，这样this对应的content的对象才能被正确回收；和`removeEventListener`一样的原理。
``` js
this.listener = () => {
    if(this.contentWrap && (window.pageYOffset > this.contentWrap.offsetHeight / 2)) {
        //...
        window.removeEventListener('scroll', this.listener, false); //Release memory(事件不一定被触发)
    }
}
window.addEventListener('scroll', this.listener, false)
```
虽然在`this.listener`内部有解除scroll事件监听的代码，但是要满足一定条件才会触发，所以在`componentWillUnmount`方法中也添加了`removeEventListener`，实现对scroll事件的解绑。

经过上面的优化后，再次对比，发现content对象可以被正常回收。
<img src="/image/heap_2_content.png" width="800px">

REFs: https://jinlong.github.io/2016/05/01/4-Types-of-Memory-Leaks-in-JavaScript-and-How-to-Get-Rid-Of-Them/
