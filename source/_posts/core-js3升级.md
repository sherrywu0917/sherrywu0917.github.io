---
title: core-js3升级处理
date: 2021-10-19 11:28:58
tags: [eventloop, 渲染]
---

### cms升级@babel/polyfill
1. 删掉@babel/polyfill
2. 升级core-js@2 到core-js@3
3. 配置babel：在`@babel/preset-env`的配置中，新增corejs配置，指定版本
``` js
 presets: [
    ['@babel/preset-env', {
        modules: false,
        useBuiltIns: 'usage',
        targets: { // 根据实际情况配置
            browsers: ['chrome >= 62']
        },
        corejs: {
            version: 3, // 使用core-js@3
            proposals: true,
        },
    }],
]
```
若还不行，检查下版本：babel-loader 版本升级到 8.0.0 以上，@babel/core 版本升级到 7.4.0 及以上。

参考：
[Babel7 转码（五）- corejs3 的更新](https://segmentfault.com/a/1190000020237817)
[作者的官方阐述](https://github.com/zloirock/core-js/blob/master/docs/2019-03-19-core-js-3-babel-and-a-look-into-the-future.md)
[Babel 7.4.0 版本的更新内容，及官方的升级建议](https://babeljs.io/blog/2019/03/19/7.4.0)
[core-js@2 向core-js@3 升级，官方的 Pull request](https://github.com/babel/babel/pull/7646)
