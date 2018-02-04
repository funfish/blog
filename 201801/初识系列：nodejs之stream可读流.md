# 前言
这是初识系列的第一篇：stream可读流。刚接触stream的时候有点难以理解，在客户端开发，基本接触不到stream，顶多也就是文档下载的时候，后端返回文件流，这个和stream沾边的东西。如此神秘，自然成为了首个研究的对象。nodejs对象里面有可读流，可写流，还有可读可写流，像HTTP响应Response对象就是可读流，而服务端的是可写流，下面介绍一下可读流Readable。

# 基本
常见用到可读流的情景是用`fs.createReadStream(path[, options])`，并通过监听可读流的`data`与`end`事件来操作，或则是用pipe方法将可读流的数据流到可写流里面。

可读流里面有两个构造函数，一个是Readable，一个是ReadableState。先看看ReadableState的构造函数：
```javascript
function ReadableState(options, stream) {
  // ...省略部分代码
  // objectMode 对象流模式，返回的是对象而不是n字节缓存，hwm：高水位标志
  var hwm = options.highWaterMark;
  var defaultHwm = this.objectMode ? 16 : 16 * 1024;
  this.highWaterMark = hwm || hwm === 0 ? hwm : defaultHwm;
  this.highWaterMark = Math.floor(this.highWaterMark);

  //BufferList是可读流的缓冲区，其结构类似与C语言的链表，操作比数组快
  this.buffer = new BufferList();
  //缓冲区大小长度
  this.length = 0;
  // pipes：目标对象流，pipesCount为目标对象流长度
  this.pipes = null;
  this.pipesCount = 0;
  // flowing模式标志，ended为可读流结束标志，endEmitted：是否已经触发ended
  // reading：是否正在正在调用this._read方法
  this.flowing = null;
  this.ended = false;
  this.endEmitted = false;
  this.reading = false;
  // 异步标志置为true，用来控制'readable'/'data'事件是否立即执行
  this.sync = true;
  // 是否需要触发readable事件，
  this.needReadable = false;
  this.emittedReadable = false;
  this.readableListening = false;
  this.resumeScheduled = false;

  this.destroyed = false;

  // 目标流不处于drain状态时，等待drain事件的数量；
  this.awaitDrain = 0;
  // 是否正在读取更多数据，maybeReadMore函数
  this.readingMore = false;
  //.. 省略编解码相关部分
}
```
BufferList，就是读取过程中操作的缓冲池。ReadableState构造函数基本上是用来控制可读流的标志，其中最常见的就是flowing。可读流的工作模式分为flowing模式和pause模式。一般直接使用readable.pipe() 方法来消费流数据，因为它是最简单的一种实现，如果你想要精细控制，那就是通过控制flowing标志以及其他来实现。

## 添加chunk
可读流的数据从哪里来？或许最常见的就是用fs模块来createReadStream，然后直接读取就好了，似乎不涉及到chunk的添加过程。但是createReadStream又是如何创建可读流的呢？到最后还是需要Readable的push这个API，不断地`push(chunk)`，也就是往缓冲区添加数据。下面介绍一下push方法：
```javascript
function addChunk(stream, state, chunk, addToFront) {
  if (state.flowing && state.length === 0 && !state.sync) {
    stream.emit('data', chunk);
    stream.read(0);
  } else {
    // update the buffer info.
    state.length += state.objectMode ? 1 : chunk.length;
    if (addToFront) state.buffer.unshift(chunk);else state.buffer.push(chunk);

    if (state.needReadable) emitReadable(stream);
  }
  maybeReadMore(stream, state);
}
```
上面是addChunk方法，而Readable的push，会先检查objectMode，若不是objectMode，当压入的数据chunk是一个Buffer, Uint8Array或者string，objectMode就可以是any了，在readableAddChunk函数里面会`state.reading = false`添加块的过程并不是在执行_read方法。
回到addChunk方法，先看看else语句，可以发现添加chunk只是修改`state.length`，同时调用push/unshift来把chunk添加到缓冲区里面。并根据情况来触发readable事件。readable事件表明会有新的动态，要么有新的数据，要么到了流的尾部。而前面if语句里面，为flowing模式，并且缓冲区没有数据，且为同步模式下，才会触发data事件，并执行`read(0)`，`read(0)`在满足条件的情况下，只是简单触发readable事件而不会读取当前缓冲区，后面会介绍到。addChunk结束部分还调用了`maybeReadMore`，在read部分会介绍到。

