---
title: webpack 打包优化
date: 2022-02-23 10:57:14
tags: webpack
---

#### webpack 打包优化

我们知道 webpack 打包优化很重要，不论是优化开发体验还是优化打包速度、体积都是很有益处的

##### 缩小文件搜索范围

###### 优化 loader 配置

`include/exclude` 将 `node_modules` 中的文件进行包括/排除

```bash
{
    rules: [{
        test: /\.js$/,
        use: {
            loader: 'babel-loader'
        },
        // exclude: /node_modules/,
        include: [path.resolve(__dirname, 'src')]
    }]
}
```

###### 优化 module.noParse 配置

如果一些第三方模块没有使用 `AMD/CommonJs` 规范，可以使用 `noParse` 来标记这个模块，这样 Webpack 在导入模块时，就不进行解析和转换，提升 `Webpack` 的构建速度

```bash
{
    module: {
        //noParse: /jquery|lodash|chartjs/,
        noParse: function(content){
            return /jquery|lodash|chartjs/.test(content)
        }
    }
}
```

对于 `jQuery`、`lodash`、`chartjs` 等一些库，庞大且没有采用模块化标准，因此我们可以选择不解析他们。

> 注意 ⚠️：被不解析的模块文件中不应该包含 require、import 等模块语句

###### 优化 resolve.alias 配置

alias 通过创建 import 或者 require 的别名，把原来导入模块的路径映射成一个新的导入路径；它和 resolve.modules 不同的的是，它的作用是用别名代替前面的路径，不是省略；这样的好处就是 webpack 直接会去对应别名的目录查找模块，减少了搜索时间。

```bash
{
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },
}
```

这样我们就能通过 import Buttom from '@/Button'来引入组件了；我们不光可以给自己写的模块设置别名，还可以给第三方模块设置别名：

```bash
{
  resolve: {
    alias: {
      'vue$': isDev ? 'vue/dist/vue.runtime.js' : 'vue/dist/vue.runtime.min.js',
    },
  },
}
```

我们在 `import Vue from 'vue'` 时，`webpack` 就会帮我们去 `vue` 依赖包的 `dist` 文件下面引入对应的文件，减少了搜索 `package.json` 的时间

###### 优化 resolve.mainFields 配置

`mainFields` 用来告诉 webpack 使用第三方模块中的哪个字段来导入模块；第三方模块中都会有一个 `package.json` 文件用来描述这个模块的一些属性，比如模块名(`name`)、版本号(`version`)、作者(`auth`)等等；其中最重要的就是有多个特殊的字段用来告诉 `webpack` 导入文件的位置，有多个字段的原因是因为有些模块可以同时用于多个环境，而每个环境可以使用不同的文件。

`mainFields` 的默认值和当前 `webpack` 配置的 `target` 属性有关：

- 如果 `target` 为 `webworker` 或 `web（默认）`，`mainFields` 默认值为 `["browser", "module", "main"]`
- 如果 `target` 为其他（包括 `node`），`mainFields` 默认值为 `["module", "main"]`

这就是说当我们 `require('vue')` 的时候，`webpack` 先去 `vue` 下面搜索 `browser` 字段，没有找到再去搜索 `module` 字段，最后搜索 `main` 字段。
　　为了减少搜索的步骤，在明确第三方模块入口文件描述字段时，我们可以将这个字段设置尽量少；一般第三方模块都采用 `main` 字段，因此我们可以这样配置：

```bash
{
    resolve: {
        mainFields: ["main"],
    }
}
```

###### 优化 resolve.extensions 配置

`extensions` 字段用来在导入模块时，自动带入后缀尝试去匹配对应的文件，它的默认值是：

```bash
{
    resolve: {
        extensions: ['.js', '.json']
    }
}
```

也就是说我们在 `require('./utils')` 时，`Webpack` 先匹配 `utils.js`，匹配不到再去匹配 `utils.json`，如果还找不到就报错。
因此 `extensions` 数组越长，或者正确后缀的文件越靠后，匹配的次数越多也就越耗时，因此我们可以从以下几点来优化：

