---
title: 菜鸟优化之路-图片webp&lazyload
date: 2018-09-27 20:17:29
tags: [前端性能优化, webp]
---
## webp图片优化
WebP是由google开发的一种图片格式，优势体现在它具有更优的图像数据压缩算法，能带来更小的图片体积，而且拥有肉眼识别无差异的图像质量；同时具备了无损和有损的压缩模式、Alpha 透明以及动画的特性，在 JPEG 和 PNG 上的转化效果都相当优秀、稳定和统一。
[webp兼容性](https://caniuse.com/#search=webp)见下图
<img src="/image/webp_caniuse.png" width="800px">
其中chrome和opera支持得比较好，safari和firefox目前不支持，但已经在实验阶段。

## 检测浏览器是否支持webp格式
### 方法1: canvas的toDataURL
```js
function checkWebp() {
    try{
        return (document.createElement('canvas').toDataURL('image/webp').indexOf('data:image/webp') == 0);
    }catch(err) {
        return  false;
    }
    //or
    !![].map && document.createElement('canvas').toDataURL('image/webp').indexOf('data:image/webp') == 0;
}
```
对比一下chrome和ie下`document.createElement('canvas').toDataURL('image/webp')`下的输出：
```js
//ie
"data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAASwAAACWCAYAAABkW7XSAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAADGSURBVHhe7cExAQAAAMKg9U9tCF8gAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAONUAv9QAAcDhjokAAAAASUVORK5CYII="
//chrome
"data:image/webp;base64,UklGRrgAAABXRUJQVlA4WAoAAAAQAAAAKwEAlQAAQUxQSBIAAAABBxARERCQJP7/H0X0P+1/QwBWUDgggAAAAHANAJ0BKiwBlgA+bTaZSaQjIqEgKACADYlpbuF2sRtACewD32ych77ZOQ99snIe+2TkPfbJyHvtk5D32ych77ZOQ99snIe+2TkPfbJyHvtk5D32ych77ZOQ99snIe+2TkPfbJyHvtk5D32ych77ZOQ99qwAAP7/1gAAAAAAAAAA"
```
<!-- more -->
基于`toDataURL`的特性，如果请求的类型不被支持，默认返回`data:image/png`。（PS:canvas的默认宽高是300px,150px，如果canvas的宽或者高为0，则`toDataURL`会返回`"data:,"`。)
所以，只有支持webp格式的浏览器调用`toDataURL('image/webp')`后返回的字符串中才包含`'data:image/webp'`。上面提供了两种写法，其中`!![].map`主要是判断是否是IE9+，以免toDataURL方法会报错。

### 方法2: 图片onload
google官网提供的，通过加载小的webp图片来判断是否支持该格式：
```js
// check_webp_feature:
//   'feature' can be one of 'lossy', 'lossless', 'alpha' or 'animation'.
//   'callback(feature, result)' will be passed back the detection result (in an asynchronous way!)
function check_webp_feature(feature, callback) {
    var kTestImages = {
        lossy: "UklGRiIAAABXRUJQVlA4IBYAAAAwAQCdASoBAAEADsD+JaQAA3AAAAAA",
        lossless: "UklGRhoAAABXRUJQVlA4TA0AAAAvAAAAEAcQERGIiP4HAA==",
        alpha: "UklGRkoAAABXRUJQVlA4WAoAAAAQAAAAAAAAAAAAQUxQSAwAAAARBxAR/Q9ERP8DAABWUDggGAAAABQBAJ0BKgEAAQAAAP4AAA3AAP7mtQAAAA==",
        animation: "UklGRlIAAABXRUJQVlA4WAoAAAASAAAAAAAAAAAAQU5JTQYAAAD/////AABBTk1GJgAAAAAAAAAAAAAAAAAAAGQAAABWUDhMDQAAAC8AAAAQBxAREYiI/gcA"
    };
    var img = new Image();
    img.onload = function () {
        var result = (img.width > 0) && (img.height > 0);
        callback(feature, result);
    };
    img.onerror = function () {
        callback(feature, false);
    };
    img.src = "data:image/webp;base64," + kTestImages[feature];
}
```
上面提供了几种webp的图片模式，如果浏览器支持webp，那么图片的宽高会大于0，从而返回true,否则返回false。
调用方法如下，可以在`callback`方法中得到检测结果，进而去进一步处理：选择动态加载webp格式的图片，或者给全局添加`webp`相关的class都可以。
```js
check_webp_feature('lossless',function(feature,result){
    alert(result); //true or false
});

```

## 使用webp
### 处理webp通常有两种方式
- 服务端处理，支持webp图片的浏览器会在请求头Accept中加上`image/webp`，服务器根据头信息返回webp或者其他格式的图片；但图片放在CDN服务器上，处理起来就很麻烦了；
- 前端先完成对是否支持webp格式的检测，再根据检测结果去选择请求的资源类型
我们产品使用的是网易云提供的服务，通过修改参数就可以去获取不同格式的图片，具体可以参考https://www.163yun.com/help/documents/114078550521466880。
其中，通过在url后拼接`?imageView&type=webp`就可以获得对应的webp图片。

### 在线生成
- 智图 [zhitu.isux.us](https://zhitu.isux.us/)
- 又拍云 [www.upyun.com/webp](https://www.upyun.com/webp)
- CloudConvert [cloudconvert.com/anything-to-webp](https://cloudconvert.com/anything-to-webp)
- iSparta [isparta.github.io/index.html](https://isparta.github.io/index.html)

## 图片lazyload
图片lazyload是常见的性能优化的一种方式。如果页面图片数量较多，一次性加载比较耗时，还会导致页面卡顿，所以，建议根据需要去加载部门图片，待页面滚动时再加载下面的图片。
可以使用一个轻量级的[lazyload库](https://github.com/verlok/lazyload)，具体使用可以参考GitHub。
通过`npm`安装`vanilla-lazyload`包，推荐的版本有：
```
npm install vanilla-lazyload@8.17.0
npm install vanilla-lazyload@10.19.0
```
注意：10.x版本使用了`IntersectionObserver API`，IE和safari不支持，所有图片会一次性加载。
简单粘贴一个示例：
``` html
<img class="lazy" alt="..." 
     data-src="../img/44721746JJ_15_a.jpg"
     width="220" height="280">
```
``` js
import LazyLoad from "vanilla-lazyload";
var myLazyLoad = new LazyLoad({
    elements_selector: ".lazy"
});
```
在线demo可以访问：https://www.andreaverlicchi.eu/lazyload/demos/container_single.html
REFS:
https://github.com/verlok/lazyload
https://www.haorooms.com/post/webp_bigpipe
https://www.zhangxinxu.com/php/microCodeDetail?id=3