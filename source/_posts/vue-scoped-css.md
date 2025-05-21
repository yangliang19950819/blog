---
title: vue scope css 的实现原理
cover: /img/scopecss-z.webp
---

### vue模块Id 每个组件里面所有的dom元素都会加上对应组建的moduleId作为属性
- 从图片中可以看到在style标签中加上了scoped后，打包的出的css部分会加上对用的属性选择器以此来实现css的隔离防止命名冲突造成的样式问题。
![avatar](/img/moduleId.webp)

### 源码位置
- vue-loader lib/loader.js中通过hash算法取文件路径或者文件路径加上传入的内容生成的哈希值作为moduleId。
```js
    ...
    const rawRequest = getRawRequest(this, options.excludedPreLoaders)
    const filePath = this.resourcePath
    const fileName = path.basename(filePath)
    const context =
        (this._compiler && this._compiler.context) ||
        this.options.context ||
        process.cwd()
    const sourceRoot = path.dirname(path.relative(context, filePath))
    const shortFilePath = path.relative(context, filePath).replace(/^(\.\.[\\\/])+/, '').replace(/\\/g, '/')
    const moduleId = 'data-v-' + hash(isProduction ? (shortFilePath + '\n' + content) : shortFilePath)
    console.log(moduleId,'🌛')
    // 模块Id 也是属性选择器 
    ...
```

![avatar](/img/moduleId1.webp)

