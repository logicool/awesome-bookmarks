前几天 webpack 作者 [Tobias Koppers](https://github.com/sokra) 发布了一篇新的文章 [webpack 4.0 to 4.16: Did you know?](https://medium.com/webpack/webpack-4-0-to-4-16-did-you-know-71e25a57fa6b)(需翻墙)，总结了一下`webpack 4`发布以来，做了哪些调整和优化。
并且说自己正在着手开发 `webpack 5`。

> Oh you are still on webpack 3. I’m sorry, what is blocking you? We already working on webpack 5, so your stack might be outdated soon…

翻译成中文就是：

![](https://user-gold-cdn.xitu.io/2018/7/27/164db1a2e089c151?w=690&h=200&f=jpeg&s=22433)

正好我需要使用一个文档生成工具 [docz](https://github.com/pedronauck/docz)(安利一波) 也最底需要`webpack 4+`，新版`webpack`性能提高了不少，而且`webpack 4` 都已经发布五个多月了，想必应该已经没什么坑了，我可以安心的按照别人写的升级攻略升级了。之前之所以迟迟不升级完全是被去年被 webpack3 坑哭了。在 `code splitting` 的情况下 `CommonsChunkPlugin`会 完全失效了。过了好一段时间才修复，欲哭无泪。

所以这次我等了快大半年才准备升级到`webpack 4` **但玩玩没想到还是遇到了不少的问题！** 很多之前遗留的问题还是没有很好地解决。
最主要问题是它的文档还是有所欠缺，已经废除了的东西如`commonsChunkPlugin`还在官方文档中到处出现，很多重要的东西却一笔带过，甚至没写，需要用户自己去看源码解决。

比如在`v4.16.0`版本中废除了`optimization.occurrenceOrder`、`optimization.namedChunks`、`optimization.hashedModuleIds`、`optimization.namedModules` 这几个配置项，换成`optimization.moduleIds` 和 `optimization.chunkIds`来代替，但文档完中全没有任何体现，所以你在新版本中还按照文档那样配置其实是没有任何效果的。

由于篇幅有些长，所以拆解成了两篇文章：

- 上篇-就是普通的在`webpack 3` 的基础上升级，要做哪些操作和遇到了哪些坑。
- 下篇-是在`webpack 4`下怎么合理的打包，并且最大化利用 `long term caching`

# 升级篇

## 默认配置

ModuleConcatenationPlugin

因为`webpack4`改了 `hook` 所以所有的`loaders`、`plugins`都需要升级。

使用命令行 `npm outdated` 可以列出所以可以更新的包。

![](https://user-gold-cdn.xitu.io/2018/7/27/164da832e18a97ef?w=890&h=346&f=jpeg&s=105563)

反正把`devDependencies`的依赖都升级一下总归没有错。

### html-webpack-plugin

用最新版本的的 `html-webpack-plugin`你可能还会遇到如下的错误

`throw new Error('Cyclic dependency' + nodeRep)`

解决方案使用 Alpha 版本，`npm i --save-dev html-webpack-plugin@next`

或者加入`chunksSortMode: 'none'`就可以了，`vue-cli@3`中默认帮你加好了。

[具体 issue](https://github.com/jantimon/html-webpack-plugin/issues/870)

其它 html-webpack-plugin 和之前使用不需要该什么东西。

## mini-css-extract-plugin

由于`webpack4`之后对 css 模块支持的逐步完善以及在处理 css 文件提取的方式上也做了些调整，之前我们首选使用的`extract-text-webpack-plugin`也完成了它的历史使命，将让位于`mini-css-extract-plugin`。

使用方式也很简单，大家看着[文档](https://github.com/webpack-contrib/mini-css-extract-plugin#minimal-example)抄就可以了。

它与`extract-text-webpack-plugin`最大的区别是：它在`code spliting`的时候会将原先内联写在每一个 js `chunk bundle`的 css，单独拆成了一个个 css 文件。

![](https://user-gold-cdn.xitu.io/2018/7/24/164cb85b234d224a?w=2534&h=98&f=jpeg&s=50714)

这样做最大的好处是 js 和 css 改动相互不会影响对方。比如我改了 js 并不会导致 css 文件的缓存失效。而且现在它会配合`optimization.splitChunks`可以拆分 css 文件，比如我单独配置了`element-ui`作为单独一个`bundle`,它会自动也将它单独打包成一个 css 文件，不会像以前默认将第三方的 css 全部打包成一个几十甚至上百 KB 的`app.xxx.css`文件了。

![](https://user-gold-cdn.xitu.io/2018/7/24/164cbd49dc148656?w=516&h=254&f=png&s=73856)

但有一个需求特别注意的地方，默认文档中它是这样配置的：

```js
new MiniCssExtractPlugin({
  // Options similar to the same options in webpackOptions.output
  // both options are optional
  filename: devMode ? "[name].css" : "[name].[hash].css",
  chunkFilename: devMode ? "[id].css" : "[id].[hash].css"
});
```

> 简单说明一下 `filename` 是指在你入口文件`entry`中引入生成出来的文件名，而`chunkname`是指那些未被在入口文件`entry`引入，但又通过按需加载（异步）模块的时候引入的文件。

**copy**代码使用之后发现情况不对！每次改动一个`xx.js`文件，它对应的 css 虽然没做任何改动，也会发生变比。仔细对比发现原来是 `hash` 惹的祸。 `6.f3bfa3af.css` => `6.40bc56f6.css`

![](https://user-gold-cdn.xitu.io/2018/7/24/164cbe27801ebf69?w=1214&h=308&f=png&s=162463)

但我这是根据官方文档来写的！为什么还有问题！后来在文档的**最最最**下面发下了这么一段话！

> For long term caching use filename: `[contenthash].css`. Optionally add [name].

不是很理解，这么关键的一句话会放在`Maintainers`还后面的地方，默认写在配置里面提示大家不是更好？有人已经开了将文档默认 `chunkhash` => `contenthash`的 文档修改[issue](https://github.com/webpack/webpack.js.org/issues/2096)。

这个真的满过分的，稍不注意就会自己的 css 文件缓存无效，很多用户其实每次修改都不会在意自己最终打包出来的 `dist`文件夹中有哪些改变。可能这个问题就一直存在了。我觉得大家觉得 webpack 难用不是不无道理的。

### 这里在简单说明一下几种 hash 的区别：

- **hash**

`hash` 和每次 `build`有关，没有任何改变的情况下，每次编译出来的 `hash`都是一样的，但当你改变了任何一点东西，它的`hash`就会发生改变。

简单理解，你改了任何东西，`hash` 就会和上次不一样了。

- **chunkhash**

`chunkhash`是根据具体每一个模块文件自己的的内容包括它的依赖计算所得的`hash`，所以某个文件的改动只会影响它本身的`hash`，不会影响其它文件。

- **contenthash**

它的出现主要是为了解决`css`文件不受`js`文件的改变而改变。比如`foo.css`被`foo.js`引用了，所以它们共用相同的`chunkhash`值。但是这样子有个问题，如果`foo.js`更改了代码，`css`文件就算内容没有任何改变，由于是该模块发生了改变，其`css`文件的`hash`也会随之改变。

这个时候我们就可以`contenthash`，保证即使`css`文件所处的模块里有任何内容的改变，只要 css 文件内容不变，那么它的`hash`就不会发生变化。

`contenthash` 你可以简单理解为 `moduleId` + `content` 生成的 `hash`。

## 热更新速度

相对 webpack 打包的速度，我更关心的本地开发热更新速度，这才是和我们每一个程序员每天打交道的东西，打包一般都扔给`CI`，自动执行，而且也没什么项目需要每天打包很多次。

`webpack 4`一直说更好的利用了`cache`提高了编译速度，但实测发现是有一定的提升，但当你一个项目，路由懒加载的页面多了之后，50+之后，热更新慢的问题会很明显，之前的[文章](https://juejin.im/post/595b4d776fb9a06bbe7dba56#heading-1)中也提到过这个问题，原以为新版本会解决这个问题，但并没有。

不过你首先要排斥你的热更新慢不是如：

- 没有使用合理的 [Devtool](https://webpack.js.org/configuration/devtool/#devtool) souce map 导致
- 没有正确使用 [exclude/include](https://webpack.js.org/configuration/module/#rule-include) 处理了不需要处理的如`node_modules`
- 在开发环境没有压缩代码`UglifyJs`、提取 css、babel polyfill、计算文件 hash 等不需要的操作

**旧方案**
最早的方案是开发环境中不是用路由懒加载了，只在线上环境中使用。封装一个`_import`函数，通过环境变区分是否需要懒加载。

开发环境：

```js
module.exports = file => require("@/views/" + file + ".vue").default;
```

生成环境：

```js
module.exports = file => () => import("@/views/" + file + ".vue");
```

但由于 webpack `import`实现机制有关，会产生一定的副作用，`@/views/`下的 `.vue` 文件都会被打包。不管你是否被依赖，会多打包一些可能永远都用不到 js 代码。

目前的解决方案思路还是一样的，只在生成模式中使用路由懒加载。但换了一种实现方式。

**新方案**

使用`babel` 的 `plugins` [babel-plugin-dynamic-import-node](https://github.com/airbnb/babel-plugin-dynamic-import-node)。它只做一件事就是：将所有的`import()`转化为`require()`，这样就可以用这个插件将所有异步组件都用同步的方式引入，并结合 [BABEL_ENV](https://babeljs.io/docs/usage/babelrc/#env-option) 这个`bebel`环境变量，让它只作用于开发环境下。将开发环境中所有`import()`转化为`require()`，这种方案解决了之前重复打包的问题，同时对代码的侵入性也很小，你平时写路由的时候只需要按照官方[文档](https://router.vuejs.org/zh/guide/advanced/lazy-loading.html)路由懒加载的方式就可以了，其它的都交给`bable`来处理，当你不想用这个方案的时候，也只要将它从`babel` 的 `plugins`中移除就可以了。

**具体代码：**

首先在`package.json`中增加`BABEL_ENV`

```json
"dev": "BABEL_ENV=development webpack-dev-server XXXX"
```

接着在`.babelrc`只能加入`babel-plugin-dynamic-import-node`这个`plugins`，并让它只有在`development`模式中才生效。

```json
{
  "env": {
    "development": {
      "plugins": ["dynamic-import-node"]
    }
  }
}
```

之后就大功告成了，路由只要像平时一样写就可以了。[文档](https://panjiachen.github.io/vue-element-admin-site/zh/guide/advanced/lazy-loading.html#%E6%96%B0%E6%96%B9%E6%A1%88)

```js
 { path: '/login', component: () => import('@/views/login/index')}
```

这样能大大提升你热更新的速度。当然你的项目本身就不大页面也不多，完全没必要搞这些。**当你的页面变化更不是你的代码速度的时候再考虑也不迟。**

## 打包速度

`webpack 4` 在项目中实际测了下，普遍能提高 20%~30%的打包速度。

本文不准备太深入的讲解这部分内容，详细的打包优化速度可以参考[slack 团队的这篇文章](https://slack.engineering/keep-webpack-fast-a-field-guide-for-better-build-performance-f56a5995e8f1)，掘金还有[译文](https://github.com/xitu/gold-miner/blob/master/TODO/keep-webpack-fast-a-field-guide-for-better-build-performance.md).

这里有几个建议来帮你加速 webpack 的打包。

首先你需要知道你目前打包慢，是慢在哪里。其实大部分打包花费的时间是在`Uglifyjs`压缩代码。和前面的提升热更新差不多，`source map`的正确与否，`exclude/include`的正确使用等等。

使用新版的`UglifyJsPlugin`的时候记住可以加上`cache: true`、`parall: true`，可以提搞代码打包压缩速度。更多配置可以参考 [文档](https://github.com/webpack-contrib/uglifyjs-webpack-plugin) 或者 vue-cli 的 [配置](https://github.com/vuejs/vue-cli/blob/dev/packages/@vue/cli-service/lib/config/uglifyOptions.js)。

编译的时候还有还有一个很慢的原因是那些第三方库。比如`echarts`、`element-ui`其实都非常的大，比如`echarts`打包完有 775kb。所以你想大大最加编译速度，可以将这些第三方库 `externals` 出去，使用`script`的方式引入，或者使用 `dll`的方式打包。经测试一般如`echarts`这样大的包可以节省十几秒到几十秒不等。

还是可以使用一些并行执行 webpack 的库：如[parallel-webpack](https://github.com/trivago/parallel-webpack)、[happypack](https://github.com/amireh/happypack)。

**顺便说一下，升级一下`node`可能有惊喜。前不久将`CI`里面的 node 版本依赖从 `6.9.2` => `8.11.3`，打包速度直接提升了一分多钟。**

总之我觉得打包时间控制在差不多的范围内就可以了，没必要过分的优化。你研究了半天，改了一堆参数发现其实也就提升了几秒，但维护成本上去了，得不偿失。还不如升级 node、升级 webpack、升级你的编译环境的硬件水平实在和简单。

比如我司`CI`使用的是腾讯云 一般的机器，这个项目也是一个很大的后台项目两百加页面，没有使用什么`happypack`、`dll`，只是升级用了最新版的`webpack4`，`node:8.11.3`。
编译速度稳定在两分多钟，完全不觉得有什么要优化的必要了。

![](https://user-gold-cdn.xitu.io/2018/7/29/164e5366dd1d9dec?w=896&h=236&f=jpeg&s=22563)

参考文章：

- [Long term caching using Webpack records](https://medium.com/@songawee/long-term-caching-using-webpack-records-9ed9737d96f2)
- [Predictable long term caching with Webpack](https://medium.com/webpack/predictable-long-term-caching-with-webpack-d3eee1d3fa31)
