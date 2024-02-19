---
title: 浏览器进程&EventLoop
date: 2021-08-02 11:28:58
tags: [eventloop, 渲染]
hidden: true
---

## 浏览器多进程
![image](https://developers.google.com/web/updates/images/inside-browser/part1/browser-arch.png)
<font color=#666 size=2>浏览器单进程和多进程的架构</font>

每新开一个tab页，就会新增一个独立的浏览器进程，分配相应的CPU和内存。
**注意：**在这里浏览器应该也有自己的优化机制，有时候打开多个tab页后，可以在Chrome任务管理器中看到，有些进程被合并了 （所以每一个Tab标签对应一个进程并不一定是绝对的）。

<!-- more -->

## 浏览器包括哪些进程
![image](https://developers.google.com/web/updates/images/inside-browser/part1/browserui.png)

### Browser进程（主进程）
浏览器的主进程（负责协调、主控），只有一个。作用有
- 负责浏览器界面显示，与用户交互，包括导航栏，书签，前进和后退
- 负责各个页面的管理，创建和销毁其他进程
- 将Renderer进程得到的内存中的Bitmap，绘制到用户界面上
- 网络资源的管理，下载等

### 插件进程
- 每种类型的插件对应一个进程，仅当使用该插件时才创建，例如flash插件。

### GPU进程
- 只有一个，一旦 renderLayer 提升为了合成层就会有自己的绘图上下文，并且会开启硬件加速，有利于性能提升

### 渲染进程
![image](https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/9614085650/aca8/658d/0939/cc340fa364785cb52c177a0c98220762.png)

**GUI渲染线程**
> 解析 HTML,CSS,构建 DOM 树和 RenderObject 树,布局和绘制
- 解析HTML文件，构建DomTree，同时下载css文件
- 解析CSS文件，构建CSSOMTree
- DomTree 和 CSSOMTree结合，生成RenderObject 树 (所以CSS加载会阻塞DOM渲染)
- 绘制页面的图层像素，再交给GPU
> 重绘和回流（现代浏览器会对频繁的回流或重绘操作进行优化：浏览器会维护一个队列,把所有引起回流和重绘的操作放入队列中,如果队列中的任务数量或者时间间隔达到一个阈值的,浏览器就会将队列清空,进行一次批处理,这样可以把多次回流和重绘变成一次。）

**JS引擎线程**（因为GUI渲染线程和JS引擎进程互斥，所以CSS加载会阻塞页面的渲染和JS的执行，JS的执行会阻塞DOM解析和页面渲染，但js的下载不会阻塞DOM解析和渲染）
>  JS 内核,负责处理 Javascript 脚本程序
- async独立于DOMContentLoaded，加载完立即执行(so乱序: 不会修改 DOM 或 CSSOM 时,推荐使用 async)
- defer脚本需要等到文档解析后（DOMContentLoaded事件前）执行

**事件触发线程**
- 归属于浏览器，用来控制`Event loop`
- 当对应的事件符合条件被触发时，该线程会把事件添加到待处理队列的队尾，等待JS引擎的处理；如setTimeout回调函数、异步请求回调函数。

**定时触发器线程**
- setInterval 与 setTimeout 所在线程
- 浏览器定时计数器并不是由JavaScript引擎计数的,（因为JavaScript引擎是单线程的, 如果处于阻塞线程状态就会影响记计时的准确）
- 因此通过单独线程来计时并触发定时（计时完毕后，添加到事件队列中，等待JS引擎空闲后执行）

**异步http请求线程**
- XMLHttpRequest 在连接后是通过浏览器新开一个线程请求
- 检测到状态变更时，如果设置有回调函数，异步线程就产生状态变更事件，将这个回调再放入事件队列中。再由JavaScript引擎执行。

## EventLoop
了解Event loop，首先看一段经典的代码，console的执行顺序应该是怎样？
``` js
console.log('script start');

setTimeout(function () {
  console.log('setTimeout');
}, 0);

Promise.resolve()
  .then(function () {
    console.log('promise1');
  })
  .then(function () {
    console.log('promise2');
  });

console.log('script end');
```
正确的执行顺序为：`script start`, `script end`,`promise1`,`promise2`,`setTimeout`。
### 为什么是这样的顺序？
想要了解为什么会是这样的执行顺序，需要先了解浏览器中宏任务(task、macrotask)和微任务(microtask)的处理机制，即`Event loop`。
- js引擎线程负责处理js代码的执行
- 事件触发线程负责管理`Event loop`的任务队列，会将`setTimeout`（也可来自浏览器内核的其他线程,如鼠标点击、AJAX异步请求等）添加到任务队列中

每一个js引擎线程都有自己的`Event loop`，所以每个线程可以独立地运行。

**宏任务**：
鼠标点击触发的事件、HTML解析、`setTimeout`、`setInterval`、`网络请求`都属于一个宏任务，在一个宏任务执行完成后，会在执行下一个宏任务执行之前触发浏览器的重新渲染。
所以`setTimeout`会在`script end`之后输出，因为`setTimeout`属于一个新的宏任务。

**微任务**：
微任务包括`mutation observer callbacks`，`promise callbacks`，在某一个宏任务执行完后，在重新渲染与开始下一个宏任务之前，就会将在它执行期间产生的所有微任务都执行完毕。
在执行js代码时，当`promise`的状态变为`settled`，会将`promise.then`、`promise.catch`的回调添加到微任务队列中。所以
`promise1`、`promise2`会在`script end`之后输出，但会在下一个宏任务`setTimeout`之前输出。

### 进阶版示例
看下面的例子，点击中间的方形，console会按照什么样的顺序输出呢？在线示例可以查看[eventloop-demo](https://codepen.io/sherrywu0917/pen/OJmJGMd)
![image](https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/9629043427/1f42/2142/5d88/bba89b3cf5c5d8a67a6cd239e6c0120c.png)
``` html
<div class="outer">
  <div class="inner"></div>
</div>
```
``` js
var outer = document.querySelector('.outer');
var inner = document.querySelector('.inner');

// 监听outer元素属性变化
new MutationObserver(function () {
  console.log('mutate');
}).observe(outer, {
  attributes: true,
});

function onClick() {
  console.log('click');

  setTimeout(function () {
    console.log('timeout');
  }, 0);

  Promise.resolve().then(function () {
    console.log('promise');
  });

  outer.setAttribute('data-random', Math.random());
}

inner.addEventListener('click', onClick);
outer.addEventListener('click', onClick);
```
正确的执行顺序为：`click`,`promise`,`mutate`,`click`,`promise`,`mutate`,`timeout`,`timeout`。
**Event loop执行过程**
1. click事件首先触发inner元素的`onClick`事件，并且冒泡到outer元素上，将outer元素的`onClick`事件添加到宏任务队列中
2. 执行inner元素的`onClick`函数，输出`click`
   1. 执行到`setTimeout`将其添加到宏任务队列中，此时的宏任务队列：[`outer onClick`, `inner setTimeout`]
   2. 执行到`Promise.resolve()`，将`promise.then`添加到微任务队列中，此时的微任务队列[`inner promise.then`]
   3. 执行到`outer.setAttribute`，将`MutationObserver callback`添加到微任务队列中，此时的微任务队列[`inner promise.then`, `inner MutationObserver callback`]
   4. `onClick`函数代码块执行完之后，开始按顺序执行所有的微任务队列，依次输出`promise`，`mutate`
3. 从宏任务队列中取出`outer onClick`，内部执行顺序和inner一致
   1. 输出`click`
   2. 把`setTimeout`添加到宏任务队列中，此时的宏任务队列为[`inner setTimeout`, `outer setTimeout`]
   3. 依次输出`promise`，`mutate`
4. 从宏任务队列中取出`inner setTimeout`，输出`setTimeout`
5. 从宏任务队列中取出`outer setTimeout`，输出`setTimeout`

### 总结
最后简单总结下一次`Event loop`的过程：
1. 执行一个宏任务（栈中没有就从事件队列中获取）
2. 执行过程中如果遇到微任务，就将它添加到微任务的任务队列中
3. 宏任务执行完毕后，立即执行当前微任务队列中的所有微任务（依次执行）
4. 当前宏任务执行完毕，开始检查渲染，然后GUI线程接管渲染
5. 渲染完毕后，JS引擎线程继续，开始下一个宏任务（从宏任务队列中获取）

REFS: 
- [Browser Architecture](https://developers.google.com/web/updates/2018/09/inside-browser-part1)
- [浏览器渲染过程与性能优化](https://juejin.cn/post/6844904040346681358?utm_source=gold_browser_extension%3Futm_source%3Dgold_browser_extension#heading-20)
- [Accelerated Rendering in Chrome](https://www.html5rocks.com/zh/tutorials/speed/layers/)
- [从浏览器多进程到JS单线程，JS运行机制最全面的一次梳理](https://juejin.cn/post/6844903553795014663)
- [无线性能优化：Composite](https://fed.taobao.org/blog/taofed/do71ct/performance-composite/?spm=taofed.blogs.blog-list.10.67bd5ac8fHy0LS)
- [史上最全！图解浏览器的工作原理](https://www.infoq.cn/article/CS9-WZQlNR5h05HHDo1b)
- [CSS GPU Animation: Doing It Right](https://www.smashingmagazine.com/2016/12/gpu-animation-doing-it-right/)
- [MDN:Concurrency model and the event loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)
- [In The Loop - JSConf.Asia](https://www.youtube.com/watch?v=cCOL7MC4Pl0)
- [Event loop: microtasks and macrotasks](https://javascript.info/event-loop)
- [JavaScript Event Loop Explained](https://medium.com/front-end-weekly/javascript-event-loop-explained-4cd26af121d4)