- extensions 数组尽量少，项目中不存在的文件后缀不要列进去
- 出现频率比较高的文件后缀优先放到最前面
- 在代码中导入文件的时候，要尽量把后缀名带上，避免查找

##### 减少打包文件

###### 提取公共代码

Webpack4 引入了 `SplitChunksPlugin` 插件进行公共模块的抽取；由于 `webpack4` 开箱即用的特性，它不用单独安装，通过 `optimization.splitChunks` 进行配置即可，官方给的默认配置参数如下：

```bash
module.exports = {
  optimization: {
    splitChunks: {
      // 代码分割时默认对异步代码生效，all：所有代码有效，inital：同步代码有效
      chunks: 'async',
      // 代码分割最小的模块大小，引入的模块大于 20000B 才做代码分割
      minSize: 20000,
      // 代码分割最大的模块大小，大于这个值要进行代码分割，一般使用默认值
      maxSize: 0,
      // 引入的次数大于等于1时才进行代码分割
      minChunks: 1,
      // 最大的异步请求数量,也就是同时加载的模块最大模块数量
      maxAsyncRequests: 30,
      // 入口文件做代码分割最多分成 30 个 js 文件
      maxInitialRequests: 30,
      // 文件生成时的连接符
      automaticNameDelimiter: '~',
      enforceSizeThreshold: 5000,
      cacheGroups: {
        vendors: {
          // 位于node_modules中的模块做代码分割
          test: /[\\/]node_modules[\\/]/,
          // 根据优先级决定打包到哪个组里，例如一个 node_modules 中的模块进行代码
          priority: -10
        },
        // 既满足 vendors，又满足 default，那么根据优先级会打包到 vendors 组中。
        default: {
          // 没有 test 表明所有的模块都能进入 default 组，但是注意它的优先级较低。
          // 根据优先级决定打包到哪个组里,打包到优先级高的组里。
          priority: -20,
          // 如果一个模块已经被打包过了,那么再打包时就忽略这个上模块
          reuseExistingChunk: true
        }
      }
    }
  }
};
```

有时候项目依赖模块比较多，`vendors.js` 文件会特别大，我们还可以对它进一步拆分，按照模块划分：

```bash

{
  //省略其他配置
  cacheGroups: {
    //涉及vue的模块
    vue: {
      test: /[\\/]node_modules[\\/](vue|vuex|vue-router)/,
      priority: 10,
      name: 'vue'
    },
    //其他模块
    vendors: {
      test: /[\\/]node_modules[\\/]/,
      priority: 9,
      name: 'vendors'
    },
    common: {
      test: /[\\/]src[\\/]/,
      priority: 5,
      name: 'common'
    }
  }
}
```

###### 动态链接 DllPlugin

`DLL` 即动态链接库（`Dynamic-Link Library`）的缩写，熟悉 `Windows` 系统的童鞋在电脑中也经常能看到后缀是 `dll` 的文件，偶尔电脑弹框警告也是因为电脑中缺失了某些 `dll` 文件；`DLL` 最初用于节约应用程序所需的磁盘和内存空间，当多个程序使用同一个函数库时，`DLL` 可以减少在磁盘和内存中加载代码的重复量，有助于代码的复用。
　　在 `Webpack` 中也引入了 `DLL` 的思想，把我们用到的模块抽离出来，打包到单独的动态链接库中去，一个动态链接库中可以有多个模块；当我们在多个页面中用到某一个模块时，不再重复打包，而是直接去引入动态链接库中的模块。
　　 `Webpack` 中集成了对动态链接库的支持，主要用到的两个插件：

- `DllPlugin`：创建动态链接库文件
- `DllReferencePlugin`：在主配置中引入打包好的动态链接库文件

我们首先使用 `DllPlugin` 来创建动态链接库文件，在项目下新建 `webpack.dll.js` 文件：

