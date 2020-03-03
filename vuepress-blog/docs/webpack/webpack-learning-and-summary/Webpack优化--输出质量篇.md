# Webpack 优化--输出质量篇

我们都知道，使用 Webpack 打包后的文件大小，会直接影响到线上用户的体验，如首屏加载时间，包越大时相应的打开时长也会越长，毕竟还是需要时间和网络下载一个包的。😅

那么，我们应该如何去使用 Webpack 更好滴优化打包后的文件呢？接下来我就来简单讲解一些常用方法。


## 使用process区分环境

在我们开发 web 时，一般都会分为两个环境，分别是开发环境和生产环境。开发环境就是用于开发者开发时调试使用的，而生产环境则是针对真实用户使用的。

**Webpack 已经内置了区分环境功能，当代码中使用 process 模块的语句时，Webpack 就会自动打包进 process 模块的代码以支持非 Node.js 环境**。因此，我们可以直接在代码中这样区分环境。

```javascript
if(process.env.NODE_ENV === 'production') {
  console.log('生产环境')
} else {
  console.log('开发环境')
}
```

在 Webpack 4 里，已经支持使用 mode 属性来配置环境。当然也可以如下配置。

```javascript
const DefinePlugin = require('webpack/lib/DefinePlugin')

module.exports = {
  plugins: [
    new DefinePlugin({
      'process.env': {
        NODE_ENV: JSON.stringify('production')
      }
    })
  ]
}
```

上述配置中，为什么要使用 JSON.stringify 方法定义？环境变量的值必须是一个由双引号包裹的字符串，即"'production'"。

另外，也可以直接在 Webpack 执行命令上带上环境参数。

```javascript
webpack NODE_ENV=production
```

这时候在上面的 new DefinePlugin 中就需要这样去获取环境参数进行定义。

```javascript
new DefinePlugin({
  'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV)
})
```



## 利用UglifyJS对JavaScript压缩，利用cssnano对css压缩

在性能提升方面，相信我们都会知道使用 GZip 对文件进行压缩，那么对于文件内容进行压缩则可以选用 UglifyJS，同时 Webpack 已经内置 UglifyJS。

在这里，可以使用 ParallelUglifyJS 开启多个进程对代码进行压缩来加快构建进程，但是 ParallelUglifyJS 底层都是使用 UglifyJS 进行工作的，所以我们有必要了解一下如何使用 UglifyJS 进行代码压缩。

**使用 UglifyJS 对代码进行压缩时，除了可以提升网页加载的速度，还可以混淆源码**。

UglifyJS 的基本原理就是，**分析 Javascript 代码语法树，理解代码的含义，从而做到去掉无效代码、去掉日志输出代码、缩短变量名等优化**。

UglifyJS 中常用的选项如下：

- sourceMap：是否为压缩后的代码生成对应的 Source Map，默认不生成，开启后耗时会大大增加。
- output.beautify：是否要保留空格和制表符，默认为保留，为了达到更好的压缩效果，可设置 false。
- output.comments：是否保留代码中的注释，默认保留，可设置为 false。
- compress.warning：是否删除没有用到的输出警告信息，默认为输出，可设置为 false。
- compress.drop_console：是否删除代码中所有的console语句。
- compress.collapse_vars：是否内嵌虽然已定义但是只用到一次的变量，如 var x = 5; y = x; 转换成 y = 5，默认为不转换，可设置为 true。
- compress.reduce_vars：是否提取出现多次但是没定义成变量去引用的静态值。如 x = 1; y = 1; 转换成 var a = 1; x = a; y = a，默认不转换，可以设置为 true。

对于使用 UglifyJs 进行配置如下：

```javascript
const UglifyJsPlugin = require('webpack/lib/optimize/UglifyJsPlugin')

module.exports = {
  plugins: [
    new UglifyJsPlugin({
      compress: {
        warnings: false,
        drop_console: true,
        collapse_vars: true,
        reduce_vars: true
      },
      output: {
        beautify: false,
        comments: false
      }
    })
  ]
}
```

Webpack 内置的 UglifyJsPlugin 用的是 UglifyJS2。**当执行 webpack --optimize-minimize 命令时，会自动注入 UglifyJSPlugin**。

需要注意的是，**UglifyJS 只理解 ES5 语法的代码，因此一般是需要结合 Babel 一起使用。若只是单纯压缩 ES6 代码，则需要用到 UglifyES**。

如果需要在 Webpack 接入 UglifyES，则需要单独安装最新版的 uglifyjs-webpack-plugin。

