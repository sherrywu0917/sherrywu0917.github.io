---
title: reading-record
date: 2018-11-18 10:49:49
tags:
---

## [你的Tree-Shaking并没什么卵用](https://juejin.im/post/5a5652d8f265da3e497ff3de#comment)
### Tree-Shaking作用：消除无用的js代码
- 与传统的DCE(dead code elimination)区别：DCE是消灭不可能执行的代码，而Tree-shaking更关注于消除没有用到的代码
### Tree-Shaking的原理
- ES6模块依赖关系是确定的，和运行时的状态无关，可以进行可靠的静态分析，故而可以在编译时正确判断到底加载了什么代码。
  - es6 module可以进行tree-shaking，require不可以，因为commonJS规范是在运行时才确定依赖关系
- 分析程序流，判断哪些变量未被使用、引用，进而删除此代码
tree-shaking对函数效果较好，但是不能消除无用的类，因为动态语言的特性，判断类的方法是否有被使用，比较困难。

但现实比较骨感，es6的代码经过babel和webpack编译打包后，产生的副作用(可能会改变外部变量)导致多余的代码并未被删掉。
babel6在编译的时候，会调用_createClass方法，使用Object.defineProperty去定义类里面的方法，原因是因为在es6的特性中，类里面声明的方法是不可枚举的。所以设置`{ "loose": false }`宽松模式，让babel编译的时候不必去严格遵循es6的特性。
此外，UglifyJS不会进行程序流分析，所以在压缩的时候无法排除掉可能有副作用的代码，所以这部分代码还是会被打包进去。
使用babel6/webpack打包可以考虑结合使用`BabelMinifyWebpackPlugin`，思路是先进行uglifyJS代码压缩，再去编译。
> 评论中指出：关于Person和Apple阐述Babel副作用的例子在Babel升级到Babel7之后确实已经不存在了，使用Babel7的正式版和Webpack4亲测。

## [http2.0 相比 1.0有哪些重大改进？](https://www.zhihu.com/question/34074946)
1. 多路复用
多路复用允许同时通过单一的 HTTP/2 连接发起多重的请求-响应消息。可以很容易的去**实现多流并行而不用依赖建立多个 TCP 连接**，HTTP/2 把 HTTP 协议通信的基本单位缩小为一个一个的帧，这些帧对应着逻辑流中的消息。
好处：
- 并行地在同一个 TCP 连接上。http2.0让所有数据流共用一个连接，有效地使用TCP连接，减少服务端的链接压力，内存占用更少，实现高带宽。
- 减少了TCP慢启动的时间。
- 可以变相的解决浏览器针对同一域名的请求限制阻塞问题。因为浏览器在同一时间针对同一域名下的请求有一定数量的限制，不同浏览器限制的数量不一样。

2. 二进制分帧
http2.0在应用层和传输层之间增加了一个二进制分帧层，在该层中，会将传输信息分割成更小的消息和帧。将http1.x的头部信息封装在到了HEADER frame，相应的Request Body封装到DATA frame中。
好处：识别这3部分就要做协议解析，http1.x的解析是基于文本。基于文本协议的格式解析存在天然缺陷，文本的表现形式有多样性，要做到健壮性考虑的场景必然很多，二进制则不同，只认0和1的组合。基于这种考虑http2.0的协议解析决定采用二进制格式，实现方便且健壮。
<!-- more -->

3. 首部压缩
支持首部压缩，使用了HPACK算法，减少了传输的header大小。客户端和服务器维护同一张头信息表，头部字段都会存入该表，之后只需要传对应的索引即可，避免了header的重复传输。

4. 服务端推送
服务器可以向客户端推送可能需要的资源，对一个请求发送多个响应。
> 「如果客户端早已在缓存中有了一份 copy 怎么办？」
> 一个推荐的解决方案是，客户端使用一个简洁的 Cache Digest 来告诉服务器，哪些已经在缓存中存在。

5. 设置资源的优先级
浏览器可以一次请求多个资源，指定一些优先级信息来帮助确定应该如何处理这些资源，然后等待服务器发回所有数据，而不是一次请求一个。如果浏览器和服务器都支持优先级，则应使用浏览器指定的规则并使用所有可用带宽来传递资源，而不会有资源之间的相互竞争。

支持http2.0的前提是使用了SSL/TLS9(安全传输层协议)，如果网站没有使用SSL/TLS，接入http2.0协议带来的性能提升大致可以被TLS带来的性能损耗所抵消。

### http1.x
线程阻塞，在同一时间，同一域名的请求有一定数量限制，超过限制数目的请求会被阻塞.
#### http1.0
每次TCP连接，只能提供一次request和response的响应，结束自动断开
#### http1.1
支持长连接(Response Headers头中出现Connection:keep-alive)，但服务器对请求的响应是串行进行的，只有处理完上一次请求之后，才会去处理下一个请求。

### https
- ssl安全协议：https在http的基础上增加了SSL(Secure Sockets Layer)层，对HTTP协议传输的数据进行加密
- ca证书：https需要申请ca证书
- 端口：http端口80，https端口443
- 所属层：http基于应用层，https一般说是位于应用层和传输层之间
#### https建立连接
1. 服务器先从认证机构申请数字证书
2. 浏览器访问网站，发送支持的加密协议
3. 服务器筛选出合适的加密协议
4. 服务器返回数字证书，证书中有密钥
5. 浏览器利用内置的顶级证书验证CA证书的正确性，解析服务器返回的数字证书得到服务器的公钥。
6. 浏览器生成一个对称加密的密钥，再使用服务器的公钥进行加密后，将加密后的密钥发送给服务器，
7. 服务器使用私钥解密，得到对称加密的密钥，使用该密钥加密数据发送给浏览器端
8. 浏览器端解密数据，SSL开始通信


### websocket
- 全双工通信，可以在浏览器中使用
- 借用了http的协议完成握手，然后再转为WebSocket（基于TCP协议，但本身属于应用层协议）
- 建立长连接 

