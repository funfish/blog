### 前言
前面介绍到 diff 方法，但是我们只是从简单的例子开始的，并没有用到组件，而组件才是最重要的部分，毕竟一切的一切可以是组件。

### 组件 Component
先看看 Preact 输出的 Component 长什么样子：
```javascript
export function Component(props, context) {
  this._dirty = true;
  this.context = context;
  this.props = props;
  this.state = this.state || {};
}

extend(Component.prototype, {
  setState(state, callback) {
    let s = this.state;
    if (!this.prevState) this.prevState = extend({}, s);
    extend(s, typeof state==='function' ? state(s, this.props) : state);
    if (callback) (this._renderCallbacks = (this._renderCallbacks || [])).push(callback);
    enqueueRender(this);
  },
  forceUpdate(callback) {
    if (callback) (this._renderCallbacks = (this._renderCallbacks || [])).push(callback);
    renderComponent(this, FORCE_RENDER);
  },
  render() {}
});
```

平时使用组件的时候，大致都是这样 `class Clockwarp extends Component` 通过 `extends` 来实现继承 `Component`，有点 `prototype` 的意思。在 `Component` 类里面有 `state/props/setState/render`，其中 `setState` 方法先判断 `state` 是不是函数，也就是这种写法：`this.setSate((preState, props) => {})` 这个时候才会会执行 `state` 方法，如果有回调，会被 push 到 `_renderCallbacks` 里面。在看看 `enqueueRender`:
```javascript
let items = [];
export function enqueueRender(component) {
  if (!component._dirty && (component._dirty = true) && items.push(component)==1) {
    (options.debounceRendering || defer)(rerender);
  }
}
export function rerender() {
  let p, list = items;
  items = [];
  while ( (p = list.pop()) ) {
    if (p._dirty) renderComponent(p);
  }
}
export const defer = typeof Promise=='function' ? Promise.resolve().then.bind(Promise.resolve()) : setTimeout;
```

`enqueueRender` 方法是为了延迟当前的组件的再渲染，采用的是 Promise方法，如果没有就用 `setTimeout` 代替，当然 `Promise.resolve()` 之后调用的 `then` 实现上是要优先于  `setTimeout` 的。

### 组件 diff 机制
上面代码可以看到 `setState/forceUpdate` 最后都会调用 `renderComponent` 方法，看名字就知道是渲染组件的意思，但是在介绍 `renderComponent` 之前，先看看上篇博客里面介绍的，`diff` 过程里面，对于组件的处理：
> 2. vnode 是 Component的形式，调用 buildComponentFromVNode 方法，最后会返回处理过的 dom 节点。
如若是组件，则会调用 `buildComponentFromVNode` 方法，而实际上， `buildComponentFromVNode` 最后也是会调用 `renderComponent` 方法，所以先看看 `buildComponentFromVNode` 的实现：
```javascript
export function buildComponentFromVNode(dom, vnode, context, mountAll) {
  let c = dom && dom._component,
    originalComponent = c,
    oldDom = dom,
    isDirectOwner = c && dom._componentConstructor===vnode.nodeName,
    isOwner = isDirectOwner,
    // props就是vnode的attribute/children/nodename.defaultProps
    // 传递最新鲜的porps，常见的子组件更新，都是依赖于props变化
    props = getNodeProps(vnode);
  while (c && !isOwner && (c=c._parentComponent)) {
    isOwner = c.constructor===vnode.nodeName;
  }
  // 如果vnode 和 dom 是由同类的组件生成则直接 setComponentProps，当然还需要!mountAll || c._component 成立。
  if (c && isOwner && (!mountAll || c._component)) {
    setComponentProps(c, props, ASYNC_RENDER, context, mountAll);
    dom = c.base;
  }
  else {
    // dom由组件生成，而vnode和生成dom的组件实例不是同一构造器生成的。则说明要卸载当前组件，替换上新的。
    if (originalComponent && !isDirectOwner) {
      unmountComponent(originalComponent);
      dom = oldDom = null;
  }
  // 根据nodeName生成新的组件 c。
  c = createComponent(vnode.nodeName, props, context);
  // 如果该类组件没有卸载过，而存在dom来diff，则将c.nextBase指向dom，后面做diff用
    if (dom && !c.nextBase) {
      c.nextBase = dom;
      oldDom = null;
    }
    setComponentProps(c, props, SYNC_RENDER, context, mountAll);
    dom = c.base;
    if (oldDom && dom!==oldDom) {
      oldDom._component = null;
      recollectNodeTree(oldDom, false);
    }
  }
  return dom;
}
```

