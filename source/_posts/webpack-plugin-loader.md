---
title: webpack loader 和 plugins   
cover: /img/webpack.webp
---

### 如何编写一个最简版的babel-loader

- babel-loader的配置，复制的babel-loader的配置
```js
const { resolve } = require('path')
module.exports = {
    module: {
        rules: [
          {
            test: /\.m?js$/,
            exclude: /node_modules/,
            use: {
              loader: resolve(__dirname,'./js-loader.js'),
              options: {
                presets: [
                  ['@babel/preset-env', { targets: "defaults" }]
                ]
              }
            }
          }
        ]
    }
}
```

- js-loader.js引入了acorn,acorn-walk,magic-string,loader-utils。loader-utils获取你配置的options，acorn解析js生成ast树，acorn-walk遍历生成的ast树，magic-string负责把字符串改写或者添加你需要的类容，pitch方法可以在loader执行前加入你所需要的内容。
```js
const acorn = require('acorn');
const walk = require('acorn-walk');
const MagicString = require( 'magic-string' );
const loadUtils = require('loader-utils');

module.exports = function(context){
    console.log(this.data.value,'👀')
    const options = loadUtils.getOptions(this);
    console.log(options,'🐶')
    const ast = acorn.parse(context);
    console.log(ast,'🍎')
    const code = new MagicString(context);
    console.log(code,'🚗')
    walk.simple(ast,{
        VariableDeclaration(node){
            console.log(node,'🚀')
            const { start } = node;
            code.overwrite(start,start + 5,'var')
        },
        ExpressionStatement(node){
            console.log(node,'🌹')
            const { end } = node;
            code.overwrite(end  -1,end,'+ "添加的元素")')
        }
    })
    return code.toString();
}

module.exports.pitch = function(r,prerequest,data){
    data.value = '人众无知'
}
```
- 生成的dist中的main.js可以看到已经用var替换了const，并添加了对应的字符。 
```js
/******/ (() => { // webpackBootstrap
/******/ 	var __webpack_modules__ = ({

/***/ "./src/index.js":
/*!**********************!*\
  !*** ./src/index.js ***!
  \**********************/
/***/ (() => {

eval("var jiong = '🤔囧';\nconsole.log(jiong+ \"添加的元素\")\n\n//# sourceURL=webpack://loader/./src/index.js?");

/***/ })

/******/ 	});
/************************************************************************/
/******/ 	
/******/ 	// startup
/******/ 	// Load entry module and return exports
/******/ 	// This entry module can't be inlined because the eval devtool is used.
/******/ 	var __webpack_exports__ = {};
/******/ 	__webpack_modules__["./src/index.js"]();
/******/ 	
/******/ })()
```


