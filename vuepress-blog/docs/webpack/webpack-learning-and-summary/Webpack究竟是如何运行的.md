# Webpack 究竟是如何运行的？

在日常开发过程中，我们是需要经常将心思放在如何使用 Webpack 优化开发体验以及输出的质量，却很少把目光放在 Webpack 的启动流程、loader 和 plugin 基本原理等等方面上。

那么，现在我们就来一起来简单缕一缕，究竟 Webpack 是怎么运行的？以及 loader 和 plugin 又是如何工作的？


## 现代构建工具对比

在探究之前，我个人觉得是很有必要说一说的，那就是现代构建工对比。为什么？

在前端快速发展的这几年，如何高效滴构建项目变得越来越重要，但是在百花争艳的构建工具里面，究竟是选用哪一款？那么现代构建工具有那么多，肯定得有它们适用场合，因此了解它们也是很有必要的。🤔

今天要讲的现代工具，主要是 **NPM Script、Grunt、Glup、Fis3、Webpack、Rollup**。下面我就来看看它们究竟是用来干啥的。

> NPM Script

如果你用过 package.json，那么你对它肯定不陌生。**NPM Script 是 NPM 内置的一个功能，允许在 package.json 文件里面使用 scripts 字段定义的任务**。举个🌰：

```javascript
{
  "scripts": {
    "start": "node main.js",
 		"prod": "node prod.js"
  }
}
```

在定义上述的任务后，我们就可以直接在项目根路径下执行如下命令：

```javascript
npm start
```

**执行 NPM 相应的命令后，会根据 package.json 执行命令对应的任务**。

毫无疑问，**NPM Script 的优点就是内置，无需安装其他依赖，缺点就是功能太简单，不能方便滴管理多个任务之间的依赖**。

> Grunt

Grunt 和 NPM Script 一样，都只是一个任务执行者。但相比之下，**Grunt 有着大量现成的插件封装了常见的任务，也能管理任务之间的依赖关系，并能自动化滴执行依赖的任务**，每个任务的具体代码和依赖关系写在 Gruntfile.js 里。

```js
module.exports = function(grunt) {

  // Project configuration.
  grunt.initConfig({
    pkg: grunt.file.readJSON('package.json'),
    uglify: {
      options: {
        banner: '/*! <%= pkg.name %> <%= grunt.template.today("yyyy-mm-dd") %> */\n'
      },
      build: {
        src: 'src/<%= pkg.name %>.js',
        dest: 'build/<%= pkg.name %>.min.js'
      }
    }
  });

  // Load the plugin that provides the "uglify" task.
  grunt.loadNpmTasks('grunt-contrib-uglify');

  // Default task(s).
  grunt.registerTask('default', ['uglify']);

};
```

在项目根目录执行命令 grunt，就会启动 uglify 功能对指定的 JavaScript 文件进行压缩，上述配置中还从源文件 package.json 中读取信息。

**Grunt 最大的优点在于能够通过大量现成插件进行功能的拓展，缺点就是集成度不高，几乎都是依赖第三方插件，无法做到开箱即用。**

Grunt 相当于是 NPM Script 的进化版，它的诞生就是弥补 NPM Script 的不足。

> Gulp

Glup 是一个基于流的自动化构建工具。除了可以管理和执行任务，还支持监听文件、读写文件。

```javascript
const { src, dest } = require('gulp');
const babel = require('gulp-babel');

exports.default = function() {
  return src('src/*.js')
    .pipe(babel())
    .pipe(dest('output/'));
}
```

上述栗子中，通过读取 src 目录下的所有 js 文件，通过 babel 插件转换后，然后写入到 output 目录下。

**Gulp 的优点就是引入了流的概念，可以更好支持读写文件以及监听文件变化，缺点和 Grunt 一样，集成度依然不高，都是需要第三方插件完成大部分功能**。

Glup 可以看成是 Grunt 的加强版，相比于 Grunt，Glup 增加了监听文件、读写文件、流式处理的功能。

> Fis3

Fis3 是一款来自百度的优秀国产构建工具。相比 Grunt 和 Glup，它集成了 Web 开发中的常用功能，主要包含如下：

- 读写文件。通过 fis.match 读文件
- 资源定位。解析文件之间的依赖关系
- 文件缓存。通过 useHash 配置输出文件时为文件 URL 加上 md5 戳，以优化浏览器的缓存
- 文件编译。通过 parser 配置文件解析器来完成文件转换，如 ES6 转换成 ES5。
- 压缩资源。通过 optimizer 配置代码压缩方法。
- 图片合并。通过 spriter 配置合并 CSS 里导入的图片到一个文件中，从而减少 HTTP 请求。

```javascript
fis.match('*.{js, css, png}', { // 为文件添加md5
  useHash: true
})
fis.match('*.ts', { // 转化ts文件
  parser: fis.plugin('typescript')
})
```

