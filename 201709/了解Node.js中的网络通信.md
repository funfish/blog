# 前言
一直想知道Node.js是如何作为一个用js语言写的后端平台。这个设定很是奇怪，前端开始涉足后端了？刚开始用api实现通信的时候，蛮简单的，框架都不用到，简单几句就能实现通信，于是借此机会研究一下Node.js的通信。

# 从net模块出发
看一个简单的例子
```javascript
require('http').createServer((req, res) => {
    res.end('hello world');
}).listen(8181);
```
这里用的是http模块通信，也就是我们最常用的部分，看[官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/http.html#http_http_createserver_requestlistener)可以发知道其返回一个http.Server实例，而该实例又继承与net.Server，net.Server的描述是用于创建TCP或者本地服务器，这不正是我们想要的吗？
通过上面例子不难发现，我们主要用到的api的接口是createServer和listen，而首先的是net.Server实例。
```javascript
// net.js中
function createServer(options, connectionListener) {
  return new Server(options, connectionListener);
}
function Server(options, connectionListener) {
  ...
  EventEmitter.call(this);
  ...
    if (typeof connectionListener === 'function') {
      this.on('connection', connectionListener);
    }
  this._connections = 0;
  ...
  this[async_id_symbol] = -1;
  this._handle = null;
  this._usingWorkers = false;
  this._workers = [];
  this._unref = false;

  this.allowHalfOpen = options.allowHalfOpen || false;
  this.pauseOnConnect = !!options.pauseOnConnect;
}
util.inherits(Server, EventEmitter);
```
这里可以发现Server实例继承EventEmitter，就像Backbone里面的model等继承Events一样，监听connection事件，并初始化this，net.createServer也是返回Server实例，看来Server很重要。再看看listen
```javascript
Server.prototype.listen = function(...args) {
  ...
  if (hasCallback) {
    this.once('listening', cb);
  }
  ...
  var backlog;
  if (typeof options.port === 'number' || typeof options.port === 'string') {
    ...
      // 对于listen(port, cb)的情况
      listenInCluster(this, null, options.port | 0, 4,
                      backlog, undefined, options.exclusive);
    return this;
  }
  ...
};
function listenInCluster(server, address, port, addressType,
                         backlog, fd, exclusive) {
  ...
  if (cluster.isMaster || exclusive) {
    server._listen2(address, port, addressType, backlog, fd);
    return; 
  }

  const serverQuery = {
  ...
  };
  ...
  cluster._getServer(server, serverQuery, listenOnMasterHandle);
  function listenOnMasterHandle(err, handle) {
    ...
    server._handle = handle;
    server._listen2(address, port, addressType, backlog, fd);
  }
}
Server.prototype._listen2 = setupListenHandle;  // legacy alias
```
上面listen方法中有不同的场景判断，上面代码仅仅列出开发中常用的listen(port， cb)的方法。 结合Node.js的document里面的server.listen方法和这里的源码，发现document里面的介绍不就是源码里面的实现吗。。。。listenInCluster方法中通过cluster.isMaster来判断是否是主线程，如果是直接server._listen2，不是的话，还要进行cluster._getServer
而setupListenHandle方法
```javascript
function setupListenHandle(address, port, addressType, backlog, fd) {
  ...
    var rval = null;
  ...
      rval = createServerHandle(address, port, addressType, fd);
  ...
    this._handle = rval;
  }
  this[async_id_symbol] = getNewAsyncId(this._handle);
  this._handle.onconnection = onconnection;
  this._handle.owner = this;
  var err = this._handle.listen(backlog || 511);
  ...
  nextTick(this[async_id_symbol], emitListeningNT, this);
}
function createServerHandle(address, port, addressType, fd) {
  ...
    handle = new TCP();
    isTCP = true;

  if (address || port || isTCP) {
    ...
    if (!address) {
      err = handle.bind6('::', port);
      ...
    } else if (addressType === 6) {
      err = handle.bind6(address, port);
    } else {
      err = handle.bind(address, port);
    }
  }
  ...
  return handle;
}
```
上面的setupListenHandle经过简化，不难发现最后落脚点在`this._handle = new TCP()`，而TCP：`const TCP = process.binding('tcp_wrap').TCP`，this._handle 正是server._handle，在初始化Server实例时候所建立的，process.binding是用来连接Node.js的内建模块，这里要看tcp_wrap.cc文件

# tcp_wrap.cc文件开始的C++和C语言
tcp_wrap.cc文件里面有这里几个方法,
```C++
TCPWrap::TCPWrap(Environment* env, Local<Object> object)
    : ConnectionWrap(env,
                     object,
                     AsyncWrap::PROVIDER_TCPWRAP) {
  int r = uv_tcp_init(env->event_loop(), &handle_);
  CHECK_EQ(r, 0);  // How do we proxy this error up to javascript?
                   // Suggestion: uv_tcp_init() returns void.
  UpdateWriteQueueSize();
}
void TCPWrap::Listen(const FunctionCallbackInfo<Value>& args) {
  TCPWrap* wrap;
  ASSIGN_OR_RETURN_UNWRAP(&wrap,
                          args.Holder(),
                          args.GetReturnValue().Set(UV_EBADF));
  int backlog = args[0]->Int32Value();
  int err = uv_listen(reinterpret_cast<uv_stream_t*>(&wrap->handle_),
                      backlog,
                      OnConnection);
  args.GetReturnValue().Set(err);
}
```
TCPWrap方法里面调用了tcp.c里面的方法，建立TCP的句柄，函数形参包括uv_loop_t，uv_tcp_t和flags标识，接着对uv_tcp_t结构体中的各项进行初始设置，并调用uv__handle_init循环；
为什么要提及listen方法？在net.js的setupListenHandle方法里面明确用到了`this._handle.listen(backlog || 511)`，在stream.c里面的uv_listen方法来监听，根据uv_stream_t结构体的type来判断是否是TCP类型，在tcp.c，uv_tcp_listen中通过uv_tcp_t结构体的flags来判断执行；

在uv.h里面定义了上面的结构体，每个结构体都有自己的成员。刚开始接触的时候，看到这里么就有种越看越乱，越看越多的感觉，这个时候开始知道有libuv；

# libuv为何物
简单来讲就是跨平台io库，整合了window下的iocp和Linux的epoll，官网上有下图
![](https://github.com/funfish/blog/blob/master/images/libuv.PNG)

在node_maic.cc里面调用了start方法，加载bootstrap_node.js文件，并同时while循环调用uv_run()，uv_run就是libuv事件循环的入口，这个方法的执行如下图
![](https://github.com/funfish/blog/blob/master/images/uv_run.PNG)
其中每一个模块和uv_run中的语句是对应，其中在window里面用`(*poll)(loop, timeout)`，而unix采用`uv__io_poll(loop, timeout)`。 
上文提到的结构体uv_xx_s/t正是libuv的观察者，其中对应的类型uv_TYPE_t中的type指定了handle的使用目的。 至于具体的机理还是看[官方文档好](http://docs.libuv.org/en/v1.x/#documentation)。
看过了文档以及api之后，再去看Node.js里面代码，well， 还是一脸懵逼。。。。

只有硬啃下来，发现就是观察体太过长了，一遍一遍的嵌套宏定义，只是到最后还是没有完全读懂libuv如何实现通信，或许以后有下一篇来阐述libuv吧

参考资料
1. [node源码详解（六） —— 从server.listen 到事件循环](https://cnodejs.org/topic/5716137fe84805cd5410ea21)
2. [uvbook 中文教程](http://luohaha.github.io/Chinese-uvbook/source/networking.html)
3. [初步研究node中的网络通信模块](http://zhenhua-lee.github.io/node/socket.html)
4. [《深入理解Node.js：核心思想与源码分析》](https://yjhjstz.gitbooks.io/deep-into-node/)