## read
Readable.prototype.read方法，用于读取缓冲池里面的数据，其开始如下：
```javascript
  if (n === 0 && state.needReadable && (state.length >= state.highWaterMark || state.ended)) {
    debug('read: emitReadable', state.length, state.ended);
    if (state.length === 0 && state.ended) endReadable(this);else emitReadable(this);
    return null;
  }
```
在开始的时候如果`n=0`，并且符合其他条件，则会执行emitReadable，接着触发readable事件，并返回null，这个时候`read(0)`并不会触发缓冲区的数据读取，只是简单的触发readable事件，这个实现还是很巧妙的。
```javascript
  n = howMuchToRead(n, state);
  if (n === 0 && state.ended) {
    if (state.length === 0) endReadable(this);
    return null;
  }
  var doRead = state.needReadable;
  debug('need readable', doRead);
  // 数据比高水线低，那就需要读
  if (state.length === 0 || state.length - n < state.highWaterMark) {
    doRead = true;
    debug('length less than watermark', doRead);
  }

  if (state.ended || state.reading) {
    doRead = false;
    debug('reading or ended', doRead);
  } else if (doRead) {
    debug('do read');
    state.reading = true;
    state.sync = true;
    if (state.length === 0) state.needReadable = true;
    this._read(state.highWaterMark);
    state.sync = false;
    if (!state.reading) n = howMuchToRead(nOrig, state);
  }
```
这里面先是重新计算n，
当`n===NaN`的时候，读取缓冲区的第一个节点或则所有缓冲数据，
当`n<=state.length`返回n，并且当`n>hwm`的时候，重新计算高水位，为最小大于n的2^x，
当`n>state.length`的时候，返回0，并使得`state.needReadable=true`，
下面则是通过needReadable来判断是否执行_read方法，_read方法是需要自定义实现的，在方法里面手动添加数据到缓冲区里面，以便后面读取数据以及判断缓冲区长度。由于_read可能是同步的方法，修改缓冲区，所以需要重新评估n，以便后面获取数据。
后面部分，获取数据：
```javascript
  var ret;
  if (n > 0) ret = fromList(n, state);else ret = null;
  if (ret === null) {
    state.needReadable = true;
    n = 0;
  } else {
    state.length -= n;
  }
  if (state.length === 0) {
    if (!state.ended) state.needReadable = true;
    if (nOrig !== n && state.ended) endReadable(this);
  }
  if (ret !== null) this.emit('data', ret);

  return ret;
```
这一部分就简单了，就是获取数据，设置needReadable，并触发data事件，供可读流监听，操作chunk。

### needReadable的作用
在read方法里面，needReadable经常被修改为true，这个有什么用呢？
在`addChunk`里面若执行if语句里面的`read(0)`，由于需要`state.length>=htm`是不会触发readable事件的，相反执行else语句，添加chunk之后，就触发readable事件，并将所有的缓冲区数据读出来，相当于把之前的数据，和本次加的数据都读出来了。`addChunk`的最后面也会通过调用maybeReadMore来执行`read(0)`，其实现如下：
```javascript
function maybeReadMore(stream, state) {
  if (!state.readingMore) {
    state.readingMore = true;
    processNextTick(maybeReadMore_, stream, state);
  }
}
function maybeReadMore_(stream, state) {
  var len = state.length;
  while (!state.reading && !state.flowing && !state.ended && state.length < state.highWaterMark) {
    debug('maybeReadMore read 0');
    stream.read(0);
    if (len === state.length)
      // didn't get any data, stop spinning.
      break;else len = state.length;
  }
  state.readingMore = false;
}
```
当添加chunk的时候，若`state.lenght<hwm`则会执行`stream.read(0)`，从而肯定会执行_read方法，导致缓冲区增加，直到缓冲区长度超过高水线，同时最后一次调用`stream.read(0)`，也不会触发readable事件。
另外在`Readable.prototype.on`函数，readable事件的处理函数里面，异步调用`read(0)`。
ps：maybeReadMore用了异步调用`maybeReadMore_`，是为了让本轮循环里面调用的`_read`执行完先，在`_read`里面的每个添加chunk的步骤，都会执行maybeReadMore函数，若同步执行`maybeReadMore_`，缓冲区数据将会远远超标。

## 数据读取ended
当数据读取结束后，要显式的执行`push(null)/unshift(null)`来调用onEofChunk方法。在onEofChunk里面，会通过emitReadable处理剩余的数据，并设置ended为true。emitReadable方法里面，通过while循环来读取剩余数据，使得state.length为0，并执行endReadable方法。
```javascript
function endReadableNT(state, stream) {
  // Check that we didn't get one last unshift.
  if (!state.endEmitted && state.length === 0) {
    state.endEmitted = true;
    stream.readable = false;
    stream.emit('end');
  }
}
```
endReadable方法异步调用endReadableNT，置readable为false，并触发end事件。这样数据读取就结束了。

## pipe管道
readable里面的pipe方法，主要部分是事件上的监听，核心部分如下：
```javascript
  dest.emit('pipe', src);

  // start the flow if it hasn't been started already.
  if (!state.flowing) {
    debug('pipe resume');
    src.resume();
  }
```
触发pipe事件，同时开启flowing模式，并在结束的时候，会调用cleanup清理掉之前监听的事件。如果在pipe的时候，src又添加数据，而目标文件不处于drain状态，就需要监听drain事件，来单独处理。
