---
title: HTML渲染和资源加载
date: 2021-03-16 20:42:41
tags: [渲染, load]
---

### 渲染树构建、布局及绘制
1. 处理HTML标记并生成DOM树
2. 处理CSS标记并生成CSSOM树
3. 将DOM树和CSSOM树合成渲染树：
   - 从 DOM 树的根节点开始遍历每个可见节点（会忽略“display: none”的元素）
   - 对于每个可见节点，为其找到适配的 CSSOM 规则并应用它们
   - 发射可见节点，连同其内容和计算的样式
4. 根据渲染树来**布局**，以计算每个节点的几何信息：计算它们在设备视口内的确切位置和大小
   - 输出是一个“盒模型”，它会精确地捕获每个元素在视口内的确切位置和尺寸：所有相对测量值都转换为屏幕上的绝对像素
5. **绘制**：将渲染树中的每个节点转换成屏幕上的实际像素
   - 绘制这一个过程应该是：RenderLayers渲染层 -> GraphicsLayer图形层（GPU）
   - 特殊的合成层，例如translate3d、will-change等
     - 3D transforms：translate3d、translateZ 等
     - video、canvas、iframe 等元素
     - 通过 Element.animate() 实现的 opacity 动画转换
     - 通过 СSS 动画实现的 opacity 动画转换
     - position: fixed
     - 具有 will-change 属性
     - 对 opacity、transform、filter、backdropfilter 应用了 animation 或者 transition
结论：CSS解析和DOM解析是可以并行的，二者完成后再去执行DOM渲染，但是CSS解析会阻塞JS的执行，JS的执行会阻塞DOM的解析。所以，要将CSS放在头部，JS资源放在尾部。

### 回流Reflow和重绘Repaint
#### 回流
 - 页面初次渲染
 - 添加或者删除可见的 dom 元素
 - 元素尺寸、位置、内容发生改变
 - 浏览器窗口大小改变
 - 元素字体大小变化
 - 激活 CSS 伪类（例如：:hover）
 - 查询某些属性或调用某些方法
    - clientWidth、clientHeight、clientTop、clientLeft （元素可视区域的宽度）
    - offsetWidth、offsetHeight、offsetTop、offsetLeft （元素包括滚动条的宽度）
    - scrollWidth、scrollHeight、scrollTop、scrollLeft （元素的实际宽度）
    - getComputedStyle()
    - getBoundingClientRect()  ({x,y,width,height,left,right,top,bottom} 其中right和bottom分别指元素相对于页面左边、上面的距离;使用document.documentElement.scrollTop和window.pageYOffset获取页面滚动的距离，window.pageYOffset IE9以下不兼容；scrollTop是该元素到视口可见内容顶部的距离，如果该元素没有垂直方向的滚动条，则该值始终为0。)
    - scrollTo()

#### 重绘
- background color visibility border-style border-radius设置等

#### 获取宽高的不同方式
    - screen.height 屏幕的高度
    - screen.availHeight 浏览器窗口在屏幕上可占用的最大高度
    - window.outerHeight window.innerHeight 浏览器的高度包括导航栏/不包括导航栏
    - element.clientWidth offsetWidth scrollWidth  针对元素的，其中offsetTop和offsetLeft是相对于position为非static的父元素offsetParent，

#### 如何优化
- **避免频繁操作样式**，最好一次性重写style属性，或者将样式列表定义为class并一次性更改class属性。
- **避免频繁操作DOM**，创建一个documentFragment，在它上面应用所有DOM操作，最后再把它添加到文档中（virtual dom思想）。
- 也可以先为元素设置display: none，操作结束后再把它显示出来。因为在display属性为none的元素上进行的DOM操作不会引发回流和重绘。
- **缓存布局信息**避免频繁读取会引发回流/重绘的属性，如果确实需要多次使用，就用一个变量缓存起来。
- 对具有复杂动画的元素使用绝对定位，使它**脱离文档流**，否则会引起父元素及后续元素频繁回流。
- **分离读写操作**

#### 渲染队列
``` js
div.style.left = '10px';
div.style.top = '10px';
div.style.width = '10px';
div.style.height = '10px';
```
理论上会发生4次回流，但是实际上只会发生1次回流，因为浏览器的渲染队列机制。当我们修改了元素的几何属性时，导致浏览器的回流或重绘时，会将操作放到渲染队列中，然后在达到一定数量或者一定时间时一次性统一渲染。
``` js
div.style.left = '10px';
console.log(div.offsetLeft);

div.style.top = '10px';
console.log(div.offsetTop);

div.style.width = '20px';
console.log(div.offsetWidth);

div.style.height = '20px';
console.log(div.offsetHeight);
```
会触发四次回流和重绘，因为在调用`offsetLeft`相关属性的时候，如果渲染队列中有数据的话，会先触发回流，因为在这样才能保证取到的数据的准确性。

#### [为什么在DOMContentLoaded事件中添加动画不成功？](https://stackoverflow.com/questions/42891628/why-dont-transitions-on-svg-work-on-domcontentloaded-without-delay)
DOMContentLoaded MDN上的定义：
> it fires when the DOM has been parsed, but before the CSSOM has been (and thus before styles have been applied)
> 在CSSOM ready之前，所以此时css样式一定没有渲染
``` js
! function() {
  // 1. 查询offsetTop属性，强行触发回流和重绘
  window.addEventListener('DOMContentLoaded', function(){
      document.body.offsetTop; // force a CSS repaint
      var logo2 = document.querySelector("svg");
      logo2.classList.add('start');
    });
  }();

  // 2. setTimeout or raf(() => raf(() => {}))
```

### 改变阻塞模式：defer 与 async
 async 与 defer 属性对inline script是无效的。
 > MDN async: For classic scripts, if the async attribute is present, then the classic script will be fetched in parallel to parsing and evaluated as soon as it is available.

 > MDN defer: This Boolean attribute is set to indicate to a browser that the script is meant to be executed after the document has been parsed, but before firing DOMContentLoaded. The defer attribute should only be used on external scripts.

##### async文件
  - 加载好就会立即执行—无论此刻是HTML解析阶段还是DOMContentLoaded触发之后，但会阻塞load事件。
  - script中代码的执行顺序不定。
  - 执行顺序：一定会在页面的load事件前执行，但可能会在DOMContentLoaded事件触发之前或之后执行。 

##### defer文件
  - 载入JavaScript文件时不阻塞HTML的解析，执行阶段被放到HTML标签解析完成之后
  - 不改变script中代码的执行顺序
  - 执行顺序：在其他没有添加defer属性的script之后执行，在DOMContentLoaded事件之前

 ``` js
 console.log(document.createElement("script").async); // true 动态创建的script默认是异步的。

 const style = document.createElement("link");
 style.rel = "stylesheet"; 
 style.href = "index.css";
 document.head.appendChild(style); // 阻塞？ 已知的是，Chrome 中已经不会阻塞渲染，Firefox、IE 在以前是阻塞的。
 ```
 