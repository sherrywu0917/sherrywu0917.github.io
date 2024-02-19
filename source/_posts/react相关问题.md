---
title: react相关问题
date: 2019-01-29 15:38:28
tags: [react]
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
 <!-- more -->

### react16生命周期
#### Mounting
- constructor()
- static getDerivedStateFromProps()
- render()
- componentDidMount

子类必须在构造函数第一行执行super(props)，否则我们无法在构造函数里拿到this对象，因为子类的实例要通过父类的构造函数完成塑造，得到与父类实例相同的属性和方法。es6的继承是先将是先将父类实例对象的属性和方法，加到this上面（所以必须先调用super方法），然后再用子类的构造函数修改this。ES5 的继承，实质是先创造子类的实例对象this，然后再将父类的方法添加到this上面（Parent.apply(this)）
不建议使用`UNSAFE_componentWillMount()`，因为它是在render之前执行，所以不会触发重新渲染。在该方法内调用setState会和constructor里的初始化state合并执行，应该放到constructor中去初始化。

#### Updating
- static getDerivedStateFromProps(nextProps, prevState) //会返回一个对象来更新当前的state对象
- shouldComponentUpdate()
- render()
- getSnapshotBeforeUpdate(prevProps, prevState) 在dom更新前，可以捕获之前的滚动位置，便于在componentDidUpdate中去恢复
- componentDidUpdate(prevProps, prevState, snapshot) 在dom更新后,snapshot的值是getSnapshotBeforeUpdate的返回值

不建议使用`UNSAFE_componentWillReceiveProps()`和`UNSAFE_componentWillUpdate()`，一次更新可能会调用多次。
`UNSAFE_componentWillReceiveProps()`经常会带来bug和不一致的问题，例如在该方法中设置input值，当父组件更新的时候，用户的输入就会被置空。
``` 
class EmailInput extends Component {
  state = { email: this.props.email };

  render() {
    return <input onChange={this.handleChange} value={this.state.email} />;
  }

  handleChange = event => {
    this.setState({ email: event.target.value });
  };

  componentWillReceiveProps(nextProps) {
    // This will erase any local state updates!
    // Do not do this.
    this.setState({ email: nextProps.email });
  }
}
```

#### Unmounting
- componentWillUnmount()

`getDerivedStateFromProps(nextProps, prevState)`每次render之前都会被触发，例如在挂载时、接收到新的props、调用了setState和forceUpdate都会触发，与`componentWillReceiveProps`只在父组件rerender时会触发不一样。此外，`getDerivedStateFromProps`方法不建议经常使用，使用前想一想是否有替代方案。

#### 错误捕获
- static getDerivedStateFromError(error)
- componentDidCatch(error, info)
错误边界(Error Boundaries) 仅可以捕获其子组件的错误。
使用static getDerivedStateFromError()在抛出错误后渲染回退UI。 使用 componentDidCatch() 来记录错误信息。

#### 异步加载
React V16.6引入的新特性lazy，以及React-loadable库。

#### getDerivedStateFromProps应用

### 加分题：数据获取为什么用 componentDidMount 而不是 constructor？
你希望听到的两个原因会是：“在渲染发生之前数据不会存在” —— 虽然不是主要原因，但它向您显示该人员了解组件的处理方式; “在 React Fiber 中使用新的异步渲染……” —— 有人一直在努力学习。

- r1: SSR模式下，componentWillMount在server端也是会被调用的，内容返回到client端后，componentWillMount会被第二次调用，如果在componentWillMount中处理数据获取则会被调用两次。
- r2: 在componentWillMount中调用setState不会触发rerender，所以一般不会被用来获取数据。
- r3: React16之后采用了Fiber架构，类似ComponentWillMount的生命周期钩子都有可能执行多次，所以不在这些生命周期中做有副作用的操作，比如请求数据。
- r4: constructor用来初始化组件，作用应该保持纯粹，不应该引入数据获取这种有副作用的操作。


