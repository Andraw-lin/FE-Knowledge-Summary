# Webpack 优化--构建速度篇

在日常的开发构建过程或生产打包构建过程中，我们都会遇到过因为构建速度非常缓慢导致耗时严重，不仅得让我们把时间浪费在等待的时间，最重要的还得会直接影响到我们的加班啊...😂

所以优化 Webpack 的构建速度显得尤为重要。当项目庞大时构建的耗时可能会变得更长，每次等待构建的耗时加起来也会是一个大数目。

为此，得总结一下如何在 Webpack 构建过程里加快速度。🤔

## 使用resolve缩少文件的搜索范围

Webpack 在启动后会**从配置的 Entry 入口出发，解析出文件中的导入语句，再递归解析**。

Webpack 在解析文件过程中，遇到导入语句时，会做两件事：

- 根据导入语句去寻找对应的要导入的文件。
- 找到对应文件后，根据匹配到的后缀名，然后使用配置中相应的 Loader 处理该文件。

在小型项目中，上述两个过程都会很快，但是对于一个庞大系统来说，寻找解析就会显得尤为吃力，这样构建速度也会慢慢凸显出来。

> Loader 中使用 include 针对性地处理文件

在 Loader 中通过配置其 include 属性，可**针对性地对主要目录进行递归解析**，可有效地避免处理其他没必要目录的递归解析。

```javascript
module: {
  rules: [
    {
      test: /\.js$/,
      use: ['babel-loader'],
      include: path.resolve(__dirname, 'src') // 只对src目录进行递归解析，其他目录一律不解析
    }
  ]
}
```

> 配置 resolve.modules 减少第三方模块查找流程

如果有了解过 NodeJs 的童鞋，相信对于其加载模块机制会了解，这里就来简单描述一下。

在 Javascript 主进程执行过程中，当遇到要加载的第三方模块时，会在当前文件对应的目录下的 node_modules 文件夹去寻找该模块，若无则会向上一层的目录下的 node_modules （即 ../node_modules 文件夹）去查找，以此类推。

在上面这个过程中，无疑会产生很多没必要的遍历查找过程，为此，**配置 resolve.modules 会直接指定在相应的目录下进行查找，以减少没必要的查找过程**。

```javascript
resolve: {
  modules: [path.resolve(__dirname, 'node_modules')]
}
```

> 配置 resolve.mainFields 减少第三方模块入口查找流程

在开发过程中安装的第三方模块，都会自带一个 package.json 文件，用于描述该模块的属性，包括入口文件的定义。

**第三方模块的入口文件可以有 browser、module、main 三个属性进行定义**。而 **reslove.mainFields 可用于配置直接从哪个属性进行入手加载第三方模块**。默认情况下，reslove.mainFields 和当前环境 target 相关联。

-  当 target 为 web 或者 webworker 时，默认值是["browser", "module", "main"]；
- 当 target 为其他情况时，值是["module", "main"]；

**由于绝大多数第三方模块都采用 main 字段来描述入口，为减少搜索入口步骤，我们可以直接配置 resolve.mainFields 为 main**。

```javascript
resolve: {
  mainFields: ["main"]
}
```

> 配置 resolve.alias 减少完整第三方模块的查找依赖流程

以 Vue 为例，在打包后，都会存在两个文件，一个是 vue.js，另一个是 vue.min.js。前者用于开发环境，而后者用于生产环境。

开发环境下，Webpack 在构建过程遇到需要引入 vue 时，会直接寻找 vue.js。我们应该知道 vue.js 会分为很多个文件组成，在执行 vue.js 时就需要根据不同场景去引入相应模块进行解析，这过程就会导致耗时大。

为此，**配置 resolve.alias 可以让 Webpack 在处理 vue 时，直接使用单独、完整的 vue.min.js 文件，从而跳过耗时的递归解析操作**。

```javascript
resolve: {
  alias: {
    'vue': path.resolve(__dirname, './node_modules/vue/dist/vue.min.js')
  }
}
```

需要注意的是，不是所有的第三方模块都可以这样处理。只有完整的第三方模块才可以，那什么是完整的第三方模块。

我们知道，一个 vue 构建项目，基本每一个源代码都需要用到，因此它是完整的，再比如 lodash.js，我们在开发过程中，只需要按需引入部分方法，而且有绝大部分的源码我们是没必要引用的，因此它是不完整的。

**配置 resolve.alias 只能在需要全部源代码的第三方模块进行使用，按需使用的第三方模块一律不能使用**。

**若强制在不完整的第三方模块中使用 resolve.alias 配置，那么在 Tree-Shaking 过程中就无法去掉没用的代码**，从而增大了包的体积，得不偿失啊...一定要慎用！！！

> 配置 resolve.extensions 减少文件后缀名匹配流程

相信很多童鞋都写过 import MyComponent from 'components/MyComponent' 这样类似语句，可以看到的是，一般情况下引入相应的 Vue 组件或 React 组件，都是不会加入相应的后缀名。

知道为啥不？

那是很多时候，在 Webpack 的 resolve.extensions 中配置了。**在导入语句中没带文件后缀时，配置 resolve.extensions 可自动带上后缀名询问相应文件是否存在，默认值是 ["js", "json"]。**

