## 函数式组件和class组件的区别
> 函数式组件捕获了渲染所使用的值。
- 像调用函数一样，每次渲染的函数式组件有属于自己的变量，原理是闭包捕获了当前的props值;
- 在函数式组件中，你也可以拥有一个在所有的组件渲染帧中**共享**的可变变量，useRef(null);
- [函数式组件与类组件有何不同？](https://overreacted.io/zh-hans/how-are-function-components-different-from-classes/)

> 相比class组件，可以自定义hooks，在不同组件间共享逻辑
- 复用逻辑，但内部的state和effects是完全隔离的
  - 组件内的每一个函数（包括事件处理函数，effects，定时器或者API调用等等）会捕获定义它们的那次渲染中的props和state。[useEffect 完整指南](https://overreacted.io/zh-hans/a-complete-guide-to-useeffect/)
- hooks中目前还不支持getSnapshotBeforeUpdate生命周期
- hooks支持HOC、renderProps

> class组件容易阻断数据源
- class组件在生命周期方法中使用props和state时很容易忘记更新
- Hooks可以方便实现数据源的更新和相应，即hooks推动你做正确的事情
- [编写有弹性的组件](https://overreacted.io/zh-hans/writing-resilient-components/)

TODO: 尝试自定义hooks
- https://blog.logrocket.com/how-to-migrate-from-hocs-to-hooks-d0f7675fd600/
- https://kentcdodds.com/blog/react-hooks-whats-going-to-happen-to-render-props

## render props
- render props是给子组件传入一个function类型的prop用作render
- 可以使用children prop字段，或者直接内嵌children的形式
``` js
<Mouse>
  {mouse => (
    <p>The mouse position is {mouse.x}, {mouse.y}</p>
  )}
</Mouse>
```
- 注意事项：当和PureComponent使用的时候, shallowEqual始终返回false，因为每次传入的function prop都是新创建的，可以将function prop定义在一个实例方法中，例如
``` js
class MouseTracker extends React.Component {
  // Defined as an instance method, `this.renderTheCat` always
  // refers to *same* function when we use it in render
  renderTheCat(mouse) {
    return <Cat mouse={mouse} />;
  }

  render() {
    return (
      <div>
        <h1>Move the mouse around!</h1>
        <Mouse render={this.renderTheCat} />
      </div>
    );
  }
}
```
- 与HOC比较
  - 只能使用指定传入的props|state参数
  - 可以传入多个render props

## React hooks
1. 为什么需要hooks
React很受欢迎，但class组件存在一些问题：
- 使用super(props)是很烦人的
- this指向的绑定比较麻烦
- 通过生命周期方法组织组件，迫使我们将互相关联的逻辑分散到不同的函数中
- 同一个生命周期函数放置很多互不关联的side effect逻辑
- class组件对于共享非可视逻辑没有很好的原始支持
1. hooks可以解决什么问题
- 首先function组件没有super和this绑定问题
- hooks可以解决function组件缺失的state管理、模拟生命周期、自定义hooks共享非可视逻辑(例如需要用HOC实现的签到弹窗逻辑可以改成自定义hooks，HOC开发不友好、props传递丢失、陷入嵌套Hell)
- 提供数据和方法缓存减少渲染范围、提供全局变量

### react hooks原理
- 初次渲染的时候，按照 useState，useEffect 的顺序，把 state，deps 等按顺序塞到 memoizedState 队列中。
- 更新的时候，按照顺序，从 memoizedState 中把上次记录的值拿出来。

Q：为什么只能在函数最外层调用 Hook？为什么不要在循环、条件判断或者子函数中调用。
A：memoizedState 数组是按 hook定义的顺序来放置数据的，如果 hook 顺序变化，memoizedState 并不会感知到。

Q：自定义的 Hook 是如何影响使用它的函数组件的？
A：共享同一个 memoizedState，共享同一个顺序。

Q：“Capture Value” 特性是如何产生的？ 闭包属性
A：每一次 ReRender 的时候，都是重新去执行函数组件了，对于之前已经执行过的函数组件，并不会做任何操作。


### hooks中过时的闭包
> 闭包创建的时候会捕获引用的外部变量的值
解决办法：
- `setCount(count => count + 1)`确保将最新状态值作为参数提供给更新状态函数
- (didUpdate, [dependsArr])，将依赖传入dependsArr

https://codesandbox.io/s/goofy-leaf-11gmt?file=/src/index.js
https://codesandbox.io/s/white-hooks-qgvm5?file=/src/index.js
https://segmentfault.com/a/1190000020805789

### useCallback和useMemo
> 性能优化不是免费的。 它们总是带来成本，但这并不总是带来好处来抵消成本。
#### 什么时候使用 useMemo 和 useCallback
- 引用相等
  - 大多数时候，你不需要考虑去优化不必要的重新渲染; 有些情况下渲染可能会花费大量时间（比如重交互的图表、动画等）
- 昂贵的计算
  - 计算素数
https://jancat.github.io/post/2019/translation-usememo-and-usecallback/

[将 React 作为 UI 运行时](https://overreacted.io/zh-hans/react-as-a-ui-runtime/)

### useEffect vs useLayoutEffect
- useEffect 是异步执行的，而useLayoutEffect是同步执行的。
- useEffect 的执行时机是浏览器完成渲染之后，而 useLayoutEffect 的执行时机是浏览器把内容真正渲染到界面之前，和 componentDidMount 等价。
> [官方文档](https://zh-hans.reactjs.org/docs/react-component.html#componentdidmount)：你可以在 componentDidMount() 里直接调用 setState()。它将触发额外渲染，但此渲染会发生在浏览器更新屏幕之前。如此保证了即使在 render() 两次调用的情况下，用户也不会看到中间状态。
> [useLayoutEffect和useEffect的区别](https://zhuanlan.zhihu.com/p/348701319)

### react优化
- 添加错误边界ErrorBoundary
- list添加key值

### React性能优化V2
React性能优化的三个方向：
- 减少计算的量。 -> 对应到 React 中就是减少渲染的节点 或者 降低组件渲染的复杂度 
- 利用缓存。-> 对应到 React 中就是如何避免重新渲染，利用函数式编程的 memo 方式来避免组件重新渲染
- 精确重新计算的范围。 对应到 React 中就是绑定组件和状态关系, 精确判断更新的'时机'和'范围'. 只重新渲染'脏'的组件，或者说降低渲染范围
#### 减少计算的量
1. 不要在render方法内部进行副作用相关的计算
2. 减少嵌套，例如使用 React Fragments 避免额外标记
3. 虚拟列表
4. 惰性渲染：懒加载react-loadable | lazy Suspense
``` js
import React, { Suspense } from 'react';

const OtherComponent = React.lazy(() => import('./OtherComponent'));

function MyComponent() {
  return (
    <div>
      <Suspense fallback={<div>Loading...</div>}>
        <OtherComponent />
      </Suspense>
    </div>
  );
}
```
#### 利用缓存
1. 不变的事件处理函数：
  - 不要使用内联函数定义
  - 在Function组件中使用useCallback来包装事件处理器，尽量给下级组件暴露一个静态的函数
  - 设计更方便处理的 Event Props
  - this绑定放到constructor中，或者使用autobind方法
2. useMemo缓存复杂的计算结果
3. 简化props，降低props变化的影响范围
4. 简化state，避免不必要的数据变动导致组件重新渲染

#### 精确重新计算的范围
所谓精细化渲染指的是只有一个数据来源导致组件重新渲染, 比如说 A 只依赖于 a 数据，那么只有在 a 数据变动时才渲染 A, 其他状态变化不应该影响组件 A。
Vue 和 Mobx 宣称自己性能好的一部分原因是它们的'响应式系统', 它允许我们定义一些‘响应式数据’，当这些响应数据变动时，依赖这些响应式数据视图就会重新渲染。在Vue和Mobx中，组件的依赖是自动追踪的，****。
1. 响应式数据的精细化渲染
   - mobx状态管理框架
   - shouldComponentUpdate 生命周期事件
   - 函数式组件React.memo | class组件PureComponent
2. 不要滥用 Context：明确状态作用域, Context 只放置必要的，关键的，被大多数组件所共享的状态
   - Context API 的更新特点，它是可以穿透React.memo或者shouldComponentUpdate的比对的，也就是说，一旦 Context 的 Value 变动，所有依赖该 Context 的组件会全部 forceUpdate.
##### context缓存问题
因为 context 会使用参考标识（reference identity）来决定何时进行渲染，这里可能会有一些陷阱，当 provider 的父组件进行重渲染时，可能会在 consumers 组件中触发意外的渲染。举个例子，当每一次 Provider 重渲染时，以下的代码会重渲染所有下面的 consumers 组件，因为 value 属性总是被赋值为新的对象：
``` js
class App extends React.Component {
  render() {
    return (
      <MyContext.Provider value={{something: 'something'}}>
        <Toolbar />
      </MyContext.Provider>
    );
  }
}
```
为了防止这种情况，将 value 状态提升到父节点的 state 里：
``` js
class App extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      value: {something: 'something'},
    };
  }

  render() {
    return (
      <Provider value={this.state.value}>
        <Toolbar />
      </Provider>
    );
  }
}

// Function组件
export function ThemeProvider(props) {
  const [theme, switchTheme] = useState(redTheme);
  const value = useMemo(() => ({ theme, switchTheme }), [theme]);
  return <Context.Provider value={value}>{props.children}</Context.Provider>;
}
```

### React.StrictMode
- 识别不安全的生命周期
- 关于使用过时字符串 ref API 的警告
- 关于使用废弃的 findDOMNode 方法的警告
- 检测意外的副作用: 通过调用多次来帮助发现副作用
- 检测过时的 context API

### react-router原理：
- 组件初始化的时候注册history.listen的监听事件
- 调用history.push、history.replace之后会触发监听，调用notifyListeners
- 根据传递进来path,url,params等参数，最终通过setState触发组件reRender
- 路由相关信息的传递依赖context从Router传递到内部的Route
``` js
class Router extends React.Component {
    componentWillMount() {
        const { children, history } = this.props
        
        //调用history.listen监听方法，该方法的返回函数是一个移除监听的函数
        this.unlisten = history.listen(() =&gt; {
          this.setState({
            match: this.computeMatch(history.location.pathname)
          });
        });
    }
    componentWillUnmount() {
      this.unlisten();
    }
    render() {
    
    }
}
```
``` js
class Route extends React.Component {
  ....
  constructor(){
  
  }
  render() {
    const { match } = this.state;
    const { children, component, render } = this.props;
    const { history, route, staticContext } = this.context.router;
    const location = this.props.location || route.location;
    const props = { match, location, history, staticContext };

    if (component) return match ? React.createElement(component, props) : null;

    if (render) return match ? render(props) : null;

    if (typeof children === "function") return children(props);

    if (children &amp;&amp; !isEmptyChildren(children))
      return React.Children.only(children);

    return null;
  }
}
```
``` js
var push = function push(path, state) {
    var action = 'PUSH';
    var location = createLocation(path, state, createKey(), history.location);

    transitionManager.confirmTransitionTo(location, action, getUserConfirmation, function (ok) {
      if (!ok) return;

      var href = createHref(location);
      var key = location.key,
          state = location.state;

      if (canUseHistory) {
        globalHistory.pushState({ key: key, state: state }, null, href);

        if (forceRefresh) {
          window.location.href = href;
        } else {
          var prevIndex = allKeys.indexOf(history.location.key);
          var nextKeys = allKeys.slice(0, prevIndex === -1 ? 0 : prevIndex + 1);

          nextKeys.push(location.key);
          allKeys = nextKeys;
          // 调用setState触发listeners更新
          setState({ action: action, location: location });
        }
      } else {
        warning(state === undefined, 'Browser history cannot push state in browsers that do not support HTML5 history');

        window.location.href = href;
      }
    });
  };

  var setState = function setState(nextState) {
    _extends(history, nextState);

    history.length = globalHistory.length;
    transitionManager.notifyListeners(history.location, history.action);
  };
```

### callback Ref的使用场景
结合`callback Ref`, 可以方便地获取dom节点的高度。代码示例可参考：
- [react-measuredref-callback](https://codesandbox.io/s/react-measuredref-callback-525p4?file=/src/index.js:89-708)
- [even if a child component displays the measured node later](https://codesandbox.io/s/818zzk8m78)
``` js
function MeasureExample() {
  const [height, setHeight] = useState(0);

  const measuredRef = useCallback((node) => {
    if (node !== null) {
      setHeight(node.getBoundingClientRect().height);
    }
    // 1. 传入[]，保证rerender的时候不会被重新触发，只在mount和unmount的时候被调用
    // 2. 不使用useRef，是因为object ref的value发生变化时，不会触发通知；使用ref callback的形式
    //    可以对延迟展示的组件同样有效
    // 3. 如果要支持resize，可以结合[ResizeObserver](https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver)
  }, []);

  return (
    <>
      <h1 ref={measuredRef}>Hello, world</h1>
      <h2>The above header is {Math.round(height)}px tall</h2>
    </>
  );
}

```