**Fis3 的优点是集成了各种 Web 开发所需的构建功能，配置简单并且开箱即用。缺点就是官方目前不再进行更新和维护，也不支持最新版的 Node.js**。

Fis3 相比 Grunt 和 Glup 构建工具来说，进一步加强了集成功能，如果将 Grunt、Glup 比作汽车的发动机话，则可以将 Fis3 比作一辆完整的汽车。

> Webpack

作为今天的主角——Webpack，可以说是目前集成功能最全、使用最广泛的构建工具。

Webpack 是一个 JavaScript 打包模块化的工具，**在 Webpack里一切文件皆模块，通过 Loader 转换文件，通过 Plugin 注入事件钩子，最后输出由多个模块组合成 Chunk 并转换成相应的文件进行输出**。

```javascript
module.exports = {
	entry: './main.js',
  output: {
    filename: '[name].js',
    path: path.resolve(__dirname, './dist')
  }
}
```

**Webpack 的优点在于专注处理模块化功能，能做到开箱即用、一步到位，能通过 Plugin 进行扩展，并且社区也活跃。缺点就是只能用于模块化开发的项目（其实这也不算是缺点，个人觉得）**。

相比之下，webpack 可以说是一个完整的集成度、完善的第三方插件管理和社区活跃的模块化构建工具。

> Rollup

Rollup 是一个和 Webpack 很类似但专注于 ES6 的模块打包工具。它的亮点在于能够针对 ES6 源码进行 Tree Shaking 以及对没用的代码进行 Scope Hoisting，从而优化输出质量。当然，Webpack 也在内部实现这些功能。

相比之下，Rollup 在用于打包 JavaScript 库时比 Webpack 更有优势，因为其打包出来的文件更小。但它的缺点就是还不够完善，在很多场景下还找不到相应的解决方案。



## Webpack构建流程

要了解 Webpack 是怎么进行构建的，首先得下列几个核心概念进行入手。

- Entry：Webpack 构建的入口，也就是输入配置。
- Output：Webpack 构建的输出配置。
- Module：模块配置，一切文件都是模块，一个模块对应一个文件。
- Chunk：代码块，一个代码块由一个或多个模块构成，用于代码合并和分割。
- Loader：模块转换器，用于将模块的原内容转换成新内容。
- Plugin：插件扩展功能器，用于监听Webpack在生命周期内广播事件时执行相应的回调逻辑。

那么，Webpack 的构建流程是如何的呢？

**其实 Webpack 的构建流程可分为三个阶段，分别是初始化阶段、编译阶段、输出阶段**。下面就简单总结一下。

1. 初始化阶段

   首先会初始化参数（shell脚本中参数和配置中参数合并），根据参数初始化 compiler 编译对象，接着就是加载配置项里的插件（创建相应插件实例，开始监听 Webpack 声明周期中的广播事件）。

2. 编译阶段

   将初始化阶段得到的 compiler 编译对象，执行 run 方法正式开始进行编译。

   首先会从配置项里的 Entry 入手，找到所有的入口模块。

   接着从入口模块开始递归寻找，将匹配到的模块使用相应的 Loader 进行转换，重复该过程直到所有的依赖模块寻找完为止。

   最后，编译结束后将得到所有转换好的模块以及各个模块之间的依赖关系。

3. 输出阶段

   根据编译阶段得到的各个模块间依赖关系，将一个个模块开始组合成一个 Chunk，并将 Chunk 转换文件，最终将文件输出到文件系统中。

在了解完 Webpack 的构建流程后，我相信你肯定和我有一个疑问，那就是**为什么不直接将转换好的模块直接转变成文件输出？而是将多个模块根据依赖关系组成 Chunk 并转变成文件再输出？**

**其实组成一个 Chunk 是有目的的，就是为了减少网络请求**。如果仅仅是把转换好的模块转变成文件来输出，是没问题的，但是在浏览器中，访问一个模块文件时，当发现它还有依赖的文件，那么就需要通过网络请求相应的依赖文件，这样就不得不耗费时间在网络请求上。另外浏览器环境不像 Nodejs 那样能够快速滴从本地加载一个模块文件，因此根据依赖关系组装成一个 Chunk 并转换成文件后，将大大减少对网络的依赖程度。



## 理解 Loader

Loader 就像一个翻译员，能将源文件经过转换后输出新的内容，并且一个文件还能经过链式的 Loader 处理。

在这里，我会先提一个问题，就是常问的，多个 Loader 的执行顺序是怎样的？为什么？

其实，**Loader 的执行顺序是从右到左的**。Webpack 是由于跟随函数式编程的缘故，才会采取了从右到左的顺序执行 Loader。

**Webpack 实现 Loader 执行选择函数式编程中的 compose 方式，而不是选择 pipe 方式**（pipe 方式实现的是从左到右，如果要实现从左到右也不难）。在函数式编程中，实现方式都是从右到左的，看个🌰：

