<h1 align="center"> webpack 相关总结</h1>

在了解原理之前我们先认识基本的构建概念和构建流程

### 📖 基本概念 

- entry 构建的入口文件
- output 构建完成的文件输出的位置
- module 模块，通过模块内的匹配规则和loader来处理文件
- loader 处理文件的文件转化器（ES6 -> ES5 等）
- plugin 插件，通过监听执行流程上的钩子函数可以做扩展功能
- chunk 多个文件组成的代码快

```js
// 简单的例子
const webpackConfig = {
  mode: _mode == "test" ? "development" : _mode,
  entry: {
    main: "./src/main.js"
  },
  output: {
    path: join(__dirname, "./dist/assets")
  },
  module: {
    // ...
  },
  optimization: {
    // ...
  },
  plugins: [..._plugins, new VueLoaderPlugin()],
  externals: {
    ..._externals
  },
  resolve: {
   // ...
  }
};
```

### 📖 HMR 原理 

<img src="/assets/webpack-hmr.jpg">

- webpack 对文件系统进行 watch 打包到内存中：（webpack-dev-middleware 调用 webpack 的 api 对文件系统 watch，当文件发生变化的时候重新进行打包，然后保存到内存中），dev 环境不生成不生成文件的原因就在于访问内存中的代码比访问文件系统中的文件更快，而且也减少了代码写入文件的开销，这一切都归功于memory-fs，memory-fs 是 webpack-dev-middleware 的一个依赖库
- devServer 通知浏览器端文件发生改变 (sockjs 是桥梁)： webpack-dev-server 调用 webpack api 监听 compile的 done 事件，当compile 完成后，webpack-dev-server通过 _sendStatus 方法将编译打包后的新模块 hash 值发送到浏览器端。
- webpack-dev-server/client 接收到服务端消息做出响应：当接收到 type 为 hash 消息后会将 hash 值暂存起来，当接收到 type 为 ok 的消息后对应用执行 reload 操作，在 reload 操作中，会根据 hot 配置决定是刷新浏览器还是对代码进行热更新（HMR）
- webpack 接收到最新 hash 值验证并请求模块代码： 首先是 webpack/hot/dev-server 监听第三步 webpack-dev-server/client 发送的 webpackHotUpdate 消息，调用 webpack/lib/HotModuleReplacement.runtime 中的 check 方法，检测是否有新的更新，在 check 过程中会利用 webpack/lib/JsonpMainTemplate.runtime 中的两个方法 hotDownloadUpdateChunk 和 hotDownloadManifest ， 第二个方法是调用 AJAX 向服务端请求是否有更新的文件，如果有将发更新的文件列表返回浏览器端，而第一个方法是通过 jsonp 请求最新的模块代码，然后将代码返回给 HMR runtime，HMR runtime 会根据返回的新模块代码做进一步处理，可能是刷新页面，也可能是对模块进行热更新。
- HotModuleReplacement.runtime 对模块进行热更新： 
  - 第一个阶段是找出 outdatedModules 和 outdatedDependencies
  - 第二个阶段从缓存中删除过期的模块和依赖
  - 第三个阶段是将新的模块添加到 modules 中，当下次调用 __webpack_require__ 方法的时候，就是获取到了新的模块代码了。

### 📖 运行流程 

webpack 就像一条生产线，要经过一系列处理流程后才能将源文件转换成输出结果。 这条生产线上的每个处理流程的职责都是单一的，多个流程之间有存在依赖关系，只有完成当前处理后才能交给下一个流程去处理。 插件就像是一个插入到生产线中的一个功能，在特定的时机对生产线上的资源做处理。

webpack 通过 Tapable 来组织这条复杂的生产线。 webpack 在运行过程中会广播事件，插件只需要监听它所关心的事件，就能加入到这条生产线中，去改变生产线的运作。 webpack 的事件流机制保证了插件的有序性，使得整个系统扩展性很好。  --吴浩麟《深入浅出webpack》

上面这段话把 webapck 做的事情解释的很清楚，实际上就是在构建的时候通过一系列的事件处理和回调来处理不同的功能。那么这种事件流是通过 Tapable 来管理的，关于 <a href="https://github.com/Pein892/learn-webpack-tapable">Tapable</a>请移步大佬的源码分析。

但是在 webpack4 中重写了事件流机制 <a href="https://webpack.js.org/api/compiler-hooks/"> webpack Hook</a>，由于源码的复杂性。在后面会通过思维导图的方式整理方便记忆。那么在<a href="https://github.com/webpack/changelog-v5/blob/master/README.md">webpack5</a> 中其实也增加了一些功能。在这里就不展开讨论了。


下面我们通过流程图来对 webpack 的执行流程作出分析

<img src="/assets/webpack.png"></img>


- 首先读取 webpack.config.js 文件，初始化本次构建的配置参数
- 初始化参数阶段开始读取配置 Entries ，遍历所有入口文件。
- 依次进入每个入口文件使用配置好的loader 对文件内容进行解析编译，生成<a hre="https://juejin.im/post/5bff941e5188254e3b31b424">AST（静态语法树）</a>
- 最后输出代码 chunk、打包好的文件等


上面的过程只是粗略的介绍了执行流程，那么接下来还是通过一张图来看一下 webpack 内部到底是如何工作的

<img src="/assets/webpack-render.jpg"></img>

通过上面的图已经很清楚的看到在 webpack 的每个阶段要做的事情熟悉整个流程有助于帮助我们理解日常的开发。下面我们就日常开发列出一些常用 loader 和 plugin

### 📖webpack 常用插件

我们通过配置的伪代码来介绍我们常用的功能几插件

### 🎄 mode

