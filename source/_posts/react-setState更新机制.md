---
title: react-setState更新机制
date: 2019-04-02 10:08:50
tags: [react, setState]
---
## setState更新机制
如果当前处于更新中，新的setState更新就会被push到dirtyComponent中，所以setState是异步的。
### V15版本为例
**step1: setState**
ReactBaseClassses.js
``` js
ReactComponent.prototype.setState = function (partialState, callback) {
  //  将setState事务放进队列中
  this.updater.enqueueSetState(this, partialState);
  if (callback) {
    this.updater.enqueueCallback(this, callback, 'setState');
  }
};
```
调用enqueueSetState将要更新的部分状态值partialState放入队列中，再将callback放到callback队列中。
<!-- more -->

**step2: enqueueSetState**
``` js
enqueueSetState: function (publicInstance, partialState) {
    // 获取当前组件的instance，Q: 这边为啥有一个'setState'
    var internalInstance = getInternalInstanceReadyForUpdate(publicInstance, 'setState');

    // 将要更新的state放入一个数组里
    var queue = internalInstance._pendingStateQueue || (internalInstance._pendingStateQueue = []);
    queue.push(partialState);

    //  将要更新的component instance也放在一个队列里
    enqueueUpdate(internalInstance);
}
```
partialState被放到一个数组中去维护，这个方法会获取当前组件的实例，即需要更新的组件，也放入到队列中。

**step3: enqueueUpdate**
ReactUpdates.js
``` js
function enqueueUpdate(component) {
  // 如果没有处于批量创建/更新组件的阶段，则处理update state事务
  if (!batchingStrategy.isBatchingUpdates) {
    batchingStrategy.batchedUpdates(enqueueUpdate, component);
    return;
  }
  // 如果正处于批量创建/更新组件的过程，将当前的组件放在dirtyComponents数组中
  dirtyComponents.push(component);
}
```
如果isBatchingUpdates标志位是true，说明当前正处于组件更新的阶段，则将当前组件实例push到dirtyComponents数组中；
反之，立刻触发更新。

**step4: batchingStrategy**
ReactDefaultBatchingStrategy.js
``` js
var ReactDefaultBatchingStrategy = {
  // 用于标记当前是否处于批量更新
  isBatchingUpdates: false,
  // 当调用这个方法时，正式开始批量更新
  batchedUpdates: function (callback, a, b, c, d, e) {
    var alreadyBatchingUpdates = ReactDefaultBatchingStrategy.isBatchingUpdates;

    ReactDefaultBatchingStrategy.isBatchingUpdates = true;

    // 如果当前事务正在更新过程中，则调用callback，既enqueueUpdate
    if (alreadyBatchingUpdates) {
      return callback(a, b, c, d, e);
    } else {
    // 否则执行更新事务
      return transaction.perform(callback, null, a, b, c, d, e);
    }
  }
};
```
调用batchedUpdates方法去批量更新的时候，还是会先去判断当前事务是否处于更新过程中，如果是，则调用传入的enqueueUpdate方法，将组件实例push到dirtyComponents中。反之，去执行批量更新。

**step5: transaction**
```
/**
 * <pre>
 *                       wrappers (injected at creation time)
 *                                      +        +
 *                                      |        |
 *                    +-----------------|--------|--------------+
 *                    |                 v        |              |
 *                    |      +---------------+   |              |
 *                    |   +--|    wrapper1   |---|----+         |
 *                    |   |  +---------------+   v    |         |
 *                    |   |          +-------------+  |         |
 *                    |   |     +----|   wrapper2  |--------+   |
 *                    |   |     |    +-------------+  |     |   |
 *                    |   |     |                     |     |   |
 *                    |   v     v                     v     v   | wrapper
 *                    | +---+ +---+   +---------+   +---+ +---+ | invariants
 * perform(anyMethod) | |   | |   |   |         |   |   | |   | | maintained
 * +----------------->|-|---|-|---|-->|anyMethod|---|---|-|---|-|-------->
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | +---+ +---+   +---------+   +---+ +---+ |
 *                    |  initialize                    close    |
 *                    +-----------------------------------------+
 * </pre>
 */ 
```
transaction对象暴露了一个perform方法，传入的anyMethod，都会被wrapper包裹，每次先执行完所有wrapper的initialize方法，再去执行anyMethod，最后去执行所有wrapper的close方法。
在ReactDefaultBatchingStrategy.js,tranction 的 wrapper有两个 FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES。
``` js
var RESET_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: function () {
    ReactDefaultBatchingStrategy.isBatchingUpdates = false;
  }
};

var FLUSH_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates)
};

var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];
```
两个wrapper的initialize方法都是空的，在callback之后，FLUSH_BATCHED_UPDATES会循环所有dirtyComponent(批量更新过程中push进来的所有组件),调用updateComponent来执行所有的生命周期方法：componentWillReceiveProps，componentShouldUpdate，componentWillUpdate，render，componentDidUpdate，完成更新，并且并执行它的pendingCallbacks，即setState设置的callback；RESET_BATCHED_UPDATES会将isBatchingUpdates标志位置为false，FLUSH_BATCHED_UPDATES 是执行flushBatchedUpdates，。