### OSI七层网络模型
应用层 表示层 会话层 传输层 网络层 数据链路层 物理层

### TCP四层模型 
应用层 传输层（TCP/UDP） 网络层 数据链路层

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
- 增加攻击难度，降低攻击后果 通过 CSP（Content Security Policy，禁止外域等等）、输入长度配置、接口安全措施等方法，增加攻击的难度，降低攻击的后果
- 主动检测和发现 可使用 XSS 攻击字符串和自动扫描工具寻找潜在的 XSS 漏洞
 
可以手动拼接字符串`jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e`去检查

## position:fixed
position:fixed 的元素将相对于屏幕视口（viewport）的位置来指定其位置。并且元素的位置在屏幕滚动时不会改变。
**但是当元素祖先的 transform 属性非 none 时，容器由视口改为该祖先。**
设置为position:fixed的元素，如果存在transform为非none的元素祖先时，会相对该元素去进行定位。因为transform设置为非none时，会创建一个堆叠上下文和包含块，会影响子元素的固定定位。
不是所有的创建新的堆叠上下文都会影响fixed定位，在最新的Chrome浏览器下，只有下面3种还会影响：
- 设置了 transform-style: preserve-3d 的元素
- perspective 值不为 none 的元素
- 在 will-change 中指定了任意 CSS 属性
但不同浏览器表现不同，所以要具体问题具体分析。


## vm wh
显示器宽度: screen.width
浏览器宽度: window.outerWidth
网页宽度: window.innerWidth

wm wh是相对与网页的宽高的，网页的宽为100vm，网页的高度为100vh

## 事件捕获 冒泡
- 对于非target节点则先执行捕获在执行冒泡
- 对于target节点则是先执行先注册的事件，无论冒泡还是捕获
A超链接上的onclick事件会先于href执行，可以联想到通常会在click事件中去阻止冒泡或者默认事件。

## http协议
1xx 消息
2xx 成功

### 3xx 重定向
- 301 Moved Permanently 永久重定向
  - 在请求的 URL 已被移除时使用。响应的 Location 首部中应该包含 资源现在所处的 URL。
  - 默认是缓存的

- 302 Found 临时重定向
  - 与 301 状态码类似；但是，客户端应该使用 Location 首部给出的 URL 来临时定位资源。将来的请求仍应使用老的 URL。
  - 默认不缓存，除非设置了Cache-Control或Expires

- 303 See Other 临时重定向
  - 303 是为了区分302而存在的。
  - 虽然 RFC 1945 和 RFC 2068 规范不允许客户端在重定向时改变请求的方法，但是很多现存的浏览器在收到302响应时，直接使用GET方式访问在Location中规定的URI，而无视原先请求的方法。因此状态码303被添加了进来，用以明确服务器期待客户端进行何种反应。重定向到新地址时，客户端必须使用GET方法请求新地址。

- 307 Temporary Redirect 临时重定向
  - 这个状态码和302相似，有一个唯一的区别是不允许将请求方法从post改为get。
  - 默认不缓存，除非设置了Cache-Control或Expires

- 308 Permanent Redirect 永久重定向
  - 此状态码类似于301（永久移动），但不允许更改从POST到GET的请求方法。
  - 默认是缓存的 

- 304 Not modified

### 4xx 客户端错误
- 400 Bad Request 请求出现语法错误。
- 401 Unauthorized 访问被拒绝，客户试图未经授权访问受密码保护的页面。
- 403 Forbidden 资源不可用。服务器理解客户的请求，但拒绝处理它
- 405 Method not allowed

### 5xx 服务端错误
- 500 Internal server error 服务器遇到了意料不到的情况，不能完成客户的请求
- 502 Bad gateway  服务器作为网关或者代理时，为了完成请求访问下一个服务器，但该服务器返回了非法的应答。
- 503 Service Unavailable 服务不可用，服务器由于维护或者负载过重未能应答。
- 504 Gateway timeout 网关超时，由作为代理或网关的服务器使用，表示不能及时地从远程服务器获得应答。

## 浏览器渲染页面的过程
### 请求加载页面的主要流程
1. 输入URL
2. 确认是否使用本地缓存
    - Expires: 服务器时间，客户端时间可能和服务器时间不一致，所以不准；
    - cache-control:max-age=484200 在这个时间内缓存有效；
3. DNS查询：根据域名查出IP地址
4. 建立TCP连接，三次握手
5. 发出HTTP请求
6. 服务端响应
    - 本地缓存过期后，向服务器询问缓存是否真的过期了，带上if-modified-since和Etag(Etag资源的实体标识，更准确)，如果缓存未过期，返回304
7. 如果可以缓存，会被存储起来
8. 客户端渲染
9. 关闭或继续保持TCP连接，断开连接需要四次挥手(client:fin server: ack server: fin client: ack)

### 为什么断开连接需要四次挥手
- client端发FIN报文
- server端返回ACK确认（为什么没有一起返回FIN，因为server端此时可能还有数据需要发送，所以需要等server端把数据发送完毕后，再去发送FIN报文关闭连接）
- server端发FIN报文
- client端返回ACK确认

 ### call和apply的作用和区别
 - 函数
 - 改变方法执行的上下文
 - call是传多个参数，apply是传入参数数组
 #### 应用
 - Array.prototype.slice.call([类数组])，将类数组如arguments、NodeList转为数组
 - 应用Object.prototype.toString.call([obj])去判断具体的类型[object Type] Type可以为Boolean，String，Object，Array，Set等等
 - 获取数组最大值，Math.prototype.max.apply(Math, [3, 4, 5])
 - 合并数组，Array.prototype.push.apply(arr1, arr2)

