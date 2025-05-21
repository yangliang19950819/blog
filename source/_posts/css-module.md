---
title: css-loader怎么实现css-module
cover: /img/css-module.webp
---

### css-loader 中 utils.js

- generateScopedName函数拿到css类名，根据传入所需的hash算法生成与每个css类名对应的hash值，最后作为编译后的类名来防止类名冲突。

```js 
var _postcssModulesScope = _interopRequireDefault(require("postcss-modules-scope"));

(0, _postcssModulesScope.default)({ 
        generateScopedName(exportName) {
        //exportName是传入的css的类名
        let localIdent;
        if (typeof getLocalIdent !== "undefined") {
          localIdent = getLocalIdent(loaderContext, localIdentName, unescape(exportName), {
            context: localIdentContext,
            hashSalt: localIdentHashSalt,
            hashFunction: localIdentHashFunction,
            hashDigest: localIdentHashDigest,
            hashDigestLength: localIdentHashDigestLength,
            regExp: localIdentRegExp
          });
        } // A null/undefined value signals that we should invoke the default
        // getLocalIdent method.


        if (typeof localIdent === "undefined" || localIdent === null) {
          localIdent = defaultGetLocalIdent(
            loaderContext, /*loader的上下文*/
            localIdentName, /*[hash:base64]*/
            unescape(exportName), /*去掉反斜杠*/
            {
            context: localIdentContext,
            hashSalt: localIdentHashSalt,
            hashFunction: localIdentHashFunction,
            hashDigest: localIdentHashDigest,
            hashDigestLength: localIdentHashDigestLength,
            regExp: localIdentRegExp
          });
          return escapeLocalIdent(localIdent).replace(/\\\[local\\]/gi, exportName);
        }
        
        return escapeLocalIdent(localIdent);
      },

function defaultGetLocalIdent(loaderContext, localIdentName /*[hash:base64]*/, localName/*css类名*/, options) {
  let relativeMatchResource = ""; // eslint-disable-next-line no-underscore-dangle

  if (loaderContext._module && loaderContext._module.matchResource) {
    relativeMatchResource = `${normalizePath( // eslint-disable-next-line no-underscore-dangle
    _path.default.relative(options.context, loaderContext._module.matchResource))}\x00`;
  }

  const relativeResourcePath = normalizePath(_path.default.relative(options.context, loaderContext.resourcePath /*css文件所在的路径*/)); // eslint-disable-next-line no-param-reassign

  //relativeResourcePath css文件的相对路径  normalizePath 转换对应系统的斜线的分割符号


  options.content = `${relativeMatchResource}${relativeResourcePath}\x00${localName}`;
  let {
    hashFunction, //md4
    hashDigest, // hex
    hashDigestLength //20
  } = options;

  //localIdentName 哪种hash算法
  const mathes = localIdentName.match(/\[(?:([^:\]]+):)?(?:(hash|contenthash|fullhash))(?::([a-z]+\d*))?(?::(\d+))?\]/i);

  if (mathes) {
    const hashName = mathes[2] || hashFunction;
    hashFunction = mathes[1] || hashFunction;
    hashDigest = mathes[3] || hashDigest;
    hashDigestLength = mathes[4] || hashDigestLength; // `hash` and `contenthash` are same in `loader-utils` context
    // let's keep `hash` for backward compatibility
    // eslint-disable-next-line no-param-reassign

    localIdentName = localIdentName.replace(/\[(?:([^:\]]+):)?(?:hash|contenthash|fullhash)(?::([a-z]+\d*))?(?::(\d+))?\]/gi, () => hashName === "fullhash" ? "[fullhash]" : "[contenthash]");
  } // eslint-disable-next-line no-underscore-dangle


  const hash = loaderContext._compiler.webpack.util.createHash(hashFunction);

  const {
    hashSalt
  } = options;

  if (hashSalt) {
    hash.update(hashSalt);
  }

  hash.update(options.content);
  const localIdentHash = hash.digest(hashDigest).slice(0, hashDigestLength).replace(/[/+]/g, "_").replace(/^\d/g, "_"); // TODO need improve on webpack side, we should allow to pass hash/contentHash without chunk property, also `data` for `getPath` should be looks good without chunk property

  const ext = _path.default.extname(loaderContext.resourcePath);

  const base = _path.default.basename(loaderContext.resourcePath);

  const name = base.slice(0, base.length - ext.length);
  const data = {
    filename: _path.default.relative(options.context, loaderContext.resourcePath),
    contentHash: localIdentHash,
    chunk: {
      name,
      hash: localIdentHash,
      contentHash: localIdentHash
    }
  }; // eslint-disable-next-line no-underscore-dangle

  let result = loaderContext._compilation.getPath(localIdentName, data);

  if (options.regExp) {
    const match = loaderContext.resourcePath.match(options.regExp);

    if (match) {
      match.forEach((matched, i) => {
        result = result.replace(new RegExp(`\\[${i}\\]`, "ig"), matched);
      });
    }
  }

  return result;
}

```


### postcss-modules-scope src中的index.js

- 在postcss-modules-scope中可以拿到我们写的css类的命名了，也可以看到generateScopedName函数生成作用域命名时，如果传入的options有这个函数,则调用传入的函数，在css-loader的utils.js中可以找到该函数。

```js
const generateScopedName =
    (options && options.generateScopedName) || plugin.generateScopedName;

function exportScopedName(name, rawName) {
        // console.log(name,'🌹') //这里传入了css的类名
        const scopedName = generateScopedName(
          rawName ? rawName : name,
          root.source.input.from,
          root.source.input.css
        );
        const exportEntry = generateExportEntry(
          rawName ? rawName : name,
          scopedName,
          root.source.input.from,
          root.source.input.css
        );
        const { key, value } = exportEntry;

        exports[key] = exports[key] || [];

        if (exports[key].indexOf(value) < 0) {
          exports[key].push(value);
        }
        return scopedName;
      }

```