## 前言
项目中经常用 Jenkins 构建项目，只要点击构建，服务端就会按照指令，重新拉取数据构建，这是很好的，只是久而久之发现一个问题：项目的构建时间从之前的飞快，到现在龟速。等待构建开发时间长是一个问题，更重要的是如果项目继续发展壮大呢？现在的 antd-pro 项目也就十来个页面，一点也不多，但是测试服构建起来，时间将近4分钟，特别吃内存。如果以后页面多很多呢？五六十个页面呢？那岂不是要二十来分钟的构建时间？内存呢，难道最后要溢出？这是难以置信的。

## 初探问题
春节期间前有空，上网查了一下方法，`webpack.optimize.UglifyJsPlugin` 几乎是千夫所指的，自带的代码丑化基本就是鸡肋，用上其他丑化插件，打包时间可以节省上30%，甚至更多，只是 antd-pro 用的是 roadhog.js，是一款接近于 create-react-app 的基础工具，能自己编写的配置参数少之又少，更不要提随意运用 webpack 的插件了。只是想要试探性的玩一下，于是在本地的 node_modules 里面修改了 roadhog 关于 webpack 的 UglifyJsPlugin 插件，结果一试，速度 duang 的一下就上来了。后来由于工作忙就没有怎么 care 构建问题。

春节期间看了几位大佬的博客，有感珠玉在前，webpack 构建问题，是要好好处理的了。

## roadhog.js问题
roadhog.js 是类似可配置的 react-create-app，只是这个可配置，也只是部分可配置的，木有办法，只能从源码开始看 webpack 配置。

在进行 `npm run build` 的时候发现终端有提示 `Creating an optimized production build...` 的字样，而且出现的时间也挺晚的，以前其他项目上面从未见过，难道是 roadhog 自己的？这个时候 webpack 居然还没有开始构建？抱着疑惑，从 roadhog 的`bin/roadhog.js`就开始打印当前时间，再在到开始webpack构建的时候再打印一次时间。

