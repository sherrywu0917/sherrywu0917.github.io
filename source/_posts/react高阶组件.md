---
title: react高阶组件
date: 2017-10-30 17:39:56
tags: [react, 高阶组件]
---
### 什么是高阶组件？
官方文档指出，高阶组件（Higher-Order Components，简称HOC）是react中对组件逻辑进行重用的高级技术。但高阶组件本身并不是React API。它只是一种模式，这种模式是由react自身的组合性质必然产生的。
具体而言，高阶组件就是一个函数，且该函数接受一个组件作为参数，并返回一个新的组件。
先看一个典型的高阶组件：
``` js
// It's a function...
function myHOC() {
  // Which returns a function that takes a component...
  return function(WrappedComponent) {
    // It creates a new wrapper component...
    class TheHOC extends React.Component {
      render() {
        // And it renders the component it was given
        return <WrappedComponent {...this.props} />;
      }
    }

    // Remember: it takes a component and returns a new component
    // Gotta return it here.
    return TheHOC;
  }
}
```
### 使用高阶组件优化代码
需求是要在网站的每个页面要添加签到请求和签到成功弹窗，显然，一般人都不会想着要在每个页面写一遍。首先，我想到的是引入一个React组件，但是要在每个页面的render方法里面添加该组件，需要改动所有页面对应组件的render方法，感觉比较繁琐且易出错。也可以通过js方法去动态创建签到成功弹窗，然后在每个页面里面引入该方法，不过同样需要改变组件内部方法，且不符合React项目风格。
最后，发现高阶组件可以满足我的需求，既实现了功能又不污染原有组件。下面是简化版的签到高阶组件：
<!-- more -->
``` js
export const signHoc = ComposedComponent => class extends Component {
    constructor(props) {
        super(props)
        this.state = {show: false, config: {}}
    }

    componentDidMount() {
        if(!this.props.showSign) return;
        this.setState({
            show: true,
            config: {
                title: '今日首次登录',
                content: `获赠<em>50</em>阅点红包`
            }
        })
    }

    render() {
        const {show, config} = this.state;
        const {showSign, ...passThroughProps} = this.props;
        return (<div>
            <ComposedComponent {...passThroughProps} />
            {show && (<MessageModal config={config}/>)}
        </div>);
    }
};
```
可以通过this.props获得传给WrappedComponent的props，在操作props的时候要确保不会破坏要传给WrappedComponent的props。在componentDidMount可以发起签到请求，此外可以根据props.showSign和签到结果决定是否显示签到组件等。当页面有其他弹窗的优先级大于签到弹窗时，props.showSign设为false，就可以控制签到弹窗是否显示。
```
export default class WrappedComponent extends Component {
    render() {
        return false
    }
}
signHoc(WrappedComponent)
```
调用signHoc函数获得组合后的组件，也可以结合@decorator简化调用。
```
@signHoc
export default class WrappedComponent extends Component {
    render() {
        return false
    }
}
```

