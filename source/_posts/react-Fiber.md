---
title: react fiber纤程
date: 2020-07-02 12:12:12
tags: [react, diff]
---
## react v16的diff算法
> React Fiber 是对React核心算法的重新实现，是React团队花费了2年的研究成果。
> 核心特性**增量式渲染**：the ability to split rendering work into chunks and spread it out over multiple frames.
### 0. react element结构
React 元素（elements）是设计好的 plain object：
``` js
{
  $$typeof:  Symbol(react.element), // React会检测该属性，避免xss攻击 
  key: null,
  props: {},
  ref: null,
  type: String | Object | Function, // 元素类型，string，function
  _owner: FiberNode, // 所属React Component
  _store: { validated: false }
}
```
为了避免用户模拟JSON注入xss攻击，React 0.14 修复手段是用 Symbol 标记每个 React 元素（element）：因为服务器返回的序列化JSON不支持 Symbol 类型（**JSON.stringify后，symbol属性和方法会被抛弃**）。所以即使服务器存在用JSON作为文本返回安全漏洞，JSON 里也不包含 Symbol.for('react.element')。React 会检测 element.$$typeof，如果元素丢失或者无效，会拒绝处理该元素。
特意用 Symbol.for() 的好处是 Symbols 通用于 iframes 和 workers 等环境中。
一个很有趣的点：如果浏览器不支持 Symbols 怎么办？React仍然会加上 $$typeof 字段以保证一致性，但只是设置一个数字而已 —— 0xeac7。为什么？因为 0xeac7 看起来有点像 「React」

### 1. react fiber纤程
React Fiber 是一种基于浏览器的**单线程调度算法**。
- 一种将 recocilation （递归 diff），拆分成无数个小任务的算法；
- 它随时能够停止，恢复。停止恢复的时机取决于当前的一帧（16ms）内，还有没有足够的时间允许计算。
- 划分任务的优先级
fiber是纤程颗粒化的概念，一个线程可以包含多个Fiber，主要是对react更新机制的优化。React16之前的版本，更新组件会一直占用主线程，如果组件树过大，则可能会导致浏览器失去响应。在React16中加入的fiber可以将同步任务拆解，每次执行完一小片后，都会把控制权交还给react负责任务调度的模块，如果有优先级更高的任务，就先执行高优先级的任务。

#### 1.1 解决问题
JavaScript 是单线程运行的，，同时只能做一件事情，这个和 DOS 的单任务操作系统一样的，事情只能一件一件的干。要是前面有一个傻叉任务长期霸占CPU，后面什么事情都干不了，浏览器会呈现卡死的状态，这样的用户体验就会非常差。
##### 对于’前端框架‘来说，解决这种问题有三个方向:
1️⃣ 优化每个任务，让它有多快就多快。挤压CPU运算量 （Vue）
2️⃣ 快速响应用户，让用户觉得够快，不能阻塞用户的交互 (React FiberNode)
3️⃣ 尝试 Worker 多线程
React 会递归比对VirtualDOM树，找出需要变动的节点，然后同步更新它们, 一气呵成。这个过程 React 称为 Reconcilation(协调)。在 Reconcilation 期间，React 会霸占着浏览器资源，一则会导致用户触发的事件得不到响应, 二则会导致掉帧，用户可以感知到这些卡顿。
协调是CPU密集型的操作，浏览器的渲染树构建、布局、绘制、资源加载(例如HTML解析)、事件响应、脚本执行都可以看作是进程，我们需要通过某些调度策略合理地分配CPU资源，从而提高浏览器的用户响应速率, 同时兼顾任务执行效率。
**所以 React 通过Fiber 架构，让自己的Reconcilation 过程变成可被中断。 '适时'地让出CPU执行权，除了可以让浏览器及时地响应用户的交互，还有其他好处:**
- 与其一次性操作大量 DOM 节点相比, 分批延时对DOM进行操作，可以得到更好的用户体验。
- 给浏览器一点喘息的机会，他会对代码进行编译优化（JIT）及进行热代码优化，或者对reflow进行修正