``` js
(function() {
    'use strict';

    return this;
}).call(null); // null

(function() {
    return this;
}).call(null); // 非严格模式下，null和undefined会被替换为全局变量，通常情况下是window
```
this的指向
- new, this指向的是创建的实例对象
- call, apply, bind
- obj.foo()
- 默认绑定。如果在严格模式下，则绑定到 undefined，否则绑定到全局对象。
- 箭头函数没有自己的 this, 它的this继承于上一层代码块的this。

 ### websocket 全双工通信

 ### webworker 多线程
 作用：可以另开子线程，处理复杂数据的计算
 **注意点**
 - 同源限制：分配给 Worker 线程运行的脚本文件，必须与主线程的脚本文件同源。
 - DOM 限制：Worker 线程所在的全局对象，与主线程不一样，无法读取主线程所在网页的 DOM 对象，也无法使用document、window、parent这些对象。但是，Worker 线程可以navigator对象和location对象。
 - 通信联系：Worker 线程和主线程不在同一个上下文环境，它们不能直接通信，必须通过消息完成。
 - 脚本限制：Worker 线程不能执行alert()方法和confirm()方法，但可以使用 XMLHttpRequest 对象发出 AJAX 请求。
 - 文件限制：Worker 线程无法读取本地文件，即不能打开本机的文件系统（file://），它所加载的脚本，必须来自网络。
 #### 海量数据的渲染
 - setTimeout/requestAnimationFrame分批处理
 - 虚拟滚动列表

### 跨域
- JSONP: 利用script标签不受跨域的限制，只支持get请求
  - client端：构建script，发起跨域get请求，在url上拼接上参数和callback
  - server端：响应请求，将结果作为callback的参数（需要对json对象序列化），并返回callback函数的代码：
    - `callback({params: xxx})`
    - 序列化与反序列化：主要解决的是数据的一致性问题，数据需要序列化以后才能在服务端和客户端之间传输
  - 浏览器会执行callback函数，并传递解析后json对象作为参数
- 代理请求方式解决接口跨域问题：node.js代理请求，nginx反向代理接口跨域
  
- postMessage：允许与其他窗口（或iframe）互相通信
  
- document.domain和iframe，如果两个页面的主域名相同，则还可以通过设置 document.domain 属性将它们认为是同源的。
  - 父页面通过ifr.contentWindow就可以访问子页面的window，子页面通过window.parent或parent访问父页面的window

- location.hash + iframe跨域

- 使用CORS(Cross-origin resource sharing)协议: CORS需要浏览器和服务器同时支持。服务端设置 Access-Control-Allow-Origin：* ，客户端设置是否发送cookie，例如XMLHttpRequest的withCredentials，fetch请求的credentials: 'same-origin', mode: 'cors'；
  - 浏览器端会在Request Headers中新增一个origin字段
  - 默认不发送cookie和http认证
  - 服务器端设置Access-Control-Allow-Credentials：允许前端带认证cookie，启用此项后，上面的域名不能为'*'，必须指定具体的域名，否则浏览器会提示
  - 区分简单请求和复杂请求，复杂请求在正式发出请求之前会有一次预检（预检有效期内只会发出一次预检）
#### 跨域资源共享标准（ cross-origin sharing standard ）允许在下列场景中使用跨域 HTTP 请求：
- 前文提到的由 XMLHttpRequest 或 Fetch 发起的跨域 HTTP 请求。
- eb 字体 (CSS 中通过 @font-face 使用跨域字体资源), 因此，网站就可以发布 TrueType 字体资源，并只允许已授权网站进行跨站调用。
- WebGL 贴图
- 使用 drawImage 将 Images/video 画面绘制到 canvas



### 跨标签页通信
- window.open 和 postMessage
    - 当指定`window.open`的第二个name参数时，再次调用`window.open('****', 'child')`会使之前已经打开的同name子页面刷新
    - 由于安全策略，异步请求之后再调用`window.open`会被浏览器阻止，不过可以通过句柄设置子页面的url即可实现类似效果
    ``` js
    // 首先先开一个空白页
    const tab = window.open('about:blank')

    // 请求完成之后设置空白页的url
    fetch(/* ajax */).then(() => {
        tab.location.href = '**url**';
        tab.postMessage('msg', '**origin**')
    })
    ```
- localStorage 与监听 window.onstorage，event对象包括key、oldValue、newValue等参数
- cookie 
- sessionStorage
- 借助server

### 判断是否是数组
- [] instanceof Array
- [].__proto__ == Array.prototype
- [].constructor == Array // constructor可以修改
- Object.prototype.toString.call([]) == '[object Array]'
- Array.isArray([])

对于自定义的类A，修改了A.prototype后，修改原型之前创建的实例的原型链不会发生改变。
``` js
function A(){}
var a = new A();
function B(){}
A.prototype = B.prototype;
a instanceof A //false
a.__proto__ == A.prototype //false
```
因为instanceof是在原型链上查找的，所以instanceof和__proto__判断实例的方式都是false

#### 类数组 vs 数组
类数组有arguments、Dom对象列表(NodeList)、带length属性的对象（如：{1: 'mon', 3: 'wed', length: 4}）
**类数组转成数组**
- Array.prototype.slice.call(arrayLike)
- [...arrayLike]
- Array.from(arrayLike)
转为数组后就可以使用数组众多的方法，例如map、filter、slice、join、reduce、sort等等。

## let const
- 不存在变量提升
- 暂时性死区
如果使用了let、const，则该区域会形成一个封闭的作用域，在变量tmp使用let声明之前使用，都会报错，但如果一个变量根本没有被声明，反而不会报错。
``` js
if (true) {
  typeof tmp; // Uncaught ReferenceError: tmp is not defined
  typeof undeclared_var;  //'undefined'
  let tmp;
}
```
- 不允许重复声明
- 块级作用域
ES5中只有全局作用域和函数作用域，ES6新增了块级作用域，let和const只在声明的块级作用域中有效。

### 函数的执行上下文

