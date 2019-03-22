---
title: reading-record
date: 2019-03-18 10:49:49
tags:
---

## [你的Tree-Shaking并没什么卵用](https://juejin.im/post/5a5652d8f265da3e497ff3de#comment)
> Tree-Shaking的原理
> - ES6的模块引入是静态分析的，故而可以在编译时正确判断到底加载了什么代码。
> - 分析程序流，判断哪些变量未被使用、引用，进而删除此代码

但现实比较骨感，es6的代码经过babel和webpack编译打包后，产生的副作用(可能会改变外部变量)导致多余的代码并未被删掉。
babel6在编译的时候，会调用_createClass方法，使用Object.defineProperty去定义类里面的方法，原因是因为在es6的特性中，类里面声明的方法是不可枚举的。所以设置`{ "loose": false }`宽松模式，让babel编译的时候不必去严格遵循es6的特性。
此外，UglifyJS不会进行程序流分析，所以在压缩的时候无法排除掉可能有副作用的代码，所以这部分代码还是会被打包进去。
使用babel6/webpack打包可以考虑结合使用`BabelMinifyWebpackPlugin`，思路是先进行uglifyJS代码压缩，再去编译。
> 评论中指出：关于Person和Apple阐述Babel副作用的例子在Babel升级到Babel7之后确实已经不存在了，使用Babel7的正式版和Webpack4亲测。

## [http2.0 相比 1.0有哪些重大改进？](https://www.zhihu.com/question/34074946)
1. 多路复用
多路复用允许同时通过单一的 HTTP/2 连接发起多重的请求-响应消息。
在http1.0和1.1协议下，浏览器在同一时间针对同一域名下的请求有一定数量的限制。不同浏览器限制的数量不一样。

2. 二进制分帧
http2.0在应用层和传输层之间增加了一个二进制分帧层，在该层中，会将传输信息分割成更小的消息和帧。将http1.x的头部信息封装在到了HEADER frame，相应的Request Body封装到DATA frame中。
http2.0让所有数据流共用一个连接，有效地使用TCP连接，减少服务端的链接压力，内存占用更少，实现高带宽。
此外，减少了TCP慢启动的时间。

3. 首部压缩
支持首部压缩，使用了HPACK算法。

4. 服务端推送
服务器可以向客户端推送可能需要的资源，对一个请求发送多个响应。
> 「如果客户端早已在缓存中有了一份 copy 怎么办？」
> 一个推荐的解决方案是，客户端使用一个简洁的 Cache Digest 来告诉服务器，哪些已经在缓存中存在。

5. 设置资源的优先级
浏览器可以一次请求多个资源，指定一些优先级信息来帮助确定应该如何处理这些资源，然后等待服务器发回所有数据，而不是一次请求一个。如果浏览器和服务器都支持优先级，则应使用浏览器指定的规则并使用所有可用带宽来传递资源，而不会有资源之间的相互竞争。

支持http2.0的前提是使用了SSL/TLS9(安全传输层协议)，如果网站没有使用SSL/TLS，接入http2.0协议带来的性能提升大致可以被TLS带来的性能损耗所抵消。
https://www.mnot.net/talks/h2fe/
https://www.w3ctech.com/topic/1563#tip7sharding
https://juejin.im/post/5c1d9b8ae51d4559746922de

## xss攻击
例如：http://xxx/search?keyword="><script>alert('XSS');</script> 
``` html
<input type="text" value="<%= getParameter("keyword") %>">

// 参数拼接后
<input type="text" value=""><script>alert('XSS');</script>">
```
跨站脚本攻击（Cross-site scripting），分为存储型(存储到数据库中)、反射型(网站服务端将恶意代码从URL中取出，拼接在HTML中返回给浏览器)和Dom型(前端浏览器拼接并执行)，需要前端处理的是Dom型。
预防：
- 合适的HTML转义，完善的转义库需要针对上下文制定多种规则，例如 HTML 属性、HTML 文字内容、HTML 注释、跳转链接、内联 JavaScript 字符串、内联 CSS 样式表等等
- 利用模板引擎 开启模板引擎自带的 HTML 转义功能
- 避免内联事件 尽量不要使用 onLoad="onload('{{data}}')"
- 避免拼接 HTML
- 时刻保持警惕 在插入位置为 DOM 属性、链接等位置时
- 增加攻击难度，降低攻击后果 通过 CSP、输入长度配置、接口安全措施等方法，增加攻击的难度，降低攻击的后果
- 主动检测和发现 可使用 XSS 攻击字符串和自动扫描工具寻找潜在的 XSS 漏洞
 
可以手动拼接字符串`jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e`去检查

## position:fixed
position:fixed 的元素将相对于屏幕视口（viewport）的位置来指定其位置。并且元素的位置在屏幕滚动时不会改变。
**但是当元素祖先的 transform 属性非 none 时，容器由视口改为该祖先。**
设置为position:fixed的元素，如果存在transform为非none的元素祖先时，会相对该元素去进行定位。因为transform设置为非none时，会创建一个堆叠上下文和包含块，会影响子元素的固定定位。
不是所有的创建新的堆叠上下文都被影响fixed定位，在最新的Chrome浏览器下，只有下面3种还会影响：
- 设置了 transform-style: preserve-3d 的元素
- perspective 值不为 none 的元素
- 在 will-change 中指定了任意 CSS 属性
但不同浏览器表现不同，所以要具体问题具体分析。


## vm wh
显示器宽度: screen.width
浏览器宽度: window.outerWidth
网页宽度: window.innerWidth

wm wh是相对与网页的宽高的，网页的宽为100vm，网页的高度为100vh