- vue-loader lib/loader.js中的getRawLoaderString对于template的处理是用的defaultLoaders.html，const templateCompilerPath = normalize.lib('template-compiler/index')，和compile = compiler.compile以及compile(html, compilerOptions)表明template-compiler/index.js调用了vue-template-compiler/build.js的compile方法来处理template，查看template-compiler/index.js可以看到导出了一个loader函数接受到了从template中解析出的html，经过vue-template-compiler/build.js中的compile方法编译成ast树，再经过组装生成code，可以看到.render('data-v-xxx',esEports)传入了moduleId，在之后的loader处理生成html时会给每个组件的元素加上属性data-v-xxx。
![avatar](/img/moduleId4.webp)
![avatar](/img/moduleId5.webp)
```js
// vue-loader lib/loader.js
...
const templateCompilerPath = normalize.lib('template-compiler/index')// 引入了template-compiler解析template
...
const defaultLoaders = {
    html: templateCompilerPath + templateCompilerOptions, 
    css: options.extractCSS
      ? getCSSExtractLoader()
      : styleLoaderPath + '!' + 'css-loader' + cssLoaderOptions,
    js: hasBuble
      ? 'buble-loader' + bubleOptions
      : hasBabel ? 'babel-loader' : ''
  }
  ...
  function getRawLoaderString (type, part, index, scoped) {
      ...
      // if user defines custom loaders for html, add template compiler to it
      if (type === 'template' && loader.indexOf(defaultLoaders.html) < 0) {
        loader = defaultLoaders.html + '!' + loader
      }
      if(type === 'template'){
        // console.log(loader,'🐟')
      }
      return injectString + ensureBang(loader)
    } else {
      // unknown lang, infer the loader to be used
      switch (type) {
        case 'template':
          return (
            defaultLoaders.html +
            '!' +
            templatePreprocessorPath +
            '?engine=' +
            lang +
            '!'
          )
        case 'styles':
          loader = addCssModulesToLoader(defaultLoaders.css, part, index)
          return loader + '!' + styleCompiler + ensureBang(ensureLoader(lang))
        case 'script':
          return injectString + ensureBang(ensureLoader(lang))
        default:
          loader = loaders[type]
          if (Array.isArray(loader)) {
            loader = stringifyLoaders(loader)
          }
          return ensureBang(loader + buildCustomBlockLoaderString(part.attrs))
      }
    }
  }
...
// template-compiler/index.js
const prettier = require('prettier')
const loaderUtils = require('loader-utils')
const normalize = require('../utils/normalize')
const compiler = require('vue-template-compiler')
const transpile = require('vue-template-es2015-compiler')
const hotReloadAPIPath = normalize.dep('vue-hot-reload-api')
const transformRequire = require('./modules/transform-require')
const transformSrcset = require('./modules/transform-srcset')

module.exports = function (html) {
  // console.log(html,'👀')
  this.cacheable()
  const isServer = this.target === 'node'
  const isProduction = this.minimize || process.env.NODE_ENV === 'production'
  const vueOptions = this.options.__vueOptions__ || {}
  const options = loaderUtils.getOptions(this) || {}
  const needsHotReload = !isServer && !isProduction && vueOptions.hotReload !== false
  const defaultModules = [transformRequire(options.transformToRequire), transformSrcset()]
  let userModules = vueOptions.compilerModules || options.compilerModules
  // for HappyPack cross-process use cases
  if (typeof userModules === 'string') {
    userModules = require(userModules)
  }
  const compilerOptions = {
    preserveWhitespace: options.preserveWhitespace,
    modules: defaultModules.concat(userModules || []),
    directives:
      vueOptions.compilerDirectives || options.compilerDirectives || {},
    scopeId: options.hasScoped ? options.id : null,
    comments: options.hasComment
  }

  // vue-template-compiler 编译成ast  此处的compiler不是webpack的compiler模块是vue-template-compiler模块
  const compile =
    isServer && compiler.ssrCompile && vueOptions.optimizeSSR !== false
      ? compiler.ssrCompile
      : compiler.compile

  const compiled = compile(html, compilerOptions)
  // console.log(compiled,'🌟')
  // tips
  if (compiled.tips && compiled.tips.length) {
    compiled.tips.forEach(tip => {
      this.emitWarning(tip)
    })
  }
  let code
  if (compiled.errors && compiled.errors.length) {
    this.emitError(
      `\n  Error compiling template:\n${pad(html)}\n` +
        compiled.errors.map(e => `  - ${e}`).join('\n') +
        '\n'
    )
    code = vueOptions.esModule
      ? `var esExports = {render:function(){},staticRenderFns: []}\nexport default esExports`
      : 'module.exports={render:function(){},staticRenderFns:[]}'
  } else {
    const bubleOptions = options.buble
    const stripWith = bubleOptions.transforms.stripWith !== false
    const stripWithFunctional = bubleOptions.transforms.stripWithFunctional
    const staticRenderFns = compiled.staticRenderFns.map(fn =>
      toFunction(fn, stripWithFunctional)
    )
    // 组装ast生成render函数
    code =
      transpile(
        'var render = ' +
          toFunction(compiled.render, stripWithFunctional) +
          '\n' +
          'var staticRenderFns = [' +
          staticRenderFns.join(',') +
          ']',
        bubleOptions
      ) + '\n'
    // prettify render fn
    if (!isProduction) {
      code = prettier.format(code, { semi: false, parser: 'babylon' })
    }
    // mark with stripped (this enables Vue to use correct runtime proxy detection)
    if (!isProduction && stripWith) {
      code += `render._withStripped = true\n`
    }
    const exports = `{ render: render, staticRenderFns: staticRenderFns }`
    code += vueOptions.esModule
      ? `var esExports = ${exports}\nexport default esExports`
      : `module.exports = ${exports}`
  }
  // hot-reload
  if (needsHotReload) {
    const exportsName = vueOptions.esModule ? 'esExports' : 'module.exports'
    code +=
      '\nif (module.hot) {\n' +
      '  module.hot.accept()\n' +
      '  if (module.hot.data) {\n' +
      '    require("' + hotReloadAPIPath + '")' +
      '      .rerender("' + options.id + '", ' + exportsName + ')\n' +
      '  }\n' +
      '}'
  }
  // console.log(code,'🌞')
  return code
}
...
// vue-template-compiler/build.js 

```


