---
title: react-setState更新机制
date: 2019-04-02 10:08:50
tags: [react, setState]
---
## setState更新机制
### V16版本为例
**setState**
``` js
Component.prototype.setState = function (partialState, callback) {
  !(typeof partialState === 'object' || typeof partialState === 'function' || partialState == null) ? invariant(false, 'setState(...): takes an object of state variables to update or a function which returns an object of state variables.') : void 0;
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```
****
``` js
var classComponentUpdater = {
  isMounted: isMounted,
  enqueueSetState: function (inst, payload, callback) {
    var fiber = get(inst);
  // However, if two updates are scheduled within the same event, we
  // should treat their start times as simultaneous, even if the actual clock
  // time has advanced between the first and second call.

  // In other words, because expiration times determine how updates are batched,
  // we want all updates of like priority that occur within the same event to
  // receive the same expiration time. Otherwise we get tearing.
   // expirationTime时间决定了如何批量更新，期望是如果两个update发生在同一个event内，他们的过期时间一致
    var currentTime = requestCurrentTime();
    var expirationTime = computeExpirationForFiber(currentTime, fiber);
    // 返回一个Update节点
    // {
    //   expirationTime: expirationTime,

    //   tag: UpdateState,
    //   payload: null,
    //   callback: null,

    //   next: null,
    //   nextEffect: null
    // }
    var update = createUpdate(expirationTime);
    // payload就是将要更新的particalState
    update.payload = payload;
    if (callback !== undefined && callback !== null) {
      {
        warnOnInvalidCallback$1(callback, 'setState');
      }
      update.callback = callback;
    }

    flushPassiveEffects();
    // 将当前要更新的通过链表的形式放到fiber.updateQueue更新队列中
    enqueueUpdate(fiber, update);
    // 在expirationTime时间内去更新，调用requestWork函数。
    scheduleWork(fiber, expirationTime);
  },
  enqueueReplaceState: function (inst, payload, callback) {
    
  },
  enqueueForceUpdate: function (inst, callback) {
    
  }
}
```
**performWork**
``` js
// requestWork is called by the scheduler whenever a root receives an update.
// It's up to the renderer to call renderRoot at some point in the future.
function requestWork(root, expirationTime) {
  addRootToSchedule(root, expirationTime);
  if (isRendering) {
    // Prevent reentrancy. Remaining work will be scheduled at the end of
    // the currently rendering batch.
    return;
  }

  if (isBatchingUpdates) {
    // Flush work at the end of the batch.
    if (isUnbatchingUpdates) {
      // ...unless we're inside unbatchedUpdates, in which case we should
      // flush it now.
      nextFlushedRoot = root;
      nextFlushedExpirationTime = Sync;
      performWorkOnRoot(root, Sync, false);
    }
    return;
  }

  // TODO: Get rid of Sync and use current time?
  if (expirationTime === Sync) {
    performSyncWork();
  } else {
    scheduleCallbackWithExpirationTime(root, expirationTime);
  }
}
```
如果这次的setState并不是由合成事件触发的，那么isBatchingUpdates将会为false。如果为false就会直接执行performSyncWork函数了，马上对这次setState进行diff和渲染了。
如果当前处在批量更新或者rendering中，直接return，不重复调用scheduleCallbackWithExpirationTime函数；会在workLoop中去检查是否还有更新任务，更新队列(updateQueue)中弹出更新任务来执行，每执行完一个‘执行单元‘，就检查一下剩余时间是否充足，如果充足就进行执行下一个执行单元，反之则停止执行，保存现场，等下一次有执行权时恢复。
``` js
function scheduleCallbackWithExpirationTime(root, expirationTime) {
  if (callbackExpirationTime !== NoWork) {
    // A callback is already scheduled. Check its expiration time (timeout).
    if (expirationTime < callbackExpirationTime) {
      // Existing callback has sufficient timeout. Exit.
      return;
    } else {
      if (callbackID !== null) {
        // Existing callback has insufficient timeout. Cancel and schedule a
        // new one.
        scheduler.unstable_cancelCallback(callbackID);
      }
    }
    // The request callback timer is already running. Don't start a new one.
  } else {
    startRequestCallbackTimer();
  }

  callbackExpirationTime = expirationTime;
  var currentMs = scheduler.unstable_now() - originalStartTimeMs;
  var expirationTimeMs = expirationTimeToMs(expirationTime);
  var timeout = expirationTimeMs - currentMs;
  callbackID = scheduler.unstable_scheduleCallback(performAsyncWork, { timeout: timeout });
}
```
performAsyncWork -> performWork -> performWorkOnRoot -> renderRoot -> workLoop
``` js
function workLoop(isYieldy) {
  if (!isYieldy) {
    // Flush work without yielding
    while (nextUnitOfWork !== null) {
      // 如何获取下一个需要更新的Fiber节点？
      // 猜测：通过dfs去遍历？ 先child -> return，判断是否需要更新？
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  } else {
    // Flush asynchronous work until there's a higher priority event
    while (nextUnitOfWork !== null && !shouldYieldToRenderer()) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  }
}
```
performUnitOfWork -> updateClassComponent等diff和更新组件 -> mountClassInstance|updateClassInstance|resumeMountClassInstance -> processUpdateQueue去遍历fiber.updateQueue，更新payload