```javascript
npm i -D uglifyjs-webpack-plugin@beta

// 配置如下
const UglifyEsPlugin = reqire('uglifyjs-webpack-plugin')

module.exports = {
  plugins: [
    new UglifyEsPlugin({
      compress: {
        warnings: false,
        drop_console: true,
        collapse_vars: true,
        reduce_vars: true
      },
      output: {
        beautify: false,
        comments: false
      }
    })
  ]
}
```

**在接入 UglifyES 时，需要 babel-loader 不能转换 ES5 代码，即需要在 .babelrc 中去掉 babel-preset-env**。

既然 JavaScript 代码可以压缩，那么 css 代码能否也压缩一下？

**如果压缩 css 代码，需要用到的压缩工具则是 cssnano，基于 PostCSS 实现的**。

举个例子，在 css 中，margin: 10px 20px 10px 20px; 会被转换成 margin: 10px 20px; 、color: #000 转换成 color: black;。

那么在 Webpack 中怎么接入 cssnano 呢？

其实**在 css-loader 中已经内置了 cssnano，若需开启，则只需要加上 minimize 选项即可**。

```javascript
module.exports = {
  module: {
    rule: [
      {
        test: /\.css$/,
        use: ExtractTextPlugin.extract({
          use: ['css-loader?minimize']
        })
      }
    ]
  }
}
```



## 配置publicPath指向CDN服务加速

相信大家对 CDN 并不陌生，**CDN 可以加速网络传输，通过将资源部署到世界各地，使用户在访问时按照就近原则从离最近的服务器上获取资源，从而加快资源的获取**。

在项目中常用的 CDN 方式是如下：

- **对于项目中 HTML 文件，直接放在项目根目录下储存，而不是放到 CDN 服务上**，同时需要关闭项目服务器的缓存，项目服务器只提供 HTML 文件和数据接口。
- **针对打包好的 JavaScript、CSS、图片等静态文件，都放在 CDN 服务上，并开启 CDN 缓存**，同时为每个文件带上一个 hash 值，如 a.asvf1231sd.js 文件。带上 hash 原因是，当发版新的代码到生产上，避免用户依旧使用的是旧版代码，每次发版都会更新相应的 hash 值告诉浏览器拉取最新的代码。

由于浏览器有一个规则是，**在同一时刻针对同一个域名的资源的并行请求有限制，一般为 4 个左右，不同的浏览器可能不同**。**为了避免将所有的 JavaScript、CSS、图片等静态文件只放在一个服务上，可以尝试将这些静态资源分散到不同的 CDN 服务上**。（即 JavaScript 文件放到单独的 js CDN 服务上，CSS 文件放到单独的 css CDN 服务上）

既然如此，Webpack 应该如何接入 CDN 呢？要接入 CDN 服务，需要实现以下几点。

- 使用 publicPath 指向 CDN 服务的绝对路径地址，而不是指向项目的服务地址上。
- 静态资源的文件名需要带上相应的 hash 值，避免被缓存。
- 将不同类似的资源放到不同的 CDN 服务上，避免资源的并行下载限制。

直接看看在 Webpack 中如何配置的。

```javascript
const path = require('path')
const ExtractTextPlugin = require('extract-text-webpack-plugin')
const {WebPlugin} = require('web-webpack-plugin')

module.exports = {
  output: {
    filename: '[name]_[chunkhash:8].js',
    path: path.resolve(__dirname, './dist'),
    // 指定存放JavaScript文件的CDN服务地址
    publicPath: '//js.cdn.com/id'
  },
  module: {
    rules: [
      {
        test: /\.css$/,
        use: ExtractTextPlugin.extract({
          use: ['css-loader?minimize'],
          // 指定存放CSS中导入的资源（如图片）的CDN服务地址
          publicPath: '//img.cdn.com/id'
        })
      },
      {
        test: /\.png$/,
        // 为输入的PNG文件加上Hash值，并指定hash的长度为8位
        use: ['file-loader?name=[name]_[hash:8].text']
      }
    ]
  },
  plugins: [
    new WebPlugin({
      // HTML模板所在位置
      template: './template.html',
      // 输出的HTML文件名
      filename: 'index.html',
      // 指定存放css文件的CDN服务地址
      stylePublic: '//css.cdn.com/id/'
    }),
    new ExtractTextPlugin({
      // 为输出的CSS文件名加上hash值，并指定hash的长度为8位
      filename: '[name]_[contentHash:8]'
    })
  ]
}
```



