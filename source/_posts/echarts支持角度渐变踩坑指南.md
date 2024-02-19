---
title: echarts支持角度渐变踩坑指南
date: 2023-10-12 19:40:59
tags: [echarts, 角度渐变]
---

## 背景

背景是在听歌数据主页的项目中，需要用一个圆环去表达听歌时段，圆环是基于echarts库实现的，最初的方案是使用线性渐变的方式去渲染颜色；后期UI侧对于线性渐变的效果不太满意，希望: `能够用颜色的递进来表达用户当前时段的投入程度，但目前最可行的线性渐变解决方案的效果还是不太能达到要求的，是否有更合适的方案能够满足角度渐变的效果呢？`

<img src="https://p6.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/30822455382/f470/cac6/258e/c06fd86566025bcf1354aaf44fb97b0c.png" width="360px">

问题的难点在于echarts目前只支持下面两种渐变方式：
- 线性渐变：LinearGradient
- 径向渐变：RadialGradient

对于是否可以支持角度渐变，首先需要调研下可行性，大致思路是调研下是否可以基于echarts的源码去扩展，以期支持角度渐变的效果。

### 可行性调研
首先把echarts源码下载到本地，发现主要分为两个包：echarts和zrender，渲染的逻辑主要集中在zrender，官网介绍：`ZRender, a high performance 2d drawing library.`。
为了快速定位到渐变渲染的位置，通过canvas的`createLinearGradient`API去全局搜索(就是这么简单粗暴)，最后发现构建渐变对象的核心逻辑主要在zrender的`canvas/helper.ts`文件中，详情见[git源码](https://github.com/ecomfe/zrender/blob/master/src/canvas/helper.ts)。
```ts
export function getCanvasGradient(this: void, ctx: CanvasRenderingContext2D, obj: GradientObject, rect: RectLike) {
    const canvasGradient = obj.type === 'radial'
        ? createRadialGradient(ctx, obj as RadialGradientObject, rect)
        : createLinearGradient(ctx, obj as LinearGradientObject, rect);

    const colorStops = obj.colorStops;
    for (let i = 0; i < colorStops.length; i++) {
        canvasGradient.addColorStop(
            colorStops[i].offset, colorStops[i].color
        );
    }
    return canvasGradient;
}
```
初步设想，在getCanvasGradient中根据obj.type去返回对应的渐变对象；为了快速验证可行性，在createLinearGradient方法中直接返回使用`ctx.createConicGradient(-0.5 * Math.PI, 187, 150)`创建的对象，colorStops也是固定值，事实验证是可行的。

### 角度渐变实现
为了实现业务代码可用的角度渐变，需要在echarts.graphic的基础上去扩展，支持`new echarts.graphic.ConicGradient(startAngle, colorStops)`的调用方式；主要的改造包括：
1. 在`zrender/graphic`文件夹下，新增`ConicGradient`类，继承自`Gradient`类，type标记为`conic`，后续会根据该type值去判断需要返回的渐变对象类型，关键代码:
``` js
import { __extends } from "tslib";
import Gradient from './Gradient.js';
var ConicGradient = (function (_super) {
    __extends(ConicGradient, _super);
    function ConicGradient(angle, colorStops, timeUnitType) {
        var _this = _super.call(this, colorStops) || this;
        _this.angle = angle == null ? 0 : angle;
        _this.type = 'conic';
        _this.timeUnitType = timeUnitType;
        _this.global = false;
        colorStops.forEach(function (colorStop) {
            colorStop.timeUnitType = timeUnitType;
        });
        return _this;
    }
    return ConicGradient;
}(Gradient));
export default ConicGradient;
```

2. 在echarts、zrender两个库中，全局查找和补充：
- 在需要export渐变类的地方，补充对`ConicGradient`类的导出；
- 对于源码中需要判断渐变类型的地方，补充对`isConicGradient`方法的调用和相关渲染逻辑

3. 最后在关键的`getCanvasGradient`方法中，创建conic渐变对象并返回，注意开始角度需要设置为-0.5 * Math.PI，因为角度渐变的起始点为水平3点钟方向，而视觉要求的起始点为垂直12点方向：

``` js
export function getCanvasGradient(ctx, obj, rect) {
    var canvasGradient;
    switch (obj.type) { 
        case 'radial':
            canvasGradient = createRadialGradient(ctx, obj, rect);
            break;
        case 'conic':
            canvasGradient = createConicGradient(ctx, obj, rect);
            break;
        default:
            canvasGradient = createLinearGradient(ctx, obj, rect);
            break;
    }
    var colorStops = obj.colorStops;
    for (var i = 0; i < colorStops.length; i++) {
        canvasGradient.addColorStop(colorStops[i].offset, colorStops[i].color);
    }
    return canvasGradient;
}

export function createConicGradient(ctx, obj, rect) {
    // ...
    // 开始角度为-0.5 * Math.PI
    return ctx.createConicGradient(-0.5 * Math.PI, x, y);
}
```
最后，在业务代码中通过itemStyle.color的配置，返回`new echarts.graphic.ConicGradient`创建的角度渐变对象。
```js
itemStyle: { // 每个扇形的样式
    color: (params) => {
        // 计算出每个扇形的开始角度
        const startAngle = xxx;
        // 获取该扇形对应的渐变值
        const colorStops = yyy;
        return new echarts.graphic.ConicGradient(startAngle, colorStops);
    },
},
```
至此，在电脑端chrome浏览器上效果非常完美，满足视觉同学的预期效果（此时，我有一个小疑问，echarts为什么不支持角度渐变？）。


### 起始角度问题
众所周知，前端在chrome上实现的效果通常很完美，但在手机端上可能是另一回事。

#### 起始角度不同
chrome上的效果：
<img src="https://p6.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/30873370228/b6c5/9963/ba4d/e7415b0bbaa82a163d000f04a40efb99.png" width="300px">

iphone上的效果：
<img src="https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/30873419595/2808/be17/8354/fb03811d82d5c83753b2eff9cc459b40.png" width="300px">

对比上面两图，可以发现在iphone上，渐变绘制的是有角度渐变的，但渐变的色值和起始位置都不对。
<img src="https://p6.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/30873448490/4ea7/a9a2/b035/a4de2e20767f73998af4c0c259f3fd4c.png" width="300px">

通过绘制单条数据可以发现，在iphone上`-0.5 * Math.PI`对应的起始点并非垂直12点方向，而是水平9点方向，相比理想位置多了`-0.5 * Math.PI`角度，说明在该手机上起始角度0度对应的就是垂直12点方向。于是，在创建渐变对象时增加了是否是移动端设备的判断：
```js
export function createConicGradient(ctx, obj, rect) {
    // ...
    const startAngle = isMobile() ? 0 : -0.5 * Math.PI;
    return ctx.createConicGradient(startAngle, x, y);
}
```
在增加上述判断后，发现在另一台iphone上绘制的效果也并不符合预期，经测试，在不同的iphone设备上起始角度不同，有的是垂直12点方向，有的是水平9点方向，应该是和ios系统有关系，但难以覆盖完全。

#### 判断起始角度
此时，需要有一个通用的方法去判断角度渐变的起始角度，在遍查API之后，发现并没有这样的方法。最后，想到可以通过绘制1/4的渐变，再去读取对应位置的颜色，进而计算出渐变的起始角度。具体的流程如下：
```js
function countDirection() {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    const x = 100;
    const y = 100;
    const radius = 50;

    if (ctx && ctx.createConicGradient) {
        // 从默认的起始角度0度开始绘制1/4圆的渐变
        const conicGradient = ctx.createConicGradient(0, x, y);
        conicGradient.addColorStop(0, '#000');
        conicGradient.addColorStop(0.25, '#fff');
        
        ctx.fillStyle = conicGradient;
        ctx.beginPath();
        ctx.arc(x, y, radius, 0, 2 * Math.PI);
        ctx.fill();

        // 获取右上角的颜色，若从水平方向开始则为[255, 255, 255, 255]，反之为垂直方向[135, 135, 135, 255]
        const imageData = ctx.getImageData(110, 90, 1, 1);
        const pixelData = imageData.data || [];
        isHorizontal = pixelData?.[0] > 240; // 存在可能为254的情况，保险起见改成大于240
    }
}

const isHorizontal = countDirection();
```
通过`ctx.getImageData`去读取右上角某一个点的位置，如果起始位置是水平方向，则取到的色值应该为白色: [255, 255, 255, 255]；反之如果起始位置是垂直方向，则右上角为渐变区域：#000 -> #fff 的渐变，色值必然不是白色。通过这种间接的方式，可以计算出角度渐变的起始方向。
- 水平方向：

<img src="https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/30875695990/bae8/5c24/01a4/fedea8507b1540349173395fed864258.png" width="180px">

- 垂直方向：

<img src="https://p6.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/30875716128/4c47/b840/3ec7/55bc755db421e8bfd6d07ab94e785d22.png" width="180px">

### 安卓Polyfill适配
成功处理了角度问题之后，在安卓机上打开，发现页面白屏，看console日志，发现有报错`ctx.createConicGradient is not defined`，此时，我大概明白了`echarts为什么不支持角度渐变`= =！。
尝试查找可用的polyfill，发现[create-conical-gradient](https://github.com/parksben/create-conical-gradient)这个库大概可以满足需求。在createConicGradient方法中增加对`ctx.createConicGradient`是否存在的判断：
```js
export function createConicGradient(ctx, obj, rect) {
    //... 
    if (ctx.createConicGradient){
        const startAngle = isHorizontal ? -0.5 * Math.PI : 0;
        canvasGradient = ctx.createConicGradient(startAngle, x, y);
    } else {
        canvasGradient = ctx.createConicalGradient(x, y, -0.5 * Math.PI, 1.5 * Math.PI);
        // 标记使用的是polyfill方式，底层对应createPattern
        canvasGradient.usePattern = true;
    }
    return canvasGradient;
}
```
该库内部使用的是`createPattern`方式去渲染，最后暴露了`get pattern()`方式去获取，在zrender需要渲染渐变的地方，需要根据`usePattern`变量去判断，若为true，则设置fillStyle为`canvasGradient.pattern`。

### Polyfill卡顿问题处理
通过增加了上述Polyfill方法在安卓机上可以渲染出角度渐变的效果，但该方案实现后动画效果有问题，卡顿非常严重。

#### 卡顿原因定位
1. 发现zrender中帧动画掉帧严重：requestAnimationFrame回调函数执行时间过长，导致动画卡顿；使用chrome的帧工具，辅助验证了的确是掉帧：
<img src="https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/30875303347/d99b/0cbd/180a/2311d67d5cf3cc03e1e14b61b4c3c757.png" width="300px">

2. 进一步定位，发现同一个渐变被调用并创建了多次，例如起始角度为0的第一个扇形，会被重复渲染10次
<img src="https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/30875252434/7fdc/9d93/30fa/82150b00d07fc3b68c6d05a5b2b82222.png" width="500px">

#### 添加缓存

尝试使用缓存解决，先给ConicGradient创建缓存对象，使用colorStops作为cacheKey，保证唯一性。
```js
const conicGradientCache = {}; // 缓存渐变对象，避免重复创建
export function createConicGradient(ctx, obj, rect) {
    const cacheKey = JSON.stringify(obj.colorStops);
    if (conicGradientCache[cacheKey]) { 
        conicGradientCache[cacheKey].fromCache = true;
        return conicGradientCache[cacheKey];
    }

    //... 
    // 创建canvasGradient对象代码，参考上面
    conicGradientCache[cacheKey] = canvasGradient;
    return canvasGradient;
}
```

<img src="https://p6.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/30875342816/2f65/ead4/3507/b06f612e4e52e895700c3a0f98fcd35a.png" width="300px">

有点效果，但收效甚微，经过定位，发现问题出在[create-conical-gradient](https://github.com/parksben/create-conical-gradient)库中；虽然canvasGradient对象是缓存了，但在使用canvasGradient.pattern时，`get pattern`方法内部每次都会重新创建一遍，所以canvasGradient.pattern也需要缓存。
于是把该库源码pull到本地，在返回pattern的地方也加上缓存：
```js
get pattern() {
    patternCache[cacheKey] = patternCache[cacheKey] || createConicalGradient(ctx2d, this.stops, ...args);
    return patternCache[cacheKey];
},
```
加上缓存后的效果，虽然比不上原生API的渲染性能，但在体感上已基本流畅。

<img src="https://p6.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/30875407445/cba2/0f0f/e5d7/96b1baf09fe491805f7573ca853c295c.png" width="300px">

### 一条线问题
此外，还发现在ios16.0-16.2版本的系统上，在渲染渐变时，会偶现有闪烁的一条线问题，如下图所示：
<img src="https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/30875484909/d6aa/a08d/1c30/a4212d1d2a5724c7bf0b1f92829952e0.png" width= "300px">

在stackoverflow上同样有人有类似的问题：[Flickering horizontal lines on the animation made with canvas only on iOS 16](https://stackoverflow.com/questions/73733470/flickering-horizontal-lines-on-the-animation-made-with-canvas-only-on-ios-16)。

目前没有找到可行的方案去解决：使用原生API的同时可以避免这样线的问题，可选的方案就是判断系统的版本，若处于有问题的版本中，则使用Polyfill的方式去渲染。

## 总结
目前浏览器对canvas的`createConicGradient`API的支持存在诸多问题，或许在未来，浏览器的支持度会越来越高，使用起来就不会像现在这样存在诸多问题。在视觉方案的选择上，也需要再慎重些，看看是否有其他可替代的方案。
