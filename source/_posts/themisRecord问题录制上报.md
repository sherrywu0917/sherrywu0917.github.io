---
title: themisRecord问题录制上报
date: 2021-07-02 12:12:12
tags: [react, diff]
---

## 背景
为了对用户在使用中遇到的相关问题及时给予反馈，尽快定位并解决用户遇到的使用问题。我们设计实现了问题上报工具，主要包括录制和展示两部分：
- ThemisRecord插件: 上报用户 uid、用户权限、API 请求&结果、错误堆栈、录屏
- 展示平台：显示录屏回放、用户、请求和错误堆栈信息

本文主要介绍ThemisRecord问题一键上报插件的实现原理。如何接入和使用详见[ThemisRecord接入使用文档](https://music-cms.hz.netease.com/themis/plugins/record)。
![image](https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/9768133317/b5d2/0140/e564/c0d698968e2e158486c1cf207733a129.png)

## 设计方案
问题一键上报插件的主要流程如下图所示，在录屏期间，插件需要分别收集用户账号信息、API请求数据、错误堆栈信息和录屏信息，并将数据上传到NOS和倾听平台。
![image](https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/9761905867/46b6/f1cf/8cb9/3e3ee413e0530ad767093af874e3f7ae.png)

<!-- more -->

### 用户账号信息
用户账号信息我们都可以通过相关接口拿到，因为问题上报同时支持C端和运营后台CMS，两者属于不同的用户体系，所以在实现的时候会区分C端和CMS用户。

### API请求数据
Web的http请求的底层最后都会通过fetch、xhr的方式去发送请求，所以如果想要获取API的requet和response，可以从底层拦截fetch、xhr请求去实现。

### 错误堆栈信息
错误堆栈信息可以帮助我们直观地定位到当前js出错的地方，为了实现对js运行错误的监听，首先明确js错误主要包括:
1.同步错误类型`error`
2.promise被reject之后触发的`unhandledrejection`
可以通过`window.addEventListener`去监听：
``` js
    window.addEventListener('error', errorEvent => {});
    window.addEventListener('unhandledrejection', promiseRejectionEvent => {});
```
需要注意的是errorEvent和promiseRejectionEvent两个事件对象的结构并不一致，需要针对两种错误类型进行不同的format处理。

### 录屏信息
录屏回放基于[rrweb](https://github.com/rrweb-io/rrweb)开源库实现。`rrweb`提供了`record`和`replay`两个方法：
1. **record**主要监听两种事件类型：
   - DOM变化：内部基于`MutationObserver`API来监听页面DOM变化，当DOM发生变化时，会触发回调方法，将当前DOM变化通过event对象传递给事件监听者；
   - 鼠标移动，鼠标交互，页面滚动等：通过addEventListener等事件绑定的形式去监听
```js
const stopFn = rrweb.record({
  emit(event) {
    // 保存获取到的 event 数据
  }
})
```
2. **replay**负责解析拿到的事件，通过将event数组还原为DOM元素，进而还原页面的用户操作
```js
const events = GET_YOUR_EVENTS

const replayer = new rrweb.Replayer(events);
replayer.play();
```
### 数据上传
录屏期间产生的所有数据会存储到`json文件`中，结束录制后会将`json文件`上传到nos获得`nosKey`，最后将`nosKey`与问题描述等信息同步到倾听平台。倾听平台会根据`noskey`下载相关json文件，并将录屏数据可视化展示。