## 使用Tree-Shaking去除多余的无用代码

**Tree Shaking 可用于剔除 JavaScript 中用不上的死代码，是基于静态的 ES6 模块化语法的**（即依赖于 import 和 export）。

简单滴来说，Tree Shaking 不支持其他模块化如 RequireJS、CommonJS 等。

接下来就来看看，是如何配置 Tree Shaking 的。🤔

首先，需要在 babel 的配置下，禁止 ES6 模块化转换成 ES5 的形式，因此在 .babelrc 文件中配置如下：

```javascript
{
  "presets": [
    "env",
    {
      "modules": false
    }
  ]
}
```

上述的 **modules: false 的含义就是关闭 babel 的模块转换功能，保留原本的 ES6 模块化语法**。

从 Webpack 2.0 开始，Webpack 自带 Tree Shaking 功能。另外，在执行 webpack 命令时带上 --display-used-exports 参数，可追踪 Tree Shaking 的工作。

**Webpack 只是指出哪些函数被用上，而哪些函数没被用上，以及腰剔除用不上的代码，则还要经过 UglifyJS 处理一番。**

在使用大量的第三方库时，会发现 Tree Shaking 似乎并不生效，原因是大部分 NPM 包中的代码都采用了 CommonJS 模块化语法。当然有些库已经考虑到这一点，在发布到 NPM 上时会同时提供两份代码，一份用于 CommonJS 模块化语法，一份采用 ES6 模块化语法，并且在 package.json 文件中会分别指出这两份代码的入口。

以 Redux 为例，在其 package.json 中会这样指明入口：

```javascript
{
  // ...
  "main": "lib/index.js", // 指明采用CommonJS模块化的代码入口
	"jsnext:main": "es/index.js" // 指明采用ES6模块化的代码入口
  // ...
}
```

在前面已经提到过，resolve.mainFields 用于配置采用哪个字段作为模块的入口描述，因此，为了让 Tree Shaking 对上面的 Redux 有效，需要配置如下：

```javascript
module.exports = {
  resolve: {
    // 针对NPM中的第三方模块优先采用jsnext:main中指向ES6模块化语法的文件
    mainFields: ['jsnext:main', 'browser', 'main']
  }
}
```

采用 jsnext:main 作为 ES6 模块化代码的入口是社区的一个约定。



## 提取公共代码和第三方代码

当每个页面的代码都将公共的代码包含进去，就会造成：

- 相同的资源被重复加载，浪费用户的流量和服务器的成本；
- 每个页面需要加载的资源太大，导致网页首屏加载缓慢，影响用户体验；

需要说明的是，**在 Webpack 3.0 中使用插件 CommonChunkPlugin 提取公共部分，在 Webpack 4.0 后则需要在 optimization.splitChunk 中进行配置提取公共部分**。在官方文档中提到，虽然 CommonChunkPlugin 可用来避免使用重复的依赖，但若想进一步的优化是不可能的。

通常情况下，会按照以下规则进行提取公共代码。

- 将项目中所用到的**第三方代码统一打包**到一个文件中（如vendors.js）。
- 在剔除第三方代码后，再找出**所有页面都依赖的公共部分代码**，将它们提取出来并放到一个文件中（如commons.js）。
- 最后再为每个页面都生成一个单独的文件，在这个文件中不再包含第三方代码以及公共部分代码。

也许你会觉得很奇怪，为什么不可以将第三方代码都放到公共部分代码中去？

其实就是为了**长期缓存第三方代码**。由于第三方代码是基本不会更新，相反页面依赖的公共部分代码则会根据业务需求的不同而发生变化，因此需区别对待。

现在我们就来看看，在 Webpack 4.0 中是如何进行提取第三方代码和页面依赖的公共部分代码的。