`buildComponentFromVNode` 有几个概念是要分清楚的，如果 `dom` 是由一个组件生成渲染的，则 `dom._component` 是指向渲染出 `dom` 的组件实例。而生成的组件实例的 `base` 属性又会指向 `dom` 节点。初次渲染时候会直接是进入下面的 `else` 语句。对于不同的组件则先卸载之前的组件，让后生成新的组件 `c`，`nextBase` 指的是卸载的同类组件的 `base` 属性，也就是上个该类组件生成的 dom 节点。为什么要这样做呢？答案是提高效率。假设组件替换是这样的 `A -> B ->A`，在 A 组件卸载的时候，就会将 A 生成的 dom 节点缓存下来，当 B 组件卸载，A 组件再次渲染的时候，这个时候就会用上之前 A 组件生成的 dom 节点，与这次 A 组件渲染出的做 diff对比，这样看是不是很高效？可以看看 `createComponnet`:
```javascript
export function createComponent(Ctor, props, context) {
  let list = components[Ctor.name],
    inst;
  if (Ctor.prototype && Ctor.prototype.render) {
    inst = new Ctor(props, context);
    Component.call(inst, props, context);
  }
  else {
    inst = new Component(props, context);
    inst.constructor = Ctor;
    inst.render = doRender;
  }
  if (list) {
    for (let i=list.length; i--; ) {
      if (list[i].constructor===Ctor) {
        inst.nextBase = list[i].nextBase;
        list.splice(i, 1);
        break;
      }
    }
  }
  return inst;
}
```

这里的 `components` 是缓存的卸载的组件集合。通过简单的判定，将生成的新组件的 `nextBase` 指向卸载的同类组件的 `nextBase`，其实也是后者 'base'了。

在创建组件之后是 `setComponentProps`:
```javascript
export function setComponentProps(component, props, opts, context, mountAll) {
  if (component._disable) return;
  component._disable = true;
  // 将vnode传入的attribute/children/nodename.defaultProps里面的ref/key传给组件。
  if ((component.__ref = props.ref)) delete props.ref;
  if ((component.__key = props.key)) delete props.key;

  if (!component.base || mountAll) {
    if (component.componentWillMount) component.componentWillMount();
  }
  else if (component.componentWillReceiveProps) {
    component.componentWillReceiveProps(props, context);
  }

  if (context && context!==component.context) {
    if (!component.prevContext) component.prevContext = component.context;
    component.context = context;
  }

  if (!component.prevProps) component.prevProps = component.props;
  component.props = props;
  component._disable = false;
  if (opts!==NO_RENDER) {
    if (opts===SYNC_RENDER || options.syncComponentUpdates!==false || !component.base) {
      renderComponent(component, SYNC_RENDER, mountAll);
    }
    else {
      enqueueRender(component);
    }
  }
  if (component.__ref) component.__ref(component);
}
```

