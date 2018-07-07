# 前言
用了Vue快一年多了(虽然中间间断好长时间)，就越发的对其周边的生态感兴趣，尤其是对Vue-router和Vuex，Vue-router是单页面应用的核心部件，基本上的路由跳转都依赖它，项目上用的比较多的Vonic也是基于于Vue-router的；而Vuex只是在状态变化较多，需要store的时候才用上。本文先介绍Vue-router(2.7.0)，有时间再介绍Vuex；

# 从示例开始
下面是官方给出的示例basic，清晰的介绍了VueRouter最基本使用方法：
```javascript
// 1. 安装插件，同时注册<router-view>和<router-link>，并且劫持$router和$route
Vue.use(VueRouter)

// 2. 定义路由组件
const Home = { template: '<div>home</div>' }
const Foo = { template: '<div>foo</div>' }
const Bar = { template: '<div>bar</div>' }

// 3. 创建路由
const router = new VueRouter({
  mode: 'history',
  base: __dirname,
  routes: [
    { path: '/', component: Home },
    { path: '/foo', component: Foo },
    { path: '/bar', component: Bar }
  ]
})
```
上面代码就可以构成最简单的Vue-router示例，当然创建好的router还需要加入Vue的option中。
可以发现一切的开始在于`Vue.use(VueRouter)`，use之后，直接使用Vue-router里面的api就好了。看看Vue里面use的用法：
```javascript
@Vue.js

Vue.use = function (plugin: Function | Object) {
  const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
  if (installedPlugins.indexOf(plugin) > -1) {
    return this
  }

  // additional parameters
  const args = toArray(arguments, 1)
  args.unshift(this)
  if (typeof plugin.install === 'function') {
    plugin.install.apply(plugin, args)
  } else if (typeof plugin === 'function') {
    plugin.apply(null, args)
  }
  installedPlugins.push(plugin)
  return this
}
```
在Vue.js里面不难发现，use方法主要功能就是执行插件，若有install方法就执行install，并在将该插件push到内部变量_installedPlugins数组里面；而Vue-router的index.js文件里面`VueRouter.install = install`，install变量从install.js文件导入，所以Vue.use(VueRouter)，相当于执行了install.js导出的install方法。
再看看install方法都做了些什么：
```javascript
Vue.mixin({
  beforeCreate () {
    if (isDef(this.$options.router)) {
      this._routerRoot = this
      this._router = this.$options.router
      this._router.init(this)
      Vue.util.defineReactive(this, '_route', this._router.history.current)
    } else {
      this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
    }
    registerInstance(this, this)
  },
  destroyed () {
    registerInstance(this)
  }
})
// 劫持$router，getter方法返回的是VueRouter
Object.defineProperty(Vue.prototype, '$router', {
  get () { return this._routerRoot._router }
})
// 劫持$router，getter方法返回的是VueRouter的路由对象
Object.defineProperty(Vue.prototype, '$route', {
  get () { return this._routerRoot._route }
})
// 注册router-view和router-link全局组件
Vue.component('router-view', View)
Vue.component('router-link', Link)
```
Vue.minxin作用是将混合对象的方法和组件合并，install.js里面则是为每个组件都添加beforeCreate钩子和destroyed钩子；在beforeCreate里面只有Vue实例化的时候才会进入true语句里面(router选项是配置在Vue对象里面)，其他的组件创建时候this.$options没有router对象，只有this.$options.parent才有router对象。如此，Vue实例化的时候，会对router进行初始化`this._router.init(this)`和'_route'的劫持。registerInstance方法是专门针对router-view组件，分析router-view组件的时候会介绍到。

# init 初始化VueRouter实例
VueRouter这个class的实例化过程中会根据配置的选项mode，判断是要进行HTML5History，HashHistory还是AbstractHistory，默认下就是HashHistory，其兼容性是最好的；
而install方法里面重要的就是调用VueRouter实例的init方法：
```javascript
init (app: any /* Vue component instance */) {
  // 判断是否已经处理过app
  // ...
  // 切换路由
  if (history instanceof HTML5History) {
    history.transitionTo(history.getCurrentLocation())
  } else if (history instanceof HashHistory) {
    const setupHashListener = () => {
      history.setupListeners()
    }
    history.transitionTo(
      history.getCurrentLocation(),
      setupHashListener,
      setupHashListener
    )
  }
  // history实例的cb
  history.listen(route => {
    this.apps.forEach((app) => {
      app._route = route
    })
  })
}
```
在init里面对于HTML5History和HashHistory，进行`history.transitionTo`而history是在前文提到的VueRouter里面实例化的，` history.getCurrentLocation()`对于hash模式，就是`window.location.hash`#符号后面的地址；而` history.setupListeners()`则是监听hashchange事件，并执行`history.transitionTo`

