---
title: react-mobx使用
date: 2020-12-02 12:12:12
tags: [react, diff]
---

### 核心
1. 封装可观察对象——observable
``` js
  const store = observable({ a: 1 });
```
2. 依赖收集——派生derivation：
   1. computed表达式
   2. when、autorun、reaction的第一个入参函数
   3. observer组件的render方法
``` js
  autorun(() => console.log(store.a);
```
3. 推导——Action触发 derivation 的重新执行
``` js
  store.a = 2; // console 2
```
### 核心原理
#### 利用Object.defineProperty方法，劫持了Observable对象的get()和set()方法。
- 在访问Observable对象的时候，会调用get()方法，注册监听。如何绑定依赖：
  - Observable对象调用`reportObserved(this)`，先将其添加到derivation监听者的可观察对象数组中observing[]
  - derivation对象调用`bindDependencies`，遍历`observing[]`，将自身实例添加到observable对象的监听者数组中observers_
  - 至此，建立了双向依赖
- 在修改Observable对象的时候，会调用set()方法，这个时候会触发监听
  - Observable对象发生变化，会遍历监听者数组observers_，触发更新
#### observe方法
- 显示调用addListener绑定依赖，在set()中调用notifyListeners触发更新
#### class组件如何做到reactive
```js
function makeComponentReactive(render: any) {
    if (isUsingStaticRendering() === true) return render.call(this)

    /**
     * If props are shallowly modified, react will render anyway,
     * so atom.reportChanged() should not result in yet another re-render
     */
    setHiddenProp(this, skipRenderKey, false)
    /**
     * forceUpdate will re-assign this.props. We don't want that to cause a loop,
     * so detect these changes
     */
    setHiddenProp(this, isForcingUpdateKey, false)

    const initialName = getDisplayName(this)
    const baseRender = render.bind(this)

    let isRenderingPending = false
    //  Reaction的两个参数(name, onInvalidate)
    const reaction = new Reaction(`${initialName}.render()`, () => {
        if (!isRenderingPending) {
            // N.B. Getting here *before mounting* means that a component constructor has side effects (see the relevant test in misc.js)
            // This unidiomatic React usage but React will correctly warn about this so we continue as usual
            // See #85 / Pull #44
            isRenderingPending = true
            if (this[mobxIsUnmounted] !== true) {
                let hasError = true
                try {
                    setHiddenProp(this, isForcingUpdateKey, true)
                    if (!this[skipRenderKey]) Component.prototype.forceUpdate.call(this)
                    hasError = false
                } finally {
                    setHiddenProp(this, isForcingUpdateKey, false)
                    if (hasError) reaction.dispose()
                }
            }
        }
    })

    reaction["reactComponent"] = this
    reactiveRender[mobxAdminProperty] = reaction
    this.render = reactiveRender


    function reactiveRender() {
        isRenderingPending = false
        let exception = undefined
        let rendering = undefined
        // 调用Reaction.prototype.track方法
        reaction.track(() => {
            try {
                // step1: 修改globalState$$1.allowStateChanges的值为false，并将之前的值保存在prev字段中，
                // step2: 执行baseRender()方法，结果为res
                // step3: 恢复globalState$$1.allowStateChanges值为prev
                // step4: 返回res结果，赋值给rendering
                rendering = _allowStateChanges(false, baseRender)
            } catch (e) {
                exception = e
            }
        })
        if (exception) {
            throw exception
        }
        return rendering
    }

    return reactiveRender.call(this)
}
```