1、创建阶段【当函数被调用，但未执行任何其内部代码之前】
- 创建作用域链（Scope Chain）
- 创建顺序：函数的形参==>>函数声明==>>变量声明
- 求”this“的值

**函数的声明比变量优先级要高，并且定义过程不会被变量覆盖，除非是赋值**
``` js
function foo3(a){
    var a = 10
    function a(){}
    console.log(a)
}
foo3(20) //'10' 创建变量的顺序是 形参a, function a(){}, var a;执行的顺序是 a=10; console

function foo3(a){
    var a 
    function a(){}
    console.log(a)
}
foo3(20) //'function a(){}' 创建变量的顺序是 形参a, function a(){}, var a;执行的顺序是 console
```

2、执行阶段
- 在当前上下文上运行/解释函数代码，并随着代码一行行执行指派变量的值。

### BFC(Block Formatting Context)
BFC是一个独立的渲染区域，内部的元素遵循一定的规则去布局，**不会影响外部元素也不会被外部元素影响**。
#### 生成BFC
- 根元素
- oveflow不为visible
- float不为none
- 绝对定位absolute和fixed
- flex元素和直接子元素
- display: inline-block、table-cell、table-caption
- grid元素和直接子元素
- display：table也认为可以生成BFC，其实这里的主要原因在于Table会默认生成一个匿名的table-cell，正是这个匿名的table-cell生成了BFC
- display 值为 flow-root 的元素

#### BFC布局规则
1. 每个元素的margin box的左边，与包含块border box的左边相接触(对于从左往右的格式化，否则相反)。即使存在浮动也是如此。
2. BFC的区域不会与float box重叠。（结合1、2，实现两栏布局|三栏布局）
3. 同一个BFC中相邻的box在垂直方向上的margin会合并 （外边距折叠）
4. 内部的box会在垂直方向上一个接一个放置
5. 计算BFC的高度时，浮动元素也参与计算（可以参考清除浮动前，外部元素的高度没有被撑起来）

#### 使用
- 清除浮动就是创建了一个新的BFC来包含浮动元素（oveflow:hidden/auto）
  - 因为BFC的特性，内部元素不应该影响外部元素布局，所以将浮动元素包含进来
- 属于同一BFC的两块级元素的外边距会折叠
    - 相邻元素之间
    - 父元素与第一个或最后一个子元素
    - 空的块级元素上下边距会折叠


### cookie、sessionStorage与localStorage
- cookie 4k 设置过期时间 每次http请求都会带上cookie 设置httponly的无法被js读取
- sessionStorage 5M 永久 仅在客户端
- localStorage  5M 会话期间 仅在客户端


### 盒模型
margin, border, padding，content 
- IE的怪异盒模型 盒子的总宽度 = margin + css设置的width  对应box-sizing的border-box
- 标准盒模型 盒子的总宽度 = margin + border + padding + css设置的width  对应box-sizing的content-box
**应用**：百分比的布局，在不同状态之间切换的button等

### css选择器优先级
- 从高到低：
  - 内联样式
  - ID 选择器（例如，#example）
  - 类选择器 (例如，.example)，属性选择器（例如，[type="radio"]）和伪类（例如，:hover，:first-child）
  - 类型选择器 (例如，div)，伪元素（例如，::after ::before）(除了before和after其他常见的都是伪类)
- !important 例外规则：此声明将覆盖任何其他声明

### flex弹性布局
有水平的主轴和垂直的交叉轴
- flex-direction
- flex-wrap 排列不下时如何换行
- flex-flow: [direction] [wrap]
- justify-content
- align-Items
- align-content: 当有多根主轴时，即item不止一行的时候，多行在交叉轴上的对齐方式
item的属性
- order 布局顺序
- flex-shrink: 收缩 默认为1，表示当空间不足时，item自动缩小
- flex-grow: 延伸 默认为0，即当有多余空间时也不放大
- flex-basis: 项目在主轴上占据的空间，默认为auto，优先级比width|height高
- align-self: 允许item有自己独特的在交叉轴上的对齐方式，默认值为auto，与父元素的align-items的值一样

### 深拷贝的实现
``` js
    const toString = Object.prototype.toString;
    function deepClone(origin) {
        let cloned = {}
        let cls = toString.call(origin)
        if(cls == '[object Date]') {
            const Ctor = origin.constructor
            return new Ctor(origin);
        }
        else if (cls == '[object Map]') {
            origin.forEach((subValue, key) => {
                result.set(key, deepClone(subValue))
            })
        }
        else if (cls == '[object Set]') {
            origin.forEach((subValue) => {
                cloned.add(deepClone(subValue))
            })
        }
        else if(typeof origin == 'object' || typeof origin == 'array') {
            for(var key in origin) {
                cloned[key] = deepClone(origin[key]);
            }
        }
        else {
            cloned = origin;
        }
        return cloned;
    }
```
### babel原理及插件开发
babel解析分为三步：
- 使用@babel/parser解析器，根据estree规范构造AST语法树
    - 默认支持最新的ECMAScript规范（ES2017）
    - JSX Flow TypeScript
- 转换AST
    - 根据一定的规则转换、修改AST（babel插件或者预置的stage-0，1，2，3，jsx等）
- 使用@babel/generator将AST转为code

### stage-0
- transform-do-expressions 支持在react的jsx语法中使用if/else语句
- transform-function-bind 提供::去实现和bind一样的作用

### stage-1
- transform-class-constructor-call
- transform-export-extensions

### stage-2
- syntax-dynamic-import 可以解析动态import语法
- transform-class-properties 支持property（不在prototype上）和static的定义
- transform-decorators 

### stage-3
- syntax-trailing-function-commas 允许参数后添加逗号
- transform-object-reset-spread rest参数... 解构
- transform-async-generator-functions transform-async-to-generator 支持async和await语法
- transform-exponentiation-operator 通过**这个符号进行幂操作

