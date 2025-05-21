---
title: webpack 优化 
cover: /img/hand-z.webp
---

### webpack 构建速度优化

- 开启多线程构建:happypack(没有维护)
- thread-loader 开启线程池

   ```js
    module.exports = {
    module: {
        rules: [
        {
            test: /\.js$/,
            include: path.resolve('src'),
            use: [
            'thread-loader',
            // your expensive loader (e.g babel-loader)
            ],
        },
        ],
    },
    };
   ```
- terser-webpack-plugin 开启多线程

   ```js
    module.exports = {
    optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
            parallel: true,
        }),
      ],
    },
    };
   ```

- 缩小打包作用域:
   1.include|exclude 确定需要解析编译的模块或排除不需要解析编译的模块

   ```js
    {
  "loader": "babel-loader",
  "options": {
        "exclude": [
        // \\ for Windows, \/ for Mac OS and Linux
        /node_modules[\\\/]core-js/,
        /node_modules[\\\/]webpack[\\\/]buildin/,
        ],
        "presets": [
        "@babel/preset-env"
        ]
      }
    }
   ```
   2. resolve.modules 告诉webpack在解析模块时应该搜索哪些目录

   ```js
   module.exports = {
    //...
    resolve: {
        modules: ['node_modules'],
    },
   };
   ```
   3. resolve.extensions 减少不必要的后缀的解析

   ```js
   module.exports = {
    //...
    resolve: {
        extensions: ['.js', '.json', '.wasm'],
    },
    };
   ```
   4. ignore-loader 例如ssr时打包的服务端的模块就可以使用该loader忽略对css的处理

   ```js
    module.exports = {
    // other configurations
    module: {
        loaders: [
        { test: /\.css$/, loader: 'ignore-loader' }
        ]
    }
    };
   ```
- 利用缓存提升二次编译速度
   1. webpack5 cache

   ```js
   module.exports = {
    cache: {
        type: 'filesystem',
        allowCollectingMemory: true,
    },
    };
   ```
   2. webpack4 cache

   2.1 babel-loader 开启缓存

   ```js
    rules: [
    {
        test: /\.jsx?$/,
        exclude: /node_modules/,
        loader: "babel-loader",
        query: {
            optional: "runtime",
            cacheDirectory: true
        }
    }
    ]   
   ```
   2.2 cache-loader 开启缓存

   ```js
    module.exports = {
    module: {
        rules: [
        {
            test: /\.ext$/,
            use: ['cache-loader', ...loaders],
            include: path.resolve('src'),
        },
        ],
    },
    };
   ```
   2.3 hard-source-webpack-plugin 开启缓存

   ```js
    var HardSourceWebpackPlugin = require('hard-source-webpack-plugin');
    module.exports = {
        context: // ...
        entry: // ...
        output: // ...
        plugins: [
            new HardSourceWebpackPlugin()
        ]
    }
   ```
- DllPlugin和DllReferencePlugin提供了拆分包的方法，可以极大地提高构建时的性能。
  DllPlugin 这个插件会生成一个名为 manifest.json 的文件，这个文件是用来让 DLLReferencePlugin 映射到相关的依赖上去的。
  通过引用 dll 的 manifest 文件来把依赖的名称映射到模块的 id 上，之后再在需要的时候通过内置的 __webpack_require__ 函数来 require 它们。 

  webacpk --config webpack_dll.config.js 会生成entry中对用的js 和 mainifest.json js需要在index.html中引用
  ```js
    // webpack_dll.config.js
    const path = require('path'); 
    const DllPlugin = require('webpack/lib/DllPlugin');
    module.exports = {
      // JS 执行入口文件
      mode:'production',
      entry: {
        // 把 React 相关模块的放到一个单独的动态链接库
        react: ['react', 'react-dom'],
        // 把项目需要所有的 polyfill 放到一个单独的动态链接库
        // polyfill: ['core-js/fn/object/assign', 'core-js/fn/promise', 'whatwg-fetch'],
      },
      output: {
        // 输出的动态链接库的文件名称，[name] 代表当前动态链接库的名称，
        // 也就是 entry 中配置的 react 和 polyfill
        filename: '[name].dll.js',
        // 输出的文件都放到 dist 目录下
        path: path.resolve(__dirname, '../dist'),
        // 存放动态链接库的全局变量名称，例如对应 react 来说就是 _dll_react
        // 之所以在前面加上 _dll_ 是为了防止全局变量冲突
        library: '_dll_[name]',
      },
      plugins: [
        // 接入 DllPlugin
        new DllPlugin({
          // 动态链接库的全局变量名称，需要和 output.library 中保持一致
          // 该字段的值也就是输出的 manifest.json 文件 中 name 字段的值
          // 例如 react.manifest.json 中就有 "name": "_dll_react"
          name: '_dll_[name]',
          // 描述动态链接库的 manifest.json 文件输出时的文件名称
          path: path.join(__dirname, '../dist', '[name].manifest.json'),
        }),
      ],
    };
  ```