```javascript
resolve: {
  extensions: ["vue"]
}
```

**配置 resolve.extensions 可针对性地减少文件后缀名的匹配流程。**

> 配置 module.noParse 忽略对未采用模块化文件的递归解析处理

形如 jQuery、ChartJS 等模块都是木有采用模块化标准的，如果要让 Webpack 解析这些文件将会耗费更多时间。

再比如前面讲到的，使用 vue.min.js 文件，由于 min.js 文件未采用模块化标准，都是包含全部源代码的。**通过配置 module.noParse 可忽略对 vue.min.js 文件的递归解析处理。**

```javascript
module: {
  noParse: [/vue\min\.js$/],
}
```



## DllPlugin构建动态链接库

在 window 系统会经常看到 DLL 后缀名文件，这些文件叫动态链接库，在一个动态链接库中保存着其他模块需调用的函数和数据。（这就有点像一张表，表里面保存了其他模块需要调用的函数以及数据对应的地址，方便查找）

要在项目中接入动态链接库的思想，需要做到以下几点。

- 将网页依赖的基础模块抽离出来，打包到一个个单独的动态链接库中，在一个动态链接库中可包含多个模块。
- 当遇到需导入的模块存在于某个动态链接库中时，该模块将不会再被打包，而是直接到动态链接库中获取。
- 页面依赖的所有动态链接库都需要被加载。

**动态链接库加快构建速度的核心思想是，大量复用模块的动态链接库只需要被编译一次，在后面的构建过程中被动态链接库包含的模块都不会重新编译，而是直接使用动态链接库中的代码**。

**动态链接库中常常包含最常用的第三方模块，如 react、react-dom 等**，所以只要不升级这些模块的版本，动态链接库就不需要重新编译。

**在 Webpack 中，已经内置支持对动态链接库**。主要通过两个内置插件接入，分别是

- DllPlugin 插件。用于打包出一个个单独的动态链接库文件
- DllReferencePlugin 插件。用于告诉 Webpack 使用了哪些动态链接库。

以 React 为例，在接入 DllPlugin 后，构建出来的目录结构是如下：

```javascript
|-- main.js
|-- polyfill.dll.js // 包含所有依赖的polyfill，如Promise、fetch等API
|-- polyfill.manifest.json
|-- react.dll.js // 包含React的基础运行模块，如react和react-dom模块
|-- react.manifest.json
```

dll.js 文件中其实蕴含着大量模块的代码，这些**模块代码都被放到一个数组中，用数组的索引作为其模块的 ID 标识，通过一个 dll_ 前缀变量全局暴露出来**，即可以通过 window 进行访问（如 window.dll_react）。

而 **manifest.json 文件是由 DllPlugin 生成的，描述了与对应的 dll.js 文件中包含哪些模块以及每个模块的路径和ID**。

动态链接库文件相关的文件需要一份独立的构建输出，用于主构建使用。新建一个 webpack_dll.config.js 文件

```javascript
// webpack_dll.config.js
const path = require('path')
const DllPlugin = reqiure('webpack/lib/DllPlugin')

module.exports = {
  entry: {
    react: ['react', 'react-dom'],
    polyfill: ['core-js/fn/object/assign', 'core-js/fn/promise', 'whatwg-fetch']
  },
  output: {
    filename: '[name].dll.js',
    path: path.resolve(__dirname, 'dist'),
    // 存放动态链接库的全局变量名称，例如react就是_dll_react
    // 加上_dll_是为了防止全局变量冲突
    library: '_dll_[name]'
  },
  plugins: [
    new Dllplugin({
      // 动态链接库的全局变量名称，需和output.library中保持一致
      // 该字段也是输出的manifest.json文件中name字段的值，如在react.manifest.json文件中就有name: "_dll_react"
      name: '_dll_[name]',
      // 定义manifest.json文件输出时的文件名称
      path: path.join(__dirname, 'dist', '[name].manifest.json')
    })
  ]
}
```

接下来看看，如何在主流程的 webpack.config.js 中配置。

```javascript
// webpack.config.js
const path = require('path')
const DllReferencePlugin = require('webpack/lib/DllReferencePlugin')

module.exports = {
  entry: {
    main: './main.js'
  },
  output: {
    filename: '[name].js',
    path: path.resolve(__dirname, 'dist')
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        use: ['babel-loader'],
        exclude: path.resolve(__dirname, 'node_modules')
      },
    ]
  },
  plugins: [
    // 告诉webpack使用了哪些动态链接库
    new DllReferencePlugin({
      manifest: require('./dist/react.manifest.json')
    }),
    new DllReferencePlugin({
      manifest: require('./dist/polyfill.manifest.json')
    })
  ]
}
```

在执行时，先将动态链接库相关文件编译出来，因为 Webpack 配置文件中定义的 DllReferencePlugin 依赖这些文件。因此在执行构建时流程如下：

- 第一次执行时，动态链接库相关文件还没编译出来，需执行 webpack --config webpack_dll.config.js 命令，先编译得到动态链接库相关文件。
- 第二次执行以后，由于动态链接库相关文件已经存在，可直接执行 webpack 命令运行项目。会发现项目构建速度比第一次快很多，因为动态链接库文件只会编译一遍。



