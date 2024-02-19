---
title: 浏览器进程&EventLoop
date: 2021-08-02 11:28:58
tags: [eventloop, 渲染]
---

## 浏览器进程
### 多进程架构
![image](https://developers.google.com/web/updates/images/inside-browser/part1/browser-arch.png)
<font color=#666 size=2>浏览器单进程和多进程的架构</font>
每新开一个tab页，就会新增一个独立的浏览器进程，分配相应的CPU和内存。需要注意的是不同的浏览器有不同的优化机制，例如chrome可能会将某些进程合并，所以每一个Tab标签对应一个进程并不一定是绝对的。

### 进程组成
![image](https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/9614085650/aca8/658d/0939/cc340fa364785cb52c177a0c98220762.png)
每一个独立的浏览器进程包括：
<!-- more -->
- Browser进程（主进程）
  - 负责浏览器界面显示，与用户交互，包括导航栏，书签，前进和后退
  - 负责各个页面的管理，创建和销毁其他进程
  - 将Renderer进程得到的内存中的Bitmap，绘制到用户界面上
  - 网络资源的管理，下载等
- 插件进程：每种类型的插件对应一个进程，仅当使用该插件时才创建，例如flash插件。
- GPU进程：负责处理GPU相关的任务
- 渲染进程（浏览器内核）：负责一个 tab 内关于网页呈现的所有事情，页面渲染，脚本执行，事件处理等，包括多个线程
  - GUI渲染线程：负责解析 HTML,CSS,构建 DOM 树和 RenderObject 树,布局和绘制
  - JS引擎线程：负责处理 Javascript 脚本程序，与GUI渲染线程**互斥**（因为GUI渲染线程和JS引擎进程互斥，所以CSS加载会阻塞页面的渲染和JS的执行，JS的执行会阻塞DOM解析和页面渲染，但js的下载不会阻塞DOM解析和渲染）
  - 事件触发线程：用来控制`Event loop`，当对应的事件符合条件被触发时，该线程会把事件添加到待处理队列的队尾，等待JS引擎的处理
  - 定时触发器线程：setInterval 与 setTimeout 所在线程
  - 异步http请求线程：XMLHttpRequest 在连接后是通过浏览器新开一个线程请求

## EventLoop
了解了浏览器的进程和线程机制之后，可以帮助我们更好地理解`Event loop`的运行机制。
> [`while(true)示例`](https://event-loop-tests.glitch.me/while-true-test.html)

<img src="https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/9909918065/e858/e218/abf5/71064751baae7abd038aef7b921b27c6.png" width="450">

`while(true)`阻止了渲染和其他的页面交互。注意：现代浏览器如chrome可能会启用GPU去处理gif的渲染，可参考[Does Inifinite Javascript Loop Block The Rendering?](https://stackoverflow.com/a/58281733/8185899)。

> [`loop-setTimeout示例`](https://event-loop-tests.glitch.me/setTimeout-loop.html)

``` js
  function loop() {
    setTimeout(loop, 0);
  }
  loop();
```

<img src="https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/9911425290/b41d/7a3a/1974/3561f57fd362105f1a05b0b41a85fccc.png" width="450">

一次只能处理一次任务，所以当它处理一个任务的时候，必须绕事件循环一圈，来接收任务队列中的下一个任务；如果有渲染任务，在任务之间会去做渲染，所以setTimeout循环没有阻止渲染。

再来看一段经典的代码，console的执行顺序应该是怎样？
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
想要了解为什么会是这样的执行顺序，需要先了解`Event loop`对宏任务(macrotask)和微任务(microtask)的处理机制。
- js引擎线程负责处理js代码的执行
- 事件触发线程负责管理`Event loop`的任务队列，会将`setTimeout`（也可来自浏览器内核的其他线程,如鼠标点击、AJAX异步请求等）添加到**任务队列**中。

**宏任务**：
鼠标点击触发的事件、HTML解析、`setTimeout`、`setInterval`、`网络请求`都属于一个宏任务，在一个宏任务执行完成后，会在执行下一个宏任务执行之前触发浏览器的重新渲染。
所以`setTimeout`会在`script end`之后输出，因为`setTimeout`属于一个新的宏任务。

**微任务**：
微任务包括`mutation observer callbacks`，`promise callbacks`，在某一个宏任务执行完后，在重新渲染与开始下一个宏任务之前，就会将在它执行期间产生的所有微任务都执行完毕。
在执行js代码时，当`promise`的状态变为`settled`，会将`promise.then`、`promise.catch`的回调添加到微任务队列中。所以`promise1`、`promise2`会在`script end`之后输出，但会在下一个宏任务`setTimeout`之前输出。

### 进阶版示例
看下面的例子，点击中间的圆形，console会按照什么样的顺序输出呢？在线示例可以查看[eventloop-demo](https://codepen.io/sherrywu0917/pen/OJmJGMd)
<img src="https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/9935625955/4036/bb8a/b07a/b5d54fbc93979d1fcd15ffe459ef14f1.png" width="450">
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

此外，若是通过js自动触发调用click，顺序会有什么不一样？可以通过示例[EventLoop-autoClick](https://codepen.io/sherrywu0917/pen/yLbXYxR?editors=1111)对比手动点击触发与js自动触发click的差别。

### 总结
最后简单总结下一次`Event loop`的过程：
1. 执行一个宏任务（栈中没有就从事件队列中获取）
2. 执行过程中如果遇到微任务，就将它添加到微任务的任务队列中
3. 宏任务执行完毕后，立即执行当前微任务队列中的所有微任务（依次执行）
4. 当前宏任务执行完毕，开始检查渲染，然后GUI线程接管渲染
5. 渲染完毕后，JS引擎线程继续，开始下一个宏任务（从宏任务队列中获取）

### REFS 
- [In The Loop - JSConf.Asia](https://www.youtube.com/watch?v=cCOL7MC4Pl0)
- [Browser Architecture](https://developers.google.com/web/updates/2018/09/inside-browser-part1)
- [浏览器渲染过程与性能优化](https://juejin.cn/post/6844904040346681358?utm_source=gold_browser_extension%3Futm_source%3Dgold_browser_extension#heading-20)
- [MDN:Concurrency model and the event loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)
- [Accelerated Rendering in Chrome](https://www.html5rocks.com/zh/tutorials/speed/layers/)
- [从浏览器多进程到JS单线程，JS运行机制最全面的一次梳理](https://juejin.cn/post/6844903553795014663)


(ps: MutationObserver的监听不会说同时触发多次，多次修改只会有一次回调被触发。)



**el元素是否会闪现一下？**
``` js
document.body.appendChild(el);
el.style.display = 'none';
```
- 不会，因为js脚本作为任务的一部分必须执行结束，浏览器才会渲染。浏览器标准刷新频率60Hz，每秒更新60次。
- RAF运行在处理CSS和绘制之前
![image](https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/9911491245/cc3a/5e60/73b7/d35a9f313730f3b6f02522182797c8e0.png)


- [无线性能优化：Composite](https://fed.taobao.org/blog/taofed/do71ct/performance-composite/?spm=taofed.blogs.blog-list.10.67bd5ac8fHy0LS)
- [史上最全！图解浏览器的工作原理](https://www.infoq.cn/article/CS9-WZQlNR5h05HHDo1b)
- [CSS GPU Animation: Doing It Right](https://www.smashingmagazine.com/2016/12/gpu-animation-doing-it-right/)
- [Event loop: microtasks and macrotasks](https://javascript.info/event-loop)
- [JavaScript Event Loop Explained](https://medium.com/front-end-weekly/javascript-event-loop-explained-4cd26af121d4)