```javascript
module.exports = {
  // ...
  optimization: {
    splitChunks: {
      chunks: 'initial', // 表示同步模块，有三个值，分别是initial(初始化，打包第三方代码用到)、all(所有块)、async(异步块，按需加载用到) 
      minSize: 30000,  // 块的最小值
      maxSize: 0, // 块的最大值
      minChunks: 1, // 拆分前必须共享模块的最小块数
      maxAsyncRequests: 5, // 按需加载时最大并行请求数
      maxInitialRequests: 3, // 入口点的最大并行请求数
      automaticNameDelimiter: '~', // 默认情况下，webpack将使用块的来源和名称生成名称（如vendors～main.js）
      name: true, // 拆分块的名称，提供true将基于块和缓存组密钥自动生成一个名称
      cacheGroups: { // 缓存模块
        vendors: { // 基本框架
          chunks: 'all',
          test: /(react|react-dom|react-dom-router|babel-polyfill|mobx)/,
          priority: 100,
          name: 'vendors',
        },
        d3Venodr: { // 将体积较大的d3单独提取包，指定页面需要的时候再异步加载
          test: /d3/,
          priority: 100, // 设置高于async-commons，避免打包到async-common中
          name: 'd3Venodr',
          chunks: 'async'
        },
        echartsVenodr: { // 异步加载echarts包
          test: /(echarts|zrender)/,
          priority: 100, // 高于async-commons优先级
          name: 'echartsVenodr',
          chunks: 'async'
        },
        'async-commons': { // 其余异步组件加载包
          chunks: 'async',
          minChunks: 2,
          name: 'async-commons',
          priority: 90,
        },
        commons: { // 其余同步加载包
          chunks: 'all',
          minChunks: 2,
          name: 'commons',
          priority: 80,
        }
      }
    }
  }
}
```

对于 splitChunks 中的 chunks，需要说明的是：

- all：不管文件是动态还是非动态载入，统一将文件分离。当页面首次载入会引入所有的包。
- inital：将异步和非异步的文件分离，如果一个文件被异步引入也被非异步引入，那它会被打包两次（注意和all区别），用于分离页面首次需要加载的包。（**第三方代码引入**）。
- async：将异步加载的文件分离，首次一般不引入，到需要异步引入的组件才会引入（即**按需引入**）。



## 提取懒加载代码（即按需加载的代码）

在上面使用 splitChunkPlugin 提取代码中，其实已经有提及到对懒加载代码的提取，就是 chunk 值为 async 。

相信用过 React、Vue 的童鞋们，肯定对 import() 方法不陌生，它就是用来实现组件的懒加载的。

**import() 返回一个 Promise，依赖于 Promise**。下面就使用 React-Router 实现组件懒加载栗子。

```javascript
import React, {PureComponent, createElement} from 'react'
import ReactDom from 'react-dom'

function getAsyncComponent(load) {
  return class AsyncComponent extends PureComponent {
    componentDidMount() {
      load().then({ default: component } => {
        this.setState({
          component
        })
      })
    }
    render() {
      const {component} = this.state
      return component ? createElement(component) : null
    }
  }
}

function App() {
  return (
  	<HashRouter>
    	<div>
    		<nav><link to="/about">About</link></nav>
    	</div>
      <Route path="/about" component={getAsyncComponent(() => import(/* webpackChunkName: about */ './pages/about'))}></Route>
    </HashRouter>
  )
}
```

以上代码需要在 Webpack 中配置好相应的 babel-loader，将源码先提交给 babel-loader 处理，再提交给 Webpack 处理其中的 import(\*) 语句。但是由于 babel-loader 并不认识 import(\*) 语句，因此会报错。

为了让 babel-loader 处理 import(\*) 语句，需要安装一个 babel 插件 babel-plugin-syntax-dynamic-import。需在 .babelrc 文件做如下配置。

```javascript
{
  "presets": [
    "env", "react"
  ],
  "plugins": [
    "syntax-dynamic-import"
  ]
}
```



## 开启Scope Hoisting

Scope Hoisting 可以让 Webpack 打包出来的代码文件更小、运行更快，又称为作用域提升。

**Scope Hoisting 实现的基本原理是，分析模块之间的依赖关系，尽可能将被打散的模块合并到一个函数中，但前提是不能造成代码冗余**。

那么要开启 Scope Hoisting，在 Webpack 中如何配置呢？**只需要用到内置的 ModuleConcatenationPlugin 插件即可开启 Scope Hoisting**。

```javascript
const ModuleConcatenationPlugin = require('webpack/lib/optimize/ModuleConcatenationPlugin')

module.exports = {
  plugins: [
    new ModuleConcatenationPlugin()
  ]
}
```

需要注意的是，Scope Hoisting 依赖源码时需采用 ES6 模块化代码，还需配置 resolve.mainFields。因为大部分 NPM 包的第三方库都是采用 CommomJS 语法，因此需配置 resolve.mainFields 指向使用 ES6 模块化代码。

```javascript
module.exports = {
  resolve: {
    mainFields: ['jsnext:main', 'browser', 'main']
  }
}
```

当然，对于采用了非 ES6 模块化的代码，Webpack 则会降级处理且不使用 Scope Hoisting 进行优化。









