#### 1.2 requestIdleCallback
requestIdleCallback：让浏览器在'有空'的时候就执行我们的回调，这个回调会传入一个期限，表示浏览器有多少时间供我们执行, 为了不耽误事，我们最好在这个时间范围内执行完毕。
浏览器在一帧内可能会做执行下列任务，而且它们的执行顺序基本是固定的:
- 处理用户输入事件
- Javascript执行
- requestAnimationFrame 调用 (会在下一次渲染之前调用)
- render Tree构建
- 布局 Layout
- 绘制 Paint
理想的一帧时间是 16ms (1000ms / 60)，如果浏览器处理完上述的任务(布局和绘制之后)，还有盈余时间，浏览器就会调用 requestIdleCallback 的回调。
但是在浏览器繁忙的时候，可能不会有盈余时间，这时候requestIdleCallback回调可能就不会被执行。 为了避免饿死，可以通过requestIdleCallback的第二个参数指定一个超时时间。
**所以setState触发的diff会在浏览器空闲的时候执行，即setState是批量触发的机制**

##### 任务优先级
- Immediate(-1) - 这个优先级的任务会同步执行, 或者说要马上执行且不能中断
- UserBlocking(250ms) 这些任务一般是用户交互的结果, 需要即时得到反馈
- Normal (5s) 应对哪些不需要立即感受到的任务，例如网络请求
- Low (10s) 这些任务可以放后，但是最终应该得到执行. 例如分析通知
- Idle (没有超时时间) 一些没有必要做的任务 (e.g. 比如隐藏的内容), 可能会被饿死

#### 1.3 一个执行单元
**将它视作一个执行单元，每次执行完一个'执行单元', React 就会检查现在还剩多少时间，如果没有时间就将控制权让出去.**
执行过程是：
1. 假设用户调用 setState 更新组件, 这个待更新的任务会先放入队列中, 然后通过 requestIdleCallback 请求浏览器调度：   
``` js
// setState的时候可以将要更新的component instance放到一个队列里，即Fiber节点，Fiber执行单元里面也有state变更相关信息
updateQueue.push(updateTask);
requestIdleCallback(performWork, {timeout});
```
2. 现在浏览器有空闲或者超时了就会调用performWork来执行任务：
``` js
// 1️⃣ performWork 会拿到一个Deadline，表示剩余时间
function performWork(deadline) {

  // 2️⃣ 循环取出updateQueue中的任务(这边应该是以Fiber为单位)
  while (updateQueue.length > 0 && deadline.timeRemaining() > ENOUGH_TIME) {
    workLoop(deadline);
  }

  // 3️⃣ 如果在本次执行中，未能将所有任务执行完毕，那就再请求浏览器调度
  if (updateQueue.length > 0) {
    requestIdleCallback(performWork);
  }
}
```
workLoop 的工作大概猜到了，它会从更新队列(updateQueue)中弹出更新任务来执行，每执行完一个‘执行单元‘，就检查一下剩余时间是否充足，如果充足就进行执行下一个执行单元，反之则停止执行，保存现场，等下一次有执行权时恢复。
``` js
// 保存当前的处理现场
let nextUnitOfWork: Fiber | undefined // 保存下一个需要处理的工作单元
let topWork: Fiber | undefined        // 保存第一个工作单元

function workLoop(deadline: IdleDeadline) {
  // updateQueue中获取下一个或者恢复上一次中断的执行单元
  if (nextUnitOfWork == null) {
    nextUnitOfWork = topWork = getNextUnitOfWork();
  }

  // 🔴 每执行完一个Fiber执行单元，检查一次剩余时间
  // 如果被中断，下一次执行还是从 nextUnitOfWork 开始处理
  while (nextUnitOfWork && deadline.timeRemaining() > ENOUGH_TIME) {
    // 下文我们再看performUnitOfWork，DFS遍历：子节点->兄弟节点->父节点 
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork, topWork);
  }

  // 提交工作，下文会介绍
  if (pendingCommit) {
    commitAllWork(pendingCommit);
  }
}
```