```bash
const path = require("path");
const webpack = require("webpack");

module.exports = {
  mode: "production",
  entry: {
    vue: ["vue", "vuex", "vue-router"],
    vendor: ["dayjs", "axios", "mint-ui"],
  },
  output: {
    path: path.resolve(__dirname, "public/vendor"),
    // 指定文件名
    filename: "[name].dll.js",
    //暴露全局变量的名称
    library: "[name]_dll_lib",
  },
  plugins: [
    new webpack.DllPlugin({
      path: path.join(__dirname, "public", "vendor", "[name].manifest.json"),
      name: "[name]_dll_lib",
    }),
  ],
};
```

这里 `entry` 设置了多个入口，每个入口也有多个模块文件；然后在 `package.json` 添加打包命令

```bash
{
    "scripts":{
        "build:dll": "webpack --config=webpack.dll.js"
    }
}
```

执行 `npm run build:dll` 后，我们在` /public/vendor` 目录下得到了我们打包后的动态链接库的文件：

```bash
├── vendor.dll.js
├── vendor.manifest.json
├── vue.dll.js
└── vue.manifest.json
```

生成出来的打包文件正好是以两个入口名来命名的，以 `vue` 为例，看一下 `vue.dll.js` 的内容：

```bash
var vue_dll_lib =
/******/ (function(modules) {
    // 省略webpackBootstrap代码
/******/ })
/******/ ({

/***/ "./node_modules/vue-router/dist/vue-router.esm.js":
/***/ (function(module, exports, __webpack_require__) {
    // 省略vue-router模块代码
/***/ }),

/***/ "./node_modules/vue/dist/vue.runtime.esm.js":
/***/ (function(module, exports, __webpack_require__) {
    // 省略vue模块代码
/***/ }),

/***/ "./node_modules/vuex/dist/vuex.esm.js":
/***/ (function(module, exports, __webpack_require__) {
    // 省略vuex模块代码
/***/ }),

/******/ });
```

可以看出，动态链接库中包含了引入模块的所有代码，这些代码存在一个对象中，通过模块路径作为键名来进行引用；并且通过 `vue_dll_lib` 暴露到全局；`vue.manifest.json` 则是用来描述动态链接库文件中包含了哪些模块：

```bash
{
    "name": "vue_dll_lib",
    "content": {
        "./node_modules/vue-router/dist/vue-router.esm.js": {
            "id": "./node_modules/vue-router/dist/vue-router.esm.js",
            "buildMeta": {}
        },
        "./node_modules/vue/dist/vue.runtime.esm.js": {
            "id": "./node_modules/vue/dist/vue.runtime.esm.js",
            "buildMeta": {}
        },
        "./node_modules/vuex/dist/vuex.esm.js": {
            "id": "./node_modules/vuex/dist/vuex.esm.js",
            "buildMeta": {}
        },
    }
}
```

`manifest.json` 描述了对应 `js` 文件包含哪些模块，以及对应模块的键名（id），这样我们在模板页面中就可以将动态链接库作为外链引入，当 `Webpack` 解析到对应模块时就通过全局变量来获取模块：

```bash
<!-- public/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <div id="app"></div>
    <!-- 引入动态链接库 -->
    <script src="./vendor/vendor.dll.js"></script>
    <script src="./vendor/vue.dll.js"></script>
</body>
</html>
```

最后我们在打包时，通过 `DllReferencePlugin` 将动态链接库引入到主配置中：

```bash
//webpack.config.js
{
    plugins: [
        new webpack.DllReferencePlugin({
            context: path.join(__dirname),
            manifest: require('./public/vendor/vendor.manifest.json')
        }),
        new webpack.DllReferencePlugin({
            context: path.join(__dirname),
            manifest: require('./public/vendor/vue.manifest.json')
        }),
    ]
}
```

