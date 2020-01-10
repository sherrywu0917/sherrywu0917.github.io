---
title: animejs动画入门
date: 2019-12-27 11:26:12
tags:
---
### 概述
anime.js是一个灵活的轻量JavaScript动画库，支持CSS属性、JS对象、DOM属性和SVG。压缩后仅6.2K，且不依赖任何第三方库，加载迅速。在[git](https://github.com/juliangarnier/anime)上Usedby数量8.5k, Star数量为33.5k。此外，anime遵从MIT开源协议,可应用于各种商业网站或app而无需付费。

### 简单的浮动动画
首先看一个简单的示例，实现了气泡的上下浮动动画，效果详见[浮动动画](https://codepen.io/sherrywu0917/pen/yLyoNQj)。
``` js
const floatAnimation = anime({
    targets: '.bubble',
    translateY: 20, // --> '20px'
    duration: 600,
    loop: true,
    direction: 'alternate',
    easing: 'easeInOutSine'
});
```
其中各个属性代表的含义如下：
- `targets`代表动画的目标对象，这边的值是css选择器，也可以是dom元素、数组，或是JS对象；
- `translateY`代表将要发生动画的属性，不带单位的时候，默认单位为px，此外还支持其他多种赋值方式：
    1. 百分比：`translateY: '100%'`
    2. 相对数值：`translateY: { value: '*=2.5',duration: 1000}`，相对于translateY的初始值去计算
    3. 设置初始值：`translateY: [100, 250]`，设置数组的形式，会将translateY的初始值设置为100px，并且从100px移动到250px
    4. 函数：`translateY: function(el, i, l) {return (l - i) * 50;}`
- `duration`代表动画的持续时间，单位为毫秒；
- `loop`定义了动画是否需要循环，默认值为false；
- `direction`的意义和animation-direction属性一致，定义是否应该轮流反向播放动画，可选的值有正向动画`normal`、反向动画`reverse`和往返动画`alternate`；
- `easing`定义了动画缓动函数，和css3中的animation-timing-function意义一致，设置该参数时可以参考该网站[Easing Functions Cheat Sheet](https://easings.net/en)去选择合适的缓动函数。


### 点击触发抖动动画，结束后恢复浮动
基于上面的漂浮动画，新增一个小功能：点击小球的时候，小球左右抖动，抖动结束后，恢复浮动动画。具体效果参见[抖动动画](https://codepen.io/sherrywu0917/pen/yLyoNQj)。
``` js
const shakeAnimation = anime({
    targets: '.bubble',
    keyframes: [
        { translateX: '25px' },
        { translateX: 0 },
        { translateX: '-25px' },
        { translateX: 0 },
        { translateX: '15px' },
        { translateX: 0 },
        { translateX: '-15px' },
        { translateX: 0 }
    ],
    duration: 600,
    delay: 300,
    endDelay: 500,
    easing: 'linear',
    begin: () => {
        floatAnimation.pause();
    },
    complete: () => {
        floatAnimation.play();
    }
});
```
这组动画的实现有几个关键点，**关键帧**-`keyframes`、**延迟设置**-`delay`、`endDelay`，**事件函数**`begin`、`complete`以及**动画控制方法**`play`和`pause`。
其中，`keyframes`是用数组定义的关键帧，和animation中用@keyframes实现动画的方式基本一致，如果关键帧内没有指定duration，则每个关键帧的持续时间将等于动画总持续时间除以关键帧数。
`delay`和`endDelay`用来设置动画的延迟时间，完整的动画过程是：
1. 先触发begin事件，暂停floatAnimation；
2. 过了delay设置的300ms后，开始执行keyframes定义的抖动动画，持续时间是600ms；
3. 抖动动画结束后500ms，会触发complete事件，floatAnimation会恢复执行。
从start事件被触发到complete事件被触发，整个动画持续的时间为300 + 600 + 500 = 1400ms。
`play`和`pause`方法用来控制动画的执行和暂停，animejs还提供了`restart`和`reverse`方法去控制动画的重新开始和反转，如果期望动画跳转到某个特定时间，可以使用`seek`方法去设置跳转时间。

### 兼容性
animejs从3.x版本开始不兼容webkitTransform属性，默认只支持transform属性，在一些低版本的手机上使用，例如ios8手机上，会遇到一些动画不生效的问题。如果需要兼容这些机型，可以给animejs库增加对webkitTrannsform属性的支持，手动修改源码，重新publish一个支持webkitTransform属性的npm包。
首先使用了animejs源码提供的`getCSSValue`方法去检测浏览器是否支持transform属性，对于不支持的，使用webkitTransform属性去替换，具体的修改可以参考下图。
<img src="/image/webkit-anime.png" width="800px">
`getCSSValue`方法的实现如下：
``` js
function getCSSValue(el, prop, unit) {
  if (prop in el.style) {
    var uppercasePropName = prop.replace(/([a-z])([A-Z])/g, '$1-$2').toLowerCase();
    var value = el.style[prop] || getComputedStyle(el).getPropertyValue(uppercasePropName) || '0';
    return unit ? convertPxToUnit(el, value, unit) : value;
  }
}
```
关键是`prop in el.style`的判断，对于不支持transform属性的值，会直接返回undefined，支持该属性的会返回属性值，没有设置过transform值的默认会返回一个字符串'none'或者'0'。
修改后重新发布了[webkit-animejs](https://www.npmjs.com/package/webkit-animejs)，有需要的可以直接使用。