#### 1.4 Fiber节点
每个 VirtualDOM 节点内部现在使用 Fiber表示，Fiber的部分结构如下所示，**将简单的树结构，变成了基于单链表的树结构**。
``` json
{
    alternate: FiberNode|null, // 替身：指向旧树中的节点
    effectTag: SideEffectTag, // 标识当前节点的副作用类型，例如节点更新、删除、移动
    nextEffect: FiberNode | null, // 和节点关系一样，React 同样使用链表来将所有有副作用的Fiber连接起来
    pendingWorkPriority: PriorityLevel, // 一个数字，标记子树上待更新任务的优先级

    type: any, // Fiber 类型信息
    stateNode: any, // 管理 instance 自身的特性，function组件值为null
    return: FiberNode|null, // 指向 Fiber Tree 中的父节点
    child: FiberNode|null, // 指向第一个子节点
    sibling: FiberNode|null, // 指向兄弟节点

    pendingProps: {}, // pendingProps在执行开始时设置，并在结束时设置memoizedProps。
    memoizedProps: {}, // 当输入的pendingProps等于memoizedProps时，它表示fiber的先前输出可以重复使用，从而防止不必要的工作。
    memoizedState: {}, // 组件实例的 state
}
```

#### 1.5 执行顺序
来看看 performUnitOfWork 的实现, 它其实就是一个深度优先的遍历：
``` js
/**
 * @params fiber 当前需要处理的节点
 * @params topWork 本次更新的根节点
 */
function performUnitOfWork(fiber: Fiber, topWork: Fiber) {
  // 对该节点进行处理
  beginWork(fiber);

  // 如果存在子节点，那么下一个待处理的就是子节点
  if (fiber.child) {
    return fiber.child;
  }

  // 没有子节点了，上溯查找兄弟节点
  let temp = fiber;
  while (temp) {
    completeWork(temp);

    // 到顶层节点了, 退出
    if (temp === topWork) {
      break
    }

    // 找到，下一个要处理的就是兄弟节点
    if (temp.sibling) {
      return temp.sibling;
    }

    // 没有, 继续上溯
    temp = temp.return;
  }
}
```
因为是单链表(A → B → C)的结构，所以在每次执行到某个节点(A → B)被中断后，下次可以从该节点(B → C)接着执行。
requestIdleCallback会让一个低优先级的任务在空闲期被调用，而requestAnimationFrame会让一个高优先级的任务在下一个栈帧被调用，从而保证了主线程按照优先级执行 fiber 单元。
> 优先级顺序为：文本框输入 > 本次调度结束需完成的任务 > 动画过渡 > 交互反馈 > 数据更新 > 不会显示但以防将来会显示的任务。

因为react fiber机制，一个任务很可能执行到一半就被其他优先级更高的任务所替代，或者因为时间原因而被终止。当再次执行这个任务时，是从头开始执行一遍，就会导致组件的某些 will 生命周期可能被多次调用而影响性能。
``` js
function beginWork(fiber: Fiber): Fiber | undefined {
  if (fiber.tag === WorkTag.HostComponent) {
    // 宿主节点diff
    diffHostComponent(fiber)
  } else if (fiber.tag === WorkTag.ClassComponent) {
    // 类组件节点diff
    diffClassComponent(fiber)
  } else if (fiber.tag === WorkTag.FunctionComponent) {
    // 函数组件节点diff
    diffFunctionalComponent(fiber)
  } else {
    // ... 其他类型节点，省略
  }
}
```
#### 1.6 副作用的收集和提交
将所有打了 Effect 标记的节点串联起来，收集完成后再提交。
在`completeUnitOfWork ——> completeWork`方法中遍历fiber节点，将标记了EffectTag的节点通过nextEffect串联起来

