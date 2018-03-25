### 前言
每次看到有人谈起 React 的 diff 机制的时候，总觉得很厉害的样子，所以自然这里也是立马就想介绍 diff 机制。

### diff 机制
以下面为例子来介绍：
```javascript
import { h, render } from 'preact';

render((
  <div id="foo">
    <span>Hello, world!</span>
    <button onClick={ e => alert("hi!") }>Click Me</button>
  </div>
), document.body);
```

render 方法的实现如下：
```javascript
import { diff } from './vdom/diff';

export function render(vnode, parent, merge) {
	return diff(merge, vnode, {}, false, parent, false);
}
```

这里面 `merge` 是需要对比的 `VNode` 节点，`vnode` 就是传入节点，`parent` 则是挂载的节点。可以发现传入到 `render` 方法里面，最终还是会调用 `diff` 方法。看看 `diff` 的实现：
```javascript
export function diff(dom, vnode, context, mountAll, parent, componentRoot) {
  // 初始化的时候才进入，每次进入diff函数都会自增，递归的level
	if (!diffLevel++) {
    // SVG处理，判断是否是在SVG里面diff。
		isSvgMode = parent!=null && parent.ownerSVGElement!==undefined;
		// 只有dom存在，且DOM没有__preactattr_属性，才为true；一般也就是初次进来的时候
		hydrating = dom!=null && !(ATTR_KEY in dom);
	}

	let ret = idiff(dom, vnode, context, mountAll, componentRoot);
	// 挂载生成ret到parent去，也就是document.body
	if (parent && ret.parentNode!==parent) parent.appendChild(ret);

	if (!--diffLevel) {
		hydrating = false;
		// 执行options.afterMount方法，和所有初次挂载的组件的componentDidMount方法
		if (!componentRoot) flushMounts();
	}

	return ret;
}
export function flushMounts() {
	let c;
	while ((c=mounts.pop())) {
		if (options.afterMount) options.afterMount(c);
		if (c.componentDidMount) c.componentDidMount();
	}
}
```

**`render` 传参 `merge`，也就是 `diff` 方法传参 `dom`，是用来和 `vnode` 做 `diff` 的前节点。**可以看到上面 `diff` 方法主要作用是生成 `ret`，并将其挂载到 `parent` 上面去，并在最顶部的递归层，一般 `componentRoot` 是 `undefined/false`，可以执行所有已经加载的组件的 `componentDidMount` 方法。 `idiff` 的实现如下：
```javascript
function idiff(dom, vnode, context, mountAll, componentRoot) {
	let out = dom,
		prevSvgMode = isSvgMode;
	// 如果vnode是空，直接处理为''
	if (vnode==null || typeof vnode==='boolean') vnode = '';
	// 若果vnode是字符串或则数字，便捷方式
	if (typeof vnode==='string' || typeof vnode==='number') {
		// 通过splitText方法来判断是不是文本节点。若果dom是文本，就直接直接dom的nodeValue替换为 vnode
		if (dom && dom.splitText!==undefined && dom.parentNode && (!dom._component || componentRoot)) {
			if (dom.nodeValue!=vnode) {
				dom.nodeValue = vnode;
			}
		}
		else {
			// 如果dom不是文本节点，就创建vnode的文本节点，并在dom的parent上替换掉dom。
			out = document.createTextNode(vnode);
			if (dom) {
				if (dom.parentNode) dom.parentNode.replaceChild(out, dom);
				recollectNodeTree(dom, true);
			}
		}
    out[ATTR_KEY] = true;
    // 这里的out是个dom节点，而不是VNode
		return out;
	}
  // 如果传入的 vnode 是函数，也就是class实例，是个组件，就返回buildComponentFromVNode的执行结果
	let vnodeName = vnode.nodeName;
	if (typeof vnodeName==='function') {
		return buildComponentFromVNode(dom, vnode, context, mountAll);
	}
	isSvgMode = vnodeName==='svg' ? true : vnodeName==='foreignObject' ? false : isSvgMode;
	vnodeName = String(vnodeName);
	// 将vnode里面填充上dom的子元素,只有dom存在且为文本的时候才不进入。
	if (!dom || !isNamedNode(dom, vnodeName)) {
		// 生成 vnodeName 的元素节点
		out = createNode(vnodeName, isSvgMode);
		if (dom) {
			// 将dom里面的节点都添加到out里面
			while (dom.firstChild) out.appendChild(dom.firstChild);
			// 最后直接用out替换掉dom
			if (dom.parentNode) dom.parentNode.replaceChild(out, dom);
			recollectNodeTree(dom, true);
		}
	}

	let fc = out.firstChild,
		props = out[ATTR_KEY],
		vchildren = vnode.children;
	// 给dome节点，将attributes属性添加到__preactattr_属性里面
	if (props==null) {
		props = out[ATTR_KEY] = {};
		for (let a=out.attributes, i=a.length; i--; ) props[a[i].name] = a[i].value;
	}
	// hydrating为false的时候，若vnode只有一个节点string就直接替换掉out的第一个子节点。自然out后面的其他节点都会被除去
	if (!hydrating && vchildren && vchildren.length===1 && typeof vchildren[0]==='string' && fc!=null && fc.splitText!==undefined && fc.nextSibling==null) {
		if (fc.nodeValue!=vchildren[0]) {
			fc.nodeValue = vchildren[0];
		}
	}
	// 只要有vchildren和out有子节点就来innerDiffNode，
	else if (vchildren && vchildren.length || fc!=null) {
		innerDiffNode(out, vchildren, context, mountAll, hydrating || props.dangerouslySetInnerHTML!=null);
	}
	//把vnode的属性都传到out，还有props吧。
	diffAttributes(out, vnode.attributes, props);
	//还原之前的isSvgMode值
	isSvgMode = prevSvgMode;

	return out;
}
```

