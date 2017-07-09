# 前言
相信大家都或多或少接触过Vue，先前就有人介绍学习Vue的源码，提到旧版本的源码行数只不过一千多行，可以一个一个commit学习下去。前段时间为了找下家，一直在用Vue做作品，效率也较以前原生JavaScript要快上许多，后来工作上手了，不禁想看看Vue源码长什么样子的，只是从第一个commit开始读起来较为费时，而Github上面Vue项目能够找到的最早branch是[Vue 0.10](https://github.com/vuejs/vue/tree/0.10)，然而在发布版本里面，可以发现最好早一版本是[0.6.0版本](https://github.com/vuejs/vue/tags?after=v0.7.2)。本文介绍也是从该本版开始，该版本较0.10的要上少40%左右。

# 目标
本文目的在于最短时间内，介绍Vue0.6.0的核心模块源码，数据双向绑定，computed计算属性等。
看完本文之后，能明白Vue框架到底做了些什么，并自行构建一个早期Vue框架的简化版本。

# 准备
在介绍Vue 源码之前，需要了解Object.defineProperty,通过这个方式，在获取对象字段值时，调用getter方法，而设置对象字段值时候，调用setter方法，来实现数据劫。Object.defineProperty是Vue的基础内容，至今不熟悉的请一定要看(Object.defineProperty MDN)[https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty]

# 思路
对于
```javascript
data: {
    a: 1,
    c: 2
}
```
如果你要设置data.a的值，并更新渲染DOM，会怎么做呢？
了解Object.defineProperty神奇的setter和getter数据劫持功能后，自然而然是有以下步骤：
1. data.a的访问器属性setter里，通知所有和data.a有关的instances；
2. instances在源码中指的是Directive构造函数生成的实例集合，其中每个实例的this.el是一个指向data.a有关联的node节点，如<div>{{a}}</div>里面的textNode节点。如果有多个这样的节点，data.a就对应多个directive实例。在进行data.a重新赋值的时候，setter被调用，执行data.a下每个directive实例的update函数，实现局部DOM更新;
3. directive实例和data.a数据的对应关系可以用Binding构造函数说明，每个Binding都含有instances数组，该数组存放的就是directive实例集合，而data.a只有一个Binding

# 实现
看了源码不难发现，创建的实例new Vue({}),指的是ViewModel构造函数，而ViewModel函数如下：
```javascript
function ViewModel (options) {
    // just compile. options are passed directly to compiler
    new Compiler(this, options)
}
所以comilper.js才是生成Vue的核心所在，下面是简化了compiler构造函数
function Compiler(vm, options) {
    let compiler = this;

    options = compiler.options = options || makeHash()

    var el = compiler.setupElement(options)

    var scope = options.scope;
    if (scope) utils.extend(vm, scope, true)

    compiler.vm = vm

    var observables = compiler.observables = [],
        computed    = compiler.computed    = [];

    compiler.bindings = makeHash()

    compiler.setupObserver()

    let keyPrefix;
    for(let key in vm) {
        keyPrefix = key.charAt(0)
        if (keyPrefix !== '$' && keyPrefix !== '_') {
            compiler.createBinding(key)
        }
    }

    compiler.compile(el, true)

    while (i--) {
        binding = observables[i]
        Observer.observe(binding.value, binding.key, compiler.observer)
    }

    if (computed.length) DepsParser.parse(computed)
}
```
上面过程分为三个阶段：

## 准备设置
1. compiler.setupElement(options)通过querySelector返回Vue的el节点
2. 我们熟悉的data，computed和metheds，在该版本中都是options.scope里面，也就是说米有data，computed和methods之分，统一在scope里面，utils.extend(vm, scope, true)则是将scope的所有字段扩展到vm下面，方便后面处理；
3. compiler.bindings = makeHash()则是给compiler的bindings设置为{}；

## observer实现
这里实现每个key的数据劫持和事件监听
4. compiler.setupObserver() 创建compiler.observer监听'get', 'set', 'mutate'动作
5. compiler.createBinding(key)，将所有的scope里面的字段key都创建Binding实例，而compiler.bindings[key]则指向创建的binding，同时若字段不含有'.'则会进行define方法，该方法先是将刚刚创建的Binding{key].value设置为vm[key]，实现每个key的Binding实例都保存下value值。如果该value是对象或则数组，如scope.a为对象，则push到compiler.observables里面；最后现实数据劫持Object.defineProperty
在define里面的Object.defineProperty，setter方法: 若新值不等于旧值，则重新对bindings[key].value设置为新值，同时进行Observer.unobserve旧值和Observer.observe新值，只有value是对象或则数组的时候，才会进入Oberver.oberver/unobserve方法，那对于普普通通的data.a等于字符串这种呢？前文说好的设置data.a的时候通知所有和data.a有关的instances，又发生在那里呢？
答案在在setter里面还通过compiler.observer触发了set事件,set事件里面就有bindings[key].update，而正如上文所说的bindings[key].instances就是directive实例，而directive实例又是什么时候添加进去的呢，且看下面

## compiler.compile的实现
6. compiler.compile(el, true)解析DOM节点，通过遍历所有的节点，对不同的属性名字如v-text采用不同指令，如果是有效的字段就返回Directive实例，这个实例会通过CompilerProto.bindDirective方法处理Directive和Binding的关系。如<div>{{a}}</div>, 会先生成一个Directive实例，该实例包含textNode节点，v-text的更新指令，在bindDirective方法中，则会判断vm是否有bindings['a'],如果有的话，直接使用compiler.bindings['a']，并将directive实例push到bindings['a'].instances，接着执行directive.update(value),而这个value就是前文提到的binding['a'].value,至此完成DOM节点和scope的关系

那对于scope.a为对象，如scope.a = {b: 1},这里的scope.a.b是无法在步骤5中实现的，只有scope.a会出现在步骤5，那scope.a.b要如何创建binding呢？
在步骤5中，进入define方法后，若检测到scope.a是对象，则将binding['a'] push到compiler.observables里面，到了步骤6，遍历到{{a.b}}的时候，会创建bindings['a.b'],但是不会进入define方法里面，无法对bindings['a.b']赋值，和Object.defineProperty数据劫持，没有赋值没有数据劫持，那又要如何实现{{a.b}}以及后面的重新设置a.b的值呢？

7. 对observables遍历，通过Observer.observe方法对scope.a下的所有字段遍历，并在observer.js里面的bind方法，对所有的字段建立Object.defineProperty数据劫持，同时触发'set'事件，最后会传递到步骤4里面的事件监听'set'事件，执行bingdings['a.b'].update,现实DOM里面的数据首次展现

通过对于普通的socpe.a = 12,而言打印创建的Vue实例有如下图
![](https://github.com/funfish/blog/blob/master/images/Vue.PNG)
compiler下创建了a: Binding构造函数，而这个实例的instances包含了一个Directive，里面存放的value 12，

## computed原理
类似于observables，在compiler里面也会创建compiler.computed数组，在define方法里除了在compiler.observables.push(binding)外，若是对象，并且有$get方法，则是computed，这个版本里面，computed是有$get方法的，well...当然还有$set; 
```javascript
scope: {
    a: 1,
    c: {
        $get: function(){
            return this.a
        }
    }
}
```
8. DepsParser.parse(computed)，若computed存在元素，则调用DepsParser.parse，这个时候DOM里面的计算属性还没有赋值！只是创建了对应的binding，和observables的遭遇是一样的。。。。在 DepsParser.parse里面，当执行computed的$get函数的时候，如遇到依赖scope.a,则会调用步骤5里面为scope.a创建的数据劫持getter，这个方法里面触发'get'事件，在步骤4里面监听到'get'事件后，触发DepsParser.observer的'get'事件，而这个'get'事件作用则是将bindings['a']，bindings['c'].deps里面，同时bindings['a'].subs里面放入bindings['c'],为何要这样做呢？在初始化的时候是不必的毕竟$get已经可以获得正确的数值了，但是当重新设置scope.a的时候，就需要通知scope.c同步更新了，这个在bindings['a'].update里面实现

自此Compiler构造函数也就大体如此，具体细节还是要多看源码

## 监听数组
在compiler.observables里面，在Observer.observe里面，可以看到当处理对象的typeOf值是对象或则数组都会进行特别处理，若是对象上文已经提过处理方法，若是数组的话又会如何呢？
这里简要概述一下：通过ExpParser.parse方法，将bindings['a[0]'].value设{$get: newFunction('this.a; return this.a[0]')}的形式，后面就和computed类似了，这里有一点需要注意，在observer.js里面，重写了['push','pop','shift','unshift','splice','sort','reverse']等数组方法，当a.push(1)的时候，会触发mutate动作，执行bindings['a'].pub()，从而通知bindings['a[0]']的instances更新，具体内容可以自己去看源码
