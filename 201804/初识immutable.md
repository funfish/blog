## 前言
Immutable.js 出来已经有很长一段时间了，只是日常项目中一直用不上。一个是平时没有怎么接触，不了解，另外一个是团队的开发节奏和习惯已经稳定下来了，要改变也不容易，了解一下也不差。

不可变的数据一旦生成，就无法被改变，这就要求数据可持久化。可是日常中的引用类型的数据，一旦改变了，就改变了，谈什么持久化数据结构呢？

## 接触immutable
感受一下immutable的不同：
```javascript
// 原本写法：
let a = [1, 2];
let b = a;
b[0] = 0;
console.log(a[0]); // 0
console.log(a === b); // true

// immutable 里面的写法
import { List } from 'immutable';
let a = List([1, 2]);
let b = a.set(0, 0);
console.log(a.get(0)); // 1
console.log(b.equals(a)) // false
```

可以直观的看到在没有使用 immutable 之前， a 与 b 都是同一引用，一旦其中的数据修改了，另一个也会跟着变化，毕竟 a 是等于 b 的。而到了 immutable，就不一样了，当你设置 `a.set(0, 0)` 的时候，并不会修改 a ，而是返回一个新的数组赋予到 b，所以 a 还是原来的 a。

**这就是 immutable 致力解决的疼点：持久化数据解构。**平时使用数据的时候，可能一不小心就会把引用类型数据修改了，导致一些隐藏比较深的问题。尤其是对后来的开发者/维护者而言，意义就更重大了。

在 immutable 里面常见的数据类型有：
1. List： 有序可重复列表，类似于 Javascript 的 Arry；
2. Map：键值对，类似于 Javascript 的 Object；