> 注意 ⚠️：动态链接库打包到 `/public/vendor` 目录下，还需要通过 `CopyWebpackPlugin` 插件将它拷贝到生成后的目录中，否则会出现引用失败的报错；打包动态链接库文件只需要执行一次，除非以后模块升级或者引入新的模块。

###### externals

我们在项目打包时，有一些第三方的库会从 `CDN` 引入（比如 `jQuery` 等），如果在 `bundle` 中再次打包项目就过于臃肿，我们就可以通过配置 `externals` 将这些库在打包的时候排除在外。

```bash
{
  externals: {
    'jquery': "jQuery",
    'react': 'React',
    'react-dom': 'ReactDOM',
    'vue': 'Vue'
  }
}
```

这样就表示当我们遇到 `require('jquery')` 时，从全局变量去引用 `jQuery`，其他几个包也同理；这样打包时就把 `jquery`、`react`、`vue` 和 `react-dom` 从 `bundle` 中剔除了。

###### Tree Shaking

`Tree Shaking` 最早由 `rollup` 实现，后来 `webpack2` 也实现了这项功能；`Tree Shaking` 的字面意思是摇树，一棵树上有一些树叶虽然还挂着，但是它可能已经死掉了，通过摇树方式把这些死掉的树叶去除。

为了让 `Tree Shaking` 生效，我们需要使用 `ES6` 模块化的语法，因为 `ES6` 模块语法是静态化加载模块，它有以下特点：

- 静态加载模块，效率比 `CommonJS` 模块的加载方式高
- `ES6` 模块是编译时加载，使得静态分析成为可能进一步拓宽 `JS` 的语法

如果是 `require`，在运行时确定模块，那么将无法去分析模块是否可用，只有在编译时分析，才不会影响运行时的状态。

使用 `ES6` 模块后还有一个问题，因为我们的代码一般都采用 `babel` 进行编译，而 `babel` 的 `preset` 默认会将任何模块类型编译成 `Commonjs`，因此我们还需要修改 `.babelrc` 配置文件：

```bash
{
  "presets": [
    [
      "@babel/preset-env",
      {
        // 添加modules：false
        "modules": false
      }
    ]
  ]
}
```

配置好 babel 后我们需要让 webpack 先将“死代码”标识出来：

```bash
{
  // 其他配置
  optimization: {
    usedExports: true,
    sideEffects: true,
  }
}
```

运行打包命令后，当我们打开输出的 bundle 文件时，我们发现虽然一些“死代码”还存在里面，但是加上了一个 unused harmony export 的标识

```bash
/* unused harmony export isFunction */
/* unused harmony export isDate */
var toString = Object.prototype.toString;
function isFunction(val) {
  return toString.call(val) === '[object Function]';
}
function isDate(val) {
  return toString.call(val) === '[object Date]';
}
```

虽然 `webpack` 给我们指出了哪些函数用不上，但是还需要我们通过插件来剔除；由于 `uglifyjs-webpack-plugin` 不支持 `ES6` 语法，这里我们使用 `terser-webpack-plugin` 的插件来代替它：

```bash
const TerserJSPlugin = require("terser-webpack-plugin");
module.exports = {
  optimization: {
    usedExports: true,
    sideEffects: true,
    minimize: true,
    minimizer: [
      new TerserJSPlugin({
        cache: true,
        parallel: true,
        sourceMap: false,
      }),
    ],
  }
}
```

这样我们发现打包出来的文件就没有多余的代码了。

> `Tree Shaking` 在生产环境（`production`）是默认开启的
> 对于我们常用的一些第三方模块，我们也可以实现 `Tree Shaking`；以 `lodash` 为例，它整个包有非常多的函数，但并不是所有的函数都是我们所用到的，因此我们也需要对它没有用到的代码进行剔除。
>
> ```bash
> //index.js
> import { chunk } from 'lodash'
> console.log(chunk([1,2,3,4], 2))
> ```
>
> 打包出来发现包的大小还是能达到 `70+kb`，如果只引用了 `chunk` 不应该有这么大；我们打开 `/node_modules/lodash/index.js` 发现他还是使用了 `require` 的模式导入导出模块，因此导致 `Tree Shaking` 失败；我们先安装使用 `ES6` 模块版本的 `lodash：npm i -S lodash-es`，然后修改引入包：
>
> ```bash
> //index.js
> import { chunk } from 'lodash-es'
> console.log(chunk([1,2,3,4], 2))
> ```