### webpack打包原理，loader原理
#### webpack构建流程
- 合并shell的参数和config.js配置的参数
- 注册所有配置的插件，让插件监听webpack生命周期
- 从entry入口开始解析文件，构建AST语法树，递归查找依赖
- 根据loader配置的规则对文件进行处理
- 递归完后得到每个文件的最终结果
- 应用plugin插件扩展webpack的功能
- 最后根据entry配置生成chunk，输出chunk
可以说一下持久化缓存的过程。
从 webpack2 开始，已经内置了对 ES6、CommonJS、AMD 模块化语句的支持。但不包括新的ES6语法转为ES5代码，这部分工作还是留给了babel及其插件。

### 技术栈以及遇到的问题
React + React-Router + Sass + Rem + Webpack打包
**问题**
- 资源同步问题 gitsubtree
- 正文页历史滚动位置：正文页和从其他页面返回第一步都是onpopstate事件（只对pushState和replaceState的页面有效）拿到key值，但是从其他页面返回，页面会reload一下，reload之前（先判断key值是否存在于历史位置中）存下当前key值和posHistory[key]值，在页面reload后，再恢复位置。
- 微信url返回问题
- 二维码无法识别问题
- 代码分片

**其他**
- polyfill引入方式


### 如何提高webpack打包速度
- 热更新或热替换
- 选择合适的devtool：sourceMap设置
- babel-loader开启缓存
- 全局script标签引入react/react-dom等第三方库
- DllPlugin和DllReferencePlugin动态链接库
- 提取公共代码
- 使用HappyPack多进程打包构建
- 优化打包文件路径配置，使用include和exclude
- ModuleConcatenationPlugin插件开启作用域提升：它将一些有联系的模块，放到一个闭包函数里面去，通过减少闭包函数数量从而加快JS的执行速度
- noParse：有时不需要解析某些模块的依赖（这些模块并没有依赖，或者并根本就没有模块化）
- 异步加载
- 只引入模块中的一部分
- 如何提高UglifyJsPlugin的压缩速度
    - cache 设置缓存
    - parallel 开启多进程并行压缩
    - sourceMap 帮助定位问题，但是可以关闭提高压缩速度


### promise
#### 错误catch问题
1. 将原本的callback形式的函数Promise化，然后通过promise.then(xxx).catch()去捕获异步操作中抛出的错误。
2. 在使用try/catch的时候，要配合async/await去使用，因为这样能保证异步函数的同步执行，这样能在try/catch方法中捕获异步函数中抛出的错误。
3. 跟传统的try/catch代码块不同的是，如果没有使用catch()方法指定错误处理的回调函数，Promise 对象抛出的错误不会传递到外层代码，即不会有任何反应。
- 有一个提案`Promise.try`可以用来统一处理同步和异步请求，支持then和catch方法，catch可以捕获所有同步和异步的错误
``` js
Promise.try(() => database.users.get({id: userId}))
  .then(...)
  .catch(...)
```

#### 方法
- 实例方法： then, catch, finally
- 静态方法： all, race, resolve, reject, allSettled
    - Promise.resolve([xxx]) 接收的参数可以是Promise实例、thenable对象、原始值、空，可以用来实现async函数
    - allSettled是ES2020新增的，返回值状态始终是fulfilled
``` js
const resolved = Promise.resolve(42);
const rejected = Promise.reject(-1);

const allSettledPromise = Promise.allSettled([resolved, rejected]);

allSettledPromise.then(function (results) {
  console.log(results);
});
// 只有allSettled会返回对应promises对象的status
// [
//    { status: 'fulfilled', value: 42 },
//    { status: 'rejected', reason: -1 }
// ]
```
### Generator 函数与 Promise 的结合 
``` js
function getFoo () {
  return new Promise(function (resolve, reject){
    resolve('foo');
  });
}

const g = function* () {
  try {
    const foo = yield getFoo();
    console.log('1:', foo);
    const bar = yield getFoo();
    console.log('2', bar);
  } catch (e) {
    console.log(e);
  }
};

function run(generator) {
    const gen = generator();
    function step(result) {
      if(result.done) {
        return result.value;
      }
      return result.value.then(function(v) {
        return step(gen.next(v));
      }, function(e) {
        return step(gen.throw(e));
      });
    }
    step(gen.next());
}

run(g);
```

## JSbridge原理
### JavaScript 调用 Native
- 注入API: 客户端通过webview提供的API，向JavaScript的Context（window）中注入对象和方法，让JavaScript调用时，直接执行相应的 Native 代码逻辑，达到 JavaScript 调用 Native 的目的
- 拦截 URL SCHEME: Web 端通过某种方式（例如 iframe.src）发送 URL Scheme 请求，之后 Native 拦截到请求并根据 URL SCHEME（包括所带的参数）进行相关操作。（需要创建请求、耗时、url长度限制、参数不够直观）
### Native 调用 JavaScript
- WebView作为自组件存在于View/Activity中，直接调用相应的API：Native 调用 JavaScript，其实就是执行拼接 JavaScript 字符串，从外部调用 JavaScript 中的方法，因此 JavaScript 的方法必须在全局的 window 上。
### JSBridge 接口实现
- callback参考JSON机制：
  - 当发送 JSONP 请求时，url 参数里会有 callback 参数，其值是 当前页面唯一 的，而同时以此参数值为 key 将回调函数存到 window 上，随后，服务器返回 script 中，也会以此参数值作为句柄，调用相应的回调函数。
https://juejin.im/post/5abca877f265da238155b6bc

## html的font-size计算
- wy: font值 / 100 = deviceWidth / 750
- taobao： font值 * r / 75 * r = deviceWidth / 750 


## rest参数和扩展运算符
``` js
function push(array, ...items) { //这个...是rest参数，将其他参数放到items数组中
  array.push(...items);  //这个...是扩展运算符，可以看作是rest的逆运算
}
```
**任何 [Symbol.Iterator] 接口的对象，都可以用扩展运算符转为真正的数组。**
``` js
var nodeList = document.querySelectorAll('div');
var array = [...nodeList];
```

