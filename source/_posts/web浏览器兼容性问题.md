---
title: web浏览器兼容性问题
date: 2017-06-15 20:19:58
tags: [兼容]
---

### IE的问题
- IE中animate作用在body上会失效，使用$('html, body')
```javascript
    $('html, body').animate({
        scrollTop: offsetTop - $('#J_nav').height()
    }, 500)
```
- $('body').scrollTop()在IE中获得的值始终是0，使用$(document).scrollTop()可以获得正确的值
- IE中通过offset获得的top值 = chrome浏览器offset的top值 + 该元素的paddingTop值。要获取元素offset值，需要兼容IE，如下：

```javascript
    \\大于等于Ver版本的IE浏览器
    isGteIE(ver) {
        let b = document.createElement('b');
        b.innerHTML = '<!--[if lte IE ' + ver + ']><i></i><![endif]-->';
        return b.getElementsByTagName('i').length === 1;
    }
    let offsetTop = $(target).offset().top;
    //IE8及以上, 修正offset
    if(self.isGteIE(8)) {
        offsetTop -= parseInt($(target).css('paddingTop'));
    }
```

### 浏览器内核
浏览器 | 内核
- | :-:
IE | Trident
Safari | webkit
Chrome | WebKit的分支—Chromium引擎
Chrome 28.0.1469.0版本之后 | 基于WebKit2—Blink引擎
Firefox | Gecko内核
Opera | Presto渲染引擎，2013年2月之后紧跟chrome引擎

浏览器市场份额
http://www.netmarketshare.com/

### 兼容IE8
IE8不支持本地视频播放，要使用在线的
为了让IE8兼容video标签，使用html5media
``` js
 <!–[if lt IE 9]>
        <script src="http://api.html5media.info/1.1.8/html5media.min.js"></script>
<![endif]–>
```

兼容IE8的图表库：highcharts
http://www.highcharts.com/demo
需要使用1.x的jQuery版本