其 API 还是很亲民的，基本上字如其名，大致接触一下就能够了解了。常用的用法这里就不做介绍了，需要了解，请移步[官方文档](http://facebook.github.io/immutable-js/)。

## List 类型
Liit 类型可以说是常用的了，尤其是和 Javascript 里面的 Arry 类似。看一下 List 里面的主要部分：
```javascript
function makeList(origin, capacity, level, root, tail, ownerID, hash) {
  const list = Object.create(ListPrototype);
  list.size = capacity - origin;
  list._origin = origin;
  list._capacity = capacity;
  list._level = level;
  list._root = root;
  list._tail = tail;
  list.__ownerID = ownerID;
  list.__hash = hash;
  list.__altered = false;
  return list;
}
```
一个空内容的 List，也就是 `List()`，是 `makeList(0, 0, SHIFT))` 生成的，而 SHIFT 的值为 5。而传入 List 括号里面的值存储在 _root 和 _tail 里面，而 _root 和 _tail 里面的数据又是有一下结构：
```javascript
// @VNode
constructor(array, ownerID) {
  this.array = array;
  this.ownerID = ownerID;
}
```

在 List 列表的生成里，就可以看到，持久化数据的形成，比如看看为何将数据分别保持在 _tail 和 _root 里面，以及是用何种方式保存；

以设置一个数据为例子，如： List([1]).set(0, 0):
```javascript
set(index, value) {
  return updateList(this, index, value);
}

function updateList(list, index, value) {
  ...
  let newTail = list._tail;
  let newRoot = list._root;
  const didAlter = MakeRef(DID_ALTER);
  if (index >= getTailOffset(list._capacity)) {
    newTail = updateVNode(newTail, list.__ownerID, 0, index, value, didAlter);
  } else {
    newRoot = updateVNode(
      newRoot,
      list.__ownerID,
      list._level,
      index,
      value,
      didAlter
    );
  }
  ...
  return makeList(list._origin, list._capacity, list._level, newRoot, newTail);
}
// SIZE = 1 << SHIFT，而 SHIFT = 5
function getTailOffset(size) {
  return size < SIZE ? 0 : ((size - 1) >>> SHIFT) << SHIFT;
}
```

可以看出在 updateList 里面，通过 _capacity 来判断，以 32位 为尺度将 _capacity 切分开来，当 index 大于 `((size - 1) >>> SHIFT) << SHIFT` 时候，更新 _trail, 否则更新 _root。例如： 当 _capacity 为 33，index 为 32 及其以下的时候，修改的都是 _root，否之则修改 _tail。这个是很好理解的，当数据量达到一定程度的时候，针对靠后的数据单独存储，而靠前的数据放在 _tail，分类处理。只是特别之处在与 _tail 的设计。

### List 里面的 32 阶 RRB-Tree
_tail 里面采用的是 RRB-Tree 的形式存储数据。这个什么树的先不介绍，先看看看怎么形成，形成的是什么，继续看上面的 updateList 方法，里面用到了 updateVNode 方法来生成 _tail:
```javascript
function updateVNode(node, ownerID, level, index, value, didAlter) {
  // MASK = 31;
  const idx = (index >>> level) & MASK;
  ...
  let newNode;
  if (level > 0) {
    const lowerNode = node && node.array[idx];
    const newLowerNode = updateVNode(
      lowerNode,
      ownerID,
      level - SHIFT,
      index,
      value,
      didAlter
    );
    if (newLowerNode === lowerNode) {
      return node;
    }
    newNode = editableVNode(node, ownerID);
    newNode.array[idx] = newLowerNode;
    return newNode;
  }
  ...
  newNode = editableVNode(node, ownerID);
  if (value === undefined && idx === newNode.array.length - 1) {
    newNode.array.pop();
  } else {
    newNode.array[idx] = value;
  }
  return newNode;
}
function editableVNode(node, ownerID) {
  if (ownerID && node && ownerID === node.ownerID) {
    return node;
  }
  return new VNode(node ? node.array.slice() : [], ownerID);
}
// @VNode
constructor(array, ownerID) {
  this.array = array;
  this.ownerID = ownerID;
}
```

在这里可以可能到 value/newLowerNode 是赋值给 `newNode.array[idx]`， 而 idx 并不是等于 index。以 _capacity 为 65 的 List 为例子，其 _level 为 5。当 index 为 60 的时候，有以下行为：
1. 第一次进入 idx = 1，level = 5，将会有 _tail.array[1] = newLowerNode;
2. 计算 1 里面的 newLowerNode，第二次进入，此时 level = 0，idx = 28，于是有 newLowerNode.array[28] = value；
可以看出这里生成个**二维的数组，其中每个子节点的长度最大为 32，于是这就构成了一个 32阶的树结构**。至于为什么阶长是 32，在代码中是这么解释的：
> Resulted in best performance after ______?

这不是逗我嘛。。。什么都没有写好吧，而且 github 里面最早的版本也是这么写的。。。。最后还好找到是有测试实验数据证明 32 也就是 SHIFT 为 5 是最佳实践。至于为什么要采用这种结构，不是本文要考虑的，将在下篇中给出来。

最后数据生成的结构如下如所示：
![](https://github.com/funfish/blog/blob/master/images/immutableTree.PNG)

get 方法里面也是差不多的，通过 _capacity 来判断。

## Map 类型结构
同样的先看看一个空的 Map() 的结构：
```javascript
function makeMap(size, root, ownerID, hash) {
  const map = Object.create(MapPrototype);
  map.size = size;
  map._root = root;
  map.__ownerID = ownerID;
  map.__hash = hash;
  map.__altered = false;
  return map;
}
```

这里就没有 _tail 结构的，所有数据都放在 _root 里面。只是 _root 里面的数据并非总是简单，采用了 Trie Nodes 的方式存储数据。 immutable 里面将对象的键值对转换为数组表示，即 [key, value] 的形式。

存储的数据分为以下级别：
1. ArrayMapNode，最简单方式，当键值对不超过8个的时候(不含嵌套的键值对)，采用这种方式，所有键值对保存在 entries 里面。同时 get/set 方法都较为简单，直接遍历一下获取就好了；
2. BitmapIndexedNode，当 ArrayMapNode 里面元素超过8个的时候，_root 会转变为 BitmapIndexedNode，BitmapIndexedNode 的子节点是 ValueNode。而 BitmapIndexedNode 里面查/增/改元素，都需要用到 bit-map(位图)算法，BitmapIndexedNode.bitmap 存储的是键名和存储顺序的位图信息。例如 get 方法，通过 BitmapIndexedNode.bitmap，以及 key 名就可以获取该键值对的排列序号，从而获取到相应的 ValueNode；
3. HashArrayMapNode，ValueNode 个数超过 16 个的时候，_root 会转变为 HashArrayMapNode 对象，其子元素为 ValueNode。而当 ValueNode 个数超过 32 个的情况时，HashArrayMapNode 的亲子元素可以是 HashArrayMapNode/BitmapIndexedNode，而 BitmapIndexedNode 的亲子元素可以是 BitmapIndexedNode/ValueNode。由此看来巨量的键值对，将有 HashArrayMapNode/BitmapIndexedNode/ValueNode 组合而成，而每个 HashArrayMapNode 最多有32个亲子元素，BitmapIndexedNode 最多有16个亲子元素。 HashArrayMapNode 类对应带的 count，代表其子元素的数量。当需要读取的时候，直接键名的哈希值，就能够实现了。。。。好像有点简单呀；
4. HashCollisionNode，这种情况相当少，比如当 {null: 1} 和 {undefined: 2} 的键名是完全不同的，但是他们的键名的哈希值却是一样的。这使得后则 {undefined: 2} 创建的时候才会有 HashCollisionNode。HashCollisionNode 包含了这两个相同哈希键名数据；

> // The hash code for a string is computed as
> // s[0] * 31 ^ (n - 1) + s[1] * 31 ^ (n - 2) + ... + s[n - 1],
(多键值的整个读写离不开键名哈希计算，而其关键步骤如上，n为键名从左到右字母的 charCodeAt)。

## withMutations 操作
每次对 immutable 的数据类型操作的时候，都会返回一个新的数据，这样就存在一个问题，如果需要像原本一样对一个数据不断0操作，如不断的向Arry 里面 push 新值。对于 List，每次都返回一个新 List 岂不是多了很多中间变量，多了很多蹩脚的操作？于是就有了 withMutations 操作。如：
```javascript
let a = List();
Array.apply(null, { length: 10 }).forEach(item => a = a.push(item));
// 上面可以用withMutations操作
a = a.withMutations(list => 
  Array.apply(null, { length: 10 }).forEach(item => list.push(item))
)
```

上面是不使用 withMutations 的方法，每次 push 之后都需要重新对 a 重新赋值，才能保证正确性。而用了 withMutations 方法之后，不在需要创建中间 List，减少了复杂度，基本上运行时间比前一种快上好几倍。当然 withMutations 不会修改 a，所以需要将中间 List 赋值到 a。

为什么 withMutations 会有这样的效果呢？以 A(List) 为例子，当使用 withMutations 时候，其首先，生成的中间变量 B(List) 的属性值就是 A 的属性值，没有任何更改，当然如果 A 没有 __ownerID， B 的 __ownerID 会被设置上。但是对 B 进行 push/pop/set 等等操作的时候，A 其实是没有受到任何影响的。
```javascript
function editableVNode(node, ownerID) {
  if (ownerID && node && ownerID === node.ownerID) {
    return node;
  }
  return new VNode(node ? node.array.slice() : [], ownerID);
}
```

如在 withMutations 操作里面 set 数据的时候， 上面 if 语句是无法通过的，因为 A 的数据里面是没后 ownerID 的，相当于给 A 的数据拷贝一次到 B 里面，但是在 withMutations 里面再次 set 的时候，这时上面的 if 语句是可以通过的，于是又能节省不少运行时间。类似的 Map 也是相似的，这里就不再提到了。

## 其他
immutable.js 类型的一层层继承关系，刚开始的看的时候会觉得有点乱，不管哪里都有 extends。细细看下来还是挺不错的。Immutable.js 还有个 Seq 类型，其特点就是懒。懒的意思是，直到获取使用了才开始计算。一开始觉得很神奇，居然有用的时候才计算的类型。。。well，后来一看这得益于一系列的 API，主要就是把调用方法函数数据什么的统统用闭包给存起来。。。。。。。好吧。

ps：immutable.js 最神奇的地方还是在于其数据结构，下篇文章将会好好讲数据结构。