## HappyPack多个进程处理文件

**在使用 HappyPack 时，可进行分解任务和管理线程**。先来看看 webpack 如何接入 HappyPack。

```javascript
const path = require('path')
const ExtractTextPlugin = require('extract-text-plugin')
const HappyPack = require('happypack')

module.exports = {
  module: {
    rules: [
      {
        test: /\.js$/,
        // 将对.js文件的处理交给id为babel的happypack实例
        use: ['happypack/loader?id=babel'],
        exclude: path.resolve(__dirname, 'dist', 'node_modules')
      },
      {
        test: /\.css$/,
        // 将对.css文件的处理交给id为babel的happypack实例
        use: ExtractTextPlugin.extract({
          use: ['happypack/loader?id=css']
        })
      }
    ]
  },
  plugins: [
    // 创建happypack实例处理任务
    new HappyPack({
      id: 'babel',
      loaders: ['babel-loader?cacheDirectory'],
      // ...其他配置项
    }),
    // 创建happypack实例处理任务
    new HappyPack({
      id: 'css',
      loaders: ['css-loader'],
      // ...其他配置项
    }),
    new ExtractTextPlugin({
      name: '[name].css'
    })
  ]
}
```

上述配置中，所有文件的处理都交给了 happypack/loader ，根据 id 来告诉 happypack/loader 选择哪个 happypack 实例处理文件。

另外，在 happypack 实例中，还支持使用三个参数，分别是

- threads：代表**开启几个子进程处理**这类型的文件，默认是3个。
- verbose：是否允许使用 happypack **输出日志**，默认是true。
- threadPool：代表共享进程池，**多个 Happypack 实例都是使用同一个共享进程池中的子进程去处理任务**，以防止资源占用过多。

一般情况下，可按如下接入 happypack。

```javascript
const happyThreadPool = HappyPack.ThreadPool({ size: 5 }) // 构造共享进程池，在进程池中包含5个子进程

modules.export = {
  plugins: [
    // 创建happypack实例处理任务
    new HappyPack({
      id: 'babel',
      loaders: ['babel-loader?cacheDirectory'],
      // ...其他配置项
      threadPool: happyThreadPool
    }),
    // 创建happypack实例处理任务
    new HappyPack({
      id: 'css',
      loaders: ['css-loader'],
      // ...其他配置项
      threadPool: happyThreadPool
    }),
    new ExtractTextPlugin({
      name: '[name].css'
    })
  ]
}
```

在使用 HappyPack 前，需如下安装：

```javascript
npm i -D happypack
```

**在 webpack 构建流程中，使用 loader 对文件的转换操作无疑是最耗时的流程**。

HappyPack 核心原理就是**将 loader 对文件的转换操作分解到多个进程中去并行处理，从而减少构建时间**。

实例化 HappyPack 时，其实就是告诉 HappyPack 核心调度器通过一系列 loader 去转换一类文件，并且指定如何为该类文件转换操作分配子进程。**在执行 webpack 时，核心调度器会将一个个任务分配给当前空闲子进程，子进程处理完毕就会将结果发送给核心调度器，它们间的数据交换就是通过进程间的通信 API 实现的**。



## ParalleUglifyPlugin多个进程压缩文件

在发生产时，我们都经常使用 uglifyJS 对源代码进行压缩处理。在压缩代码过程中，需要先将 js 代码解析成抽象语法树 AST，接着再去应用各种规则分析和处理 AST，这过程计算量可谓是巨大的，耗时特别大。

**使用 uglifyJS 会对文件进行一个一个的压缩，并不会并行处理，这无疑是耗时最大的**。借鉴于 HappyPack 多进程分配任务优点，**使用 ParalleUglifyPlugin 可以开启多个子进程，将对多个文件的压缩工作分配给多个子进程完成，每个子进程其实都是使用 uglifyJS 进行压缩，但是区别就是并行处理**。

```javascript
const ParalleUglifyPlugin = require('webpack-parallel-uglify-plugin')

module.exports = {
  plugins: [
    // 使用ParallelUglifyPlugin并行压缩输出Javascript代码
    new ParallelUglifyPlugin({
      // 传递给uglifyJS的参数
      uglifyJS: {
        output: {
          // 紧凑输出
          beautify: false,
          // 删除所有注释
          comments: false
        },
        compress: {
          // 在UglifyJs删除没有用到的代码时不输出警告
					warnings: false,
          // 删除所有的console语句，可兼容IE浏览器
          drop_console: true,
          // 内嵌已定义但是只用到一次的变量
          collapse_vars: true
        }
      }
    })
  ]
}
```

当然 ParalleUglifyPlugin 还支持其它参数，就不一一列举，可自行观察其官方文档。需要注意的是，**ParalleUglifyPlugin 同时内置了 UglifyJS 和 UglifyES，UglifyES 可并行压缩 ES6 代码**。

在使用 ParalleUglifyPlugin 时，需安装：

```javascript
npm i -D ParalleUglifyPlugin
```