### 结合Context API使用
`React 16.3`引入了全新的`Context API`，主要提供的API有：`React.createContext`, `Provider`和`Consumer`，详见[context](https://reactjs.org/docs/context.html)。
这儿记录下如何结合高阶组件和Context API去实现全局变量的设置，如主题theme。
step1: 定义高阶组件，通过`ThemeContext.Consumer`获取变量并且传递给组件ComposedComponent。
``` js
export const ThemeContext = React.createContext({
    theme: localStorage.getItem('N_reader_theme') || 'light',
    changeTheme: () => {}});

export const themeHoc = ComposedComponent => {
    props => {
        <ThemeContext.Consumer>
            {({theme, changeTheme}) => <ComposedComponent {...props} theme={theme} changeTheme={changeTheme}/>}
        </ThemeContext.Consumer>
    }
}
```

step2: 使用`ThemeContext.Provider`传递当前theme、changeTheme给整个子树，任何子组件都可以读到这些变量的值，不需要显式的通过props传递。
``` js
import {ThemeContext} from 'themeHoc.js'

export default class App extends Component {
    changeTheme(theme) {
        localStorage.setItem('N_reader_theme', theme);
        document.querySelector('body').className = document.querySelector('body').className.replace(/theme--\w*/, 'theme--' + theme);
        this.setState({
            theme: theme
        })
    }

    render() {
        return (
            <ThemeContext.provider value={{theme: this.state.theme, changeTheme: this.changeTheme}}>
                <Content />
            </ThemeContext.provider>
        )
    }
}
```

step3: 通过decorator的方式添加`@themeHoc`标记，Content组件可以直接从props中读取theme和changeTheme函数。
``` js
import {themeHoc} from 'themeHoc.js'

@themeHoc
export default class Content extends Component {
    render() {
        return (
            <div theme={props.theme}>
                <a href="javascript:;" className={props.theme} onClick={props.changeTheme.bind(this, 'blue')}>blue</a>
                <a href="javascript:;" className={props.theme} onClick={props.changeTheme.bind(this, 'yellow')}>yellow</a>
            </div>
        )
    }
}
```

### 公约

#### 最大化使用组合
高阶组件可以接受一个或多个参数：
```
//仅接收包裹组件一个参数
const NavbarWithRouter = withRouter(Navbar);
//接收包裹组件和额外的参数
const CommentWithRelay = Relay.createContainer(Comment, config);
//connect(commentListSelector, commentListActions)返回了一个高阶组件，最后作用于Comment组件
const ConnectedComment = connect(commentSelector, commentActions)(Comment);
```
#### 包装显示名字以便于调试
高价组件创建的容器组件在React Developer Tools中的表现和其它的普通组件是一样的。为了便于调试，可以选择一个好的名字，确保能够识别出它是由高阶组件创建的新组件还是普通的组件。
最常用的技术就是将包裹组件的名字包装在显示名字中。所以，如果你的高阶组件名字是 withSubscription，且包裹组件的显示名字是 CommentList，那么就是用 withSubscription(CommentList)这样的显示名字：
``` js
function withSubscription(WrappedComponent) {
  class WithSubscription extends React.Component {/* ... */}
  WithSubscription.displayName = `WithSubscription(${getDisplayName(WrappedComponent)})`;
  return WithSubscription;
}

function getDisplayName(WrappedComponent) {
  return WrappedComponent.displayName || WrappedComponent.name || 'Component';
}
```
### 注意事项
#### 静态方法需要被复制
当你通过 HOC 创建组件的时候，原有组件被容器组件包裹。这意味着容器组件不含有原有组件的静态方法。
``` js
// 创建静态方法
WrappedComponent.staticMethod = function() {/*...*/}
// Now apply an HOC
const EnhancedComponent = enhance(WrappedComponent);

// 新组件没有静态方法
typeof EnhancedComponent.staticMethod === 'undefined' // true
```
为了解决这个问题，你需要在包裹之前把静态方法拷贝给容器组件：
``` js
function enhance(WrappedComponent) {
  class Enhance extends React.Component {/*...*/}
  // Must know exactly which method(s) to copy :(
  Enhance.staticMethod = WrappedComponent.staticMethod;
  return Enhance;
}
```
也可以使用 hoist-non-react-statics 来自动拷贝非原生 React 方法的静态方法。

#### ref不会被传递
尽管高阶组件的约定是把所有的 props 传递给被包裹的组件，但它并不能传递 ref。那是因为 ref 实际上并不是 prop —— 就像 key 一样，它们会被 React 特殊对待。如果你给 HOC 传递了 ref，它将指向容器组件而不是被包裹的组件。
如果一定要用到ref，可以将ref当做普通的props传递给组件：
``` js
function Field({ inputRef, ...rest }) {
  return <input ref={inputRef} {...rest} />;
}

// 将 Field 在高阶组件里包裹起来
const EnhancedField = enhance(Field);

// 在组件的 render 方法内..
<EnhancedField
  inputRef={(inputEl) => {
    // This callback gets passed through as a regular prop
    this.inputEl = inputEl
  }}
/>

// 在高阶组件中就可以访问组件实例和dom节点了...
this.inputEl.focus();
```