## 路由匹配
看看transitionTo如何实现：
```javascript
transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const route = this.router.match(location, this.current)
  this.confirmTransition(route, () => {
    this.updateRoute(route)
    onComplete && onComplete(route)
    this.ensureURL()
  // ...
  }, err => {
  // ...
  })
}
```
在transitionTo中，对于hash模式，传参是路径字符串(location)，和监听的回调函数(onComplete/onAbort)；第一步调用VueRouter实例的match方法，返回一个匹配的route对象。在介绍route对象之前，需要先了解`create-route-map.js`，里面的路由字典生成函数createRouteMap，其返回:
```javascript
return {
  pathList,
  pathMap,
  nameMap
}
```
pathList:是自然是示例中routes的path集合，pathMap则是每个path对应的路由记录对象字典，nameMap则是每个name对应的路由记录对象字典；路由记录对象里面其他选项都较好理解，里面的regex用了'path-to-regexp'模块，可以对路由记录对象里面的path处理为正则表达式，方便和当前路由进行配对；另外路由记录里面还有parent选项，当routes下面某个路由有children的时候parent指的就是上一级的路由记录对象。
回过头来，继续看match方法，该方法传入的参数是当前路由hash部分和current对象，current对象可以追溯到route.js里面的`Object.freeze(route)`，返回的是冻结了的路由对象，值得注意的是这个路由对象的matched，matched数组是所有传入createRoute的record路由记录对象及其所有父路由记录对象。在所有初始化的过程中，this.current的path就是'/'。
match里面现实如下：
```javascript
  function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location
    if (name) {
    // ...
    } else if (location.path) {
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // no match
    return _createRoute(null, location)
  }
```
normalizeLocation方法则是对当前hash和当前路由对象做比较，生成path，query，hash三个键以及`_normalized: true`，_normalized可以用于判断是否已经对当前hash和路由对象对比过了，在match的else if语句里面，可以看到对pathlist进行遍历，存储的路由记录对象的regex和生成的path对比，若能匹配上，则对location对象的params为path里面解析出来的参数；最后match会返回_createRoute函数，该函数在匹配的路由记录对象没重定向和别名时，会返回一个路由对象。而这个路由对象和match传参里面的current同出自createRoute方法，返回的结构自然也是一样的，于是就有猜想this.current会不会赋值为normalizeLocation生成的location呢？结果发现还真是这样。