## Symbol作用
- 属性名冲突

## 解构赋值
解构赋值默认会调用Iterator 接口，除此之外扩展运算符、for…of循环、yield*、Map、set、Array.from、Promise.all、Promise.race等等任何接收数组为参数的场合，都调用了Iterator接口。
``` js
//数组解构
let [a, b, c] = [1, 2, 3];

//对象结构
let {foo} = {foo: 1}

//字符串结构
const [a, b, c, d, e] = 'hello';

//数值和布尔值 会被转成对象
let {toString: s} = 123;
s === Number.prototype.toString // true

let {toString: s} = true;
s === Boolean.prototype.toString // true

//函数参数的解构赋值
function add([x, y]){
  return x + y;
}

add([1, 2]); // 3
```

## 对象的遍历
- for...in循环：只遍历对象自身的和继承的可枚举的属性。
- Object.keys()：返回对象自身的所有可枚举的属性的键名。
- JSON.stringify()：只串行化对象自身的可枚举的属性。，
- Object.assign()： 忽略enumerable为false的属性，只拷贝对象自身的可枚举的属性。

- Object.getOwnPropertyNames(obj)：返回对象自身的所有可枚举+不可枚举的属性（但不包括Symbol属性）的键名。
- Object.getOwnPropertySymbols(obj)：返回对象自身的所有Symbol属性的键名。
- Reflect.ownKeys(obj)：返回对象自身的所有键名。

- for of / for in可以中断
- forEach 不能中断
- map不能中断
### 自身可枚举属性
- 用class A extends B创建的对象实例，可以返回A和B的所有属性attr，因为继承是父类先创建了this实例，然后子类修改this实例
- getter在class中是不可枚举的，class的方法都是不可枚举的
- getter在普通对象中是可枚举的，如var a = {get a(){ return 1;}}; 这个时候序列化也是可以拿到值的"{"a":1}"
**`JSON.stringify`首先会调用toJSON方法，然后undefined、方法、Symbol等会被忽略，仅序列化自身可枚举的属性**

## 具备Iterator接口的数据结构
- Map
- Set
- Array
- String
- arguments
- NodeList
- TypedArray

**对象不具备Iterator属性，所以不支持for...of..遍历**
因为对象key值顺序不定，此外有补充Map数据结构，可变因素很多：是否是原型链上的、是否可枚举、是否要包括Symbol属性。
遍历顺序：先遍历出整数属性（integer properties，按照升序），然后其他属性按照创建时候的顺序遍历出来。
``` js
let codes = {
  "c": "China",
  "49": "Germany",
  "41": "Switzerland",
  "44": "Great Britain",
  "4.5": "Great Britain",
  "+5": "Great Britain",
  "-5": "Great Britain",
  "b": "British",
  "1": "USA"
};

for(let code in codes) {
  console.log(code); // 1, 41, 44, 49, c, 4.5, +5, -5, b
}
```

### Map和WeakMap的区别
- WeakMap只能使用对象作为键名
- WeakMap没有遍历操作
 因为没有办法列出所有键名，某个键名是否存在完全不可预测，跟垃圾回收机制是否运行相关。这一刻可以取到键名，下一刻垃圾回收机制突然运行了，这个键名就没了，为了防止出现不确定性，就统一规定不能取到键名。
- WeakMap无法清空，即不支持clear方法
- WeakMap只支持set、get、has、delete4个API

### Object和Map的区别
- Object的key值都会默认转为string
``` js
var object = {};
object[4] = 40;
object[{}] = 100;
object[NaN] = 50;
object[Symbol.key] = 60; 
for(var key in object){
    console.log(typeof(key));
}
//string
//string
//string
//string
console.log(object);
//{4: 40, [object Object]: 100, NaN: 50, undefined: 60}
```
- Map的键值不局限于字符串，可以是各种类型的值（包括对象）
- Object不具备Iterator属性，所以不支持for...of遍历

## generator和yeild
``` js
function* f() {
    yield 1;
    var n1 = yield 2;
    console.log(n1)
    var n2 = yield 3;
    console.log(n2)
    return 'end'
}
var g = f();
g.next(); // {value: 1, done: false}
g.next(); // {value: 2, done: false}
g.next(2); // 2 {value: 3, done: false}
g.next(); // undefined {value: 'end', done: true} 
g.next(); // {value: undefined, done: true}
```
如果有return值，会将return语句后面的表达式的值，作为返回的对象的value属性值，但此时的遍历已经结束，done为true。for...of只会返回done为false的遍历值。
yield表达式本身没有返回值，或者说总是返回undefined。next方法可以带一个参数，该参数就会被当作上一个yield表达式的返回值。
Generator函数只有在调用next()方法的时候，才会被执行。

## 按顺序完成异步操作
### promise
``` js
function logInOrder(urls) {
  // 远程读取所有URL
  const textPromises = urls.map(url => {
    return fetch(url).then(response => response.text());
  });

  // 按次序输出
  textPromises.reduce((chain, textPromise) => {
    return chain.then(() => textPromise)
      .then(text => console.log(text));
  }, Promise.resolve());
}
```

### async
https://objcer.com/2017/10/12/async-await-with-forEach/
``` js 顺序进行
async function logInOrder(urls) {
  for (const url of urls) {
    const response = await fetch(url);
    console.log(await response.text());
  }
}
```
``` js
async function logInOrder(urls) {
  // 并发读取远程URL 并发执行，因为只有async函数内部是继发执行，外部不受影响
  const textPromises = urls.map(async url => {
    const response = await fetch(url);
    return response.text();
  });

  // 按次序输出  for..of循环内部使用了await，因此实现了按顺序输出。
  for (const textPromise of textPromises) {
    console.log(await textPromise);
  }

  //Promise.all(textPromises).then(result => console.log(result)); 并发
}
```