#### 1.7 两个阶段的拆分
首先，看React的渲染，包括两个阶段：协调阶段(reconciliation)和提交(渲染)阶段(commit)。
1. 调度阶段react根据数据更新virtual DOM，再运用diff算法找到需要VDOM change。这一部分的工作是可以拆分的。
- constructor
- componentWillMount 废弃
- componentWillReceiveProps 废弃
- static getDerivedStateFromProps
- shouldComponentUpdate
- componentWillUpdate 废弃
- render
2. 提交阶段根据计算出的所有diff去一次性更新真实的DOM。
- getSnapshotBeforeUpdate() 严格来说，这个是在进入 commit 阶段前调用
- componentDidMount
- componentDidUpdate
- componentWillUnmount
也就是说，在协调阶段如果时间片用完，React就会选择让出控制权。因为协调阶段执行的工作不会导致任何用户可见的变更，所以在这个阶段让出控制权不会有什么问题。
需要注意的是：因为协调阶段可能被中断、恢复，甚至重做，⚠️React 协调阶段的生命周期钩子可能会被调用多次!, 例如 componentWillMount 可能会被调用两次。
因此建议 协调阶段的生命周期钩子不要包含副作用. 索性 React 就废弃了这部分可能包含副作用的生命周期方法，例如componentWillMount、componentWillUpdate. v17后我们就不能再用它们了, 所以现有的应用应该尽快迁移.

Reconciliation完成后，会有一个旧树和一个WIP树，对于需要变更的节点，都打上了'标签'。在提交阶段，React 就会将这些打上标签的节点应用变更。

#### 1.8 双缓冲技术
WIP树就是一个缓冲，它在Reconciliation 完毕后一次性提交给浏览器进行渲染。它可以减少内存分配和垃圾回收，WIP 的节点不完全是新的，比如某颗子树不需要变动，React会克隆复用旧树中的子树。另外一个重要的场景就是异常的处理。
**你可以将 WIP 树想象成从旧树中 Fork 出来的功能分支，你在这新分支中添加或移除特性，即使是操作失误也不会影响旧的分支。当你这个分支经过了测试和完善，就可以合并到旧分支，将其替换掉. 这或许就是’提交(commit)阶段‘的提交一词的来源吧？**

### 中断和恢复
**到目前为止：⚠️更新任务还是串行执行的，我们只是将整个过程碎片化了. 对于那些需要优先处理的更新任务还是会被阻塞**
**但React Fiber已经实现了分片执行**
**实际情况是，在 React 得到控制权后，应该优先处理高优先级的任务。**也就是说中断时正在处理的任务，在恢复时会让位给高优先级任务，原本中断的任务可能会被放弃或者重做。 =》 下一步是**启用 Concurrent 模式（实验版本）**: 开启Concurrent 模式后，之前 deprecated 的生命周期方法就彻底不能用了。
**Concurrent 模式通过使渲染可中断来修复此基本限制。**
(React Fiber 实际上非常复杂，不管执行的过程怎样拆分、以什么顺序执行，最重要的是保证状态的一致性和视图的一致性，这给了 React 团队很大的考验，以致于现在都没有正式release出来。?)

#### Concurrent Mode
- 快速响应用户操作和输入，提升用户交互体验
- 让动画更加流畅，通过调度，可以让应用保持高帧率
- 利用好I/O 操作空闲期或者CPU空闲期，进行一些预渲染。比如离屏(offscreen)不可见的内容，优先级最低，可以让 React 等到CPU空闲时才去渲染这部分内容。这和浏览器的preload等预加载技术差不多。?
- 用Suspense 降低加载状态(load state)的优先级，减少闪屏。 比如数据很快返回时，可以不必显示加载状态，而是直接显示出来，避免闪屏；如果超时没有返回才显式加载状态。?

REFs:
- [这可能是最通俗的 React Fiber(时间分片) 打开方式](https://juejin.im/post/5dadc6045188255a270a0f85#heading-5)
- [[译]以面试官的角度来看React工作面试](https://juejin.im/post/5bca74cfe51d450e9163351b)
- [浅析 React Fiber](https://juejin.im/post/5be969656fb9a049ad76931f)
- [浅谈React 16中的Fiber机制](https://tech.youzan.com/react-fiber/)
- [React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture)
- [从 0 实现 React 系列(一)：React 的架构设计](https://my.oschina.net/u/4088983/blog/4545968)