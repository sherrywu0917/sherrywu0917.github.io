---
title: react相关问题
date: 2019-01-29 15:38:28
tags:
---

### 什么是JSX?——浏览器是如何识别它的？
JSX是facebook普及的一种标记语言，通过babel/TSC等工具会编译为`React.createElement`function。所以在React每个组件中，虽然没有显式用到React，但都需要`import React from 'react'`。

#### JSX是如何区分React Component和HTML元素的？
通过元素首字母的大小写，如果首字母大写，则认为是React组件，小写的话则会被认为是HTML元素。可以在[online Babel compiler](https://babeljs.io/repl/#?presets=react&code_lz=GYVwdgxgLglg9mABACwKYBt1wBQEpEDeAUIogE6pQhlIA8AJjAG4B8AEhlogO5xnr0AhLQD0jVgG4iAXyJA)中试一下。
``` js
function hello() {
  return <div>Hello world!</div>;
}

// after babel compiler
function hello() {
  return React.createElement(
    "div",
    null,
    "Hello world!"
  );
}
```

``` js
function hello() {
  return <div>Hello world!</div>;
}

// after babel compiler
function hello() {
  return React.createElement(
    Div,
    null,
    "Hello world!"
  );
}
```
此外，把一个组件赋给`this.component`并且写`<this.component />`也会起作用。


### react16生命周期
#### Mounting
- constructor()
- static getDerivedStateFromProps()
- render()
- componentDidMount

不建议使用`UNSAFE_componentWillMount()`

#### Updating
- static getDerivedStateFromProps()
- shouldComponentUpdate()
- render()
- getSnapshotBeforeUpdate()
- componentDidUpdate()

不建议使用`UNSAFE_componentWillReceiveProps()`和`UNSAFE_componentWillUpdate()`
`UNSAFE_componentWillReceiveProps()`经常会带来bug和不一致的问题。

#### Unmounting
- componentWillUnmount()

`getDerivedStateFromProps(props, state)`每次render之前都会被触发，与`componentWillReceiveProps`只在父组件rerender时会触发不一样。此外，`getDerivedStateFromProps`方法不建议经常使用，使用前想一想是否有替代方案。


### 加分题：数据获取为什么用 componentDidMount 而不是 constructor？
你希望听到的两个原因会是：“在渲染发生之前数据不会存在” —— 虽然不是主要原因，但它向您显示该人员了解组件的处理方式; “在 React Fiber 中使用新的异步渲染……” —— 有人一直在努力学习。

- r1: SSR模式下，componentWillMount在server端也是会被调用的，内容返回到client端后，componentWillMount会被第二次调用，如果在componentWillMount中处理数据获取则会被调用两次。
- r2: 在componentWillMount中调用setState不会触发rerender，所以一般不会被用来获取数据。
- r3: React16之后采用了Fiber架构，类似ComponentWillMount的生命周期钩子都有可能执行多次，所以不在这些生命周期中做有副作用的操作，比如请求数据。
- r4: constructor用来初始化组件，作用应该保持纯粹，不应该引入数据获取这种有副作用的操作。

#### react fiber纤程
fiber是纤程颗粒化的概念，一个线程可以包含多个Fiber，主要是对react更新机制的优化。React16之前的版本，更新组件会一直占用主线程，如果组件树过大，则可能会导致浏览器失去响应。在React16中加入的fiber可以将同步任务拆解，每次执行完一小片后，都会把控制权交还给react负责任务调度的模块，如果有优先级更高的任务，就先执行高优先级的任务。
##### 拆什么
首先，看React的渲染，包括两个阶段：调度阶段(reconciliation)和渲染阶段(commit)。
- 调度阶段react根据数据更新virtual DOM，再运用diff算法找到需要VDOM change。这一部分的工作是可以拆分的。
- 渲染阶段根据计算出的所有diff去一次性更新真实的DOM。
组件比较庞大时，diff运算会比较耗时，不可控，所以需要拆分的是调度阶段。

##### 怎么拆
fiber tree的部分结构如下所示，将简单的数结构，变成了基于单链表的树结构。
``` json
{
    alternate: Fiber|null, //在fiber更新时克隆出的镜像fiber，对fiber的修改会标记在这个fiber上
    nextEffect: Fiber | null, // 单链表结构，方便遍历 Fiber Tree 上有副作用的节点
    pendingWorkPriority: PriorityLevel, // 标记子树上待更新任务的优先级

    stateNode: any, // 管理 instance 自身的特性
    return: Fiber|null, // 指向 Fiber Tree 中的父节点
    child: Fiber|null, // 指向第一个子节点
    sibling: Fiber|null, // 指向兄弟节点
}
```

##### 执行顺序
因为是单链表(A → B → C)的结构，所以在每次执行到某个节点(A → B)被中断后，下次可以从该节点(B → C)接着执行。
requestIdleCallback会让一个低优先级的任务在空闲期被调用，而requestAnimationFrame会让一个高优先级的任务在下一个栈帧被调用，从而保证了主线程按照优先级执行 fiber 单元。
> 优先级顺序为：文本框输入 > 本次调度结束需完成的任务 > 动画过渡 > 交互反馈 > 数据更新 > 不会显示但以防将来会显示的任务。

因为react fiber机制，一个任务很可能执行到一半就被其他优先级更高的任务所替代，或者因为时间原因而被终止。当再次执行这个任务时，是从头开始执行一遍，就会导致组件的某些 will 生命周期可能被多次调用而影响性能。

REFs:
- [[译]以面试官的角度来看React工作面试](https://juejin.im/post/5bca74cfe51d450e9163351b)
- [浅析 React Fiber](https://juejin.im/post/5be969656fb9a049ad76931f)
- [浅谈React 16中的Fiber机制](https://tech.youzan.com/react-fiber/)