`setComponentProps` 里面现实执行组件的 `componentWillMount/componentWillReceiveProps` 方法。将 `props` 传给组件的 `props`。接着是进行 `renderComponent` 方法，这个时候传参已经是 `component, opts, mountAll, isChild`，没有`vnode` 了。`renderComponent` 方法在 `setState` 也有提到，是更新组件的最重要的步骤，而 `renderComponent` 关键点就是会修改组件的 `base` 也就是 dom，接下来看看 `renderComponent` 方法实现：
```javascript
export function renderComponent(component, opts, mountAll, isChild) {
  if (component._disable) return;
  let props = component.props,
    state = component.state,
    context = component.context,
    previousProps = component.prevProps || props,
    previousState = component.prevState || state,
  previousContext = component.prevContext || context,
  // base和nextBase是 dom，或者undefined/null
    isUpdate = component.base,
    nextBase = component.nextBase,
    initialBase = isUpdate || nextBase,
    initialChildComponent = component._component,
    skip = false,
  rendered, inst, cbase;
  
  if (isUpdate) {
    component.props = previousProps;
    component.state = previousState;
    component.context = previousContext;
    // 通过shouldComponentUpdate判断是不是要执行更新
    if (opts!==FORCE_RENDER
      && component.shouldComponentUpdate
      && component.shouldComponentUpdate(props, state, context) === false) {
      skip = true;
    }
    else if (component.componentWillUpdate) {
      component.componentWillUpdate(props, state, context);
    }
    component.props = props;
    component.state = state;
    component.context = context;
  }

  component.prevProps = component.prevState = component.prevContext = component.nextBase = null;
  component._dirty = false;

  if (!skip) {
    // 渲染结果先。传入poprs，state。
    rendered = component.render(props, state, context);

    if (component.getChildContext) {
      context = extend(extend({}, context), component.getChildContext());
    }

    let childComponent = rendered && rendered.nodeName,
      toUnmount, base;
    // 如果render结果还是组件的话，继续render就好了，但是首次render要建立父组件和子组件关系。
    if (typeof childComponent==='function') {
      let childProps = getNodeProps(rendered);
      inst = initialChildComponent;
      // 说明执行过了，和上次渲染一样，inst是rendered的实例class。为再次进入的时候，要求是同一组件，key也要一样。
      if (inst && inst.constructor===childComponent && childProps.key==inst.__key) {
        // 对于组件更新，则重新获取其props，再来就好了
        setComponentProps(inst, childProps, SYNC_RENDER, context, false);
      }
      // 第一次进来的时候/不相同的时候
      else {
        toUnmount = inst;

    component._component = inst = createComponent(childComponent, childProps, context);
    // nextBase 的传递
        inst.nextBase = inst.nextBase || nextBase;
        inst._parentComponent = component;
        setComponentProps(inst, childProps, NO_RENDER, context, false);
        // 指明inst是子关系，不用重复继承和执行钩子函数与componentDidMount，因为已经可以了
        renderComponent(inst, SYNC_RENDER, mountAll, true);
      }

      base = inst.base;
    }
    else {
      cbase = initialBase;
      // 如果有component._component，说明上次里面生成的rendered是function，而cbase是该function生成的节点，
      // 在本轮中rendered已经不是function了，故需要设置子组件_component为null。
      toUnmount = initialChildComponent;
      if (toUnmount) {
        cbase = component._component = null;
      }

      if (initialBase || opts===SYNC_RENDER) {
        // 需要先置为null，在重新指向
        if (cbase) cbase._component = null;
        base = diff(cbase, rendered, context, mountAll || !isUpdate, initialBase && initialBase.parentNode, true);
      }
    }
  // 生成的dom和原本的dom不一样，并且子组件也不一样的情况下
    if (initialBase && base!==initialBase && inst!==initialChildComponent) {
      let baseParent = initialBase.parentNode;
      // 如果执行了上面else里的diff，那diff中initialBase，已经被rendered替代了，initialBase没有挂载在任何节点上了，并且parentNode为null了。
      // 所以下面的命令是针对上面typeof childComponent==='function'的情况的？但是在该情况的renderCompnent里面也会进入diff，
      // 所以是根本进不来的？
      if (baseParent && base!==baseParent) {
        // 这里是用本轮的base 替换掉上轮的base
        baseParent.replaceChild(base, initialBase);
        // 如果没有_component，为何要清理base？有base/nextbase，自然应该是要有_component的。
        // 已经replace了，为何还要remove base呢？
        if (!toUnmount) {
          // 防止在recollectNodeTree过程里面_component被unMounted，而是直接remove节点就好了
          initialBase._component = null;
          recollectNodeTree(initialBase, false);
        }
      }
    }
    // 发生的情况只能是rendered从组件变为普通vnode或者其他组件，所以要卸载掉子组件。
    if (toUnmount) {
      unmountComponent(toUnmount);
    }

    component.base = base;
    if (base && !isChild) {
      let componentRef = component,
        t = component;
      while ((t=t._parentComponent)) {
        (componentRef = t).base = base;
    }
    // 指明dom的_component 指向组件，而_componentConstructor指向组件的构造器
      base._component = componentRef;
      base._componentConstructor = componentRef.constructor;
    }
  }
  if (!isUpdate || mountAll) {
    mounts.unshift(component);
  }
  else if (!skip) {
    if (component.componentDidUpdate) {
      component.componentDidUpdate(previousProps, previousState, previousContext);
    }
    if (options.afterUpdate) options.afterUpdate(component);
  }
  if (component._renderCallbacks!=null) {
    while (component._renderCallbacks.length) component._renderCallbacks.pop().call(component);
  }

  if (!diffLevel && !isChild) flushMounts();
}
```