- vue-loader lib/loader.js中的getRawLoaderString来处理styles，并把moduleId传递进去，其中引入了styleCompilerPath执行了lib/style-compiler/index.js。
```js
const styleCompilerPath = normalize.lib('style-compiler/index')

...

function getRawLoaderString (type, part, index, scoped) {
    let lang = part.lang || defaultLang[type]
    let styleCompiler = ''
    if (type === 'styles') {
      // style compiler that needs to be applied for all styles
      styleCompiler =
        styleCompilerPath +
        '?' +
        JSON.stringify({
          // a marker for vue-style-loader to know that this is an import from a vue file
          vue: true,
          id: moduleId,
          scoped: !!scoped,
          hasInlineConfig: !!query.postcss
        }) +
        '!'
      // normalize scss/sass/postcss if no specific loaders have been provided
      if (!loaders[lang]) {
        if (postcssExtensions.indexOf(lang) !== -1) {
          lang = 'css'
        } else if (lang === 'sass') {
          lang = 'sass?indentedSyntax'
        } else if (lang === 'scss') {
          lang = 'sass'
        }
      }
    }
    let loader =
      options.extractCSS && type === 'styles'
        ? loaders[lang] || getCSSExtractLoader(lang)
        : loaders[lang]
    
    const injectString =
      type === 'script' && query.inject ? 'inject-loader!' : ''

    if (loader != null) {
      if (Array.isArray(loader)) {
        loader = stringifyLoaders(loader)
      } else if (typeof loader === 'object') {
        loader = stringifyLoaders([loader])
      }
      if (type === 'styles') {
        // add css modules
        loader = addCssModulesToLoader(loader, part, index)
        // inject rewriter before css loader for extractTextPlugin use cases
        if (rewriterInjectRE.test(loader)) {
          loader = loader.replace(
            rewriterInjectRE,
            (m, $1) => ensureBang($1) + styleCompiler
          )
        } else {
          loader = ensureBang(loader) + styleCompiler
        }
      }
      // if user defines custom loaders for html, add template compiler to it
      if (type === 'template' && loader.indexOf(defaultLoaders.html) < 0) {
        loader = defaultLoaders.html + '!' + loader
      }
      return injectString + ensureBang(loader)
    } else {
      // unknown lang, infer the loader to be used
      switch (type) {
        case 'template':
          return (
            defaultLoaders.html +
            '!' +
            templatePreprocessorPath +
            '?engine=' +
            lang +
            '!'
          )
        case 'styles':
          loader = addCssModulesToLoader(defaultLoaders.css, part, index)
          return loader + '!' + styleCompiler + ensureBang(ensureLoader(lang))
        case 'script':
          return injectString + ensureBang(ensureLoader(lang))
        default:
          loader = loaders[type]
          if (Array.isArray(loader)) {
            loader = stringifyLoaders(loader)
          }
          return ensureBang(loader + buildCustomBlockLoaderString(part.attrs))
      }
    }
  }
```

- vue-loader 中的lib/style-compiler/index.js引入处理scopeId的lib/style-compiler/plugins/scope-id.js，这里plugins.push(scopeId({ id: query.id }))把scope-id作为插件push到postcss(plugins)postcss的插件中。
```js
const postcss = require('postcss')
const loaderUtils = require('loader-utils')
const loadPostcssConfig = require('./load-postcss-config')
const trim = require('./plugins/trim')
const scopeId = require('./plugins/scope-id')
module.exports = function (css, map) {
  this.cacheable()
  const cb = this.async()
  const query = loaderUtils.getOptions(this) || {}
  let vueOptions = this.options.__vueOptions__
  if (!vueOptions) {
    if (query.hasInlineConfig) {
      this.emitError(
        `\n  [vue-loader] It seems you are using HappyPack with inline postcss ` +
          `options for vue-loader. This is not supported because loaders running ` +
          `in different threads cannot share non-serializable options. ` +
          `It is recommended to use a postcss config file instead.\n` +
          `\n  See http://vue-loader.vuejs.org/en/features/postcss.html#using-a-config-file for more details.\n`
      )
    }
    vueOptions = Object.assign({}, this.options.vue, this.vue)
  }
  loadPostcssConfig(this, vueOptions.postcss)
    .then(config => {
      const plugins = config.plugins.concat(trim)
      const options = Object.assign(
        {
          to: this.resourcePath,
          from: this.resourcePath,
          map: false
        },
        config.options
      )
      // add plugin for vue-loader scoped css rewrite
      if (query.scoped) {
        plugins.push(scopeId({ id: query.id }))
      }
      // source map
      if (
        this.sourceMap &&
        !this.minimize &&
        vueOptions.cssSourceMap !== false &&
        process.env.NODE_ENV !== 'production' &&
        !options.map
      ) {
        options.map = {
          inline: false,
          annotation: false,
          prev: map
        }
      }
      return postcss(plugins)
        .process(css, options)
        .then(result => {
          if (result.messages) {
            result.messages.forEach(({ type, file }) => {
              if (type === 'dependency') {
                this.addDependency(file)
              }
            })
          }
          const map = result.map && result.map.toJSON()
          cb(null, result.css, map)
          return null // silence bluebird warning
        })
    })
    .catch(e => {
      console.error(e)
      cb(e)
    })
}
```
- vue-loader lib/style-compiler/plugins/scope-id.js中的postcss的插件，使用selector.insertAfter方法把id插入到生成虚拟节点上，并且从postcss-selector-parser引入selectorParser来解析样式的选择器。
```js
const postcss = require('postcss')
const selectorParser = require('postcss-selector-parser')