`reaction.track`流程关键代码：
// 调用trackDerivedFunction方法，进一步去收集依赖，注册观察者
```js
Reaction$$1.prototype.track = function (fn) {
    startBatch$$1(); // globalState$$1.inbatch++ 
    var notify = isSpyEnabled$$1(); // !!globalState$$1.spyListeners.length
    var startTime;
    if (notify) { // spy监听所有mobx中的事件（深层监听），log作用
        startTime = Date.now();
        spyReportStart$$1({
            name: this.name,
            type: "reaction"
        }); // 触发globalState$$1.spyListeners[i][event] 
    }
    this._isRunning = true;
    var result = trackDerivedFunction$$1(this, fn, undefined);
    this._isRunning = false;
    this._isTrackPending = false;
    if (this.isDisposed) {
        // disposed during last run. Clean up everything that was bound after the dispose call.
        clearObserving$$1(this);
    }
    if (isCaughtException$$1(result))
        this.reportExceptionInDerivation(result.cause);
    if (notify) { // spy监听所有mobx中的事件（深层监听），log作用
        spyReportEnd$$1({
            time: Date.now() - startTime
        });
    }
    // 检查globalState$$1.pendingUnobservations数组中的observable对象是否还有观察者，即observers的长度是否为0
    // 若为0，将该对象变成unobserve
    endBatch$$1();
};

/**
 * Executes the provided function `f` and tracks which observables are being accessed.
 * The tracking information is stored on the `derivation` object and the derivation is registered
 * as observer of any of the accessed observables.
 */
// derivation是Reaction 或者 ComputedValue
// f是render、等方法
// 执行f方法，收集依赖
function trackDerivedFunction$$1(derivation, f, context) {
    // pre allocate array allocation + room for variation in deps
    // array will be trimmed by bindDependencies
    changeDependenciesStateTo0$$1(derivation);
    derivation.newObserving = new Array(derivation.observing.length + 100);
    derivation.unboundDepsCount = 0;
    derivation.runId = ++globalState$$1.runId;
    // 为什么prevTracking?
    var prevTracking = globalState$$1.trackingDerivation;
    globalState$$1.trackingDerivation = derivation;
    var result;
    if (globalState$$1.disableErrorBoundaries === true) {
        result = f.call(context);
    }
    else {
        try {
            result = f.call(context);
        }
        catch (e) {
            result = new CaughtException$$1(e);
        }
    }
    globalState$$1.trackingDerivation = prevTracking;
    bindDependencies(derivation);
    return result;
}

// 如何收集依赖？
// - 在访问Observable对象的时候，会调用get()方法，注册监听。如何绑定依赖：
// - Observable对象的get方法中会调用`this.reportObserved()`，先将其添加到derivation观察者的可观察对象数组newObserving[]
// array对象如何收集依赖？
function reportObserved$$1(observable$$1) {
    var derivation = globalState$$1.trackingDerivation;
    if (derivation !== null) {
        /**
         * Simple optimization, give each derivation run an unique id (runId)
         * Check if last time this observable was accessed the same runId is used
         * if this is the case, the relation is already known
         */
        if (derivation.runId !== observable$$1.lastAccessedBy) {
            observable$$1.lastAccessedBy = derivation.runId;
            derivation.newObserving[derivation.unboundDepsCount++] = observable$$1;
            if (!observable$$1.isBeingObserved) {
                observable$$1.isBeingObserved = true;
                observable$$1.onBecomeObserved();
            }
        }
        return true;
    }
    else if (observable$$1.observers.length === 0 && globalState$$1.inBatch > 0) {
        queueForUnobservation$$1(observable$$1);
    }
    return false;
}

/**
 * diffs newObserving with observing.
 * update observing to be newObserving with unique observables
 * notify observers that become observed/unobserved
 */
// - derivation对象调用`bindDependencies`，diff newObserving和old observing，
// 1. 对于old observing中不在NewObserving中的数组，移除derivation对该对象的监听
// 2. 给新增的observing，给derivation添加对每个对象的监听，将derivation添加到observable对象的监听者数组中observers_
// - 至此，建立了双向依赖
// - 在修改Observable对象的时候，会调用set()方法，这个时候会触发监听
function bindDependencies(derivation) {
    // invariant(derivation.dependenciesState !== IDerivationState.NOT_TRACKING, "INTERNAL ERROR bindDependencies expects derivation.dependenciesState !== -1");
    var prevObserving = derivation.observing;
    var observing = (derivation.observing = derivation.newObserving);
    var lowestNewObservingDerivationState = exports.IDerivationState.UP_TO_DATE;
    // Go through all new observables and check diffValue: (this list can contain duplicates):
    //   0: first occurrence, change to 1 and keep it
    //   1: extra occurrence, drop it
    var i0 = 0, l = derivation.unboundDepsCount;
    for (var i = 0; i < l; i++) {
        var dep = observing[i];
        if (dep.diffValue === 0) {
            dep.diffValue = 1;
            // 保证observing被观察者数组中前i0个都是新增的
            if (i0 !== i)
                observing[i0] = dep;
            i0++;
        }
        // Upcast is 'safe' here, because if dep is IObservable, `dependenciesState` will be undefined,
        // not hitting the condition
        if (dep.dependenciesState > lowestNewObservingDerivationState) {
            lowestNewObservingDerivationState = dep.dependenciesState;
        }
    }
    observing.length = i0;
    derivation.newObserving = null; // newObserving shouldn't be needed outside tracking (statement moved down to work around FF bug, see #614)
    // Go through all old observables and check diffValue: (it is unique after last bindDependencies)
    //   0: it's not in new observables, unobserve it
    //   1: it keeps being observed, don't want to notify it. change to 0
    l = prevObserving.length;
    while (l--) {
        var dep = prevObserving[l];
        if (dep.diffValue === 0) {
            removeObserver$$1(dep, derivation);
        }
        dep.diffValue = 0;
    }
    // Go through all new observables and check diffValue: (now it should be unique)
    //   0: it was set to 0 in last loop. don't need to do anything.
    //   1: it wasn't observed, let's observe it. set back to 0
    // 处理新增的observing，添加监听
    while (i0--) {
        var dep = observing[i0];
        if (dep.diffValue === 1) {
            dep.diffValue = 0;
            addObserver$$1(dep, derivation);
        }
    }
    // Some new observed derivations may become stale during this derivation computation
    // so they have had no chance to propagate staleness (#916)
    if (lowestNewObservingDerivationState !== exports.IDerivationState.UP_TO_DATE) {
        derivation.dependenciesState = lowestNewObservingDerivationState;
        derivation.onBecomeStale();
    }
}

// 如何触发依赖
// - set的时候，先检查值是否发生了更新，如果更新了，就触发`this.reportChanged`方法，
// - 如果有显示调用observe添加的，则hasListeners为true，也会触发notifyListeners
    Atom$$1.prototype.reportChanged = function () {
        startBatch$$1();
        propagateChanged$$1(this);
        endBatch$$1();
    };

/**
 * NOTE: current propagation mechanism will in case of self reruning autoruns behave unexpectedly
 * It will propagate changes to observers from previous run
 * It's hard or maybe impossible (with reasonable perf) to get it right with current approach
 * Hopefully self reruning autoruns aren't a feature people should depend on
 * Also most basic use cases should be ok
 */
// Called by Atom when its value changes
function propagateChanged$$1(observable$$1) {
    // invariantLOS(observable, "changed start");
    if (observable$$1.lowestObserverState === exports.IDerivationState.STALE)
        return;
    observable$$1.lowestObserverState = exports.IDerivationState.STALE;
    var observers = observable$$1.observers;
    var i = observers.length;
    while (i--) {
        var d = observers[i];
        if (d.dependenciesState === exports.IDerivationState.UP_TO_DATE) {
            if (d.isTracing !== TraceMode$$1.NONE) {
                logTraceInfo(d, observable$$1);
            }
            d.onBecomeStale();
        }
        // 修改derivation的状态，后面的shouldCompute$$1需要根据该状态判断是否需要更新
        d.dependenciesState = exports.IDerivationState.STALE;
    }
    // invariantLOS(observable, "changed end");
}

// 调用derivation的onBecomeStale方法 -> runReactions$$1
    Reaction$$1.prototype.onBecomeStale = function () {
        this.schedule();
    };
    Reaction$$1.prototype.schedule = function () {
        if (!this._isScheduled) {
            this._isScheduled = true;
            // 将当前需要更新的reaction添加到pendingReactions数组中
            globalState$$1.pendingReactions.push(this);
            runReactions$$1();
        }
    };

/**
 * Magic number alert!
 * Defines within how many times a reaction is allowed to re-trigger itself
 * until it is assumed that this is gonna be a never ending loop...
 */
// 批量更新reaction，在更新reaction的时候，可能会有新的reaction被触发。
// 加了一个单次限制，最多100个reaction一起批量更新
var MAX_REACTION_ITERATIONS = 100;
var reactionScheduler = function (f) { return f(); };
function runReactions$$1() {
    // Trampolining, if runReactions are already running, new reactions will be picked up
    if (globalState$$1.inBatch > 0 || globalState$$1.isRunningReactions)
        return;
    reactionScheduler(runReactionsHelper);
}
function runReactionsHelper() {
    globalState$$1.isRunningReactions = true;
    var allReactions = globalState$$1.pendingReactions;
    var iterations = 0;
    // While running reactions, new reactions might be triggered.
    // Hence we work with two variables and check whether
    // we converge to no remaining reactions after a while.
    while (allReactions.length > 0) {
        if (++iterations === MAX_REACTION_ITERATIONS) {
            console.error("Reaction doesn't converge to a stable state after " + MAX_REACTION_ITERATIONS + " iterations." +
                (" Probably there is a cycle in the reactive function: " + allReactions[0]));
            allReactions.splice(0); // clear reactions
        }
        var remainingReactions = allReactions.splice(0);
        for (var i = 0, l = remainingReactions.length; i < l; i++)
            remainingReactions[i].runReaction();
    }
    globalState$$1.isRunningReactions = false;
}


 /**
 * internal, use schedule() if you intend to kick off a reaction
 */
// 调用runReaction去更新
Reaction$$1.prototype.runReaction = function () {
    if (!this.isDisposed) {
        startBatch$$1();
        this._isScheduled = false;
        // 根据derivation.lowestObserverState判断下是否需要更新
        if (shouldCompute$$1(this)) {
            this._isTrackPending = true;
            try {
                // 触发new Reaction实例时传入的onInvalidate方法
                // 对于Reaction组件是调用Component.prototype.forceUpdate方法去强制更新
                this.onInvalidate();
                if (this._isTrackPending && isSpyEnabled$$1()) {
                    // onInvalidate didn't trigger track right away..
                    spyReport$$1({
                        name: this.name,
                        type: "scheduled-reaction"
                    });
                }
            }
            catch (e) {
                this.reportExceptionInDerivation(e);
            }
        }
        endBatch$$1();
    }
};

// Array数组依赖的收集，Object.defineProperty，属性是
// 对于数据的操作，是基于ObservableArray的实例上面，ObservableArray该类自定义的一些方法被触发后，
// 【spliceWithArray -> notifyArraySplice ｜ 或者直接触发】 -> reportChanged && notifyListeners
var ENTRY_0 = createArrayEntryDescriptor(0);
function createArrayEntryDescriptor(index) {
    return {
        enumerable: false,
        configurable: false,
        get: function () {
            return this.get(index);
        },
        set: function (value) {
            this.set(index, value);
        }
    };
}
function createArrayBufferItem(index) {
    // obj的key值都会被转成string，可以通过arr[index]直接获取
    Object.defineProperty(ObservableArray$$1.prototype, "" + index, createArrayEntryDescriptor(index));
}
```