这里的 `idiff` 方法，看着比较复杂，实际上还是对传参 vnode 进行分类判断，分为下面几种情况：
1. 简单的类型 string/number 之类的，直接用 vnode 替换掉 dom 元素，返回 vnode 的文本节点。
2. vnode 是 Component的形式，调用 buildComponentFromVNode 方法，最后会返回处理过的 dom 节点。
3. vnode 只有一个节点，且为文本，并且 dom 情况也是一样的，就替换掉 nodeValue。如若不是则调用 innerDiffNode 方法，来 diff dom 和 vnode。

这里面第一种情况是最基础的，`vnode` 是文本，就要替换掉对应的 `dom`，第二种情况是组件的方式，这里先不谈。第三种比较麻烦，是多个子节点情况，如若 `dom` 存在并且为文本节点，`out` 变量就是 `dom` 这个文本节点，否则 `out` 会是 `vnodeName` 的元素空节点，随后将`dom` 子节点转移到 `out`下面。接着设置 `out` 的 `__preactattr_` 属性。

在第三种情况时，对于前一种简答情况，如果 `dom` 是 `<div>123</div>`，而 vnode 的 children 属性为文本的话，例如：`vnode = {nodeName: 'SPAN', Children: ['sb'].....}`，则生成的 `out` 为 `<span>sb</span>`，这种是简单的情况。复杂情况下需要调用到 `innerDiffNode` 方法。在介绍 `innerDiffNode` 之前，先看看 `idiff` 方法最下面的 `diffAttributes` 方法：
```javascript
function diffAttributes(dom, attrs, old) {
	let name;
	for (name in old) {
		if (!(attrs && attrs[name]!=null) && old[name]!=null) {
			setAccessor(dom, name, old[name], old[name] = undefined, isSvgMode);
		}
	}
	for (name in attrs) {
		if (name!=='children' && name!=='innerHTML' && (!(name in old) || attrs[name]!==(name==='value' || name==='checked' ? dom[name] : old[name]))) {
			setAccessor(dom, name, old[name], old[name] = attrs[name], isSvgMode);
		}
	}
}
```

`diffAttributes` 方法就是将 `vnode` 里面 `attribute` 和 `props` 属性添加到 `out` 里面，最后返回的是 `out` 元素而不是 vnode！`setAccessor` 基本就是些条件语句，根据出入的属性名，来分类处理，看看就好了。就这样**将 vnode 里面的 `attribute` 属性添加到 `out` 里面**。