```javascript
// 摘自网上例子
const compose = (...fns) => x => fns.reduceRight((v, f) => f(v), x);
const add1 = n => n + 1; //加1
const double = n => n * 2; // 乘2
const add1ThenDouble = compose(
  double,
  add1
);
add1ThenDouble(2); // 6
// ((2 + 1 = 3) * 2 = 6) 
```

一个 Loader 就是一个 Node.js 模块，这个模块需要导出一个函数。导出函数的工作就是获得处理前的原内容，对原内容执行处理后返回处理后的内容。

那么**如果要编写一个简单的 Loader ，需要怎样子操作呢？**

由于 Loader 是运行在 Node.js 中，所以我们可以调用任意的 Node.js 自带的 API，或者安装第三方模块进行调用，先看个最简单的编写栗子：

```javascript
const sass = require('node-sass')
module.exports = function(source) {
  return sass(source)
}
```

另外，Webpack 还提供了一些 API 供 Loader 调用，简单列举一下常用的 API。

- this.callback。用于 webpack 和 loader 间的通信，在 loader 使用时，告诉 webpack 返回结果。
- this.context。假如当前 loader 处理的文件是 \/src\/main\.js ，则 this.context 等于 \/src。
- this.resource。当前处理文件的完整请求路径。
- this.resourceQuery。当前处理文件的 queryString。
- this.traget。相当于 Webpack 配置中的 Target。
- this.emitFile。输出一个文件。
- this.clearDependcies。清除当前正在处理文件的所有依赖。
- this.addDependcies。当前处理文件添加其依赖的文件。

那么编写完 Loader 后，如何让 Webpack 加载本地 Loader？

1. NPM link

   NPM link 专门用于开发和调试本地的 NPM 模块的，能做到不发布模版的情况下，将本地的一个正在开发的模版的源码链接到项目 node_modules 目录下，让项目可以直接使用本地的 NPM 模块。

   要完成 NPM 的步骤，需按以下实现：

   - 确保在本地开发好 NPM 模块中 package.json 已经配置好。
   - 在本地的 NPM 模块目录下执行 npm link 命令，将本地模块注册到全局。
   - 在项目根目录下执行 npm link loader-name，将第二步注册到全局的本地 NPM 模块链接到项目的 node_modules 下，其中 loader-name 是指在第一步的 package.json 文件中配置的模块名称。

2. ResolveLoader

   为了让 Webpack 加载放在本地的 loader，可以直接配置 resolveLoader.modules。

   ```js
   module.exports = {
     resolveLoader: {
       modules: ['node_modules', './loaders/']
     }
   }
   ```

   上面配置中，当在 node_modules 中找不到相应的 loader 时，便会从 loaders 文件夹中去寻找。



## 理解 Plugin

在 Webpack 运行的生命周期中会广播许多事件，Plugin 可以监听这些事件，在合适的时机通过 Webpack 提供的 API 改变输出结果。

在开发 Plugin 时最常用的两个对象是 Compiler 对象和 Compilation 对象，它们是 Plugin 和 Webpack 之间的桥梁。

- Compiler 对象：包含了 Webpack 环境的所有配置信息（即 options、loaders、plugins 等信息），可以理解为 Webpack 实例。
- Compilation 对象：包含了当前 Webpack 的模块资源、编译生成资源、变化的文件等。每当监测到一个文件发生变化，便有一次新的 Compilation 对象被创建。

那么，**Compiler 对象和 Compilation对象之间的区别是什么？Compiler 对象代表了整个 Webpack 从启动到关闭的生命周期，而 Compilation 对象只代表一次新的编译**。

另外，**Webpack 通过 Tagable 来组织这条复杂的生产线，在 Webpack 的运行过程中会广播事件，Plugin 只需要监听它关心的事件，就能加入这条生产线中，并能改变生产线的运作**。

Webpack 的事件流机制采用的是观察者模式，和 Node.js 中的 EventEmitter 非常相似，并且 Compiler 对象和 Compilation 对象都继承自 Tapable，可在 Compiler 对象和 Compilation 对象上广播和监听事件。

```js
// 广播事件
compiler.apply('eventName', params)

// 监听事件
compiler.plugin('eventName', function(params) { 
  //... 
})
```

因此，在编写一个 Plugin 时，可如下操作：

```js
// TextPlugin.js
class TextPlugin {
  constructor(options) { // 在构造函数中获取用户为该插件传入的配置
    // ...
  }
  apply(compiler) { // Webpack会调用TextPlugin实例的apply方法为插件实例传入compiler对象
    compiler.plugin('eventName', function(params) {
      // ...
    })
  }
}
module.exports = TextPlugin // 导出plugin

// webpack.config.js
const TextPlugin = require('./TextPlugin')
module.exports = {
  plugins: [
    new TextPlugin()
  ]
}
```





















