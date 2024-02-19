---
title: Themis-CMS体验度量方案
date: 2023-05-20 12:12:12
tags: [Themis，CMS体验度量方案]
---

## 背景
Themis为CMS体验优化项目，目标是提高CMS相关的体验和性能，主要的服务用户是：策划、开发、运营，主要工作包括：
- 脚手架移动化改造和场景组件封装
- 性能监控
- 效能工具的开发
本文主要介绍‘性能监控’的设计方案和实现，为了可以有效地衡量cms的性能数据，我们会从多个角度去度量应用相关的数据，进而分析出页面存在的体验问题并治理。

## 架构图
Themis性能监控处理了从数据生成、收集、计算到可视化的整个链路，流程架构图如下所示，主要包括三方面：
- 场景指标定义
- 数据收集&上报：ThemisLogger SDK
- 监控数据可视化：ThemisAnalysis 平台
<img src="https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/15009464919/17fe/2097/e368/4220300df2199192169a19dae63496da.png" width="560">


## 场景指标定义
首先明确CMS侧主要的场景可以归纳为表格和表单两种，在「场景组件封装」中，我们对这类交互进行了标准化定义，详情可以参考[场景规范](https://music-cms.hz.netease.com/themis/standards/situations/table)。
那么问题的关键就转化为：如何对这两种场景进行度量？
以表格场景为例，在用户操作的时候，我们会重点关注操作的易用性、页面渲染速度和高频行为等，从中我们抽象出表格场景的核心指标有：
- 查询效率
- 页码切换性能
- 场景初始化性能
- 筛选区操作效率
- 用户高频行为

为了度量出具体的核心指标，我们需要去定义具体的行为类型、指标计算规则，记录行为队列。

### 行为类型定义
首先针对关注的表格核心指标，定义行为类型，例如在表格场景中搜索的点击`search`，表格渲染结束`RenderEnd`
``` js
const ActionType = {
    PageChange: 'pageChange', // 页码变化
    Search: 'search', // 搜索点击
    SceneInit: 'sceneInit', // 场景初始化
    RenderEnd: 'renderEnd', // 所有的表格内容渲染完成
    FilterValueChange: 'filterValueChange', // 搜索条件变更
}
```
### 行为计算规则定义
我们关注的表格核心指标有：
- 查询耗时 = 渲染完成时间 - 搜索触发时间（RenderEnd - Search）
- 页码切换耗时 = 渲染完成时间 - 页码切换时间（RenderEnd - PageChange)
- 场景初始化耗时 = 渲染完成时间 - 场景初始化时间（RenderEnd - SceneInit)
- 筛选表单操作时长 = 搜索开始时间 - 表单开始操作时间 (Search - filterValueChange)
- 用户高频行为 = sort([页面]-[表格]-[行为])

#### 计算前置关系
针对已经定义的行为类型，如果要计算【行为1】到【行为2】之间的耗时，则可以将【行为1】放到【行为2】对应的前置行为数组。
``` js
/**
 * 定义需要计算的[行为1]_[行为2]的耗时
 */
window.Themis.PreActionMap = {
    '行为2': [
        '行为1',
        // ...
    ]
}
```
对于【行为2】，可能有多个行为会触发，例如针对页面渲染的场景，【RenderEnd行为】可以是Search、PageChange、SceneInit多个行为触发，这些行为都可以放到【RenderEnd行为】对应的前置行为数组。
``` js
window.Themis.PreActionMap = {
    [ActionType.RenderEnd]: [
        ActionType.PageChange,
        ActionType.Search,
        ActionType.SceneInit,
        ActionType.FilterValueChange,
    ]
}
```

#### 计算顺序
在计算耗时的时候，需要区分两种顺序：
- NEAR: 最近的触发行为去计算，如RenderEnd行为耗时的计算(RenderEnd - 最近一次触发渲染的时间)
- FAR: 最远的触发行为去计算，如`筛选表单操作时长`指标的计算（搜索开始时间 - 表单开始操作时间）
``` js
window.Themis.CountRule = {
    NEAR: [ActionType.RenderEnd],
    FAR: [ActionType.Search]
} 
```

### 行为队列
定义好行为类型和计算规则后，在触发需要打点的行为时，场景组件会将action对象push到`window.Themis.ActionQueue`数组中，action对象结构如下：
```js
 /**
   * @def {object} action 行为对象
   * @def {string} action.name 行为名 必须
   * @def {number} action.time 行为发生时间戳 必须
   * @def {string} action.scene 所属场景 必须
   * @def {string} action.url 所属页面 必须 location.href
   * @def {object} [action.attributes] 其他行为属性，必须是可枚举 可选
   */
 const action = {
    name: ActionType.FilterValueChange,
    time: 1605531890117,
    scene: 'Table' ,
    url: 'http://cms.qa.igame.163.com/pagexxx',
    attributes: {
        attr: 1
    },
 }
```
若用户多次触发行为，则会持续向行为队列中push action对象，该行为队列存储在本地，在满足预设的条件时会被ThemisLogger SDK收集、格式化处理并上报到服务端。