### innerDiffNode
`idiff` 第三种情况的复杂情况下下会调用 `innerDiffNode` 方法，实际上就是对 `vnode` 的子元素和 `out` 的子元素进行递归对比。先看看 `innerDiffNode` 的实现：
```javascript
function innerDiffNode(dom, vchildren, context, mountAll, isHydrating) {
	let originalChildren = dom.childNodes,
		children = [],
		keyed = {},
		keyedLen = 0,
		min = 0,
		len = originalChildren.length,
		childrenLen = 0,
		vlen = vchildren ? vchildren.length : 0,
		j, c, f, vchild, child;
	// 对比的dom有children的时候
	if (len!==0) {
		for (let i=0; i<len; i++) {
			let child = originalChildren[i],
				props = child[ATTR_KEY],
				// key就是指平时写map循环的时候，数组里面的子vnode用来区分的key。如果child由component生成，则用component的__key。否则用props传入的key，如<div key={1}></div>这里面的key。
				key = vlen && props ? child._component ? child._component.__key : props.key : null;
			// child有__preactattr_属性，也就是之前有添加过__preactattr_属性，可以看idff方法里面的第三种。
			if (key!=null) {
				keyedLen++;
				keyed[key] = child;
			}
			// 如果child存在__preactattr_，或则child为文本，就将Dom的child缓存到children里面
			else if (props || (child.splitText!==undefined ? (isHydrating ? child.nodeValue.trim() : true) : isHydrating)) {
				children[childrenLen++] = child;
			}
		}
	}
	// vnode子节点长度不0
	if (vlen!==0) {
		for (let i=0; i<vlen; i++) {
			vchild = vchildren[i];
			child = null;
			// 试着去寻找vchild和keyed里面保存的相同之处，也就是key，如果vnode的子节点的key值，在out里面能找到的话，说明他们是应该一一对应的，child就是out里面对应的节点。
			let key = vchild.key;
			if (key!=null) {
				if (keyedLen && keyed[key]!==undefined) {
					child = keyed[key];
					keyed[key] = undefined;
					keyedLen--;
				}
			}
			// 按out里面存在的子节点，依次和vnode里面的子节点排排坐对比。依次来，如果是相同的node类型，就找到了对应的out节点。并且后面的undefined设置和自加自减操作都是为了优化循环；
			else if (!child && min<childrenLen) {
				for (j=min; j<childrenLen; j++) {
					if (children[j]!==undefined && isSameNodeType(c = children[j], vchild, isHydrating)) {
						child = c;
						children[j] = undefined;
						if (j===childrenLen-1) childrenLen--;
						if (j===min) min++;
						break;
					}
				}
			}
			// 对比生成新的child
			child = idiff(child, vchild, context, mountAll);

			f = originalChildren[i];
			if (child && child!==dom && child!==f) {
				if (f==null) {
					dom.appendChild(child);
				}
				else if (child===f.nextSibling) {
					removeNode(f);
				}
				else {
					dom.insertBefore(child, f);
				}
			}
		}
	}
	// 下面两个步骤都是移除节点
	if (keyedLen) {
		for (let i in keyed) if (keyed[i]!==undefined) recollectNodeTree(keyed[i], false);
	}
	while (min<=childrenLen) {
		if ((child = children[childrenLen--])!==undefined) recollectNodeTree(child, false);
	}
}
```

`innerDiffNode` 的目的就是要 `vnode` 的每个 `vchild` 和能与其对应上的 `out` 下面的 `child` 进行对比，也就是调用 `diff` 方法，从而实现子节点之间的对比。在 `innerDiffNode` 里面对比找出 `child` 的过程，看上面代码中的解释就好了。在通过 `idiff` 方法生成新的 `child` 后，`child` 会被加入到 `out` 里面。从而一步步将 `vnode` 的 `children` 移入作为 `out` 的节点。在遍历了所有的 `vnode` 的 `children` 之后，还需要对下面两种 `out` 的子节点移除：
1. out 子节点里面带 `key` 属性的节点，如果没有匹配上 vnode 的 children ，需要移除；
2. 再次进入循环的 child 节点或则是首次进入非空字符串的文本节点，如果没有匹配上 vnode 的 children 也会被移除掉。

### 总结
diff 机制基本就是不断的遍历子节点和 vnode，来实现对比不同。将 vnode 里面的内容添加到 dom 里面，而将 dom 里面不需要的多余的子节点移除掉。所以这里还需要理解整体的移除机制，以及组件生成对比的机制，将在下篇文章里面介绍到。