module.exports = postcss.plugin('add-id', ({ id }) => root => {
  const keyframes = Object.create(null)
  root.each(function rewriteSelector (node) {
    if (!node.selector) {
      // handle media queries
      if (node.type === 'atrule') {
        if (node.name === 'media' || node.name === 'supports') {
          node.each(rewriteSelector)
        } else if (/-?keyframes$/.test(node.name)) {
          // register keyframes
          keyframes[node.params] = node.params = node.params + '-' + id
        }
      }
      return
    }
    node.selector = selectorParser(selectors => {
      selectors.each(selector => {
        let node = null
        selector.each(n => {
          // ">>>" combinator
          if (n.type === 'combinator' && n.value === '>>>') {
            n.value = ' '
            n.spaces.before = n.spaces.after = ''
            return false
          }
          // /deep/ alias for >>>, since >>> doesn't work in SASS
          if (n.type === 'tag' && n.value === '/deep/') {
            const prev = n.prev()
            if (prev && prev.type === 'combinator' && prev.value === ' ') {
              prev.remove()
            }
            n.remove()
            return false
          }
          if (n.type !== 'pseudo' && n.type !== 'combinator') {
            node = n
          }
        })
        selector.insertAfter(node, selectorParser.attribute({
          attribute: id
        }))
        console.log(selector,'🍎')
      })
    }).process(node.selector).result
  })

  // If keyframes are found in this <style>, find and rewrite animation names
  // in declarations.
  // Caveat: this only works for keyframes and animation rules in the same
  // <style> element.
  if (Object.keys(keyframes).length) {
    root.walkDecls(decl => {
      // individual animation-name declaration
      if (/-?animation-name$/.test(decl.prop)) {
        decl.value = decl.value.split(',')
          .map(v => keyframes[v.trim()] || v.trim())
          .join(',')
      }
      // shorthand
      if (/-?animation$/.test(decl.prop)) {
        decl.value = decl.value.split(',')
          .map(v => {
            const vals = v.trim().split(/\s+/)
            const i = vals.findIndex(val => keyframes[val])
            if (i !== -1) {
              vals.splice(i, 1, keyframes[vals[i]])
              return vals.join(' ')
            } else {
              return v
            }
          })
          .join(',')
      }
    })
  }
})

```
![avatar](/img/moduleId2.webp)

- postcss-selector-parser/dist/index.js中导出了./processor.js。
```js
var _processor = require('./processor');
var _processor2 = _interopRequireDefault(_processor);
...
function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { default: obj }; }
...
var parser = function parser(processor) {
    return new _processor2.default(processor);
};
...
exports.default = parser;
module.exports = exports['default'];
```

- postcss-selector-parser/dist/processor.js函数导出了一个立即执行函数返回了Processor对象，打印selectors可以看到此时的选择器已经被添加了属性选择器。
```js
'use strict';
exports.__esModule = true;
var _createClass = function () { function defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } } return function (Constructor, protoProps, staticProps) { if (protoProps) defineProperties(Constructor.prototype, protoProps); if (staticProps) defineProperties(Constructor, staticProps); return Constructor; }; }();
var _parser = require('./parser');
var _parser2 = _interopRequireDefault(_parser);
function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { default: obj }; }
function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }
var Processor = function () {
    function Processor(func) {
        _classCallCheck(this, Processor);
        this.func = func || function noop() {};
        return this;
    }
    Processor.prototype.process = function process(selectors) {
        console.log(selectors,'🌲')
        var options = arguments.length > 1 && arguments[1] !== undefined ? arguments[1] : {};
        var input = new _parser2.default({
            css: selectors,
            error: function error(e) {
                throw new Error(e);
            },
            options: options
        });
        this.res = input;
        this.func(input);
        return this;
    };
    _createClass(Processor, [{
        key: 'result',
        get: function get() {
            return String(this.res);
        }
    }]);
    return Processor;
}();
exports.default = Processor;
module.exports = exports['default'];
```
![avatar](/img/moduleId3.webp)
- 未完的疑问，当yarn dev 打包开发模式时在processor.js没有添加对应的属性选择器。