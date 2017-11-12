>本文大部分内容是基于[初识NodeJS服务端开发（Express+MySQL）](http://www.alloyteam.com/2015/03/sexpressmysql/)

# 前言
本来只是想学习一下MySQL，毕竟隔几天就可以看到隔壁小伙伴在操作数据库有mySQL和Redis，好像还有mongodb？一直挺向往后端的，加上最近想自己打造一个个人博客,从数据库，服务器部署维护，后端nodejs实现集成，最后到前端展示，这些都想一一落实，于是开始数据库的学习。

正如本文开头所说的，大部分内容都是基于AlloyTeam的那边篇博客，本来没有必要写的，但是是第一次打通前后端的数据鸿沟，纪念一下，还是发表一下吧。

# 开始数据库MySQL
数据库安装可以直接去官网下载community版本，目前最新版本是8.0.2；
笔者环境是win10，在开始之前，先熟悉一下MySQL的基本命令
```
启动服务: net start MySQL

停止服务: net stop MySQL
```
上面这些命令都是要到命令行也就是终端里面键入的；
还有对数据库的操作，可以自行上网搜索教程，这里推荐[21分钟 MySQL 入门教程](http://www.cnblogs.com/mr-wid/archive/2013/05/09/3068229.html#c1)，如果只是初接触是够用的了。
在终端里面访问MySQL需要提供root账号或者自己额外设置的账号命令为`mysql -uroot -p`，回车后输入密码就可以访问自己的MySQL了，接着就是创建数据库，创建表和插入数据这些过程。这上面一切都是在终端实现的，其实MySQL也有自己的图形界面，世面上有很多，而官方下载里面提供了
MySQL Workbench软件，在里面你能更方便的访问查看数据库； 

# 设置express
如AlloyTeam里面介绍到的，通过安装全局的express来初始化搭建脚手架，快速建立工程。后面通过那篇博客的介绍也可以完成基本的MySQL和express的结合，但是这里提醒一下，由于里面代码没有给全，没有对util说明介绍，所以这里可以把userDao.js里面修改一下：
```javascript
// userDao.js
// 删除下面这行代码
var $conf = require('../conf/conf'); 

// 将连接池的创建修改为：
var pool  = mysql.createPool($conf.mysql);
```
对于express不熟悉的话，可以看阮老师的[express框架](http://javascript.ruanyifeng.com/nodejs/express.html)，当然少不了官方api文档；

# 接口设置
如果只是想要体验一下epress和MySQL，基本到这里就可以结束了，但是日常开发里面，哪有直接在地址栏输入访问数据的呢？正常的开发肯定是通过Ajax访问数据的啦！(开玩笑，现在谁还用Ajax。。。。。)

## express设置环境
需求如此，自然要实现，首先是express设置。由于express配置里面用的是jade这类模板的开发，给人的感觉就像回到了以以前后端返回组装好的模板给到浏览器一样。为了方便直接用html文件开发，也就是最简单的访问`/`返回`index.html`静态资源文件。返回文件，就用到express的api：
```javascript
// routes/index.js
res.sendFile(path.resolve(__dirname,'../views/index.html'))
```
由于我们的路由文件`index.js`在routes文件夹里面，此时的`__dirname`并不是指向工程根目录，而是routes文件夹，所以需要path模块来拼接成需要的路径。这里需要注意到的是由于在app.js文件里面设置了`view engine`为`jade`，所以这里sendFile的时候不能省略文件的后缀`.html`要不然，无法识别html文件，会报错。
So，这样就回到了前端熟悉的环境了。

## 前端请求数据
在`index.js`里面用fetch请求数据，这里需要自定义路径，由于是前后端都是我们负责，这里就省去了工作中等待后端提供接口的过程，so，nice！的体验。
前端fetch的数据地址为`'/users/queryAll'`，而后端这里，需要在`users.js`里面添加如下代码：
```javascript
router.get('/queryAll', function(req, res, next) {
    userDao.query(req, res, next);
})
```
对于userDao里面添加一个query的方法，形式上和`userDao.add`很接近，只是在数据返回不能只是简单的返回成功或失败，接口的定义最好是按照RESETfull来设计，这里我则按照平时工作上的接口来返回数据来设计：
```javascript
res.status(200).send({
    code:200,
    data: {
        list: ret
    },
    msg: '成功'
})
```
这里的ret就是MySQL里面查询的数据，一般还是通过list字段返回。后端基本上就可以返回数据了。接下来就是前端的数据展示了。很简单是不是？

这里提个小插曲，fetch是可以远程获取数据的，但是获取得到的是Reseponse对象，而不是data，所以这里需要再进一步操作，通过`res.json()`解析成Promise对象。这个promise对象通过then继续操作，其传参才是后端返回的数据；

参考：
1. [初识NodeJS服务端开发（Express+MySQL）](http://www.alloyteam.com/2015/03/sexpressmysql/)