### react事件注册分发
事件注册的时候，react 把所有事件都委托到了document上，减少注册事件的数量，降低内存占用。
事件被触发后：
- 冒泡到document处，找到触发的dom和对应的react component
- 当前触发的事件会被加入batchedUpdates批处理队列中
- 在事件队列中，事件分发的核心handleTopLevel保证了子元素在父元素前面（此处分析的是trapBubbledEvent）。
- 事件执行的时候，首先找到合适的plugin（v16版本有5种）构造对应的合成事件，第一次触发new 创建的事件对象会放到缓存池中，下次直接从对象池中取。
- 最后，就是拿到与事件相关的元素实例和回调函数。

#### onClickCapture
如果想要注册捕获事件，可以使用onClickCapture。但React的合成事件都是统一注册在document元素上的，且只有冒泡阶段，但合成事件会区分捕获和冒泡两种类型，来保证合成事件的执行顺序。此外，**原生事件的执行都会早于合成事件的执行**，因为合成事件都要等到事件冒泡到document上，才会执行。
执行顺序是：原生捕获事件 -> 原生事件冒泡 -> 合成事件捕获 -> 合成事件冒泡 



### react 源码

#### Component和PureComponent比较
React组件继承了Component或者PureComponent，这两个父类的定义在ReactBaseClasses.js中。
``` js
function ComponentDummy() {}
ComponentDummy.prototype = Component.prototype;

/**
 * Convenience component with default shallow equality check for sCU.
 */
function PureComponent(props, context, updater) {
  this.props = props;
  this.context = context;
  // If a component has string refs, we will assign a different object later.
  this.refs = emptyObject;
  this.updater = updater || ReactNoopUpdateQueue;
}

const pureComponentPrototype = (PureComponent.prototype = new ComponentDummy());
pureComponentPrototype.constructor = PureComponent;
// Avoid an extra prototype jump for these methods.
Object.assign(pureComponentPrototype, Component.prototype);
pureComponentPrototype.isPureReactComponent = true;
```
从源码上看PureComponent继承了Component，新增了isPureReactComponent变量。
在shouldComponentUpdate函数中，可以看到如果isPureReactComponent为true，会shollow比较新旧props和state是否相等，都相等的话会返回false。而Component始终会返回true。
```
if (type.prototype && type.prototype.isPureReactComponent) {
  return (
    !shallowEqual(oldProps, newProps) || !shallowEqual(oldState, newState)
  );
}
```
在使用PureComponent时要注意：

1. 尽量不要在render函数中给子组件绑定新的函数，如下面
``` js
<ChildComponent onclick={() => this.handleClick()}/>
```
这样每次render都会创建一个新的函数，改变了子组件的props参数，使子组件重新渲染。

2. 尽量不要在render方法内生成数据
生成新的数据，例如新的list数组，也会带来不必要的重新渲染。
``` js
let list = [xxx];
...
{list.map(item => (<LiItem value={item}/>))}
<ChildComponent onclick={() => this.handleClick()}/>
```
因此相比于 Component ，PureComponent 有性能上的更大提升：
- 减少了组件无意义的重渲染（当 state 和 props 没有发生变化时），当结合 immutable 数据时其优更为明显；
- 隔离了父组件与子组件的状态变化；

3. 若是数组和对象等引用类型，则要引用不同，才会渲染
4. 如果prop和state每次都会变，那么PureComponent的效率还不如Component，因为你知道的，进行浅比较也是需要时间

## key值的作用
- 提高diff的效率
- 避免就地复用

## 完全受控组件 & 完全非受控组件
为了避免外部props覆盖掉内部state的变化
- 完全受控组件：子组件没有state，值的获取和修改都交给父组件去处理
- 完全非受控组件：子组件维护自己的state，props值只用来初始化state，给子组件添加key值，当props发生变化的时候，修改子组件的key值，会重新创建一个子组件而不是更新已有的子组件