结果这个过程要花上2931ms，还是可以接受的，只是明明第一次的时候记得等了很久的，为什么这次只要3s不到？后面又试了几次，耗时均3s左右，后来想起了[Webpack 构建性能优化探索](https://github.com/pigcan/blog/issues/1)里面提到的初次构建和再次构建的问题，一般再次构建耗时都要比初次构建的要少。会不会第一次比较慢是初次构建，后面都是再次构建呢？初次构建和再次构建有什么区别？百度和谷歌都没有查询到答案，只有该博客提到比较多。为了再现问题，well，重启电脑，再次 `npm run build` 不就是初次构建吗？结果还正如此。

初次构建从进入 roadhog.js 到开始执行 webpack，居然花了30268ms，没有看错居然要30s的时间，这还没有算是 webpack 运行的时间，我的天呀。

### 时间都去哪儿了-roadhog.js
多次尝试初次加载，大概都在30s时间左右，而再次加载平均3s不到，十倍的差距，初次加载如此龟速，时间都去哪儿了？

粗看代码，并没有什么特别耗时间的计算，于是到处用`console.log()`打印时间，发现原来是几个requre的地方特别耗时间，分别如下：
```javascript
var _webpack = require('webpack');

var _getConfig = require('./utils/getConfig');

return (0, _applyWebpackConfig2.default)(require('./config/webpack.config.prod')(argv, appBuild, c, paths), process.env.NODE_ENV);
```

这三行代码基本占用80%以上的时间，尤其是最后一句读取webpack配置，耗时将近14s，**原来时间都花在读取这些文件上面了。**这下瞬间明白了，roadhog.js 在初次加载上面表现很差劲，就是这个原因了。对比一下 Vue 的脚手架 Vue-cli，Vue-cli 里面从开始到进入 webpack 初次构建只要13s左右。这 roadhog.js 简直是慢慢慢。

再次加载由于node的缓存机制，之前require的内容都可以缓存下来，所以只是读取一个缓存的过程，时间可以缩短到3s不到。而正式服上面，一周构建一次基本就算频繁了，每次构建前期工作都要花30s，有必要这么复杂吗？看看新出的 parcel，秒杀。

## 数据报备
本机硬盘环境：

2.3 GHz Intel Core i5 / 4 核
4 GB DDR3

项目数据:

1526 个 node_modules 文件，package.json 中项目依赖 58 个

运行数据:

在 node@8.9.0 npm5.5 下整个webpack初次构建耗时达 92229ms，再次构建 86422ms

## 走进webpack优化
按照[Webpack 构建性能优化探索](https://github.com/pigcan/blog/issues/1)里面给出的思路，对于webpack的优化，可以从四个维度考量：
> * 从环境着手，提升下载依赖速度；
> * 从项目自身着手，代码组织是否合理，依赖使用是否合理，反面提升效率；
> * 从 webpack 自身优化手段着手，优化配置，提升 webpack 效率；
> * 从 webpack 可能存在的不足着手，优化不足，进一步提升效率。

从环境出发这一点，是因为不同的nodejs版本和npm版本，有着显著的性能差异来的。可以这么认为最新版本的nodejs/npm自然有更优秀的性能。由于项目本身用的就是最新版本的环境，所以这里也不加以分析了。

### 从项目中出发
首先用比较常规的方法，通过 `webpack-bundle-analyzer` 来查看 webpack 体积过大问题，结果如下图所示：
![](https://github.com/funfish/blog/blob/master/images/bundle-analyzer.PNG)

图挺好看的，乍一看没有什么特别的地方，好像每个打包文件都是由诸多细文件组成的。并从文件大小来看压缩过后都在1M以下，无可厚非。但是细心对比下，还是有不少发现。

**案例1：为了实现小功能而引用大型 lib**

这里用 `webpack-bundle-analyzer` 来查看打包过大问题，但是在引用的时候，却发现roadhog原本自身就用了 `webpack-visualizer-plugin` 插件，只是在analyze指令下才能进入分析，整理之后webpack配置如下：
```javascript
var BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;
var Visualizer = require('webpack-visualizer-plugin');

plugins: [
  new Visualizer({
    filename: './statistics.html'
  })],
  new BundleAnalyzerPlugin()
]
```

这里 `webpack-bundle-analyzer` 可以给予直观的整体感受，而 `webpack-visualizer-plugin`则细化到每个文件中，每个模块的百分比。

首先看到的是：最大的文件打包压缩后是815.9kb，相对于其他较大的文件大出了整整400kb，这里肯定是有什么问题，细看之后，发现用了支付宝的 G2，在代码中的体现是:
```javascript
import G2 from 'g2';
// 将整个G2都引入进来了，导致文件过大
```

遗憾的是目前G2没有实现按需加载的功能，在[issue](https://github.com/antvis/g2/issues/364)里面也只是表示正在讨论而已(庆幸这里只是用了G2，没有用到Data-set)。

仔细看了每个js文件打包构造后，发现有个文件也用了 moment 模块，在印象中是基本没有用到的。moment 模块大小为53.2kb，而在总的打包文件中占 131.6kb。正同 Webpack 构建性能优化探索所说的，如果不想简单实现，就采用 fecha 库来代替 moment，fecha 要比 moment 小很多。只是替换后，发现 moment 体积并没有降低多少，由于出处是在 index.js 文件里面，可能的地方只有 dva 了。只是 dva 怎么可能用到moment？完全不可能的，他的package.json里面也同样没有用到。通过排除法最后定位到如下：
```javascript
// @src/index.js
import 'moment/locale/zh-cn';

// @src/router.js
import { LocaleProvider } from 'antd';
```

第一个行代码是直接使用了 moment 模块，该代码看着作用不大，而且查阅Ant-Design-Pro的历史版本，均没有发现在index.js里面使用 `moment/locale/zh-cn`。细心观察，发现在 index.js里面使用了 `moment/locale/zh-cn` 之后，其他几处用到moment的地方，生成文件都没有明显 moment 包，这些文件的体积基本上要减少一个 moment 的大小。这个`moment/locale/zh-cn`，还能降低其他文件体积。
第二行代码，是在 router.js 文件里面，由于使用了 LocaleProvider 组件，这个组件通过源码可以发现直接引用了 moment 模块 `import * as moment from 'moment'`。当然同样也起到了 `moment/locale/zh-cn` 的效果，能降低其他原本含 moment 文件的体积。

**案例2：废弃依赖没有及时删除**

项目中用的是 Ant Design，import 的时候，组件是按需加载的，并不会整个引入 Ant Design，但是由于敏捷开发周期较短，新建页面不会从零开始写，基本都是移植相似的页面，由此导致了Ant Design组件的乱引用。

由于 G2 的不可按需加载，以及 moment 在 Ant-Design 中的作用，工程的打包体积和打包时间没有较大的减少。

### 从 webpack 自身优化点出发
webpack 本身也提供了许多优化的插件，但是由于经常接触不多，许久后容易遗漏，导致再次学习的成本高。一个好的脚手架，是相当重要的。

#### webpack 自带的优化
webpack 就有不少内置的插件。

* CommonsChunkPlugin

CommonsChunkPlugin 可以从 module 提取公共 chunk，实现降低模块大小，有利于整体工程打包后的瘦身。
CommonsChunkPlugin 这个插件在Vue-cli中也有用到，如下：
```javascript
// split vendor js into its own file
new webpack.optimize.CommonsChunkPlugin({
  name: 'vendor',
  minChunks: function (module, count) {
    // any required modules inside node_modules are extracted to vendor
    return (
      module.resource &&
      /\.js$/.test(module.resource) &&
      module.resource.indexOf(
        path.join(__dirname, '../node_modules')
      ) === 0
    )
  }
}),
new webpack.optimize.CommonsChunkPlugin({
  name: 'manifest',
  chunks: ['vendor']
})
```

把相同的 chunk 提取出来，命名 vendor 与 manifest，前者是常说的公共 chunk 部分，后者是由于代码变动导致 chunk 的 hash 值变化，导致公共部分在每次打包时都会有不一样的 hash 值，使得客户端无法缓存 vendor。**由于代码变动导致 hash 变化，而生成的代码，自然而然的会落在最后配置的 commonschunk 上面，**所以这部分可以单独提取，命名为 manifest。

在roadhog里面，刚开始看以后没有CommonsChunkPlugin的配置，想着赶紧提个issue，但是后面发现，是通过`common.js`引入，只有在roadhog里面配置了multipage选项为true的时候，才执行CommonsChunkPlugin插件。其代码如下：
```javascript
var name = config.hash ? 'common.[hash]' : 'common';
ret.push(new _webpack2.default.optimize.CommonsChunkPlugin({
  name: 'common',
  filename: name + '.js'
}));
```

通过 CommonsChunkPlugin 插件，node.js 在打包的时候，峰值内存增加了40M，就是约5.4%的内存，打包时间延长了大约6s，而构建后项目体积基本不变，what？有点震惊。只有负面效果。。。。看构建文件，只提取了一个公共文件，大小1kb，而且内容为一句普通的错误打印。为什么人与人之间没有相互的chunk可以提取呢？

通过反复查 roadhog/ant-design/ant-design-pro 的 issue 都没有类似的问题，似乎用了 `babel-plugin-antd` 对 antd 进行按需加载，没有办法将其提取到 vendor 里面了。如若不想不想按需加载，直接用 cdn 不就好了。但是现在想的是只要单独的提取antd里面几个涉及CRUD的重要组件：表格，form，日历这几个组件能否实现单独打包到vendor？难道是我打开方式不对吗？大神里面少 7s，我还多了 6s。。。。

在这个[issue](https://github.com/ant-design/ant-design/issues/8038)里面看到了这种写法，顿时觉得没错，就是她了。**entry 里面设置多入口，CommonsChunkPlugin里面再提取。**
```javascript
entry: {
  //...
  antd: [ //build the mostly used components into a independent chunk,avoid of total package over size.
      'antd/lib/button',
      'antd/lib/icon',
      'antd/lib/breadcrumb',
      'antd/lib/form',
      'antd/lib/menu',
      'antd/lib/input',
      'antd/lib/input-number',
      'antd/lib/dropdown',
      'antd/lib/table',
      'antd/lib/tabs',
      'antd/lib/modal',
      'antd/lib/row',
      'antd/lib/col'
  ]
},
//...
new webpack.optimize.CommonsChunkPlugin({
    names: ['antd'],
    minChunks: Infinity
}),
```

咦？见证奇迹的时候到了，构建后的项目大小居然小了，整整3M，少了36.64%，厉害了。更惊讶的是，峰值内存减少了180M，减少了24.3%，打包时间减少了26s，直接下降到59196ms，减少25%；这牛逼了。

仔细对比一下，发现原来减少的部分并不是我以为的 antd 组件，antd 组件反而在每个打包文件里面的体积都要更大了，大概多了几kb，而减少的部分却是一些 `_rc` 开头的组件，这 CommonsChunkPlugin 也是厉害，按需加载部分没有单独打包起来，反而打包了这些组件背后的引用，如 `rc-table`。为什么这些组件最后还是没有完整的打包在antd里面呢？难道每次用的都不同？
```javascript
import { DatePicker } from 'antd';
// / babel-plugin-import 会帮助你加载 JS 和 CSS 转变成下面内容
import DatePicker from 'antd/lib/date-picker';  // 加载 JS
import 'antd/lib/date-picker/style/css';        // 加载 CSS
```

这没看来，只是典型的引入组件，以及引入css模块而已。这是必然会被打包到公共模块的呀。看了未丑化的代码，发现用同一个组件的话，生成的不同文件 antd 的组件内容是一样的，不存在组件内部不一样导致没有打包在一起的情况。折腾许久后尚未解决，不晓得有没有大神知道。

而且 roadhog.js 的方式不允许添加新的入口，只能直接改源代码。。。这项目要怎么上线呢？难道每次都要自己改一遍？这就是__**约定和可配置**__的问题所在了，后面大神的博客也有讨论到，最后的思想还是约定为若干模块，可自选配置，来适合不同的场景。

* DedupePlugin/OccurrenceOrderPlugin

这两个功能在webpack里面很常见，以至于已经被移除了，默认加载包含在 webpack 2 里面了。

**CommonsChunkPlugin**对项目的优化还是很实在的，能减少不必要的打包，不仅是体积，更多的是从内存和时间上。

#### webpack外引入的优化
前面提到的 `webpack-bundle-analyzer` 和 `webpack-visualizer-plugin` 插件就是从 webapck 外部引入的，可以很直观的看。

externals 的设置在 Vue 项目里面用的比较多，其中主要 externals 的是 `axios, Vue, Vonic, Vue-router` 这些。本身体积也不大，而且作为单页面应用还是很需要的。

但是到了 Ant Design Pro 项目，由于 Ant Desgin 项目本身 `CSS + JS` 就要1.5M，对于首屏的影响是显著的。虽然可以通过浏览器缓存/cdn缓存的方式来自然优化，但是首次体验还是不行，还是按照官网上的介绍来吧。

* DllPlugin 和 DLLReferencePlugin

按照官网上的介绍：DLLPlugin 和 DLLReferencePlugin 用某种方法实现了拆分 bundles，同时还大大提升了构建的速度。具体原理则是将特定的第三方 NPM 包模块提前构建再引入就好了。通过在 webpack 外进行配置，DllPlugin 负责配置输出引用文件 manifest.json，而  DLLReferencePlugin 在webpack的正常配置里面用 manifest.json 就好了。可以避免每次都对 npm 包打包，明明它们就不会改动，直接引用不是更好吗。

在 roadhog.js 里面实现就有点那个了，按照 sorrycc 作者的意思，在生产环境使用 DllPlugin 是不合适，打包大量的 npm 包后，**会延长首屏时间，与按需加载矛盾**。这点就和 CommonsChunkPlugin 是相同，都是提取第三方库，而且 DllPlugin 是一次打包即可，以后重复用引用，而 CommonsChunkPlugin 是每次打包都要重复提取公共部分，那这两个又有什么区别？

一般 DllPlugin 打的包会包含很多 npm 包，导致体积很大，首次加载自然不好，而且若以后更新某个包，会导致客户端重新下载整个 DllPlugin 的生成文件，对于产品迭代是不友善的。反观 CommonsChunkPlugin，一般提取的公共部分体积较小，例如antd主要组件提取，不到500kb，除非大版本升级，否则客户端是不会重新请求 vendor.js 文件的。

基于上面的观点 DllPlugin 一般用于提升本地开发的编译速度，就是启动项目开发的时候能够快点。只是一天能够启动多少次项目呢，基本都是热更新为主吧。。。。。这么看好像意义不大，就是开发人员的自 hight 而已。

发现原来roadhog自己也有 DllPlugin 的配置，只要在 config 里面添加 `dllPlugin: true` 就可以了，当然也是**仅仅限于开发环境**，肯定不是生产环境。很是方便，这里就不详细介绍了，感兴趣的可以自行看看这个[issue](https://github.com/sorrycc/roadhog/issues/2)。

### 从 webpack 不足出发

* HappyPack

使用 HappyPack，可以利用 node.js 的多进程能力，来提升构建速度。在 webpack 打包的时候，经常可以看到 CPU 和内存飚的非常高，内存可以理解，但是 CPU 为何会如此之高呢？只能说明有大量计算，而 node.js 的单进程在大量计算面前是单薄的。可以在 webpack 中定义 loader 规则来转换文件，**让HappyPack来处理这些文件，实现多进程处理转换**。

设置如下：
```javascript
new HappyPack({
    threads: 4,
    loaders: [{
      loader: 'babel-loader',
      options: babelOptions
    }],
})
{
  test: /\.(js|jsx)$/,
  include: paths.appSrc,
  // loader: 'babel',
  loader: 'happypack/loader',
  options: babelOptions
}
```

只是运行结果却不让人满意，打包时间/内存什么都和原先的数据几乎相当。难道和 CommonsChunkPlugin 的时候一样，又是打开方式不正确？于是按照官网说的加个 id 试试，结果立马报错，提示__`AssertionError: HappyPack: plugin for the loader '1' could not be found! Did you forget to add it to the plugin list?`__，看到有 issue 提出将 loader 里面的 options 改为 query 就可以了，只是官方提示 webpack 2+ 需要使用 options 来代替query ，最后试了一下也是报错，报错的根由是 happyloader 没有获取到查询的识别 id。回头看了下源码，`query = loaderUtils.getOptions(this) || {}`这句话不就是获取 loader 的 option 配置吗，里面怎么可能有 id 呢？里面就是 babelOptions，不可能有 id 的。接着看 loader-utils 的源码，这个就是简单的获取查询到的 query，没有毛病，难道是 HappyPack 用错了？

折腾好久后，差不多都要放弃了，我定了定神，重新理一遍，看到了 rules 里面的配置：
```javascript
loaders: [{
  loader: 'happypack/loader?id=js',
  options: babelOptions
}],
```
options 选项是 roadhog 原先就有的，而 laoder 原先是 `babel`，后面改为了 happypack 的设置。这个时候眼睛一亮 loader 设置里面有个问号 `?`，这个不就是 query 吗？那 options 呢？loader-utils 里面获取的是这个 query 还是 option？注释掉试一试？完美成功了。。。。原来如此简单。

**用了 happypack 之后，不能在 rules 里面的相关 loader 中配置 options，相反只能在 happypack 插件中配置 options！**

well， 然而什么都没有变呀，设置了缓存也没有用，速度/内存什么的都和之前一摸一样。这个时候看到了(在 roadhog 中尝试支持happypack)[https://github.com/sorrycc/roadhog/issues/122]里面大神说了社区版本有问题。。。。。。虽然不知道具体的原因，但是实际效果是对 js 文件用 HappyPack 的配置，是没有起到想象中的多进程计算的优点的，原因或许出在 babel/HappyPack 身上了，最后还是落到了单线程计算上。具体就不分析了，有空可以在研究一下。

* uglifyPlugin

uglifyPlugin 是生产环境中必备的，毕竟压缩丑化代码，不仅可以降低客户端加载项目体积，降低打开时间，而且可以防止反向编译工程的可能性。在本文的开头就提到过，首次优化就是针对 uglifyPlugin 的，而且效果显著。

使用 `webpack.optimize.UglifyJsPlugin` 的时候，平均下来 webpack 的构建时间要达到 86s 左右。当**不进行代码压缩丑化**的话，构建时间下降了 68s 左右，并且构建时候，node.js 占用内存峰值下降了 380M 多，可以说不压缩丑化的话，效果是非常好的。但是项目体积却基本是原本的三倍之大，这是难以容忍的。webpack自带的uglifyPlugin，如此笨拙，要如何处理呢？

对 `webpack.optimize.UglifyJsPlugin` 在里面添加 `cache: true` 的配置也是没有什么效果，看了下官网介绍的另外一个 UglifyJsPlugin 插件，上面写着 `webpack =< v3.0.0` 已经**包含 UglifyjsWebpackPlugin 的 0.4.6 版本**了，如果想要安装最新版本才按照下面介绍的来。发现本地安装的 webpack 版本是 3.11.0，自然是内置 0.4.6 版本。1.0.0 版本是会在 webpack 4.0.0 里面安排的。那如果直接用 `uglifyjs-webpack-plugin` 最新版本呢？

安装 `uglifyjs-webpack-plugin 1.2.2`，设置配置如下：
```javascript
new UglifyJsPlugin({
  cache: true,
  uglifyOptions: {
    compress: {
      warnings: false
    },
    output: {
      comments: false,
      ascii_only: true
    },
    ie8: true,
  }
})
```

初次构建的时候，构建时间较之前多40s，也就是多了46.5%，有点夸张的多，内存还好，峰值基本和用 0.4.6 版本的一样。但是 再次构建呢？**构建时间居然下降了68s**，而且内存峰值和未用代码压缩丑化的时候相似，也就是减少了 380M，实在厉害，牛逼哄哄。

还可以开启并行，也就是多进程工作，设置 `parallel: true`，设置之后测试，初次构建时间居然比普通的再次构建时间要少10s，但是问题也很明显 CPU 在平时的时候峰值基本在 45% 左右，而多进程后，**CPU 的峰值居然很长一段时间都在 100%**，内存也是达到了 1300+M，实在恐怖，如果正式服这么用不晓得会不会爆炸呢？hahaha。parallel 除了可以设置为 true 以外，还能设置成进程数，于是试了等于 2 的时候，CPU 运行峰值接近 95%，而内存峰值在 1100+M，也算是相对较好的数据，只是 CPU 还是接近于爆表。

对于再次构建 parallel 自然是起不到作用的，这里有不得不提另外一个插件 `webpack-parallel-uglify-plugin` （下载量比另外一款 `webpack-uglify-parallel` 多上一倍，肯定使用这个嘛）。试了一下，初次构建基本和 `uglifyjs-webpack-plugin 1.2.2` 一致，只有构建时间快 7s。

综上所诉，对于服务器 CPU 豪华的可以考虑平行压缩丑化，一般时候用  `uglifyjs-webpack-plugin 1.2.2` 多进程就不用设置，使能 cache 就好了，初次构建会慢点，再次构建的话，速度就上天了。

* UglifyJsPlugin 与 CommonsChunkPlugin

最后自然也是要让两者合并试一试，效果如何呢？和为优化之前相比，初次构建，内存减少 120+M，构建时间基本一样，构建项目大小自然还是少了 3M。咋一看好像不怎么样，但是要知道这是用上了UglifyJsPlugin，有缓存的！结果再次构建数据如所想的一样，速度和内存数据，和没有用代码压缩丑化基本一致！

**这样 uglifyjs-webpack-plugin 与 CommonsChunkPlugin 在生产环境自然是很好的选择。**

** 总结
本文主要是按照(Webpack 构建性能优化探索)[https://github.com/pigcan/blog/issues/1]介绍到的方法实践一遍，可谓收获颇多。但是还是遗留下两个问题：
1. CommonsChunkPlugin 对 Ant Design 提取问题；
2. HappyPack 没有效果的问题；

另外前几天 webpack 4 已经正式发布了，大佬们都迫不及待的想介绍一波。初看一下，从之前的**配置化**思想更多转化为**约定俗成**，这确实是前端发展的趋势。尤其是这类工具，每次都学习一下，成本过高，看看`create-react-app`，基本都封装好webpack，上手用就可以了。而2017年的明星项目 parcel 更是夸张，直接跑，没有什么配置的。还是期待 webpack 4 的传说中的提升98%的速度吧。Twitter上面的数据基本在提升60%多。(Twitter数据)[https://twitter.com/TheLarkInn/status/964607133495865351?ref_src=twsrc%5Etfw&ref_url=https%3A%2F%2Fmedium.com%2Fmedia%2Ff45612237403efcbcc6212011da27699%3FpostId%3D6cdb994702d4]。

对 webpack 4 感兴趣的可以看这篇(翻译)[http://www.zcfy.cc/article/x1f3bc-webpack-4-released-today-webpack-medium]（国外大佬们都迫不及待要介绍了，国内还在过春节元宵节，哈哈哈哈）

## 参考
1. (Webpack 构建性能优化探索)[https://github.com/pigcan/blog/issues/1]