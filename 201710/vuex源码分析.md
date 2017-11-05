# 前言
前文分析了Vue-router，感觉后劲十足，于是开始分析Vuex。在项目上，Vuex也是常客。它可以很好的管理状态，尤其是跨组件的时候，Vue的单向数据流使得子组件无法修改prop，经常用$emit和$on的话组件是要多难看就多难看。当组件切换，数据需要缓存总不能一直依赖于向上级组件emit传递数据吧？如果要更好的管理状态，Vuex是个很好的选择。Vuex代码量较Vue-router少了很多，而且也没有flow的校验机制，看起来更加习惯了。这里介绍的Vuex版本号为2.4.1。

# 从示例开始
```javascript
Vue.use(Vuex)
const state = {
  count: 0
}

const mutations = {
  increment (state) {
    state.count++
  },
  decrement (state) {
    state.count--
  }
}

const actions = {
  increment: ({ commit }) => commit('increment'),
  decrement: ({ commit }) => commit('decrement'),
}
// getters are functions
const getters = {
  evenOrOdd: state => state.count % 2 === 0 ? 'even' : 'odd'
}
export default new Vuex.Store({
  state,
  getters,
  actions,
  mutations,
})
```
上面示例基本上包含了最常用的mutations，getters和actions了。可以发现这一切从Vue.use(Vuex)开始的，对Vue.use不熟悉的可以看上一篇的Vuex-router中对Vue.use的介绍。
Vuex中用到了install方法来提供Vuex的使用环境。和Vue-router不同，Vuex的主要代码功能都在store.js文件里面(这对查阅代码友好度明显提到了不少)。install过程里面用到了Vue.mixin，并用到了beforeCreate钩子，使得Vue实例化和组件加载的时候都可以调用到钩子。设计如下：
```javascript
const version = Number(Vue.version.split('.')[0])
if (version >= 2) {
  Vue.mixin({ beforeCreate: vuexInit })
} else {
  const _init = Vue.prototype._init
  Vue.prototype._init = function (options = {}) {
    options.init = options.init
      ? [vuexInit].concat(options.init)
      : vuexInit
    _init.call(this, options)
  }
}

function vuexInit () {
  const options = this.$options
  // store injection
  if (options.store) {
    this.$store = typeof options.store === 'function'
      ? options.store()
      : options.store
  } else if (options.parent && options.parent.$store) {
    this.$store = options.parent.$store
  }
}
```
可以看到这里对Vue的版本分别做了处理，本版是2.0.0及以上的都会采用Vue.mixin的方法，而低版本的，则将修改Vue的内部_init方法，来添加$store至根。高级别的版本则采用mixin的方法，同样也是添加`this.$store`。在Vue-router里面是采用数据劫持的方法，来通知更新，顺便提供this.$router，对于状态管理而言，数据劫持显然是不需要的，仅仅提供入口`this.$store`就够了，这样为全局提供了访问store对象的方法，可以轻松得使用`this.$store.commit, this.$store.state`之类的方法。