### useStaticRendering
我们知道可以通过使用@observer，将react组件转换成一个监听者(Reactions)，这样在被监听的应用状态变量(Observable)有更新时，react组件就会重新渲染。而对于服务端的React组件，我们只需要它被渲染一次，而不需要组件监听模型的状态。事实上，如果服务端React组件像客户端组件一样监听模型的状态变化，就会造成严重的内存泄漏问题。官方提供了useStaticRendering方法，用于避免mobx服务端渲染的内存泄漏问题; 该方法只需要在server启动时设置一次。

### 与Redux的区别
REDUX的好处
- 通过action,reducer来完成应用的数据流管理，逻辑简单清晰
- reducer函数式的设计，让我们代码变得可观测，可回溯
- action的设计，特别适用于多数据联动的场景
REDUX的弊端
- 啰嗦代码太多
- reducer无复用性
- 性能优化太繁琐(当然可以引入immutable来解决，但是是不是又增加了学习成本,应用的复杂度和代码量呢)
- app state和store state的划分
- 脏检查，参考https://tech.youzan.com/mobx_vs_redux/

1. **store数量**：redux将数据保存在单一的store中，mobx将数据保存在分散的多个store中
2. **普通对象vsobservable对象**：redux使用plain object保存数据，需要手动处理变化后的操作；mobx适用observable保存数据，数据变化后自动处理响应的操作
3. **状态是否可变**：redux使用不可变状态，这意味着状态是`只读`的，不能直接去修改它，而是应该返回一个新的状态，同时使用纯函数；mobx中的状态是`可变`的，可以直接对其进行修改
4. **函数式**：mobx相对来说比较简单，在其中有很多的抽象，mobx更多的使用面向对象的编程思维；redux会比较复杂，因为其中的函数式编程思想掌握起来不是那么容易，同时需要借助一系列的中间件来处理异步和副作用
5. **是否可追溯**：mobx中有更多的抽象和封装，调试会比较困难，同时结果也难以预测；而redux提供能够进行时间回溯的开发工具，同时其纯函数以及更少的抽象，让调试变得更加的容易

场景辨析:
基于以上区别,我们可以简单得分析一下两者的不同使用场景.
mobx更适合**数据不复杂的应用**: mobx难以调试,很多状态无法回溯,面对复杂度高的应用时,往往力不从心.
redux适合有**回溯需求的应用**: 比如一个画板应用、一个表格应用，很多时候需要撤销、重做等操作，由于redux不可变的特性，天然支持这些操作.
mobx适合短平快的项目: mobx上手简单,样板代码少,可以很大程度上提高开发效率.
当然mobx和redux也并不一定是非此即彼的关系,你也可以在项目中用redux作为全局状态管理,用mobx作为组件局部状态管理器来用.
