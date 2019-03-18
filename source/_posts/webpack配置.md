---
title: webpack配置
date: 2017-06-27 15:06:13
tags: [webpack, 前端打包, require]
---

### dll动态链接库优化打包
为dll单独配置一个webpack.dll.config.js文件，在entry中引入你要打包到vendor的模块，通常是一些通用的不涉及业务的库。
``` javascript
var webpack = require('webpack');

module.exports = {
    entry: {vendor: ['react', 'react-dom']},
    output: {
        path: './dist',
        filename: '[name].bundle.js',
        library: '[name]'
    },
    plugins:[
        new webpack.DllPlugin({
            path: './dist/[name]-manifest.json',
            name: '[name]',
            context: __dirname,
        })
    ]
}
```
**webpack.DllPlugin**的选项中，path是manifest文件的输出路径；name是dll暴露的对象名，要跟output.library保持一致。执行`webpack --config webpack.dll.config.js`后会在dist文件夹下面生成<font color=#F44336>vendor.manifest.json</font>和<font color=#F44336>vendor.bundle.js</font>。其中vendor.manifest.json长这样：

``` javascript
{
  "name": "vendor",
  "content": {
    "./node_modules/react/react.js": 1,
    "./node_modules/react/lib/React.js": 2,
    "./node_modules/process/browser.js": 3,
    "./node_modules/object-assign/index.js": 4,
    "./node_modules/react/lib/ReactBaseClasses.js": 5,
    "./node_modules/react/lib/reactProdInvariant.js": 6,
    "./node_modules/react/lib/ReactNoopUpdateQueue.js": 7,
    "./node_modules/fbjs/lib/warning.js": 8,
    ......
```
webpack将每个库进行了编号，之后dll user可以读取该文件，根据这个索引来引用。

<!-- more -->

之后，在webpack.config.js文件中添加DllReferencePlugin的插件配置。
``` javascript
    plugins:[
        new webpack.DllReferencePlugin({
            context: __dirname,//需要与webpack.dll.config.js中DllPlugin上下文一致
            manifest: require('./dist/vendor-manifest.json')//DllPlugin输出的manifest.json文件
        })
    ]
```
配置完成后，每次业务逻辑的修改只需要执行`webpack --config webpack.config.js`，不需要重复对dll进行打包，可以节约打包时间。


### 使用require.ensure分包
``` javascript
  require.ensure([/*预加载模块*/], function(require) {
    const Promise = require('es6-promise').Promise;
  }, 'promise');
```
webpack会将es6-promise模块打包成promise.chunk.js(require.ensure的第三个参数和chunkFilename决定了打包后的文件名)。
每个require.ensure会把前面数组里面的模块和内部require的模块都打包到一个文件内，异步加载。
可以在ensure的[]数组中加入想要预加载的模块，也可以在function内部使用require.include对文件进行预加载。


### webpack-dev-server热替换
> webpack-dev-server 主要提供两个功能：
- 为静态文件提供服务
- 自动刷新和热替换(HMR)

webpack-dev-server提供了两种自动刷新模式：iframe和inline，默认模式是iframe。inline方式会将webpack-dev-server客户端加入到webpack入口文件的配置中，配置方式有CLI和NodeJs API两种。

##### CLI方式
```javascript
webpack-dev-server --env development --port=8081 --hot --inline
```

##### Node.js API方式
手动将webpack-dev-server客户端配置到webpack打包的入口文件中
修改文件webpack.config.dev.js，添加webpack/hot/dev-server，添加插件HotModuleReplacementPlugin：
``` javascript
var webpack = require("webpack");
var webpackBase = require("./webpack.config.base.js");


var cfg = Object.assign(webpackBase, {
    devtool: "cheap-module-eval-source-map"
});

//entry
Object.getOwnPropertyNames((webpackBase.entry || {})).map(function (name) {
    cfg.entry[name] = []
        //添加HMR文件
        .concat("webpack/hot/dev-server")
        .concat("webpack-dev-server/client?http://localhost:9390")
        .concat(webpackBase.entry[name])
});

//plugins
cfg.plugins = (webpackBase.plugins || []).concat(
    new webpack.optimize.OccurrenceOrderPlugin(),
    //添加HMR插件
    new webpack.HotModuleReplacementPlugin(),
    new webpack.NoErrorsPlugin()
)

module.exports = cfg;
```

根目录添加文件devServer.js，用于创建服务器实例
``` javascript
var path = require("path");
var webpack = require("webpack");
var webpackDevServer = require("webpack-dev-server");
var webpackCfg = require("./webpack.config.dev.js");

var compiler = webpack(webpackCfg);

//init server
var app = new webpackDevServer(compiler, {
    //注意此处publicPath必填
    publicPath: webpackCfg.output.publicPath,
    //HMR配置
    hot:true
});

app.listen(9390, "localhost", function (err) {
    if (err) {
        console.log(err);
    }
});

console.log("listen at http://localhost:9390");
```
修改package.json中scripts配置，通过执行devServer.js文件启动服务器：
``` javascript
"scripts":{
    "start":"node devServer.js"
}
```


### 基础版webpack配置
1. 开发环境下

``` javascript
var webpack = require('webpack');
var path = require('path');

module.exports = {
    entry: {iread: "./iread/src/index.js",
            icartoons: "./icartoons/src/index.js"},
    output: {
        path: "./dist", //资源文件引用的目录
        publicPath: '../dist/',  //相对于html中,指定资源文件引用的目录
        filename: "[name].bundle.js",
        chunkFilename: '[name].chunk.js'
    },
    module: {
        loaders: [
            { test: path.resolve('./','lib/zepto.js'), loader: "exports?window.$!script" },
            { test: /\.css$/, loader: "style-loader!css-loader" },
            { test: /\.(png|jpg)$/, loader: "url-loader" },
            { test: /.jsx?$/, loader: 'babel-loader',exclude: /node_modules/, query: { presets: ['es2015', 'react'] } }
         ]
    },
    plugins:[
        new webpack.ProvidePlugin({
            $: path.resolve('./','lib/zepto.js')
        }),
        new webpack.DllReferencePlugin({
            context: __dirname,//需要与webpack.dll.config.js中DllPlugin上下文一致
            manifest: require('./dist/vendor-manifest.json')//DllPlugin输出的manifest.json文件
        })
    ]
}
```

2. 生产环境下

```javascript
var webpack = require('webpack');
var path = require('path');

module.exports = {
    entry: {iread: './iread/src/index.js',
            icartoons: './icartoons/src/index.js'},
    output: {
        path: "./dist",
        filename: "[name].bundle.min.js"
    },
    module: {
        loaders: [
            { test: path.resolve('./','lib/zepto.js'), loader: "exports?window.$!script" },
            { test: /\.css$/, loader: "style-loader!css-loader" },
            { test: /\.(png|jpg)$/, loader: "url-loader" },
            { test: /.jsx?$/, loader: 'babel-loader',exclude: /node_modules/, query: { presets: ['es2015', 'react'] } }  
         ]
    },
    plugins:[
        new webpack.ProvidePlugin({
            $: path.resolve('./','lib/zepto.js')
        }),
        new webpack.DefinePlugin({
            'process.env': {
              'NODE_ENV': JSON.stringify('production')
            }
        }),
        new webpack.optimize.UglifyJsPlugin({
          compress: {
            warnings: false
          }
        })
    ]
}
```
