---
title: webpack常用插件
date: 2023-07-02 12:12:12
tags: [webpack, 插件]
---

## webpack运行机制
- 初始化配置参数 -> 绑定事件钩子回调 -> 确定Entry逐一遍历 -> 使用loader编译文件 -> 输出文件

## html-webpack-plugin 和 script-ext-html-webpack-plugin
- html-webpack-plugin生成的html文件，其中<script>都是默认同步类型的
- 需要使用script-ext-html-webpack-plugin，可以定义<script>的引入类型或者inline script

## 自定义插件
- Compiler暴露了和webpack整个生命周期相关的钩子：负责文件监听和启动编译。Compiler 实例中包含了完整的 webpack 配置，全局只有一个 Compiler 实例。
``` js
//基本写法
compiler.hooks.someHook.tap(...)
//如果希望在entry配置完毕后执行某个功能
compiler.hooks.entryOption.tap(...)
//如果希望在生成的资源输出到output指定目录之前执行某个功能
compiler.hooks.emit.tap(...)
```
- Compilation暴露了与模块和依赖有关的粒度更小的事件钩子
模块会经历加载(loaded),封存(sealed),优化(optimized),分块(chunked),哈希(hashed)和重新创建(restored);compilation是Compiler生命周期中的一个步骤，使用compilation相关钩子的通用写法为:
``` js
compiler.hooks.compilation.tap('SomePlugin',function(compilation, callback){
    compilation.hooks.someOtherHook.tap('SomeOtherPlugin',function(){
        ....
    })
});
```
### HtmlWebpackPlugin提供的事件钩子
- 开始生成HTML之前勾子(HtmlWebpackPlugin BeforeHtmlGeneration)
这一阶做一些资源归类工作，主要产出物为asset资源原始对象，该对象为插入HTML头尾部的JS,CSS资源列表及其路径。

- 在HTML开始处理之前勾子(HtmlWebpackPlugin BeforeHtmlProcessing)
生成不包括JS和CSS的纯HTML结果，产出物为html字符串。

- 添加资源处理HTML勾子(HtmlWebpackPluginAlterAssetTags)
组装要插入HTML页面中的JS，CSS等资源结构，
如果要在生成HTML页面中加入自定义或者WEBPACK不支持的<script>标签等，可操作该对象。

- HTML处理完毕勾子(HtmlWebpackPluginAfterHtmlProcessing)
HTML页处理完成阶段，JS,CSS完成插入，已生成可直接打包用的文本结构，一般要输出自定义内容在此处实现。

- 勾子任务处理完毕发送事件时(HtmlWebpackPluginAfterEmit)
处理任务最后阶段，可通过返回的html属性访问source和size属性。

## webpack-dev-server vs webpack-dev-middleware
`webpack-dev-server`结合了`express`、`webpack-dev-middleware`和`webpack-hot-middleware`的功能，其中
- express负责启动node服务
- webpack-dev-middleware通过webpack提供的`watch`API监听代码变化，重新编译打包，写到内存中
- webpack-hot-middleware客户端使用[EventSource](https://developer.mozilla.org/zh-CN/docs/Server-sent_events/Using_server-sent_events)长轮询(`webpack-dev-server`在服务端和客户端建立websocket长连接)
  - 服务端监听`compiler`钩子的`done`事件，在文件打包完成后，调用监听函数
  - 监听函数中调用`eventStream.publish`，将事件类型、文件hash等打包信息，通过事件流的形式通知客户端。（有趣的事，如果一直没有更新，服务端会间隔(默认)10s发生消息给客户端，是一个心形字符，代表心跳）
  - 客户端监听`onmessage`事件，根据接收到的消息，去进行不同的处理
  - 客户端调用check方法坚持是否有更新，如果配置了模块热更新，HMR runtime根据新模块代码决定是热更新还是重刷页面；如果没配置模块热更新，则直接重刷页面(**HMR 是可选功能，只会影响包含 HMR 代码的模块。**例如`style-loader`实现了HMR的module.hot接口，当它通过 HMR 接收到更新，它会使用新的样式替换旧的样式。)
``` js
    compiler.hooks.done.tap('webpack-hot-middleware', onDone);

    // init方法
    source = new window.EventSource(options.path);
    source.onopen = handleOnline;
    source.onerror = handleDisconnect;
    source.onmessage = handleMessage;
```
### WebSockets or Server-Sent Events
- 明显的区别是：WebSockets是双向通信，Server-Sent Events是单向通信
- Server-Sent Events API是服务器向客户端发送事件，客户端负责事件监听
