---
title: SSR渲染
date: 2023-09-17 12:12:12
tags: [SSR]
---

# webpack打包
nodejs遵循 commonjs规范，文件的导入导出如下：
``` js
// 导出
module.exports = someModule
// 导入
const module = require('./someModule')
```
而我们通常所写的 react代码是遵循 esModule规范的，文件的导入导出如下：
``` js
// 导出
export default someModule
// 导入
import module from './someModule'
```
所以想要让 react代码兼容于服务器端，就必须先解决这两种规范的兼容问题，实际上 react是可以直接以 commonjs规范来书写的，例如：
``` js
const React = require('react')
```
使用webpack编译的作用是：
- 将 jsx编译为 node认识的原生 js代码
- 将 esModule代码编译成 commonjs的

# 客户端如何处理ssr渲染的数据
### server端
ReactDOM.renderToString返回的HTML片段插入到ejs模板中，再通过node的response.render返回给浏览器
### client端
- ReactDOM.render()会将挂载dom节点的所有子j节点全部清空掉，再重新生成子节点。
- ReactDOM.hydrate()则会复用挂载dom节点的子节点，并将其与react的virtualDom关联上。
初次渲染可以调用两种方法：ReactDOM.render和ReactDOM.hydrate。后者就是直接告诉ReactDOM需要hydrate，目前来说如果你调用的是render，但是 React 会调用下面的方法检测是否可以hydrate，如果可以他会提醒你应该使用hydrate。
``` js
const ELEMENT_NODE = 1;
const DOCUMENT_NODE = 9;
const ROOT_ATTRIBUTE_NAME = 'data-reactroot';

function getReactRootElementInContainer(container) {
  if (!container) {
    return null;
  }

  if (container.nodeType === DOCUMENT_NODE) {
    return container.documentElement;
  } else { // 检查container内是否已经返回了第一个child
    return container.firstChild;
  }
}

// container是ReactDOM.render(<App/>, container)传入的第二个参数
function shouldHydrateDueToLegacyHeuristic(container) {
  var rootElement = getReactRootElementInContainer(container);
  return !!(rootElement && rootElement.nodeType === ELEMENT_NODE && rootElement.hasAttribute(ROOT_ATTRIBUTE_NAME));
}
```
从根节点开始DFS遍历，dom节点从ReactDOM.render的container元素开始，检查fiber节点是否可以复用nextInstance，nextInstance对应于DOM节点，可以复用的话，会将fiber.stateNode赋值为nextInstance。另外注水阶段，会绑定事件。

# 流式渲染
## 客户端识别
### Transfer-Encoding
Transfer-Encoding 消息首部指明了将 entity 安全传递给用户所采用的编码形式。
``` js
Transfer-Encoding: chunked
```
数据以一系列分块的形式进行发送。 Content-Length 首部在这种情况下不被发送。。在每一个分块的开头需要添加当前分块的长度，以十六进制的形式表示，后面紧跟着 '\r\n' ，之后是分块本身，后面也是'\r\n' 。终止块是一个常规的分块，不同之处在于其长度为0。终止块后面是一个挂载（trailer），由一系列（或者为空）的实体消息首部构成。

### ngnix配置
开启流式渲染，首先在response的头部需要加一个X-Accel-Buffering字段，告诉浏览器我要采用流式传输；然后就可以分块输出了。
首先nginx.conf需要加如下代码，让X-Accel-Buffering透传
location / { proxy_pass_header X-Accel-Buffering; proxy_pass http://node;}
然后在返回的时候，在response的头部设置：
``` js
res.set({
   'X-Accel-Buffering': 'no',
   'Content-Type': 'text/html; charset=UTF-8'
});
```
`x-accel-buffering: no`设置此连接的代理缓存，将此设置为no将允许适用于Comet和HTTP流式应用程序的无缓冲响应。将此设置为yes将允许响应被缓存。默认yes。设置为no可以关闭缓存，让浏览器以chunk的形式解析。

## 服务端返回
### [response.write](http://nodejs.cn/api/http.html#http_request_flushheaders)
``` js
response.write(chunk[, encoding][, callback])#
```
- chunk <string> | <Buffer>
- encoding <string> 默认值: 'utf8'。
- callback <Function>
- 返回: <boolean>
如果调用此方法并且尚未调用 response.writeHead()，则将切换到隐式响应头模式并刷新隐式响应头。
这会发送一块响应主体。**可以多次调用该方法以提供连续的响应主体片段。**
第一次调用 response.write() 时，它会将缓冲的响应头信息和主体的第一个数据块发送给客户端。 第二次调用response.write() 时，Node.js 假定数据将被流式传输，并分别发送新数据。 也就是说，响应被缓冲到主体的第一个数据块。

``` js
response.flushHeaders()
```
出于效率原因，Node.js 通常会缓冲请求头，直到调用 request.end() 或写入第一个请求数据块。 然后，它尝试将请求头和数据打包到单个 TCP 数据包中。这通常是期望的（它节省了 TCP 往返），但是可能很晚才发送第一个数据。request.flushHeaders() 绕过优化并启动请求。

``` js
response.end()
```
此方法向服务器发出信号，表明已发送所有响应头和主体，该服务器应该视为此消息已完成。 必须在每个响应上调用此 response.end() 方法。

### response.render
app.render负责生成视图，但是没有能力返回给客户端，需要借助res.send|res.write。
res.send和res.write的区别：res.send can only be called once, since it is equivalent to res.write + res.end()
伪代码：
``` js
res.render = function(view, locals, cb){
    app.render(view, locals, function(err, html){
        if(typeof cb !== 'undefined'){
            return cb(err, html);
        }
        res.send(html);
    });
};
```