```js
const webpackConfig  = {
  mode: _mode == "test" ? "development" : 'production',
}
```

通过 mode 可以控制我们是开发还是线上环境通过控制环境来区分执行的文件如果 production 除了我们配置的以外 webpack 会默认开启以下插件

- FlagDependencyUsagePlugin（编译时标记依赖）
- FlagIncludedChunksPlugin（标记chunks,防止chunks多次加载）
- ModuleConcatenationPlugin（scope hoisting，预编译功能，提升代码在浏览器中的执行速度）
- NoEmitOnErrorsPlugin（在编译时候遇到错误可以跳过输出阶段，确保输出资源不会包含错误）
- OccurrenceOrderPlugin（给生成的 chunkid 排序）
- SideEffectsFlagPlugin（module.rules 的SideEffects 标志）
- uglifyjs-webpack-plugin（删除未引用代码并压缩 js） 


### 🎄 entry

单页面入口配置

```js
const webpackConfig  = {
  mode: _mode == "test" ? "development" : 'production',
  entry: {
    main: "./src/main.js"
  }
}
```

多页面入口配置

```js
const webpackConfig  = {
  mode: _mode == "test" ? "development" : 'production',
  entry: {
    pageOne: './src/pageOne/index.js',
    pageTwo: './src/pageTwo/index.js',
    pageThree: './src/pageThree/index.js'
  }
}
```

### 🎄 output

```js
const webpackConfig  = {
  //...
  output: {
    path: join(__dirname, "./dist/assets")
  }
}
```

也可以对输出文件进行 hash 和 publicPath 设置，publicPath 可以设置 cdn 地址这样可以优化加载文件时间

```js
const webpackConfig  = {
  // ...
  output: {
    filename: "scripts/[name].[contenthash:5].bundule.js",
    publicPath: "/assets/"
  }
}
```

### 🎄 module

```js
// loaders
const webpackConfig  = {
  // ...
  module: {
    // 创建模块时，匹配请求的规则数组
    // 通过这些规则可以对模块(module)应用 loader 和或者修改解析器(parse)
    rules: [
      {
        test: /\.js$/,  // 匹配规则
        exclude: /(node_modules|bower_components)/,   // 排除文件
        use: {
          loader: "babel-loader"  // 使用的loader
        }
      },
      {
        test: /\.vue$/,
        loader: "vue-loader"
      },
      {
        test: /\.(png|jpg|gif)$/i,
        use: [
          {
            loader: "url-loader",
            options: {
              // 小于 10kB(10240字节）的内联文件
              limit: 5 * 1024,
              name: _modeflag
                ? "images/[name].[contenthash:5].[ext]"
                : "images/[name].[ext]"
            }
          }
        ]
      }
    ]
  },
}
```
更多配置参数请参考<a href="https://webpack.docschina.org/configuration/module/#module-rules">webpack 官网</a>

### 🎄 plugin

```js
// plugins
const webpackConfig  = {
  // ...
  plugins: [
    //分析当前包的大小
    new BundleAnalyzerPlugin(),
    // 创建html入口文件
    new HtmlwebpackPlugin({
      filename: "index.html",
      template: "./src/index-dev.html"
    }),
    // webpack构建错误和警告通知
    new webpackBuildNotifierPlugin({
      title: "前端项目",
      logo: resolve("./favicon.png"),
      suppressSuccess: true
    }),
    // 友好提示插件
    new FriendlyErrorsPlugin({
      compilationSuccessInfo: {
        messages: ["You application is running here http://localhost:8080"],
        notes: ["请配置代理之后运行本服务"]
      },
      //提示好的工具 进行更详细的展现
      onErrors: function(severity, errors) {},
      clearConsole: true
    })
  ]
}
```
上面只是列举了几个开发环境中常用的 plugin，其实还有很多例如线上环境可以配置 OptimizeCssAssetsPlugin 、MiniCssExtractPlugin可以进行css 优化，ProgressBarPlugin 进度条等在这里就不列举了。

### 🎄 optimization

```js
// 优化配置
const webpackConfig = {
  // ...
  optimization: {
    runtimeChunk: {
      name: "runtime"
    },
    splitChunks: {
      chunks: "async",
      minSize: 30000,
      minChunks: 1,
      maxAsyncRequests: 5,
      maxInitialRequests: 3,
      name: false,
      cacheGroups: {
        commons: {
          chunks: "initial",
          minChunks: 2,
          maxInitialRequests: 5,
          minSize: 0,
          name: "commons"
        }
      }
    }
  }
}
```

关于更多参数及更多 plugins 请参考<a href="https://webpack.docschina.org/plugins/split-chunks-plugin/">webapck 官网</a>

### 🎄 externals

```js
// 外部扩展
const webpackConfig = {
  // ...
  externals: {
    vue: "Vue",
    "vue-router": "VueRouter",
    vuex: "Vuex",
    axios: "axios"
  }
}
```

### 🎄 resolve
```js
// 别名设置
const webpackConfig  = {
  // ...
  resolve: {
    alias: {
      "@": resolve("src")
    },
    extensions: [".js", ".vue"]
  }
}
```

### 🎄 devServer

```js
// 开发环境启动服务配置
const webpackConfig = {
  devServer: {
    quiet: true,
    open:true,
    historyApiFallback: true,
    proxy: {
      "/api": "http://localhost:3000"
    },
    hot: true,
    contentBase: join(__dirname, "../dist/assets"),
  }
}
```

以上就是在 webpack 开发中常用的配置，更多配置及优化请参考官网。

### 📖 webpack优化

<img src="/assets/webpack.jpg">

## 参考文章
<a href="https://github.com/gwuhaolin/dive-into-webpack/">深入浅出 webpack </a>