---
title: npm转盘组件开发
date: 2018-02-09 14:39:53
tags: [npm包, 组件开发]
---
### 准备工作
在开始开发之前，首先要准备好node环境和npm账户，nodeJS基本都有安装好，要做的就是注册npm：
``` bash
$ npm adduser
Username: your name
Password: your password
Email: (this IS public) your email
```
查询当前账号或者登陆别的用户：
``` bash
$ npm whoami
$ npm login
```

### 包管理工具lerna
项目组采用了[lerna](https://github.com/lerna/lerna)包管理工具，可以在一个仓库下管理多个npm包，基于monorepo理念。
> Monorepo is a unified source code repository used by an organisation to host as much of its code as possible.
Monorepo 它是一种管理 organisation 代码的方式，在这种方式下会摒弃原先一个 module 一个 repo 的方式，取而代之的是把所有的 modules 都放在一个 repo 内来管理。

lerna安装和初始化
``` bash
npm install lerna -g
lerna init
```
初始化后，目录里面会自动生成/packages、lerna.json和package.json。
执行`lerna bootstrap`会自动为项目进行npm install和npm link操作，npm的作用是边开发边试用，具体使用参考[npm link](http://javascript.ruanyifeng.com/nodejs/npm.html#toc18)。
<!-- more -->
执行`lerna publish`可以发布版本。
lerna有两种模式的管理方式，项目仓库选用默认的Fixed/Locked模式，在lerna.json里的version字段是本仓库版本，packages下的包的版本等于或者小于本仓库的版本
有两种情况：
- 如果某个包的版本更新了，原版本是1.0.2，本仓库版本是1.0.9，则这个包的更新后的版本是1.0.10，而不是1.0.3
- 如果有个包引入了重大改动（BREAKING CHANGE）, 则所有的包的major version都会改变
其中commit message的编写遵循[Conventional Commits ](https://conventionalcommits.org/)，主要包括fix/feat/BREAKING CHANGE三种类型，以及对文档的修改`docs:`，或者其他`style:, refactor:, perf:, test:, chore:`。

### 组件开发
#### 定义组件配置项和方法
组件对外暴露的类是PrizeWheel，该class的构造函数支持传入`rotateSelector`(将要旋转的元素对应的选择器)和`config`(转盘相关配置)两个参数，config中支持对奖项、旋转元素类型、转速等的配置。此外，该class提供了旋转到指定位置和终止旋转两个方法。
初始化时，_countPrizeAngleMap方法会计算出每个奖项的目标角度。
``` javascript
_countPrizeAngleMap() {
    const {prizeQueue, rotator, inSector} = this.config;
    let len = prizeQueue.length;
    prizeQueue.forEach( (level, index) => {
        let diff = 0, targetAngle = 0;
        if(inSector) {
            diff = - 360/len/2;
        }
        if(rotator == 'pointer') { //指针旋转
            targetAngle = 360/len * (index + 0.5) + diff;
        }
        else {  //转盘旋转
            targetAngle = 360 - (360/len * (index + 0.5) + diff);
        }
        this.prizeAngleMap.set(level, targetAngle);
    })
}
```
prizeQueue是一个数组，对应转盘上的奖项队列，按照顺时针方向。rotator是旋转类型：'pointer'(Default) - 指针旋转、'wheel' - 转盘旋转。inSector代表初始位置是否在扇区内：false(Default)、true，false代表与扇区的边缘线重合，true代表在扇区的正中位置。旋转的情况有四种(N从0开始计数)：
- 旋转的是指针，inSector为false：指针在边缘线，则第N个奖项的目标角度 = 每个扇区的角度 * (N + 0.5)；
- 旋转的是指针，inSector为true：指针在扇区中央，旋转的角度相对上一种少了半个扇区，则第N个奖项的目标角度 = 每个扇区的角度 * (N + 0.5) - 每个扇区的角度 * 0.5；
- 旋转的是扇区，inSector为false(或true)：目标角度都是 = 360 - (指针旋转情况下的角度)

具体的转盘效果封装在了WheelEffect类中，该类的关键点是提供了一个缓动函数(参考的网上代码)：
``` js
function easing(t, b, c, d) { return -c * ((t=t/d-1)*t*t*t - 1) + b; }
```
其中参数含义：
- t: 当前动画已经持续的时间
- b: 开始的角度
- c: 一共要旋转的角度和
- d: 动画要持续的时间
间隔特定的时间去旋转该方法返回的角度，实现动效。

#### 组件发布
期望通过`import PrizeWheel from 'nw-prize-wheel'`的方式引入组件，所以在prize-wheel.js中是这样export的：
``` js
export default class PrizeWheel {
}
```
但在打包成js文件时，通过script标签嵌入，发现直接`new PrizeWheel()`报错，提示undefined，在网上找了一圈，发现需要另外新建一个index.js，在index.js中:
``` js
    module.exports = require('./src/prize-wheel.js').default;
```
webpack配置添加library和libraryTarget。
``` js
  output: {
      path: __dirname + '/lib',
      filename: '[name].js',
      library: 'PrizeWheel',
      libraryTarget: 'this'
  }
```
期望webpack提供诸如`libraryTarget: 'default'`这样的配置来解决问题，但目前不支持，只能用上述方法来曲线救国。

#### 组件API文档化
``` bash
jsdoc2md --template readme.hbs --files src/prize-wheel.js > readme.md
```
结合了[jsdoc](http://usejsdoc.org/index.html)和jsdoc2md可以将自定义的模板和js注释生成一个API文档，具体的使用方式详见官网。

最终实现的组件详情[nw-prize-wheel](https://www.npmjs.com/package/nw-prize-wheel)