## 内存泄露
> 定义：不再用到的内存没有及时释放，就叫做内存泄露。
垃圾回收，如果一个变量的引用计数为0，则表示这个值不再用到了，因此可以将这块内存释放。
#### 防止内存泄露
- WeakMap和WeakSet
- 引用置为null
- 移除事件绑定
- 小心全局变量
- 小心闭包

#### 常见的内存泄露场景
1. 意外的全局变量
2. 定时器的处理函数没有及时释放，没有调用clearInterval方法
3. 脱离 DOM 的引用
4. 闭包上下文绑定后没有被释放

## Promise和setTimeout区别
### 事件循环: 
#### 同步任务VS异步任务
当我们设置一个延迟函数的时候，当前脚本并不会阻塞，它只是会在浏览器的事件表中进行记录，程序会继续向下执行。当延迟的时间结束之后，会将回调函数添加至**事件队列（task queue）**中，事件队列拿到了任务过后便将任务压入**执行栈（stack）**当中，执行栈执行任务，输出 'setTimeout'。
#### 如何判断结束？
**JS引擎**的monitoring process进程，会持续不断的检查主线程执行栈是否为空，一旦为空，就会去Event Queue那里检查是否有等待被调用的函数。
由于 JS 是单线程的，同步执行任务会造成浏览器的阻塞，所以我们将 JS 分成一个又一个的任务，通过不停的循环来执行事件队列中的任务。这就使得当我们挂起某一个任务的时候可以去做一些其他的事情，而不需要等待这个任务执行完毕。所以事件循环的运行机制大致分为以下步骤：
1. 检查事件队列是否为空，如果为空，则继续检查；如不为空，则执行 2；
2. 取出事件队列的首部，压入执行栈；
3. 执行任务；
4. 检查执行栈，如果执行栈为空，则跳回第 1 步；如不为空，则继续检查；

#### 宏任务和微任务
- MacroTask: setTimeout, setInterval, setImmediate, I/O, 网络请求
- MicroTask: process.nextTick, promise的then和catch, Object.observe, MutationObserver
requestAnimationFrame
UI rendering

