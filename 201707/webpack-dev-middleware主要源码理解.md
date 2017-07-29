# 前言
每次用Vue-cli的时候，都会觉得配置的nodejs服务器很是让人省心，几个项目下来，都只用关心工程前端部分，用久了便想探其究竟。后来才发现原理用了express框架，代码也挺简单的，但是里面用的一个中间件webpack-dev-middleware，刚开始看的时候却不知有何用处？既然是express框架，又用的是SPA，路由不需要express来分发，那webpack-dev-middleware有何用处？抱着这样的疑问，看源码去吧

# 思路
一般express中间件的结构如下：
```javascript
App.use((req, res, next) => {
  if (nextNeeded) {
    // 根据不同的请求处理，并传递到下一个next()，形成处理流
    next();
  } else { 
    // 根据不同的请求处理
  }
});
```
而webpackDevMiddleware也大体如此，只是返回一个Promise对象
在webpackDevMiddleware中的返回Promise前，有这两句：
```javascript
var filename = getFilenameFromUrl(context.options.publicPath, context.compiler, req.url);
if(filename === false) return goNext();
```
getFilenameFromUrl这个名字就可以知道这是个返回匹配url路径文件名字，若不存在这个filename则goNext，而goNext里面若在webpackDevMiddleware中没有配置serverSideRender这个选项，则next进入下一个中间件。getFilenameFromUrl这个看源码就知道了，那问题的关键在于返回的Promise如何才能满足webpage开发和express框架？
Promisel里面的try出现了下面一句：
```javascript
res.statusCode = res.statusCode || 200;
if(res.send) res.send(content);
else res.end(content);
resolve();
```
咦？这不就是最普通不过的res.send/end方法吗？那为何要大费周章的写个Promise呢？
用的确实是send和end方法，但是里面的content，却大有文章，一般用response返回的时候，直接返回本地文件就好了，用webpack的时候，部分文件却是自动生成的内存文件如index.html, app.js之类的，这些文件是不能直接本地读取并获得的。在processRequest里面的try里面有如下:
```javascript
var stat = context.fs.statSync(filename);
```
这个context.fs是在Share.js里面的生成的
```javascript
fs = compiler.outputFileSystem = new MemoryFileSystem();
```
MemoryFileSystem用的是memory-fs npm模块，这个模块就是一个内存文件模块，而这里用到的statSync方法，则是返回一个是否是文件/目录的对象，若都不是就抛出MemoryFileSystemError，所以思路就清楚了，当不存在内存文件的时候，如直接在html里面引用到的文件js/css/picture，就直接抛出Error，进入goNext，从而进入下一个中间件。若不抛出Error，则在try外部通过 `context.fs.readFileSync(filename)` 读取内存文件，如webpack生成的内存文件app.js等，从而选择性发送content。
至于剩下的响应头设置和webpackDevMiddleware的其他方法，聪明的你，自己挖掘吧

# 疑问
在返回的Promise里面，先是进行了`shared.handleRequest`，这里面会有若filename是文件才进行processRequest，而在processRequest的try里面，若filename不是文件，是路径才进行里面的改写filename，这样的逻辑设计是为何？是文件，又不是文件的，如何进入里面呢？