`renderComponent` 函数比较长。首先是第一个 if 语句里面，判断是否是 `isUpdate`，判断依据是有无 `component.base`，初次加载组件的时候，组件本身是没有 `base` 属性，最多才有 'nextBase' 属性，所以 `isUpdate` 用来区分是否是更新组件，如果是的话，执行组件的 `shouldComponentUpdate` 与 `componentWillUpdate` 方法，并判断是否执行后面一长串的渲染。

`rendered = component.render(props, state, context)` 先是得出要渲染的 VNode，如果 `rendered` 还是组件，则还要进行判断是否前后两次渲染子组件不一样或则是初次渲染。如果不是组件则主要是执行 `diff` 函数，生成新的 dom 节点 `base`。 最后由于自组件变更或则消失，则卸载子组件。将生成的 `base` dom 节点指向组件的 `base` 属性，而 dom 节点还要新增 `_component/_componentConstructor` 属性。最后如果是初次加载则将组件放入初次加载组件的队列里面，准备执行 `componentDidUpdate afterUpdate`。底部的 `flushMounts` 则是，当是最顶部的一次 diff 递归进入尾声了，就执行 `options.afterMount` 和所有初次加载组件的 `componentDidMount` 方法。另外还有 `component._renderCallbacks`，在组件状态变化，也就是在用 `setState` 的时候，如果存在第二个参数 `callback`， 则会在这个时候执行 `callback`。

### 回收机制
之前最早遇到回收问题是出现在 diff 函数上面，经常可以看到 `recollectNodeTree(dom, true)` 这句话，看看 `recollectNodeTree` 方法：
```javascript
export function recollectNodeTree(node, unmountOnly) {
  let component = node._component;
  if (component) {
    unmountComponent(component);
  }
  else {
    if (node[ATTR_KEY]!=null && node[ATTR_KEY].ref) node[ATTR_KEY].ref(null);
    // 移除node节点
    if (unmountOnly===false || node[ATTR_KEY]==null) {
      removeNode(node);
    }
    // 再来一次遍历
    removeChildren(node);
  }
}
export function removeChildren(node) {
  node = node.lastChild;
  while (node) {
    let next = node.previousSibling;
    recollectNodeTree(node, true);
    node = next;
  }
}
```

这里可以看到，对于非组件，则可以要先执行节点的 ref，这个 ref是什么？在组件 `setComponentProps` 最后一行代码还有个 `component.__ref`。其实这两个都是一样的，都是节点 `attribute` 里面的 `ref` 属性，也就是说组件初次加载的时候，或则回收组件/Dom 的时候都会执行，如果不仅仅是卸载还会在父节点上移除 `node`。从而实现回收。最后的 `removeChildren` 只是遍历用的。基本上只要涉及到老节点的回收都会用到 `recollectNodeTree` 里面。在 `innerDiffNode` 最后也会对没有用的节点回收。除了节点 diff 的问题外，还有组件卸载的时候，也会调用 `removeChildren` 方法：
```javascript
export function unmountComponent(component) {
  if (options.beforeUnmount) options.beforeUnmount(component);
  let base = component.base;
  component._disable = true;
  if (component.componentWillUnmount) component.componentWillUnmount();
  component.base = null;
  let inner = component._component;
  if (inner) {
    unmountComponent(inner);
  }
  else if (base) {
    if (base[ATTR_KEY] && base[ATTR_KEY].ref) base[ATTR_KEY].ref(null);
    component.nextBase = base;
    removeNode(base);
    collectComponent(component);
    removeChildren(base);
  }
  if (component.__ref) component.__ref(null);
}
```

上面可以看出卸载组件的时候，会调用 `componentWillUnmount` 方法，接着执行 ref ，将卸载的组件生成的节点转移到 `nextBase` 上面，再执行 `removeChildren`。

最后还有一个贯穿所有 diff 机制的传参 `mountAll`，具体作用就算是组件更新，也将其作为初次加载，就是能执行 `componentWillMount 与 componentDidMount` 方法。

### 总结
这次的 Preact 之旅就到这里，代码虽少，但是还是活灵活现的展示了 diff 功能。源码里面有着无数行空行，以及代码解释，然而整体大小才 1000 行多点，min 之后更是只有 10kb 大小，相当袖珍。大家有机会还是去接触一下。