在某一个macrotask执行完后，在重新渲染与开始下一个宏任务之前，就会将在它执行期间产生的所有microtask都执行完毕（在渲染前）。**微任务会在执行栈执行完后立即执行，而宏任务要等到下一次的event loop才会被执行**。
> 在node环境下，process.nextTick的优先级高于Promise，也就是说：在宏任务结束后会先执行微任务队列中的nextTickQueue，然后才会执行微任务中的Promise执行机制：
一次事件循环过程：
1. 执行一个宏任务（栈中没有就从事件队列中获取）
2. 执行过程中如果遇到微任务，就将它添加到微任务的任务队列中
3. 宏任务执行完毕后，立即执行当前微任务队列中的所有微任务（依次执行）
4. 当前宏任务执行完毕，开始检查渲染，然后GUI线程接管渲染(规范允许浏览器自己选择是否更新视图，根据回流和重绘的规则？https://www.html5rocks.com/zh/tutorials/internals/howbrowserswork)
5. 渲染完毕后，JS引擎线程继续，开始下一个宏任务（从宏任务队列中获取）
``` js
setTimeout(() => {console.log('test');}, 1);
setTimeout(() => {console.log('test2');}, 4);

requestAnimationFrame(() => {
    console.log('raf');
    requestAnimationFrame(() => {
        console.log('raf2');
    });
});
// 输出的可能顺序 1
// raf - 第一次raf 和 test的时间取决于当前帧还剩下的时间，如果<=4ms则先执行raf
// test
// test2
// raf2 - 下一次渲染之前有16ms的时间，可以执行两次宏任务

// 输出的可能顺序 2
// test - 第一次raf 和 test的时间取决于当前帧还剩下的时间，如果>4ms则先执行test
// raf
// test2
// raf2 - 下一次渲染之前有16ms的时间，可以执行两次宏任务

```

#### 任务优先级
- 来自不同任务源的任务会进入到不同的任务队列。其中setTimeout与setInterval是同源的。
- 微任务队列优先级： process.nextTick > Promise.then > Object.observe(废弃属性) > MutationObserver
- 宏任务队列优先级： setTimeout/setInterval> 异步IO > setImmediate
对于 UI rendering 来说，浏览器会在每次清空微任务队列会根据实际情况触发。

REF:
[前端基础进阶（十二）：深入核心，详解事件循环机制](https://www.jianshu.com/p/12b9f73c5a4f)
[这一次，彻底弄懂 JavaScript 执行机制](https://juejin.im/post/59e85eebf265da430d571f89#heading-4)

## js实现new
``` js
function instance(Cls, ...params) {
    let obj = Object.create(Cls.prototype);
    let res = Cls.apply(obj, params);
    //如果是非primitive类型的返回值，会return这个对象
    if (typeof res === "object" || typeof res === "function") { 
    	return res;
    }
    return obj
}
```

## 闭包
按值传递
https://juejin.im/post/58cf180b0ce4630057d6727c#heading-1

## 相等操作符(==)
相等操作符会对操作值进行隐式转换后进行比较：
- 如果一个操作值为布尔值，则在比较之前先将其转换为数值  //false == 0 true
- 如果一个操作值为字符串，另一个操作值为数值，则通过Number()函数将字符串转换为数值  //'0' == 0 true
- 如果一个操作值是对象，另一个不是，则调用对象的valueOf()方法，得到的结果按照前面的规则进行比较 //new Number(1) == 1 true
- null与undefined是相等的  //null == undefined true
- 如果一个操作值为NaN，则相等比较返回false  //NaN == NaN false
- 如果两个操作值都是对象，则比较它们是不是指向同一个对象 //var a = {a: 1}; var b = {a: 1}; a == b; false

Object.valueof()返回值为该对象的原始值,不同类型对象的valueOf()方法的返回值不同。


### XMLHttpRequest readyState和status的状态
``` js
const ReadyState = {
  UNSENT: 0, // Client has been created. open() not called yet.
  OPENED: 1, // open() has been called.	
  HEADERS_RECEIVED: 2,	// send() has been called, and headers and status are available.
  LOADING: 3, //	Downloading; responseText holds partial data.
  DONE: 4, // The operation is complete.
}

var oReq = new XMLHttpRequest();
console.log('UNSENT', xhr.readyState); // readyState will be 0
oReq.onprogress = function () {
    console.log('LOADING', xhr.readyState); // readyState will be 3
};
oReq.onload = function reqListener() {
  if(this.status === 200) {
    var data = JSON.parse(this.responseText);
    console.log(data);
  }
  console.log('DONE', xhr.readyState); // readyState will be 4
}
oReq.open('get', './api/some.json', true);
console.log('OPENED', xhr.readyState); // readyState will be 1
oReq.send();
```
#### `application/json; charset=utf-8`和`application/x-www-form-urlencoded; charset=utf-8`的区别：
1. `application/json`告诉webServer post请求传递的是JSON数据类型:
``` js
{ Name : 'John Smith', Age: 23}
```
2. `application/x-www-form-urlencoded`告诉webServer post请求传递的数据会被拼接到url上
``` js
Name=John+Smith&Age=23
```

### [「前端进阶」高性能渲染十万条数据(时间分片)](https://juejin.im/post/5d76f469f265da039a28aff7#heading-1)
> 性能关键：渲染耗时
- 分片渲染，一次渲染20条，在setTimeout或requestAnimationFrame回调中触发，requestAnimationFrame性能好于setTimeout
- 先append到documentFragment，直接append到document每次都会触发回流，并且会计算样式表（当然现在浏览器的优化已经做的很好了， 当append元素到document中后，没有访问 getComputedStyle 之类的方法时，现代浏览器也可以把样式表的计算推迟到脚本执行之后）。
**setTimeout 和闪屏现象**
- setTimeout的执行时间可能比设定的要晚。每次timeout后会将callback放到事件队列中，需要在当前主线程执行完成后才会去检查事件队列。
- 刷新频率受屏幕分辨率和屏幕尺寸的影响，因此不同设备的刷新频率可能会不同，而setTimeout只能设置一个固定时间间隔，这个时间不一定和屏幕的刷新时间相同。
以上两种情况都会导致setTimeout的执行步调和屏幕的刷新步调不一致。
在setTimeout中对dom进行操作，必须要等到屏幕下次绘制时才能更新到屏幕上，如果两者步调不一致，就可能导致中间某一帧的操作被跨越过去，而直接更新下一帧的元素，从而导致丢帧现象。

### performance.timing
在 Navigation Timing Level 2 草案中，已经废弃了 PerformanceTiming 接口，并且提供了新的接口 PerformanceNavigationTiming 代替其功能。
为什么被废弃？因为 W3C 给我们提供了更全面、更强大的一个性能分析矩阵，比单一的 performance.timing 更加强大，能帮助我们从各个方面分析前端页面性能。


### pushstate 和 popstate应用
- 回退缓存
- 回退挽留弹窗
``` js
/**
 * onpopstate是在history.back之后调用的，这个时候event.state是返回后的history的状态，之前的state已经被pop出去了。
 **/
window.onpopstate = function(event) {
  alert("location: " + document.location + ", state: " + JSON.stringify(event.state));
};

history.pushState({page: 1}, "title 1", "?page=1");
history.pushState({page: 2}, "title 2", "?page=2");
history.replaceState({page: 3}, "title 3", "?page=3");
history.back(); // alerts "location: http://example.com/example.html?page=1, state: {"page":1}"
history.back(); // alerts "location: http://example.com/example.html, state: null
history.go(2);  // alerts "location: http://example.com/example.html?page=3, state: {"page":3}
```

### preload vs Prefetch
preload 是一个声明式 fetch，可以强制浏览器在不阻塞 document 的 onload 事件的情况下请求资源。
Prefetch 告诉浏览器这个资源将来可能需要，会在浏览器空闲的时候请求资源。

### H5离线缓存
ServiceWorker实现离线资源缓存：https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API/Using_Service_Workers
参考示例https://github.com/mdn/sw-test
caches.open() 方法来创建了一个叫做 v1 的新的缓存，将会是我们的站点资源缓存的第一个版本。它返回了一个创建缓存的 promise，当它 resolved的时候，我们接着会调用在创建的缓存示例上的一个方法  addAll()，这个方法的参数是一个由一组相对于 origin 的 URL 组成的数组，这些 URL 就是你想缓存的资源的列表。
``` js
// 在install事件回调中添加站点资源缓存
self.addEventListener('install', function(event) {
  event.waitUntil(
    caches.open('v1').then(function(cache) {
      return cache.addAll([
        '/sw-test/',
        '/sw-test/index.html',
        '/sw-test/style.css',
        '/sw-test/app.js',
        '/sw-test/image-list.js',
        '/sw-test/star-wars-logo.jpg',
        '/sw-test/gallery/bountyHunters.jpg',
        '/sw-test/gallery/myLittleVader.jpg',
        '/sw-test/gallery/snowTroopers.jpg'
      ]);
    })
  );
});
```
``` js
// fetch事件监听资源请求
self.addEventListener('fetch', function(event) {
  event.respondWith(
    // 当命中缓存的时候，直接使用，如果没有命中缓存，去服务器进行网络请求，并把请求回来的资源放到缓存中
    caches.match(event.request).then(function(resp) {
      return resp || fetch(event.request).then(function(response) {
        return caches.open('v1').then(function(cache) {
          cache.put(event.request, response.clone());
          return response;
        });  
      });
    }).catch(function() {
      // 当请求没有匹配到缓存中的任何资源的时候，以及网络不可用的时候，兜底方案
      return caches.match('/sw-test/gallery/myLittleVader.jpg');
    })
  );
});
```