#### 如何获取nextUnitOfWork
step1: dfs先去获取需要更新的子节点child
``` js
function bailoutOnAlreadyFinishedWork(current$$1, workInProgress, renderExpirationTime) {
  cancelWorkTimer(workInProgress);

  if (current$$1 !== null) {
    // Reuse previous context list
    workInProgress.contextDependencies = current$$1.contextDependencies;
  }

  if (enableProfilerTimer) {
    // Don't update "base" render times for bailouts.
    stopProfilerTimerIfRunning(workInProgress);
  }

  // Check if the children have any pending work.
  // 通过childExpirationTime时间去检查当前Fiber节点的children是否有更新任务
  // 如果children需要更新，返回child
  var childExpirationTime = workInProgress.childExpirationTime;
  if (childExpirationTime < renderExpirationTime) {
    // The children don't have any work either. We can skip them.
    // TODO: Once we add back resuming, we should check if the children are
    // a work-in-progress set. If so, we need to transfer their effects.
    return null;
  } else {
    // This fiber doesn't have work, but its subtree does. Clone the child
    // fibers and continue.
    cloneChildFibers(current$$1, workInProgress);
    return workInProgress.child;
  }
}
```
step2: dfs先访问兄弟节点siblings，遍历完siblings，再返回父节点return
```js
function completeUnitOfWork(workInProgress) {
  // Attempt to complete the current unit of work, then move to the
  // next sibling. If there are no more siblings, return to the
  // parent fiber.
  while (true) {
    // The current, flushed, state of this fiber is the alternate.
    // Ideally nothing should rely on this, but relying on it here
    // means that we don't need an additional field on the work in
    // progress.
    var current$$1 = workInProgress.alternate;
    {
      setCurrentFiber(workInProgress);
    }

    var returnFiber = workInProgress.return;
    var siblingFiber = workInProgress.sibling;

    if ((workInProgress.effectTag & Incomplete) === NoEffect) {
      if (true && replayFailedUnitOfWorkWithInvokeGuardedCallback) {
        // Don't replay if it fails during completion phase.
        mayReplayFailedUnitOfWork = false;
      }
      // This fiber completed.
      // Remember we're completing this unit so we can find a boundary if it fails.
      nextUnitOfWork = workInProgress;
      if (enableProfilerTimer) {
        if (workInProgress.mode & ProfileMode) {
          startProfilerTimer(workInProgress);
        }
        nextUnitOfWork = completeWork(current$$1, workInProgress, nextRenderExpirationTime);
        if (workInProgress.mode & ProfileMode) {
          // Update render duration assuming we didn't error.
          stopProfilerTimerIfRunningAndRecordDelta(workInProgress, false);
        }
      } else {
        nextUnitOfWork = completeWork(current$$1, workInProgress, nextRenderExpirationTime);
      }
      if (true && replayFailedUnitOfWorkWithInvokeGuardedCallback) {
        // We're out of completion phase so replaying is fine now.
        mayReplayFailedUnitOfWork = true;
      }
      stopWorkTimer(workInProgress);
      resetChildExpirationTime(workInProgress, nextRenderExpirationTime);
      {
        resetCurrentFiber();
      }

      if (nextUnitOfWork !== null) {
        // Completing this fiber spawned new work. Work on that next.
        return nextUnitOfWork;
      }

      if (returnFiber !== null &&
      // Do not append effects to parents if a sibling failed to complete
      (returnFiber.effectTag & Incomplete) === NoEffect) {
        // Append all the effects of the subtree and this fiber onto the effect
        // list of the parent. The completion order of the children affects the
        // side-effect order.
        if (returnFiber.firstEffect === null) {
          returnFiber.firstEffect = workInProgress.firstEffect;
        }
        if (workInProgress.lastEffect !== null) {
          if (returnFiber.lastEffect !== null) {
            returnFiber.lastEffect.nextEffect = workInProgress.firstEffect;
          }
          returnFiber.lastEffect = workInProgress.lastEffect;
        }

        // If this fiber had side-effects, we append it AFTER the children's
        // side-effects. We can perform certain side-effects earlier if
        // needed, by doing multiple passes over the effect list. We don't want
        // to schedule our own side-effect on our own list because if end up
        // reusing children we'll schedule this effect onto itself since we're
        // at the end.
        var effectTag = workInProgress.effectTag;
        // Skip both NoWork and PerformedWork tags when creating the effect list.
        // PerformedWork effect is read by React DevTools but shouldn't be committed.
        if (effectTag > PerformedWork) {
          if (returnFiber.lastEffect !== null) {
            returnFiber.lastEffect.nextEffect = workInProgress;
          } else {
            returnFiber.firstEffect = workInProgress;
          }
          returnFiber.lastEffect = workInProgress;
        }
      }

      if (true && ReactFiberInstrumentation_1.debugTool) {
        ReactFiberInstrumentation_1.debugTool.onCompleteWork(workInProgress);
      }

      if (siblingFiber !== null) {
        // If there is more work to do in this returnFiber, do that next.
        return siblingFiber;
      } else if (returnFiber !== null) {
        // If there's no more work in this returnFiber. Complete the returnFiber.
        workInProgress = returnFiber;
        continue;
      } else {
        // We've reached the root.
        return null;
      }
    } else {
      if (enableProfilerTimer && workInProgress.mode & ProfileMode) {
        // Record the render duration for the fiber that errored.
        stopProfilerTimerIfRunningAndRecordDelta(workInProgress, false);

        // Include the time spent working on failed children before continuing.
        var actualDuration = workInProgress.actualDuration;
        var child = workInProgress.child;
        while (child !== null) {
          actualDuration += child.actualDuration;
          child = child.sibling;
        }
        workInProgress.actualDuration = actualDuration;
      }

      // This fiber did not complete because something threw. Pop values off
      // the stack without entering the complete phase. If this is a boundary,
      // capture values if possible.
      var next = unwindWork(workInProgress, nextRenderExpirationTime);
      // Because this fiber did not complete, don't reset its expiration time.
      if (workInProgress.effectTag & DidCapture) {
        // Restarting an error boundary
        stopFailedWorkTimer(workInProgress);
      } else {
        stopWorkTimer(workInProgress);
      }

      {
        resetCurrentFiber();
      }

      if (next !== null) {
        stopWorkTimer(workInProgress);
        if (true && ReactFiberInstrumentation_1.debugTool) {
          ReactFiberInstrumentation_1.debugTool.onCompleteWork(workInProgress);
        }

        // If completing this work spawned new work, do that next. We'll come
        // back here again.
        // Since we're restarting, remove anything that is not a host effect
        // from the effect tag.
        next.effectTag &= HostEffectMask;
        return next;
      }

      if (returnFiber !== null) {
        // Mark the parent fiber as incomplete and clear its effect list.
        returnFiber.firstEffect = returnFiber.lastEffect = null;
        returnFiber.effectTag |= Incomplete;
      }

      if (true && ReactFiberInstrumentation_1.debugTool) {
        ReactFiberInstrumentation_1.debugTool.onCompleteWork(workInProgress);
      }

      if (siblingFiber !== null) {
        // If there is more work to do in this returnFiber, do that next.
        return siblingFiber;
      } else if (returnFiber !== null) {
        // If there's no more work in this returnFiber. Complete the returnFiber.
        workInProgress = returnFiber;
        continue;
      } else {
        return null;
      }
    }
  }

  // Without this explicit null return Flow complains of invalid return type
  // TODO Remove the above while(true) loop
  // eslint-disable-next-line no-unreachable
  return null;
}
```

**如果数据的到达速度快于帧速率，我们可以合并和批量更新**
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
这段代码可以得知，enqueueSetState 做了两件事： 
1、将新的state放进数组里 
2、用enqueueUpdate来处理将要更新的实例对象
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
如果isBatchingUpdates标志位是true，说明当前正处于组件更新的阶段，就不会立刻去更新组件，而是将当前组件实例push到dirtyComponents数组中；反之，立刻触发更新。
所以，不是每一次的setState都会同步更新组件，**setState是一个异步的过程，它会集齐一批需要更新的组件然后一起更新**。

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

    // 如果当前事务正在更新过程中，则调用callback，即enqueueUpdate
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

**在组件生命周期中或者react事件绑定中，setState是通过异步更新的。在延时的回调(promise.then、setTimeout)或者原生事件绑定的回调中调用setState不一定是异步的。**
https://blog.logrocket.com/simplifying-state-management-in-react-apps-with-batched-updates/

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

### setState注意点
- 调用完，最新的值通过this.state通常是拿不到的
- 多次调用setState