##### 缓存

我们知道 webpack 会对不同的文件调用不同的 loader 进行解析处理，解析的过程也是最耗性能的过程；我们每次改代码也只是修改项目中的少数文件，项目中的大部分文件改动的次数不是那么频繁；那么如果我们将解析文件的结果缓存下来，下次发现同样的文件只需要读取缓存就能极大的提升解析的性能。

###### cache-loader

`cache-loader` 可以将一些对性能消耗比较大的 `loader` 生产的结果缓存在磁盘中，等下次再次打包时如果是相同的代码就可以直接读取缓存，减少性能消耗。

> 注意 ⚠️：保存和读取缓存也会产生额外的性能开销，因此 `cache-loader` 适合用于对性能消耗较大的 `loader`，否则反而会增加性能消耗

`cache-loader` 的使用也非常简单，安装后在所需要缓存的 loader 前面添加即可（因为 loader 加载的顺序是反向的），比如我们需要给 babel-loader 添加缓：

```bash
{
  //省略其他代码
  rules: [
    {
      test: /\.js/,
      use: [
        {
          loader: 'cache-loader'
        },
        {
          loader: "babel-loader",
        },
      ],
    },
  ],
}
```

然而我们发现第一次打包的速度并没有发生明显变化，甚至可能还比原来打包的更慢了；同时还多了 `/node_modules/.cache/cache-loader/` 这个目录，看名字就是一个缓存文件；但是从第二次打包开始，直接减少了 `75%` 的耗时

###### HardSourceWebpackPlugin

`HardSourceWebpackPlugin` 也可以为模块提供缓存功能，同时也是将文件缓存在磁盘中

首先通过 `npm i -D hard-source-webpack-plugin` 来安装插件，并且在配置中添加插件：

```bash
var HardSourceWebpackPlugin =
    require('hard-source-webpack-plugin');
module.exports = {
  plugins: [
    new HardSourceWebpackPlugin()
  ]
}
```

一般 `HardSourceWebpackPlugin` 默认缓存是在 `/node_modules/.cache/hard-source/[hash]` 目录下，我们可以设置它的缓存目录和何时创建新的缓存哈希值。

```bash
module.exports = {
  plugins: [
    new HardSourceWebpackPlugin({
      //设置缓存目录的路径
      //相对路径或者绝对路径
      cacheDirectory: 'node_modules/.cache/hard-source/[confighash]',
      //构建不同的缓存目录名称
      //也就是cacheDirectory中的[confighash]值
      configHash: function(webpackConfig) {
        return require('node-object-hash')({sort: false}).hash(webpackConfig);
      },
      //环境hash
      //当loader、plugin或者其他npm依赖改变时进行替换缓存
      environmentHash: {
        root: process.cwd(),
        directories: [],
        files: ['package-lock.json', 'yarn.lock'],
      },
      //自动清除缓存
      cachePrune: {
        //缓存最长时间（默认2天）
        maxAge: 2 * 24 * 60 * 60 * 1000,
        //所有的缓存大小超过size值将会被清除
        //默认50MB
        sizeThreshold: 50 * 1024 * 1024
      },
    })
  ]
}
```

#### 参考

[Webpack 配置全解析（优化篇）](https://juejin.cn/post/6858905382861946894)
[带你深度解锁 Webpack 系列(优化篇)](https://zhuanlan.zhihu.com/p/121820574)
[Webpack 官方文档 - Tree Shaking](https://webpack.docschina.org/guides/tree-shaking/)
