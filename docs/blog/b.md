本文主要讲述的是怎么最合理化的运用游览器缓存，和 webpack 存在了几年的一个问题斗智斗勇的过程。本文部分内容可能会随着 webpack5 的发布变得毫无意义。

## 合理划分公共模块

以下内容都会以 [vue-element-admin](https://github.com/PanJiaChen/vue-element-admin) 为例子

`webpack 4` 最大的改动就是废除了 `CommonsChunkPlugin` 引入了 `optimization.splitChunks`。

webpack 4 的 `Code Splitting` 它最大的特点就是配置简单，如果你的 `mode` 是 `production`，那 `webpack 4` 就会自动开启 `Code Splitting`。

![](https://user-gold-cdn.xitu.io/2018/7/24/164cac10a2222794?w=3348&h=1880&f=jpeg&s=720643)

如上图所示，在我没配置任何东西的情况下，`webpack 4` 就智能的

它内置的代码分割策略是这样的：

- 新的 chunk 是能被共享的或者是来自 node_modules 的模块
- 新的 chunk 在压缩之前大于 30kb
- 并行加载的请求数小于等于 5 个
- 初始页面加载小于等于 3 个

这个规则对于大部分简单的应用来说基本已经够用了，但对于复杂一点的页面或者说用到第三方库比较多项目来说还是有不少的问题的。就拿 [vue-element-admin](https://github.com/PanJiaChen/vue-element-admin) 来说，它是一个基于 [element-ui](https://github.com/ElemeFE/element) 的管理后台，所以它里面会用到如 `echarts`、`xlsx`、`dropzone`各种插件。但如上图所示的那样，由于`element-ui`被大量页面引用，它默认将其打包到了 `app.js` 之中。这样做是不合理的，因为你 `app.js`还含有你的`router.js`你的路由声明、`store`、`utils` 公共函数，`icons`图标这些等等。但这些又是平时开发中经常会修改的东西，比如我新增了一个`utils.xxxfcuntion` 全局函数，或者我修了一个路由的`path`，都会导致`app.1.js` => `app.2.js`。这样会让 `element-ui`和 `vue/react` 等一系列打包在其中，但并没有发生修改的第三方库的缓存也失效了，这是非常不合理的。所以我们需要自己来优化一下缓存的策略。

我们现在的策略是将那些第三方比较大，且更新频率适中的组件库剥离。

讲这个项目的基础 libs 如`vue`、`vuex`、`vue-router`、 `axios`等构成这个项目最基础的组件库打成一个 libs，他们一般升级频率不高，而且基本上升级有都会一起联动升级。

最后我们还需要整理一下我们的`app.js`，它主要包含我们的`icons`、`router`路由声明，`store`、`utils` 这种全局都会使用而且更新频率很高的模块。

当然这种优化都是因项目而异的。比如上面中你的`icons`包含很多图标很大，而且改动非常频繁，你还和其它公用模块一起打在`app.js`里面显然不合理了，你需要将它单独拆成一个包。但如果你有过分的追求柯里化，什么都单独的打成一个 chunk，可能你一个页面需要加载十几个`.js`文件，如果你还不是`HTTP/2`的情况下，请求的阻塞还是很明显的。所以这里资源的加载策略并没什么完全的银弹，都需要结合自己的项目找到最合适的拆包策略。

比如支持`HTTP/2`的情况下，你可以使用 `webpack4.15.0` 新增的 [maxSize](https://webpack.js.org/plugins/split-chunks-plugin/#splitchunks-maxsize)，它能将你的`chunk`在`minSize`的范围内更加合理的拆分，这样可以更好地利用`HTTP/2`来利用长期缓存(注意一下是否支持`HTTP/2`，缓存策略是有所区别的)。

[Predictable long term caching with Webpack](https://medium.com/webpack/predictable-long-term-caching-with-webpack-d3eee1d3fa31)(在 medium 上需要翻墙)，现在基本上网上关于`long-term-caching`的文章都是基于这篇文章的，虽然这篇文章写了快一年多了，但它的东西还是值得一读的，它的作者是 `webpack`第二贡献多的 [timse](https://github.com/timse)，现在基本关于这方面的问题，基本都是 issue 里面贴一下这篇文章，并且来一句

之前作者页发文章说了，他在优化更好的 `long term caching`，

> We plan to add another way to assign module/chunk ids for long term caching, but this is not ready to be told yet.

在 webpack 的 issue 和源码中也经常见到 `Long term caching will be improved in webpack@5`和`TODO webpack 5 xxxx`。真心希望`webpack 5`能真正的解决前面几个问题，并且让它更加的`out-of-the-box`，更加的简单和智能，就像`webpack4`的`optimization.splitChunks`，你基本不用做什么，它就能很好的帮你拆分好`bundle`，同时又给你非常的自由发挥空间。

不让大部分人看到`webpack`都是如下图的状态。

![](https://user-gold-cdn.xitu.io/2018/7/27/164db54515df8fc8?w=1440&h=2626&f=jpeg&s=370108)

## 有效使用浏览器长缓存

[issues/1315](https://github.com/webpack/webpack/issues/1315)

## webpack records

可能很多人连这个配置项都没有注意过，不过早在 2015 年就已经被设计出来让你更好的利用 cache。[官方文档](https://webpack.js.org/configuration/other-options/#recordspath)

要使用它配置也很简单：

```js
recordsPath: path.join(__dirname, "records.json");
```

开启这个选项可以生成一个 JSON 文件，其中含有 webpack 的 `records` 记录 - 即「用于存储跨多次构建(across multiple builds)的模块标识符」的数据片段。可以使用此文件来跟踪在每次构建之间的模块变化。

等于每次构建都是基于上次构建的基础上进行的。所以这时候你在添加或者删除 `chunk`，并不会导致之前所说的乱序，这样就能最大限度的利用浏览器缓存了。

简单看一下构建出来的 JSON 长啥样。

```json
{
  "modules": {
    "byIdentifier": {
      "demo/vendor.js": 0,
      "demo/vendor-two.js": 1,
      "demo/index.js": 2,
      ....
    },
    "usedIds": {
      "0": 0,
      "1": 1,
      "2": 2,
      ...
    }
  },
  "chunks": {
    "byName": {
      "vendor-two": 0,
      "vendor": 1,
      "entry": 2,
      "runtime": 3
    },
    "byBlocks": {},
    "usedIds": [
      0,
      1,
      2
  }
}
```

我们和之前一样，在路由里面添加一个懒加载的页面，打包对比后发现 id 并不会像之前那样按照遍历到的顺序插入了，而是基于之前的 id 依次累加了。

![](https://user-gold-cdn.xitu.io/2018/7/29/164e4a7f0bf37f02?w=1566&h=996&f=jpeg&s=221104)

但这个方案不被大家知晓主要原因就是维护这个`records.json`比较麻烦，如果你是在本地打包运行`webpack`的话，你只要将`records.json`当做普通文件上传到`github`、`gitlab`或其它版本控制仓库。

但现在一般公司都会将打包放在 `CI`里面，用`docker`打包，这时候这份`records.json`存在哪里就是一个问题了。它不经需要每次打包之前先读取你这份 json，打包完之后它还需要再更新这份 json。存在本地或者找一个服务器存，存在什么地其它方都感觉怪怪的。

如果你使用 `Circle CI`可以使用它的`store_artifacts`，相关[教程](https://medium.com/@songawee/long-term-caching-using-webpack-records-9ed9737d96f2)。使用了之后还是放弃了这个方案，使用成本略高。前端打包应该更加的纯粹，不需要依赖太多其它的东西。
