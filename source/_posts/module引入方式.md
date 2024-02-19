### UMD模式
UMD (Universal Module Definition), 希望提供一个前后端跨平台的解决方案(支持AMD与CommonJS模块方式)。

#### 实现原理
1. 先判断是否支持Node.js模块格式（exports是否存在），存在则使用Node.js模块格式。
2. 再判断是否支持AMD（define是否存在），存在则使用AMD方式加载模块。
3. 前两个都不存在，则将模块公开到全局（window或global）。
``` js
// if the module has no dependencies, the above pattern can be simplified to
(function (root, factory) {
    if (typeof exports==="object"&&typeof module!=="undefined") {
        // Node CommonJS. Does not work with strict CommonJS, but
        // only CommonJS-like environments that support module.exports,
        // like Node.
        module.exports = factory();
    } else if (typeof define === 'function' && define.amd) {
        // AMD. Register as an anonymous module.
        define([], factory);
    } else {
        // Browser globals (root is window)
        root.returnExports = factory();
  }
}(this, function () {

    // Just return a value to define the module export.
    // This example returns an object, but the module
    // can return a function as the exported value.
    return {};
}));
``` 
如果没有定义root参数，也可以通过下面的方式去赋值给全局变量：
``` js
    var g;
    if(typeof window!=="undefined"){
        g=window
    }else if(typeof global!=="undefined"){
        g=global
    }else if(typeof self!=="undefined"){
        g=self
    }else{
        g=this
    }
```

#### ES Module与CommonJS
1. 语法差异：
   - ES Module使用import、export
   - CommonJS使用require、module.exports（exports）
2. 依赖关系的确认：
   - CommonJS模块是对象，是运行时加载，运行时才把模块挂载在exports之上（加载整个模块的所有），加载模块其实就是查找对象属性；
   - ES Module是在编译时确定依赖关系（设计思想：尽量的静态化）。
3. CommonJS 模块输出的是一个值的拷贝，ES6 模块输出的是值的引用。
   - CommonJS模式中，exports的一个参数，即使该参数发生了变化，引入的地方拿到的值始终不变
   - 在ES Module中，export的参数如果发生了变化，在引入的地方也会同步更新；引用的参数是read-only的（因为是引用，约定是不能在外部修改）。
4. 只执行一次
    - 多次require，CommonJS返回的是module的缓存
    - ES6 module多次import，只会在首次import中执行
5. 加载方式：
    - commonJS模块就是对象，整体加载模块（即加载的所有方法）
    - ES6 模块不是对象，而是通过export命令显式指定输出的代码，再通过import命令输入。
6. 运行环境：AMD/CMD是CommonJS在浏览器端的解决方案。
    - CommonJS node端是同步加载（代码在本地，加载时间基本等于硬盘读取时间）。
    - AMD/CMD是异步加载（浏览器必须这么做，代码在服务端）

#### CommonJs中exports和module.exports的差别
``` js
var module = { exports: {} };
var exports = module.exports;

// your code

return module.exports;
```
这意味着exports的正确使用方法, 只有exports.A = B这种挂变量的形式。
modules.exports一旦被直接赋值, 如modules.exports = 1, 也会断开它和exports之间的联系, 导致exports失去意义. 
参考[https://www.jianshu.com/p/434c247759bc](https://www.jianshu.com/p/434c247759bc)

#### AMD vs CMD
AMD规范采用异步方式加载模块，模块的加载不影响它后面语句的运行。所有依赖这个模块的语句，都定义在一个回调函数中，等到加载完成之后，这个回调函数才会运行。
``` js
/** 网页中引入require.js及main.js **/
<script src="js/require.js" data-main="js/main"></script>

/** main.js 入口文件/主模块 **/
// 首先用config()指定各模块路径和引用名
require.config({
  baseUrl: "js/lib",
  paths: {
    "jquery": "jquery.min",  //实际路径为js/lib/jquery.min.js
    "underscore": "underscore.min",
  }
});
// 执行基本操作
require(["jquery","underscore"],function($,_){
  // some code here
});

// 定义模块
define(function () {
  // some code here
});

// 定义的模块本身也依赖其他模块
define(["jquery"], function () {
  // some code here
});
```

``` js
/** AMD写法 **/
define(["a", "b", "c", "d", "e", "f"], function(a, b, c, d, e, f) { 
     // 等于在最前面声明并初始化了要用到的所有模块
    a.doSomething();
    if (false) {
        // 即便没用到某个模块 b，但 b 还是提前执行了
        b.doSomething()
    } 
});

/** CMD写法 **/
define(function(require, exports, module) {
    var a = require('./a'); //在需要时申明
    a.doSomething();
    if (false) {
        var b = require('./b');
        b.doSomething();
    }
});

/** sea.js **/
// 定义模块 math.js
define(function(require, exports, module) {
    var $ = require('jquery.js');
    var add = function(a,b){
        return a+b;
    }
    exports.add = add;
});
// 加载模块
seajs.use(['math.js'], function(math){
    var sum = math.add(1+2);
});
```
**AMD 推崇依赖前置、提前执行，CMD推崇依赖就近、延迟执行**


### commonjs vs es6 modules
两个重大差异：
- CommonJS 模块输出的是一个值的拷贝，ES6 模块输出的是值的引用。
- CommonJS 模块是运行时加载，ES6 模块是编译时输出接口。
  - 第二个差异是因为 CommonJS 加载的是一个对象（即module.exports属性），该对象只有在脚本运行完才会生成。而 ES6 模块不是对象，它的对外接口只是一种静态定义，在代码静态解析阶段就会生成。
- 是否缓存数据
  - JS 引擎对脚本静态分析的时候，遇到模块加载命令import，就会生成一个只读引用。等到脚本真正执行时，再根据这个只读引用，到被加载的那个模块里面去取值。原始值变了，import加载的值也会跟着变。因此ES6模块不会缓存运行结果，而是动态地去被加载的模块取值，并且变量总是绑定其所在的模块。
- export通过接口，输出的是同一个值。不同的脚本加载这个接口，得到的都是同样的实例。
  - 如果多次重复执行同一句import语句，那么只会执行一次，而不会执行多次。
  - 下面代码中，虽然foo和bar在两个语句中加载，但是它们对应的是同一个my_module实例。也就是说，import语句是 Singleton 模式[es6-modules-single-instance-pattern](https://k94n.com/es6-modules-single-instance-pattern)。
  ``` js
  import { foo } from 'my_module';
  import { bar } from 'my_module';

  // 等同于
  import { foo, bar } from 'my_module';
  ```
  