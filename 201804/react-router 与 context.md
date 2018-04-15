## 前言
最近我司要上线一个 Hybird 上的 SPA，17 年年底的时候已经写过 demo 给产品和 leader 看了，近期准备要上线。问题在于，当时准备仓促，又想要玩一玩 react，导致了用的版本是比较成熟的，嗯。。。。意思就是比较老的版本，react-router 是 3.x 版本，而 react 也只是 16.0 而已。对于有追求的我而言，升级势在必行。

## 问题所在
在 Vue 应用里面用 Vue-router 就是一个 routes 的事情，甚至连 routes 都可以不是嵌套解构，直接一维路由，毕竟业务少。到了之前写的 react 也是采用了这种方式，如下：
```javascript
ReactDOM.render(
  <Router routes={RouteConfig} history={hashHistory}></Router>,
  document.getElementById('root')
);
```

RouteConfig 基本上也是一维结构，传入到 routes 就好了。然而 routes 这个 props 已经在 react-router 4 里面消失掉了，之前版本采用的是静态路由来配置的，而到了 react-router 4，则采用动态组件。。。如果还需要静态的方式可以采用 `react-router-config`。这个变化使得 react-router 4 升级变得麻烦，由路由配置变成动态映射，这里还有官网提到的[哲学](https://reacttraining.com/react-router/web/guides/philosophy)。吐槽一下：开始看官网教程的时候，感觉像一坨屎一样，东一块西一块的，不知道在说什么，这也是去年选 react-router 版本的时候直接放弃 V4 的原因。最近看这个官网，却越看越好，觉得写得相当的优秀用心，赞一个。


对于 APP 上面的页面过渡动画效果，则采用 `react-addons-css-transition-group` 的 ReactCSSTransitionGroup 组件，这也是比较成熟的方法了，这也是之前官网推荐的方式。让而到了当你点开[npm上的介绍](https://www.npmjs.com/package/react-addons-css-transition-group)时候，发现原来 `react-addons-css-transition-group` 已经不被推荐了：
> The code in this package has moved. We recommend you to use CSSTransitionGroup from react-transition-group instead.
>
> In particular, its version 1.x is a drop-in replacement for the last released version of react-addons-css-transition-group.

然而现实是无情的，只能使用 `react-addons-css-transition-group` 的 V 1.x 版本，这对于一个前端工程师怎么可以容忍呢？新版本里面肯定有适合的 API 嘛，为什么一定要用 ReactCSSTransitionGroup 呢？然而官网一开始看也是烂得不能入眼（可能是英文的缘故没有耐心看）。最后还是在 react-router 4 的[官网](https://reacttraining.com/react-router/web/example/animated-transitions)里面找到解决办法。

**只是一开始傻乎乎的用，抄也没有抄全，部分按照自己的思路走，经常报错**，只有全抄过来才对。。妈呀太可怕了。于是乎想要看看研究一下 react-router 4 的设计！

### 从 Router 出发的 Context
react-router 4 里面依然有 Router，而且这个 Router 是组件，精简一下，Router 代码如下：
```javascript
class Router extends React.Component {
  static contextTypes = {
    router: PropTypes.object
  };
  static childContextTypes = {
    router: PropTypes.object.isRequired
  };
  getChildContext() {
    return {
      router: {
        ...this.context.router,
        history: this.props.history,
        route: {
          location: this.props.history.location,
          match: this.state.match
        }
      }
    };
  }
  state = {
    match: this.computeMatch(this.props.history.location.pathname)
  };
  computeMatch(pathname) {
    return {
      path: "/",
      url: "/",
      params: {},
      isExact: pathname === "/"
    };
  }
  render() {
    const { children } = this.props;
    return children ? React.Children.only(children) : null;
  }
}
``` 

去除掉提示性报错，PropTypes以及服务端渲染中需要在 componentWillMount/componentWillUnmount 操作 `this.unlisten` 部分，就只剩下这么一点点了。 Router 组件有子节点就渲染，没有就 null，简答吧。那要 Router 有何用？别急看看大头 childContextTypes/getChildContext 这又是什么？

React 是有自己的 Context API 的，只是不建议开发者使用，并称之为实验性特性，可能移除，不熟悉 Redux/MobX 的最好都不要碰 Context API，俨然是不让人用的样子。Context API 使用还挺简单的，只要在 context 的提供者组件上申明一下就好了，包括 childContextTypes 以及 getChildContext 方法，这样在子组件里面在定义声明一下 contextTypes 就能够使用了。子组件里面怎么使用呢？通过 contextTypes 声明后直接用 `this.context` 就能够访问了。上面的 Router 中，其子组件在声明后，若要访问 Router 中的 router，直接用 `this.context.router` 就好了，是不是很简单！甚至在组件的生命周期里面也有 Context 传过来，这岂不是非常好，这样就不用一直 props 参数到子组件了，用 Context API 就好了，为何官方是不推荐使用的呢？

官网提到：
> 问题在于，组件提供的context值改变，后代元素如果 shouldComponentUpdate 返回 false 那么context的将不会更新。这使得使用context的组件完全失控，所以基本上没有办法可靠的更新context。

这就是问题所在了，所以是不推荐的。嗯。。。至于 react-router 4 里面这么用嘛。。。。反正也没有用到 shouldComponentUpdate 钩子，而且大神这么用还显得非常溜呢。再查 Context API 的时候，忽然发现原来上个月 React 16.3 有了全新的 Context API，**不再是不建议使用了**。

### React 16.3 Context API
> Context provides a way to pass data through the component tree without having to pass props down manually at every level.

看到官网介绍，大大觉得这个功能很实在，这不就解决了父子/孙子组件之间的通信问题了吗，呀，那那 redux 是不是可以不用了。。。。当然不是。这里先不说为什么，先看看 React 16.3 Context API 有什么：
1. React.createContext<T>(defaultValue: T)
2. Provider: React.ComponentType<{value: T}>
3. Consumer: React.ComponentType<{children: (value: T)=> React.ReactNode}>

第一个是创建方法，生成一个 { Provider, Consumer } 对。Consumer 将从最近的 Provider 中读取 context value，如果没有匹配的 Provider，那将从 defaultValue 中读取值。是不是也很简单？但是意义却是非凡的。Provider 和 Consumer 可以放在自己想想要用的组件上面，不用顾虑组件的层级关系。可以通过对创建的 ` React.createContext` import 到想要用的组件就好了。这不就是相当于穿梭机嘛。数据飞来飞去多有趣。官网给了很好的[例子](https://reactjs.org/docs/context.html#when-to-use-context)，这里就不介绍具体用法了。反正记住：创建的{ Provider, Consumer } 对，Provider 组件提供 context value， 而这个 value 是以 props 的形式进入 Consumer 的子组件的，是不是很骚，不是 `this.context`，而是通过 `this.props` 传入的！还是下面这个简单例子吧：
```javascript
const ThemeContext = React.createContext({
  name: 'ni'
});
<ThemeContext.Provider value={name: 'noNi'}>
  <ThemeContext.Consumer>
  { context => (
    <span>{ context.name }</span>
  )}
  </ThemeContext.Consumer>
</ThemeContext.Provider>
```

上面例子是不推荐用的，太暴殄天物了，这里是只是简单介绍一下形式而已，**Context API 的优势在于多层次的嵌套组件！**Provider 组件里面的传值 value 一般不是固定的嘛，要不然传值干嘛？一般传入 state/props 作为 value，state/props 一变化就可以在触发组件 Provider/Consumer 更新了。

可以看出这个 Context API 出来，和 Redux 的功能似乎有点重叠，都是通信问题。只是很明显的是 Redux 和 Context API 是有区别的，Redux 分离了数据和视图，而 Context API 还是在视图层做文章，并且过于灵活，不利于团队开发，不如 Redux 的数据控制来的规范清晰。并且 Redux 更多的是存储数据，Context API 更多还是一个状态的变化，一个从父组件传递到子组件的状态而已，这么看来更像是 state。另外呢，**对于 SPA 还有个问题，当页面切换的时候，需要传递给下个页面的信息，可以通过路由拼接参数传递，或者就是用 Redux 存储了，而这里 Context API 完全没有用武之地。。。还是很悲哀的。。**这么看来 Redux 还是很会有必要的。为此还有一篇[澄清的采访](http://blog.isquaredsoftware.com/2018/03/redux-not-dead-yet/)。当然对于小项目嘛，这个 Context API 完全是福利呀，为了传递个 props 而已，就不要用沉重麻烦的 redux 啦，多幸福。

### Router 与 Route
说 Context 好像说远了，回到 react-router 4 里面，Router 组件通过 Context API(老版本) 给组件传递了 Context，也就是 router，看看 router 是什么：
```javascript
  getChildContext() {
    return {
      router: {
        ...this.context.router,
        history: this.props.history,
        route: {
          location: this.props.history.location,
          match: this.state.match
        }
      }
    };
  }
```

里面有 history，这个也就是 
```javascript
import { Router } from 'react-router'
import createBrowserHistory from 'history/createBrowserHistory'

const history = createBrowserHistory();
<Router history={history}>
  <App/>
</Router>
```

通过 createBrowserHistory 方法创建的 history，并传入给到 Router组件。Context 里面第二个是 route，可以看出 router.location 就是 history 里面的 location，而 route.match 是 Router 组件里面 state.match，这个 match 在后面会介绍到。

上面就是 Router 组件了。常见的 Route 组件 写法：
```javascript
<Router>
  <div>
    <Route exact path="/" component={Home}/>
    <Route path="/news" component={NewsFeed}/>
  </div>
</Router>
```

再来看看 Route 组件代码：
```javascript
class Route extends React.Component {
  // 已简化部分代码
  static contextTypes = {
    router: PropTypes.shape({
      history: PropTypes.object.isRequired,
      route: PropTypes.object.isRequired,
      staticContext: PropTypes.object
    })
  };
  static childContextTypes = {
    router: PropTypes.object.isRequired
  };
  getChildContext() {
    return {
      router: {
        ...this.context.router,
        route: {
          location: this.props.location || this.context.router.route.location,
          match: this.state.match
        }
      }
    };
  }
  state = {
    match: this.computeMatch(this.props, this.context.router)
  };
  computeMatch({ computedMatch, location, path, strict, exact, sensitive }, router) {
    if (computedMatch) return computedMatch;// Switch 组件已经帮我们计算好了
    const { route } = router;
    // 传props有location，就用location，没有，就用 context.router.route
    const pathname = (location || route.location).pathname; 
    return matchPath(pathname, { path, strict, exact, sensitive }, route.match);
  }
  componentWillReceiveProps(nextProps, nextContext) {
    this.setState({
      match: this.computeMatch(nextProps, nextContext.router)
    });
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
    if (children && !isEmptyChildren(children))
      return React.Children.only(children);

    return null;
  }
}
```

可以看出 Route 组件的重点有三处：
1. 获取传递过来的 Context，并 getChildContext，新建 route。
2. match变化，在 route 的 props 更新的时候，修改 match。
3. 更如 props 的不同，分别以 component/render/Children 的方式渲染子组件。

这里重点看看 match，这个 match 是当前地址与该 Route 组件匹配关系。我们来看看 computeMatch 里面的 matchPath 方法：
```javascript
import pathToRegexp from "path-to-regexp";
const matchPath = (pathname, options = {}, parent) => {
  if (typeof options === "string") options = { path: options };
  const { path, exact = false, strict = false, sensitive = false } = options;
  // 此时的parent就是 Router 里面的 state.math！如果是当前路由是根路径的话为 match 为 true，否之为 false；为根路径正常渲染好了
  if (path == null) return parent;

  const { re, keys } = compilePath(path, { end: exact, strict, sensitive });
  // match：当前定义的路由pathname是否匹配 Route 的 props.path
  const match = re.exec(pathname);
  // 不匹配，则说明该 Route 组件不会渲染任何子组件
  if (!match) return null;

  const [url, ...values] = match;
  const isExact = pathname === url;
  // 完全比配情况
  if (exact && !isExact) return null;

  return {
    path, // the path pattern used to match
    url: path === "/" && url === "" ? "/" : url, // the matched portion of the URL
    isExact, // whether or not we matched exactly
    params: keys.reduce((memo, key, index) => {
      memo[key.name] = values[index];
      return memo;
    }, {})
  };
};
const compilePath = (pattern, options) => {
  // 去除缓存机制的主要部分
  const keys = [];
  const re = pathToRegexp(pattern, keys, options);
  const compiledPattern = { re, keys };
  return compiledPattern;
};
```

可以看出 matchPath 方法，主要还是依赖于 `path-to-regexp` 包，这个在 vue-router 里面有经常看到。通过这个包，可以匹配传入的 pathname，是否匹配 Route 组件 props 过来的 path。如果不匹配则 match 为 null，在渲染的时候如果是 component/render 的方式 match 为 null，Route 组件的渲染结果就是 null，也就是**意味着该 Route 组件不匹配 pathname。**

computeMatch 方法里面的 pathname 变量，这个 pathname 可以是传入参数 location 的，也可以是 context.router.route 的 patchname，而后者就是 history.pathname，也是当前地址。**这个当前地址和传参 location.pathname 会不一样吗？**会的！当然会！通过控制 Route 的 props.location 就可以修改 matchPath 的传参 pathname，从而当前地址与该 Route 的匹配关系 变更为props过来的 location.pathname 与该 Route 的匹配关系。是不是有种为所欲为的 feel。

Route 里面还有个值得注意的 props，是 exact，意思为是否全匹配。如果不设置，如果当要求的 `pathname = '/one'` 时，path 为 '/'，'/one' 或则 '/one/two' 的这个三个 Route 组件都会被匹配到。只有全匹配的时候才能精准匹配 '/one' 的组件。这也是为什么上面的 Route 写法里面会有一个 div 包裹两个 Route 组件，毕竟可能两个都匹配上，而 Router 组件只能有一个 child。

### Switch 组件
Switch 出场率还是很高的，而且 switch 这个单词也很形象，切换路由。常见的用法就是用 Swithc 组件，包裹 n 个 Route 组件。Switch 组件只会渲染一个 Route 组件，如果不是精准的 Route 组件，也只会渲染一个，所以美名为切换：只能留一个的意思。
```javascript
class Switch extends React.Component {
  // 已简化部分代码
  static contextTypes = {
    router: PropTypes.shape({
      route: PropTypes.object.isRequired
    }).isRequired
  };
  render() {
    const { route } = this.context.router;
    const { children } = this.props;
    const location = this.props.location || route.location;

    let match, child;
    React.Children.forEach(children, element => {
      if (match == null && React.isValidElement(element)) {
        const {
          path: pathProp,
          exact,
          strict,
          sensitive,
          from
        } = element.props;
        const path = pathProp || from;

        child = element;
        match = matchPath(
          location.pathname,
          { path, exact, strict, sensitive },
          route.match
        );
      }
    });
    return match
      ? React.cloneElement(child, { location, computedMatch: match })
      : null;
  }
}
```

这个 Switch 组件还是蛮简单的，对子节点 Route 组件遍历，和 Route 里面计算 math 的相同，都是调用 matchPath 方法，只是不同的地方在于传入 matchPath 的 pathname 参数是 Swith 自己的 props.location.patchname，而不是 Route 自己的 props.location.pathname。当计算出 match 之后，若符合就不会继续遍历其他 Route 组件了，从而实现仅渲染单个 Route 组件。

在回头看看 Route 组件里面的 computeMatch 方法，当有 props.computedMatch 的时候，直接返回 computedMatch，不会继续下面自己的 matchPath 方法了，这个 props.computedMatch 正是 Switch 计算出来 match 变量。直接让 Route 组件自己的 props.location 无效化。**这就说明了 Switch 组件对其下面 Route 组件的渲染不仅是单独渲染，更有筛选作用。**

这个时候在看看开头说的项目中遇到的 SPA 路由切换问题，官网给出的[方案](https://reacttraining.com/react-router/web/example/animated-transitions) 采用的方式，通过 Switch 组件的 props.location 实现对其下面的 Route 组件渲染。当路由变化的时候，前一个 Route 组件不会立马消失，而是在 Switch 组件下继续现实。同时新的 Route 也会继续出现，从而实现过渡效果。

### 其他组件
Redirct 组件里面需要注意的是 props.from 是在 Switch 组件里面生效的，`const path = pathProp || from`，来计算 computedMatch，最后的结果则是
从 match 的里面得出 to 的最后路径，而不是简简单单的 '/nameList/:id'，要具体到 id 是多少。当然这里还是用到了 `path-to-regexp` 模块。

StaticRouter 针对服务端渲染的问题，有新的 Context 传过来 staticContext，在 Redirct 里面用得比较多。

withRouter 是个 HOC 高阶组件，意在通过修饰者语句将组件传入 withRouter 里面。[官网](https://reactjs.org/docs/higher-order-components.html#static-methods-must-be-copied-over)里面介绍了一点就是将 Route 组件里面 match location history 传给过来目标组件，用处还是挺好的。

MemoryRouter 和 Promt 组件都比较简单，这里就不介绍了。

## 参考
1. [官方文档 react-router 4](https://reacttraining.com/react-router/core/guides/philosophy)
2. [官方文档 react 16.3 context](https://reactjs.org/docs/context.html)