## 数据收集&上报
ThemisLogger SDK负责收集页面相关的性能数据并上报，主要工作包括：
- 场景行为数据收集
- 指标数据计算&上报
- 异常ErrorCode捕获
- 请求耗时上报
- 页面停留时间上报
该SDK可以通过[Puzzle](https://music-fet.hz.netease.com/puzzle/web/projects/1/apps)平台去接入，通过动态js下发的方式去引入SDK，无需入侵项目代码。
### 场景指标上报
#### 场景指标上报时机
可以触发场景指标上报的条件有多个，用于处理不同的场景，包括：
- 定时10s上报一次：最通用的上报触发条件
- 页面离开时触发上报：处理页面被关闭的情况
- 行为队列长度超过100触发上报: 用于处理数据量过大的情况
定时上报、页面离开时触发上报都很好处理，难点在于如何监听行为队列长度的变化，对行为队列的修改内置在场景组件中，与SDK无直接通信，并且不希望在场景组件中耦合上报相关逻辑，保持代码职责的专一性。鉴于行为队列是一个数组，首先封装了`observeArray`方法对`window.Themis.ActionQueue`会用到的数组方法原型进行重写，这样，在对该数组进行`push`、`pop`等操作时，就可以在回调方法中去判断数据长度
```js
/**
 * 监听数组
 * @param {array} arr 
 * @param {function} callback 
 */
export function observeArray(arr, callback) {
  const arrayMethods = Object.create(Array.prototype);
  const newArrProto = [];
  [
    'push',
    'pop',
    'shift',
    'unshift',
    'splice',
    'sort',
    'reverse'
  ].forEach(method => {
    let original = arrayMethods[method];
    newArrProto[method] = function (...args) {
      const result = original.apply(this, args);
      callback && callback()
      return result;
    }
  });
  Object.setPrototypeOf(arr, newArrProto);
}
```
此外，在SDK初始化的时候，`window.Themis.ActionQueue`可能尚未被初始化，针对此情况，定义了`observeObj`方法去监听`window.Themis.ActionQueue`的赋值，内部逻辑基于`Object.defineProperty`的`setter`和`getter`去实现。
``` js
/**
 * 监听对象
 * @param {object} obj 
 * @param {string} key 
 * @param {function} callback 
 */
export function observoObj(obj, key, callback) {
  if (!obj || !key) {
    return;
  }
  let newValue;
  Object.defineProperty(obj, key, {
    set: function(value) {
      newValue = value;
      if(Array.isArray(value)) {
          observeArray(obj[key], callback);
      }
      callback && callback(newValue);
    },
    get: function() {
      return newValue;
    }
  });
}

/**
 * 监听window.Themis.ActionQueue的变化
 */
function observeActionQueue() {
    if (window.Themis.ActionQueue) {
        observeArray(window.Themis.ActionQueue, onActionQueueChange);
    } else {
        observeObj(window.Themis, 'ActionQueue', onActionQueueChange);
    }
}
```
#### 场景指标计算
在满足指标上报条件后，会对行为队列的数据进行计算，根据前一章定义的计算规则将其转换为`Event`数据结构：
``` ts
interface IEvent {
    name: string; // eg: 'cmsAction'｜'errorCode'|'requestTimeConsume'等
    value: number; // 耗时时间戳
    attributes: {
        actionName: string; // eg: `${ActionType.Search}_${ActionType.RenderEnd}`
        ... // 其他属性
    }
}
```
在计算的过程中，需要对行为队列中的数据进行过滤，对于满足计算规则的行为队列上报为`cmsAction`类型，对于异常的行为队列也需要上报，类型为`errorActionQueue`，因为异常的行为队列，很可能反应了页面存在的一些问题。

### 请求数据处理
因为CMS底层调用的`fetch`方法，要捕获请求耗时、异常ErrorCode、返回数据等请求相关数据，可以通过拦截`fetch`方法，置入功能插件去做处理，核心代码如下：
```js
const interceptFetch = () => {
    const fetch = window.fetch;
    window.fetch = (...args) => {
        /**
         * 顺序从下向上执行
         */
        const config = args[1] || {};
        const plugins = [
            errorPlugin(config),
            dataFormatter(config),
            timeConsumePlugin(config),
        ];
        return composePlugin(...plugins)(fetch.apply(window, args));
    };
};
```
设计实现了三个功能插件：
- `timeConsumePlugin`时间耗时插件
- `dataFormatter`数据格式化插件
- `errorPlugin`错误捕获插件

当请求发出的时候，会按顺序逐个执行功能插件，执行的顺序为：`timeConsumePlugin -> dataFormatter -> errorPlugin`，需要注意的关键点有：
- 在`timeConsumePlugin`插件中，需要通过闭包的方式去记录下当前请求的开始时间，在请求返回的时候，拿到返回时间，进而计算出请求耗时时间；
- `dataFormatter`插件对response数据处理时，不能直接调用`response.json()`，因为response只能做一次json处理，为了避免对CMS项目代码逻辑的影响，我们需要首先对response进行clone，再做json格式化处理，即`response.clone().json()`；此外，还需要注意，response返回的格式可能是blob的，不支持json转化，针对blob格式的response，做直接返回处理；
- `errorPlugin`插件负责捕获异常返回的http code，在上报收集的结果中，发现有一些已知的正常code，例如无权限异常code、未登录code，为了屏蔽这些code值对度量的影响，SDK提供了支持配置code白名单的功能；
- 在对请求数据的收集中，需要关注的业务相关的API，所以对于一些公共的API：日志上报、登录相关等接口，需要在上报的时候忽略这部分接口，与error code相似，SDK支持配置API白名单的功能。

## 监控数据可视化
在收集到SDK上报的数据后，为了更好地衡量CMS平台的性能，[ThemisAnalysis](https://music-cms.hz.netease.com/themis-analysis/chart/home)监控数据可视化平台对数据从不同维度做了数据聚合处理，包括指标维度、场景维度、请求维度、错误维度、页面维度等，以及抽象出CMS平台的健康模型；此外支持错误日志查询，在遇到问题的时候可以方便地查询出对应的请求数据。

### 健康模型
健康模型综合了场景指标、请求成功率和请求耗时数据，计算出健康指数的具体得分，给开发一个比较直观的关于平台健康度的反馈。
![](https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/15045900125/9dab/d7ea/7b93/b408a64ba63857093e01bce54c22c67f.png)
健康指数总分由场景指标性能得分、场景指标规范性、请求成功率得分、请求平均响应时间得分按照一定的占比计算得出，具体计算公式为：
> 健康指数总分 = 场景指标性能得分 * x% + 场景指标规范性 * y% + 请求成功率得分 * z% + 请求平均响应时间得分 * m%

各项指标具体的得分，主要是与各项指标推荐的基准值，根据我们预设的公式去计算，如果有明显不合理的地方，再去做动态调整，保证分数的合理性。
以场景指标性能得分为例，简单介绍下计算方法：
> 场景指标性能得分 = (指标 1 得分 + 指标 2 得分 + ...) / 指标数量

场景指标性能得分为所有场景指标的平均分，综合考虑了该应用相关的所有指标。其中要计算单个指标得分，首先计算出`指标耗时/指标基准值`的比值，再根据这个比值按照梯度线性计算而得。例如当该`比值<=1`时，很明显得分应该为`100`；比值越大，说明耗时越长，页面渲染耗时越长，反映在得分上越低。

### 问题分析
ThemisAnalysis平台会根据错误频率统计请求的高频错误，点击查看可以进一步查看错误详情，包括异常API地址、请求参数以及被影响的用户：
<img src="https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/15048205515/b8f9/ef54/d53a/b9b13d5ee84433c75cf0f83d8d101956.png" width="560px">

确认了存在的异常后，通常情况下期望可以看到用户操作的上下文，所以会将SDK上报的请求相关数据汇总，支持查询操作；通过uid可以查询到具体的某个用户的操作链路，最大程度地帮助开发复现并解决问题。
除此之外，Themis项目-效能工具的开发中包括`问题上报插件`，目前音乐的CMS都可以使用该插件，推荐使用，入口如下：
![](https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/15048248406/b7b1/d59f/6e2c/bef24fb46cccc6b56159e02ad82a8d6f.png)
结合该插件，可以帮助我们还原问题现场，相关原理可以参考我发表的另一篇文章[rrweb 带你还原问题现场](https://juejin.cn/post/7049909933168394277/)。

## 总结
本文主要介绍了CMS性能监控的方案，简要介绍了数据格式的定义、监控SDK对数据的收集以及可视化平台的实现，希望这篇文章可以在相似场景中带给你一些小小的灵感。