REFS: https://imweb.io/topic/5b189d04d4c96b9b1b4c4ed6

### react合成事件中的setState
react的合成事件还是会通过 ReactUpdates.batchedUpdates 去批量更新，ReactUpdate中的batchedUpdates，还是调用了batchingStrategy的batchedUpdates方法。如果在事件中调用setState方法，也会进入dirtyComponent流程，即所谓的异步。
``` js
function batchedUpdates(callback, a, b, c, d, e) {
  ensureInjected();
  return batchingStrategy.batchedUpdates(callback, a, b, c, d, e);
}
```

### 原生事件绑定和setTimeout中setState
原生事件绑定不会通过合成事件的方式处理，自然也不会进入更新事务的处理流程。setTimeout也一样，在setTimeout回调执行时已经完成了原更新组件流程，不会放入dirtyComponent进行异步更新，其结果自然是同步的。

**在组件生命周期中或者react事件绑定中，setState是通过异步更新的。在延时的回调或者原生事件绑定的回调中调用setState不一定是异步的。**


### setState为什么设计为异步？
#### 1.保证内部的一致性
假设state是同步更新的，下面的代码可以按预期工作。
``` js
console.log(this.state.value) // 0
this.setState({ value: this.state.value + 1 });
console.log(this.state.value) // 1
this.setState({ value: this.state.value + 1 });
console.log(this.state.value) // 2
```
然而，这时你需要将状态提升到父组件`this.props.onIncrement()`，以供多个兄弟组件共享。再重复上述代码：
``` js
console.log(this.props.value) // 0
this.props.onIncrement();
console.log(this.props.value) // 0
this.props.onIncrement();
console.log(this.props.value) // 0
```
输出和预期不一致，state虽然立即更新了，但没有重新渲染父组件的时候，this.props属性并没有立即更新。如果要立即更新this.props，则要放弃批处理(根据情况的不同，性能可能会有显著的下降)。

#### 2.性能优化
React 会依据不同的调用源，给不同的 setState() 调用分配不同的优先级。调用源包括事件处理、网络请求、动画等。
假设你在一个聊天窗口，你正在输入消息，TextBox 组件中的 setState() 调用需要被立即应用。然而，在你输入过程中又收到了一条新消息。更好的处理方式或许是延迟渲染新的 MessageBubble 组件，从而让你的输入更加顺畅，而不是立即渲染新的 MessageBubble 组件阻塞线程，导致你输入抖动和延迟。

#### 3.更多的可能性
异步更新 state 是有可能实现下面这种设想的前提。
假设你从一个页面导航到到另一个页面，你只需要简单的调用 setState() 去渲染一个新的页面，React “在幕后”开始渲染这个新的页面。想象一下，不需要你写任何的协调代码，如果这个更新花了比较长的时间，你可以展示一个加载动画，否则在新页面准备好后，让 React 执行一个无缝的切换。此外，在等待过程中，旧的页面依然可以交互，但是如果花费的时间比较长，你必须展示一个加载动画。

### react状态管理
- 区分local state、global state、global store
- props实现状态共享
- context API实现全局状态，例如userId、theme、language等
- render function作为props的方式
- react children方式
- react高阶组件实现状态共享
- redux、mobx状态管理