### tapable webpack处理钩子函数的库，负责把插件挂起到compiler上。
```js
// webpack/lib/Compiler.js  这里的this指代的是compiler实例
this.hooks = Object.freeze({
			/** @type {SyncHook<[]>} */
			initialize: new SyncHook([]),
			/** @type {SyncBailHook<[Compilation], boolean>} */
			shouldEmit: new SyncBailHook(["compilation"]),
			/** @type {AsyncSeriesHook<[Stats]>} */
			done: new AsyncSeriesHook(["stats"]),
			/** @type {SyncHook<[Stats]>} */
			afterDone: new SyncHook(["stats"]),
			/** @type {AsyncSeriesHook<[]>} */
			additionalPass: new AsyncSeriesHook([]),
			/** @type {AsyncSeriesHook<[Compiler]>} */
			beforeRun: new AsyncSeriesHook(["compiler"]),
			/** @type {AsyncSeriesHook<[Compiler]>} */
			run: new AsyncSeriesHook(["compiler"]),
			/** @type {AsyncSeriesHook<[Compilation]>} */
			emit: new AsyncSeriesHook(["compilation"]),
			/** @type {AsyncSeriesHook<[string, AssetEmittedInfo]>} */
			assetEmitted: new AsyncSeriesHook(["file", "info"]),
			/** @type {AsyncSeriesHook<[Compilation]>} */
			afterEmit: new AsyncSeriesHook(["compilation"]),

			/** @type {SyncHook<[Compilation, CompilationParams]>} */
			thisCompilation: new SyncHook(["compilation", "params"]),
			/** @type {SyncHook<[Compilation, CompilationParams]>} */
			compilation: new SyncHook(["compilation", "params"]),
			/** @type {SyncHook<[NormalModuleFactory]>} */
			normalModuleFactory: new SyncHook(["normalModuleFactory"]),
			/** @type {SyncHook<[ContextModuleFactory]>}  */
			contextModuleFactory: new SyncHook(["contextModuleFactory"]),

			/** @type {AsyncSeriesHook<[CompilationParams]>} */
			beforeCompile: new AsyncSeriesHook(["params"]),
			/** @type {SyncHook<[CompilationParams]>} */
			compile: new SyncHook(["params"]),
			/** @type {AsyncParallelHook<[Compilation]>} */
			make: new AsyncParallelHook(["compilation"]),
			/** @type {AsyncParallelHook<[Compilation]>} */
			finishMake: new AsyncSeriesHook(["compilation"]),
			/** @type {AsyncSeriesHook<[Compilation]>} */
			afterCompile: new AsyncSeriesHook(["compilation"]),

			/** @type {AsyncSeriesHook<[Compiler]>} */
			watchRun: new AsyncSeriesHook(["compiler"]),
			/** @type {SyncHook<[Error]>} */
			failed: new SyncHook(["error"]),
			/** @type {SyncHook<[string | null, number]>} */
			invalid: new SyncHook(["filename", "changeTime"]),
			/** @type {SyncHook<[]>} */
			watchClose: new SyncHook([]),
			/** @type {AsyncSeriesHook<[]>} */
			shutdown: new AsyncSeriesHook([]),

			/** @type {SyncBailHook<[string, string, any[]], true>} */
			infrastructureLog: new SyncBailHook(["origin", "type", "args"]),

			// TODO the following hooks are weirdly located here
			// TODO move them for webpack 5
			/** @type {SyncHook<[]>} */
			environment: new SyncHook([]),
			/** @type {SyncHook<[]>} */
			afterEnvironment: new SyncHook([]),
			/** @type {SyncHook<[Compiler]>} */
			afterPlugins: new SyncHook(["compiler"]),
			/** @type {SyncHook<[Compiler]>} */
			afterResolvers: new SyncHook(["compiler"]),
			/** @type {SyncBailHook<[string, Entry], boolean>} */
			entryOption: new SyncBailHook(["context", "entry"])
		});
    // tapable库
    const {
			SyncHook,
			SyncBailHook,
			SyncWaterfallHook,
			SyncLoopHook,
			AsyncParallelHook,
			AsyncParallelBailHook,
			AsyncSeriesHook,
			AsyncSeriesBailHook,
			AsyncSeriesWaterfallHook
		 } = require("tapable");

		1.SyncHook 同步串行 不关心监听函数的返回值
		2.SyncBailHook 同步串行 只要监听函数中有一个函数的返回值不为null,则跳过剩下所有的逻辑
		3.SyncWaterfallHook 同步串行 上一个监听函数的返回值可以传递给下一个监听函数
		4.SyncLoopHook 同步循环 当监听函数被触发的时候，如果该监听函数返回true时则这个监听函数会反复执行，如果返回undefined 则表示退出循环
		5.AsyncParallelHook 异步并发 不关心监听函数的返回值
		6.AsyncParallelBailHook 异步并发 只要监听函数的返回值不为null，就会忽略后面的监听函数执行，直接跳跃到callAsync等触发函数绑定的回调函数，然后执行这个被绑定的回调函数
		7.AsyncSeriesHook 异步串行 不关心callback的参数
		8.AsyncSeriesBailHook 异步串行 callback的参数不为null，就会直接执行callAsync等触发函数绑定的回调函数
		9.AsyncSeriesWaterfallHook 异步串行 上一个监听函数中的callback(err,data)的第二个参数，可以作为下一个监听函数的参数
```