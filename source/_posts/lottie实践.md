---
title: lottie实践
date: 2019-01-28 10:27:41
tags: lottie
---

## lottie简介
> [Lottie](http://airbnb.io/lottie/) is a mobile library for Android and iOS that parses Adobe After Effects animations exported as json with Bodymovin and renders them natively on mobile and on the web!

Lottie是一个库，可以解析使用AE制作的动画（需要用bodymovin导出为json格式），支持web、ios、android和react native。
在web侧，lottie-web库可以解析导出的动画json文件，并将其以svg或者canvas的方式将动画绘制到我们页面中。关于lottie-web的详细信息，也可以参见[https://github.com/airbnb/lottie-web](https://github.com/airbnb/lottie-web)。
<img src="http://imweb-io-1251594266.file.myqcloud.com/FvWDhZoyPK5wVQecu5qFYEDdGuJe"/>

## lottie-web引用方式
### npm包
``` js
import lottie from 'lottie-web'
lottie.loadAnimation({
  container: element, // the dom element that will contain the animation
  renderer: 'svg',
  loop: true,
  autoplay: true,
  path: 'data.json' // the path to the animation json
});
```
### js外链
多个版本的lottie库都可以从[cdnjs-bodymovin](https://cdnjs.com/libraries/bodymovin)找到
``` html
<script src="https://cdnjs.com/libraries/bodymovin" />
<script>
    bodymovin.loadAnimation({
      container: element, // the dom element that will contain the animation
      renderer: 'svg',
      loop: true,
      autoplay: true,
      path: 'data.json' // the path to the animation json
    });
</script>
```
通过外链引入的方式，既可以通过`bodymovin`去调用，也可以使用`lottie`变量，二者等价，都是js库提供的全局变量。

## lottie-web实践
lottie库文件比较大，gzip压缩后也有60k，再加上动画对应的json文件，直接引入会给项目代码增大不少体积，影响页面加载速度。

文件 | 大小 | gzip后
- | :-:| :-:
lottie.js | 513k  |  92k
lottie.min.js | 237k  |  60k
lottie_light.js (lottie_web轻量版，仅支持svg渲染)  |  345k  |  60k
lottie_lignt.min.js | 144k  |  39k

所以，如果动画只需要支持svg渲染，则可以引入light版本的库文件，gzip压缩后缩减到39k。
为了进一步降低影响，可以使用code splitting的方式，以async的方式异步加载js，这样就不会阻塞浏览器解析html、执行js脚本以及展示css布局。
``` js
import(/* webpackChunkName: "lottie-light" */ '../lib/lottie_light.min.js').then(() => {
    window.lottie && window.lottie.loadAnimation({
        container: this.headerDom,
        renderer: 'svg',
        loop: true,
        autoplay: true,
        animationData: require('../data/ribbon.json')
    });
})
```
示例demo可以参见codepen上的[ribbon-lottie](https://codepen.io/sherrywu0917/pen/XPNavE)，测试数据`ribbon.json`可以下载[彩带json数据]( https://easyreadfs.nosdn.127.net/1548645419902/ribbon.json)。

