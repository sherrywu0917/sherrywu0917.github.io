---
title: cmd小工具开发
date: 2018-05-31 10:02:52
tags:
---
### cmd项目初始化
首先新建一个文件夹，并且初始化package.json，同时在bin文件夹下面创建入口文件index.js。
``` bash
    npm init #初始化'package.json'文件
```
index.js文件头部有`#!/usr/bin/env node`，叫做[shebang](https://en.wikipedia.org/wiki/Shebang_(Unix))，指定脚本解释程序为`node`。
package.json文件中bin字段配置了cmd命令将执行的js文件，如下所示：
``` js
{
  "name": "mkfile-cli",
  "version": "1.0.7",
  "main": "index.js",
  "bin": {
    "mk": "bin/index.js"
  },
  "repository": {
    "type" : "git",
    "url" : "https://xxx"
  },
  "author": "sherrywu",
  "license": "ISC",
}
```
#### 运行
- 可以运行`node bin/index.js`命令
- 通过`npm link`的方式
- 还可以通过`npm install . -g`安装到全局

### cmd命令行工具开发
#### commander.js
[Commander](https://github.com/tj/commander.js)是一个轻量级、强大的命令行开发框架，提供了封装好的API，帮助用户快速开发命令行工具。
##### 安装：
``` bash
npm install commander --save
```

#### option
option()方法的定义

- option(flags, description, fn, defaultValue)

  - flags <String> : 自定义参数，格式为"-shortFlag, --longFlag null|`<value>`|[value]|`<value>`..`<value>`"

1. -shortFlag："-"后面跟的是自定义参数的短标志，一般用longFlag的第一个字母(区分大小写)
2. --longFlag ："--"后面跟的是自定义参数的长标志，shortFlag和longFlag必须同时存在，
3. null|`<value>`|[value]：有3种情况
  - null——可以不带参数
  - `<value>`——“<>”修饰，参数必须。
  - [value]——”[]“修饰, 参数可选。
4. description <String> : 对flags参数的描述
5. fn <Function|Mixed> : 自定义处理参数的方法，如果传入的不是一个方法，会先判断是否为一个正则表达式，如果不是，则视为defaultValue（默认值），
6. defaultValue <Mixed> ：自定义参数默认值
7. 返回值 <Object>：commander对象

``` js
program
    .version('1.0.1')
    .option('-n, --new <filename>', '创建文件')
    .parse(process.argv);
if(program.new || program.n) {
  console.log('文件名：' + (program.new || program.n))
}
```
通过`mk -h`可以查看帮助文档，-h 和 -V都是commander封装好的命令:
``` bash
$ mk -h

  Usage: index [options]

  Options:

    -V, --version         output the version number
    -n, --new <filename>  创建文件
    -h, --help            output usage information
```

#### on事件监听
此外还可以自定义help文档，通过监听--help触发定义的回调方法。
``` bash
program.on('--help', function () {
    console.log('  自定义的例子:')
    console.log('')
    console.log('    输出命令  mk -n test -k activityList')
    console.log('    输出命令  mk --new test')
    console.log('')
})
.parse(process.argv);
```
`program.parse` 会解析命令行参数以及触发回调方法，因为nodejs的emit会立刻触发事件，所以将该方法放在命令及事件监听的最后面。


#### 参数
运行`mk -n test -k activityList`命令，可以通过program.n和program.k分别获得文件名'test'和模板key值'activityList'。

#### 主要流程
运行`mk -n test -k activityList`，读取`mk.json`文件中定义的模板,根据key值读取此次创建文件的模板来源：
- dir: 创建文件的目标位置
- suffix: 文件后缀
- tpl: 文件模板，模板内容可以自定义，使用的模板语法是[nunjucks](https://mozilla.github.io/nunjucks/)。
``` json
{
    "fileList" : [
        {"dir": "test/js", "suffix": "js", "tpl": "tpl/js.tpl"},
        {"dir": "test/css", "suffix": "css", "tpl": "tpl/css.tpl"}
    ],
    "activityList" : [
        {"dir": "activity/js", "suffix": "js", "tpl": "tpl/js.tpl"}
    ]
}
```

### npm包开发
首先要`npm adduser`来注册个npm账号，如果已经有账号，可以使用`npm login`来登录。
通过`npm version <update_type>`对版本进行管理，更新了版本号后发布：
```
npm publish
```
最终发布的npm包见[mkfile-cli](https://www.npmjs.com/package/mkfile-cli)
