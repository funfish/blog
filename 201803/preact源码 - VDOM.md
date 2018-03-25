## 前言
在工作上开始用 React 开发已经有四个多月了，不禁想看看 React 和 Vue 本质上有什么区别。当然一个是 jsx 文件，一个是 vue 文件，两个处理起来肯定是不一样的。想了想以后项目发展越来越大，肯定是要以 React 为主体的，深入了解 React 是必须的，尤其是 React 已经发展到 React 16 了，新特性都不晓得怎么用呢。为了减少初学习 React 源码的陡度，想着还是从 Preact 开始好了，毕竟后者声称兼容 React 而且，关键是体积小！

## Babel 与 JSX
在进入 Preact 的介绍前，又必须要说说虚拟 DOM，这虚拟 Dom 听着神奇，在 Vue 里面也主角。所以必要介绍一下。

在 React 官网的开始学习教程上有下面这段代码
```javascript
ReactDOM.render(
  <h1>Hello, world!</h1>,
  document.getElementById('root')
);
```

刚看到时候可以很明显的发现这是给 `ReactDOM.render` 传入两个参数，一个是 H1 标签，一个是 DOM 节点，后者很好理解，就是最平常的获取一个 id 为 root 的 DOM 节点，可是前者又是什么？在 Javascript 的所有类型里面是没有这种东西的，难不成是自己落后了？于是在 Chrome 的控制台里面打印，看一下，来一发 `console.log(<div></div>)`，结果立马返回 `Uncaught SyntaxError: Unexpected token <` 这就提示说语法错误了，那是那里出错了呢？试试了试在 Preact 的 index 页面打印 `console.log(<div></div>)`，代码却能够正常跑起来，而且 `console.log(typeof <div></div>)` 居然是 `'object'` 奇了怪了？

难道两处代码是不一样的？难不成 Preact 做了什么特别处理？随查看 Preact 的源码，但是没有任何迹象，传入的参数根本就没有做什么处理，而且传进来马上就会语法错误了，怎么可能执行呢？

最后打开控制台，查看 Sources 的打包文件，发现原来 `console.log(typeof <div></div>)` 变成了下面：
```javascript
console.log(_typeof((0, _preact.h)('div', null)));

// 转变一下结果就是
console.log(typeof preact.h('div', null));
```

这。。。。。又是为什么呢？中间怎么这么多变化呢？在生成的代码阶段就不是简单的 `div` 了，那是哪里发生的呢？忽然想起以前讲 webpack 介绍的 Babel 降级问题，难道这里也是？查看 Babel 的配置文件 .babelrc，如下：
```javascript
{
  "presets": ["es2015", "stage-0"],
  "plugins": [
    ["transform-react-jsx", { "pragma": "h" }]
  ]
}
```

看看下面的配置，在看看转义的那句话，这不就是将 jsx 用 h 程序转变的意思吗？在 package.json 里面也看到了 `babel-plugin-transform-react-jsx`，这个包是用于将 JSX 转换为 React 函数，用法则是在 `.babelrc` 里面设置 `"pragma": "dom"`，后者是替换的函数名字，默认是 `React.createElement`，而在 Preact 中则需要设置为 h 函数。官网里面也有介绍到对于 Babel 5 和 Babel 6 的设置。

## h 函数
上文中的 `div` 变成了 h 函数的实现，h 函数在 Vue 里面也经常可以看到，更不要提 React，那 `h` 函数是什么呢？
```javascript
export function h(nodeName, attributes) {
	let children=EMPTY_CHILDREN, lastSimple, child, simple, i;
	for (i=arguments.length; i-- > 2; ) {
		stack.push(arguments[i]);
	}
	if (attributes && attributes.children!=null) {
		if (!stack.length) stack.push(attributes.children);
		delete attributes.children;
  }
  // 存在子节点，如text节点或者其他h函数
	while (stack.length) {
		if ((child = stack.pop()) && child.pop!==undefined) {
			for (i=child.length; i--; ) stack.push(child[i]);
		}
		else {
			if (typeof child==='boolean') child = null;

			if ((simple = typeof nodeName!=='function')) {
				if (child==null) child = '';
				else if (typeof child==='number') child = String(child);
				else if (typeof child!=='string') simple = false;
			}
      // 最终子节点都会推入children里面
      // 而对于简单节点则直接相加就好了
			if (simple && lastSimple) {
				children[children.length-1] += child;
			}
			else if (children===EMPTY_CHILDREN) {
				children = [child];
			}
			else {
				children.push(child);
			}

			lastSimple = simple;
		}
  }
  // p就是最终生成的 Vnode
	let p = new VNode();
	p.nodeName = nodeName;
	p.children = children;
	p.attributes = attributes==null ? undefined : attributes;
	p.key = attributes==null ? undefined : attributes.key;
  // 对生成的Vnode，都用options的vnode方法来处理
	// if a "vnode hook" is defined, pass every created VNode to it
	if (options.vnode!==undefined) options.vnode(p);

	return p;
}

export function VNode() {}
```

上面代码还是很好理解的，下面举个简单例子，对于 `h('DIV', {id: 'abc'}, h('SPAN', null))` 怎会被转换为以下 Vnode：
```javascript
{
  nodeName: 'DIV',
  attributes: {id: 'abc'},
  children: [
    {
      nodeName: 'SPAN',
      attributes: undefined,
      children: null,
      key: undefined,
    }
  ]
  key : undefined,
}
```

在 `h` 函数里面的 `while` 循环里面做了一件很特别的事情，本来只要将子节点统统 push 到 children 里面就好了，但这里通过相邻节点是否是简单方式 `simple/lastSimple`，若果是数字，字符串等，则直接合并在一起。这样有什么好处呢？减少要 diff 的节点，就是减少计算量，毕竟虚拟 Vnode 里面主要内容也是字符串等简单类型。而 `VNode` 构造函数，则是简单的一个实例而已。

## 总结
这里介绍了 VNode 的一些入门东西，但是这是后面学习 diff 机制的基础，也能够清晰知道 Preact 的操作对象不是 Document 上面的节点，而是一个个虚拟的 VNode，