- module.noParse 不需要解析的独立的模块或库如jq,lodash

  ```js
  module.exports = {
    //...
    module: {
        noParse: /jquery|lodash/,
    },
  }
  ```


### webpack 打包体积优化

- 代码压缩

  1. terser-webpack-plugin 开启parallel:true 多进程并行压缩
  2. mini-css-extract-plugin 将css提取生成css文件 通过optimize-css-assets-webpack-plugin插件(webapck5使用 css-minimizer-webpack-plugin插件) 使用cssnano压缩css 

- 提取公共资源
  1. externals 将可以用cdn引入的资源 不打包直接通过cdn引入 如antd momentjs vue等

  ```js
    <script
    src="https://code.jquery.com/jquery-3.1.0.js"
    integrity="sha256-slogkvB1K3VOkzAI8QITxV3VzpOnkeNVsKvtkYLMjfk="
    crossorigin="anonymous">
    </script>

    externals: {
        jquery: 'jQuery'
    }
  ```
  2. SplitChunksPlugin 对公共的chunk进行提取打包

  ```js
    module.exports = {
    //...
    optimization: {
        splitChunks:{
            chunks:'all',
            maxInitialRequests:3,
            maxAsyncRequests:5,
            minChunks:3,
            minSize:{
                javascript:30 * 1024,
                style:30 * 1024
            },
            maxSize:{
                javascript:110 * 1024,
                style:110 * 1024
            },
            cacheGroups:{
                common:{
                    chunks:'all',
                    name:'common',
                    priority:-20,
                    enforce:true,
                    reuseExistingChunk:true
                }
            }
        }
    };
  ```


- tree shaking
  
  1. unCss 和 mini-css-extract-plugin配合使用 移除无用的css
  2. 尽量使用ES6的export导出语法对模块进行导出，提高treeshaking的效率 webpack5使用prepack进行计算清除
  3. 禁用babel-loader的模块依赖解析，否则webpack接受到的就都是转换过的CommonJS形式的模块，无法进行tree-shaking
  4. 开sideeffect:false告诉webpack无论导出的包里面有没有副作用只要没用到都tree shking掉

- 图片压缩 
  1. 使用基于Node图片压缩的库 imgaemin 配置image-webpack-loader  

- scope hoisting
  1. 构建后的代码会存在大量闭包，造成体积增大，运行代码时创建的函数作用域变多，内存开销大。
  scope hoisting 将所有模块的代码按照引用顺序放在一个函数作用域里，然后适当的重命名一些变量防止变量名冲突

  2. scope hoisting 必须是ES6的语法，因为有很多第三方库仍采用CommonJS语法，为了充分发挥 scope hoisting
  的作用，需要配置mainFields对第三方模块优先采用jsnext:main中指向的ES6模块化语法


- webpack5 的新特性

1. 支持实验性的toplevelawit
2. 优化异步chunk名称冲突
3. 废除了很多loader 如对图片处理 {test:/\.(png|jpg|svg)$/,type:'asset'}
4. 去掉了webpack的polyfill
5. 使用prepack计算清除

 