# Store
Store.js里面最主要的就是Store类，这个也是之前提到的`this.$store`对象。先看看constructor方法：
在构造里面先判断有无使用install方法，没有则intall一下，接着是断言有无安装Vue，是否支持Promise和是否是通过new创建Store的实例。另外在install过程里面还有是否重复安装Vuex的断言，这个场景会发生在已经先使用Vuex了，但是没有用`Vue.use(Vuex)`来显式安装Vuex，如果再加上`Vue.use(Vuex)`就会有这样的提示，尤其是在开发环境和生产环境配置中。
Store初始化过程，有`this._modules = new ModuleCollection(options)`，这个_modules就是Store集合的意思了。Vuex有modules的概念，允许对store进行分割形成不同的模块，每个模块都可以有自己的state，getter，mutation和action，甚至还可以嵌套子模块。于是将这些模块包括根模块一起放入modules里面。this._modules的一个重要api就是注册添加一个模块：
```javascript
register (path, rawModule, runtime = true) {
  if (process.env.NODE_ENV !== 'production') {
    assertRawModule(path, rawModule)
  }

  const newModule = new Module(rawModule, runtime)
  if (path.length === 0) {
    this.root = newModule
  } else {
    const parent = this.get(path.slice(0, -1))
    parent.addChild(path[path.length - 1], newModule)
  }

  // register nested modules
  if (rawModule.modules) {
    forEachValue(rawModule.modules, (rawChildModule, key) => {
      this.register(path.concat(key), rawChildModule, runtime)
    })
  }
}
```
这里还可以看到this._modules.root就是根模块，并且对于子模块的，还会被添加到父模块parent的_children对象里面；到这里可以发现this.modules.root和原先的store很像，只是单独分离出state，并且将子模块改为了_children关系，并将_rawMoudule赋值为整个传过来模块，同时为this._modules和每个module都添加不少方法，这些方法自然是为后面做准备的。
在谈commit和dispatch方法之前，先看看后面的模块安装和StoreVM的设置
```javascript
installModule(this, state, [], this._modules.root)
resetStoreVM(this, state)

function installModule (store, rootState, path, module, hot) {
  const isRoot = !path.length
  const namespace = store._modules.getNamespace(path)
  // 命名空间字典的添加
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module
  }
  // 设置state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }
  const local = module.context = makeLocalContext(store, namespace, path)
  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })
  // 下面省略部分是通过module提供的方法分别对action和getter进行registe
  // 以及对子模块modules的遍历式得注册mutation/action/getter

  // ...
}
```
对于modules而言，[官方文档](https://vuex.vuejs.org/zh-cn/modules.html)有介绍到，模块内部的 action，mutation和getter是注册在全局命名空间的，如果想要独立的空间，比如有命名重复的情况下，可以使用`namespaced: true`来注册单独的空间；同时访问的时候也也要加上模块的名字，否则否则无法定位到。
接着看state的设置，对于if条件语句，若是子模块并且非hot，会获取子模块的亲父级模块，并通过Vue.set方法将该子模块的state添加到亲父模块state里面，这是响应式的，会被Vue劫持到。后面部分就是对action/getter/mutation的注册添加了，这部分后面在讲。


后面是resetStoreVM：
```javascript
function resetStoreVM (store, state, hot) {
  const oldVm = store._vm
  // bind store public getters
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    computed[key] = () => fn(store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })
  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent
  // enable strict mode for new vm
  if (store.strict) {
    enableStrictMode(store)
  }
  // 如果存在oldVM对其进行销毁
  // ... 
}
```
刚看到这里时候可能会惊奇何时来的_vm？事实上这个_vm正是这里的核心，_vm是个Vue实例，并将`_vm.data.$$state`指向的option中的state。细心的话还可以发现在Store类中，其中的Store.state：如下
```javascript
get state () {
  return this._vm._data.$$state
}
```
state返回的正是`_vm.data.$$state`，这个也就是平时所用的`this.$store.state`。观察resetStoreVM还可以发现通过遍历wrappedGetters，来将wrappedGetters中的方法通过_vm.computed的形式添加到`store.getters`里面，这么复杂的办法有什么好处呢？而且为什么只是专门处理getter，没有对mutation和action进行这样的处理？getter的方法是对state进行处理提取过滤，而computed是依赖于data的，当data更新的时候computed就自动计算，同样这里也是的，当state更新的时候，通过computed的方法，getter不就自动计算更新了吗？只是这样就有点麻烦。。。。。要新建一个Vue实例，关于_vm，更多的可以点[这里](https://github.com/vuejs/vuex/issues/849)

## commit和dispatch
在介绍之前先看看前面忽略的，在installModule方法里面对mutation/getter/action等方法的添加机制。
对于registerMutation：
```javascript
function registerMutation (store, type, handler, local) {
  // 内部的_mutations[type]保存对应的mutattion方法
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function wrappedMutationHandler (payload) {
    // 在mutation方法里面传入local.state和payload，
    // wrappedMutationHandler只需要payload，符合commit时，仅需传入type和payload
    handler.call(store, local.state, payload)
  })
}
```
上面方法添加了store._mutations[type]，而handler传参里面的local.state又是什么呢？回头看可以发现这里调用了makeLocalContext，生产local变量，makeLocalContext代码这里就不贴出来了。local.state就是对应path的state变量，只是是通过数据劫持的方法获得的，代码中说明getters和state对象都必须要懒加载，因为可能被vm更新影响到，这里是不是指_vm重新创建的时候造成的影响呢？由于namespaced的问题，local里面的dispatch和commit都做了特别处理，但是还是使用store的dispatch和commit的方法，只是传参做了修改。
对于registerAction，类似与mutation，采用了store._actions[type]来保存handler数组，但由于action有用于异步的情况，所以若返回的action不是Promise类型，则进行Promise包装。同时action的传参不是local.state，而是传入local的本身的所有字段和store的getters以及state，这也符合action的基本应用。
对于registerGetter，这里比较简单直接采用`store._wrappedGetters[type] = handler`的形式，而registerMutation是采用数组的形式。所以对于重复名字的getter就会有告警``[vuex] duplicate getter key: ${type}`。

回到commit方法和dispatch，在Store类构造的时候，有如下:
```javascript
this.dispatch = function boundDispatch (type, payload) {
  return dispatch.call(store, type, payload)
}
this.commit = function boundCommit (type, payload, options) {
  return commit.call(store, type, payload, options)
}
```
这里面定义commit方法和dispatch方法，这两个就是`$store.commit`和`$store.dispatch`，而commit这个方法处理起来也是比较简单，就是将_mutations里面对应方法名都执行一遍，并传递payload进去。同时还将_subscribers里面的函数都遍历执行。_subscribers是通过subscribe这个api添加进来:
```javascript
subscribe (fn) {
  const subs = this._subscribers
  if (subs.indexOf(fn) < 0) {
    subs.push(fn)
  }
  return () => {
    const i = subs.indexOf(fn)
    if (i > -1) {
      subs.splice(i, 1)
    }
  }
}
```
该方法可以添加订阅函数，每当mutation执行的时候，所有订阅函数都会执行，值得一提的时候在devtool.js文件里面用到了:
```javascript
store.subscribe((mutation, state) => {
  devtoolHook.emit('vuex:mutation', mutation, state)
})
```
当使用devtoolHook的时候(这个也涉及到Vue官方推荐的浏览器插件工具Vue devtools)能在每个mutation动作结束后，触发vuex:mutation事件，并在devtools插件内打印动作
还可以看出这个subscribe设计很巧妙，subscribe直接运行是添加订阅函数，而其返回函数就是disSubscribe，就是将订阅函数去除掉，由于不常用，所以就没有直接给出api了，厉害的很。

dispatch该动作类似的，也是调用之前存在_actions里的handlers，只是由于handles可能有多个，并且是异步的原因，若是多个的话需要用`Promise.all`来执行；

# 其他Api
日常用的比较多的是registerModule/unregisterModule，两个过程是类似的，注册新模块的时候需要重新installModule和resetStoreVM，这个时候就会将老的_vm delete掉，重新实例化Vue给到_vm。
mapState/mapMutations/mapGetters/mapActions等api结构类似。以mapState为例子:
```javascript
export const mapState = normalizeNamespace((namespace, states) => {
  const res = {}
  normalizeMap(states).forEach(({ key, val }) => {
    res[key] = function mappedState () {
      let state = this.$store.state
      let getters = this.$store.getters
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapState', namespace)
        if (!module) {
          return
        }
        state = module.context.state
        getters = module.context.getters
      }
      return typeof val === 'function'
        ? val.call(this, state, getters)
        : state[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```
normalizeNamespace来调整参数，再通过normalizeMap将传入的state调整为`{ key, val: key }`结构，并根据情况返回。这几个api还是很容易懂的。

# 结束
一周下来写了两篇源码分析，Vuex的代码和Vue-router相比还是很良心的，没有Vue-router里面那么多弯弯绕绕，Vuex简单明了多了。