## 确认切换以及_route劫持
上面提到transitionTo中执行的路由确认，生成新的路由对象route，接着confirmTransition结构如下所示：
1. 创建abort中止方法，判断当前current对象是否和路由对象route是相同路由，如果是则返回中止函数
2. 创建执行队列queue针对current和route，按需执行
3. 创建迭代器iterator，在iterator里面执行hook，而hook是queue队列中的函数
4. 执行runQueue，迭代上文3中的iterator，并在最后执行回调
confirmTransition中的queue队列如下:
```javascript
const queue: Array<?NavigationGuard> = [].concat(
  // beforeRouteLeave方法
  extractLeaveGuards(deactivated),
  // 全局路由切换前动作
  this.router.beforeHooks,
  // beforeRouteUpdate方法
  extractUpdateHooks(updated),
  // beforeEnter方法
  activated.map(m => m.beforeEnter),
  // 异步组件
  resolveAsyncComponents(activated)
)
```
其中在Vue-router的[官方文档](https://router.vuejs.org/zh-cn/advanced/navigation-guards.html)里面介绍了组件内部的方法beforeRouteEnter，beforeRouteUpdate ，beforeRouteLeave，可以对应queue里面的两个方法，而queue里面的beforeEnter，是写在routes里面的方法名beforeEnter；至于文档里面提到的beforeRouteEnter，则对应runQueue方法内部，执行的extractEnterGuards方法，也是最后执行的钩子；
迭代器iterater的是否进入下一步迭代，是由传入hook里面的to来确定的(这个to为何物？要具体到每个方法的next函数传参)。
在transistorTo中，传给confirmTransition的除了route，还有onComplete，确认切换的回调函数，代码如下：
```javascript
// confirmTransition的onComplete方法
this.updateRoute(route)
onComplete && onComplete(route)
this.ensureURL()

// fire ready cbs once
if (!this.ready) {
  this.ready = true
  this.readyCbs.forEach(cb => { cb(route) })
}
```
好奇的你估计会问怎么onComplete里面还有个onComplete，后面这个回调是transistorTo自己的，也就是我们前文提到的`history.setupListeners`，至于updateRounte方法，则如下所示：
```javascript
updateRoute (route: Route) {
  const prev = this.current
  this.current = route
  this.cb && this.cb(route)
  this.router.afterHooks.forEach(hook => {
    hook && hook(route, prev)
  })
}
```
将获得的路由匹配中创建的路由对象route指向this.current，这也涉及到我们前面所说的两者都是由createRoute生成的；this.cb，该方法在init初始里面的末端有涉及如下：
```javascript
history.listen(route => {
  this.apps.forEach((app) => {
    app._route = route
  })
})
```
Vue实例化的时候，也初始化history.cb，实现对_route的赋值修改，但是其并没有在初始化的时候执行，Vue实例化中history.cb的赋值是在transitionTo之后的，也就是在updateRoute之后，但是在后面的路由跳转中，因为history.cb已经初始化，则会执行history.cb()。这也就实现了install过程里面对$route的数据劫持，其返回this._routerRoot._route就是route路由对象。
至于ensureURL，这个就神奇了，Vue-router中是以最新的路由对象为标准来修改hash的，为了确保window.location.hash的正确性，会在确认切换路由回调里面再次确认当前hash是否与当前路由对象的记录一直，不一致的话，以最新的路由对象为标准再次修改window.location.hash。
在Vue实例化中beforeCreate有一下一句：
```javascript
Vue.util.defineReactive(this, '_route', this._router.history.current)
```
defineReactive这是Vue里面观察者劫持数据的方法， 而这里是劫持_route，当_route触发setter方法的时候，则会通知到依赖的组件；

# 组件
在install的过程里面已经将router-link和router-view两个组件注册好了，稍微看一下源码就不难发现，这两个组件用的都是render方法渲染组件
对于router-link，默认标签tag为a标签，也是h函数的第一个参数，而数据对象data，有on和attrs，on是router-link里prop过来的事件，默认为click事件；而attrs处理时候，调用了`router.resolve(this.to, current, this.append)`在index.js里面的resolve方法也是调用了match方法，返回匹配的路由route，虽然和transistorTo方法里match传参格式不同，但是结果都是返回路由对象route。
在h函数创建Vnode的时候，data.class还会根据传参，当前路由来设置对应的class样式。
router-link里面还会自动创建a标签，并且当click事件触发的时候会调用内部的handler函数，当props的replace为false的时候，会触发transitionTo方法，并切换路由，点击a标签当然要触发跳转，而该transitionTo的回调则是修改window.location.hash的方法，从而修改地址栏的hash。当然由于前文提到的在Vue实例化过程中，我们在transitionTo的回调里面用了setupListeners去监听hashchange事件，所以在hashchange监听函数里面也会调用transitionTo方法，但是因为此时路由对象已经是最新的得了，所以不会进一步切换。

对于router，值得注意的部分是registerRouteInstance，也是最开始的install里面提到的，beforeCreate和destroyed都可能触发这个方法。registerRouteInstance其功能和路由对象里面的match：记录路由对象的instances相关联，就是会将对当前的router-view组件添加到对应的路由记录的instance里面，并在router-view组件destoryed的时候将该instance置为undefined；而这个instance的主要作用是在confirmTransition中的queue中使用到的，以及[issue#750](https://github.com/vuejs/vue-router/issues/750)里面提到的。

# History
上文提到的都是HashHistory下的，当然其实还有HTML5History模式，HTML5History顾名思义，用的HTML5的特性，老版本的浏览器会有兼容问题，所以默认情况下是hash模式，可以自己手动开启；
HTML5提供了两个api:
1. history.pushState()
2. history.replaceState()
分别添加和更新浏览器的历史纪录，pushState方法会在transitorTo的回调里面调用，类似于hash模式下的pushHash，而replaceState则类似与replaceHash方法。在init初始化的时候，还有HTML5History还有直接对事件popstate监听，popstate类似于hashchange事件，同样的也会有transitionTo调用，主要作用也是监听浏览器的前进后退功能，基本上是大同小异的；

至于AbstractHistory就更简单了，不是用于浏览器的，自然没有window.location的负担，没有浏览器的后退前进按钮，所以历史浏览记录用个数组和index代替就好了。实现简单，这里就不再谈了

ps: 附上Vue-router 0.4.0 src/transition.js里面对router-view切换时候组件处理的思路，2.7.0版本已经没有这部分注释了
>  A router view transition's pipeline can be described as
    follows, assuming we are transitioning from an existing
    <router-view> chain [Component A, Component B] to a new
    chain [Component A, Component C]:
     A    A
     | => |
     B    C  
    1. Reusablity phase:
      -> canReuse(A, A)
      -> canReuse(B, C)
      -> determine new queues:
         - deactivation: [B]
         - activation: [C]  
    2. Validation phase:
      -> canDeactivate(B)
      -> canActivate(C)
    3. Activation phase:
      -> deactivate(B)
      -> activate(C) 
    Each of these steps can be asynchronous, and any
    step can potentially abort the transition.
   
   
参考资料
1. [ajax与HTML5 history pushState/replaceState实例](http://www.zhangxinxu.com/wordpress/2013/06/html5-history-api-pushstate-replacestate-ajax/)