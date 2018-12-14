## Node 模块化机制 require 源码解析

* [Cycles](#cycles)
* [Cache](#cache)
    * 千万不要试图 delete require.cache['id'] 来实现热重载 《[一行 delete require.cache 引发的内存泄漏血案](https://zhuanlan.zhihu.com/p/34702356)》
* [require](#require-源码解析)


### Cycles
`a.js`:
```javascript
console.log('a 开始');
exports.done = false;
const b = require('./b.js');
console.log('在 a 中，b.done = %j', b.done);
exports.done = true;
console.log('a 结束');
```

`b.js`:
```javascript
console.log('b 开始');
exports.done = false;
const a = require('./a.js');
console.log('在 b 中，a.done = %j', a.done);
exports.done = true;
console.log('b 结束');
```


`main.js`:
```javascript
console.log('main 开始');
const a = require('./a.js');
const b = require('./b.js');
console.log('在 main 中，a.done=%j，b.done=%j', a.done, b.done);
```


```bash
$ node main.js
main 开始
a 开始
b 开始
在 b 中，a.done = false
b 结束
在 a 中，b.done = true
a 结束
在 main 中，a.done=true，b.done=true
```

### Cache
`a.js`:
```javascript
module.exports = new Date();
```

`main.js`:
```javascript
console.log(require('./a.js'));
console.log(require('./a.js'));
setTimeout(()=>console.log(require('./a.js')),5000)
```

```bash
三次输出的值每次都是一样的
```

### require 源码解析
require 源码的位置在 [lib/internal/modules/cjs/loader.js](https://github.com/nodejs/node/blob/master/lib/internal/modules/cjs/loader.js)

`Module` 构造函数

```javascript
function Module(id, parent) {
  this.id = id;
  this.exports = {};
  this.parent = parent;
  updateChildren(parent, this, false);
  this.filename = null;
  this.loaded = false;
  this.children = [];
}
```

`Module` 对象的属性是什么
`a.js`:

```javascript
module.exports = exports = {
    a: 123
};
console.log('module.id: ', module.id);
console.log('module.exports: ', module.exports);
console.log('module.parent: ', module.parent);
console.log('module.filename: ', module.filename);
console.log('module.loaded: ', module.loaded);
console.log('module.children: ', module.children);
console.log('module.paths: ', module.paths);

```
```
$ node a.js
module.id:  .
module.exports:  { a: 123 }
module.parent:  null
module.filename:  /Users/catchme/Sites/rpc/module/a.js
module.loaded:  false
module.children:  []
module.paths:  [ '/Users/catchme/Sites/rpc/module/node_modules',
  '/Users/catchme/Sites/rpc/node_modules',
  '/Users/catchme/Sites/node_modules',
  '/Users/catchme/node_modules',
  '/Users/node_modules',
  '/node_modules' ]

```

如果没有父模块，直接调用当前模块，parent 属性就是 null，id 属性就是一个点。filename 属性是模块的绝对路径，path 属性是一个数组，包含了模块可能的位置。另外，输出这些内容时，模块还没有全部加载，所以 loaded 属性为 false 。

`b.js`:
```javascript
var a = require('./a.js');
```

```
$ node b.js
module.id:  /Users/catchme/Sites/rpc/module/a.js
module.exports:  { a: 123 }
module.parent:  Module {
  id: '.',
  exports: {},
  parent: null,
  filename: '/Users/catchme/Sites/rpc/module/b.js',
  loaded: false,
  children: 
   [ Module {
       id: '/Users/catchme/Sites/rpc/module/a.js',
       exports: [Object],
       parent: [Circular],
       filename: '/Users/catchme/Sites/rpc/module/a.js',
       loaded: false,
       children: [],
       paths: [Array] } ],
  paths: 
   [ '/Users/catchme/Sites/rpc/module/node_modules',
     '/Users/catchme/Sites/rpc/node_modules',
     '/Users/catchme/Sites/node_modules',
     '/Users/catchme/node_modules',
     '/Users/node_modules',
     '/node_modules' ] }
module.filename:  /Users/catchme/Sites/rpc/module/a.js
module.loaded:  false
module.children:  []
module.paths:  [ '/Users/catchme/Sites/rpc/module/node_modules',
  '/Users/catchme/Sites/rpc/node_modules',
  '/Users/catchme/Sites/node_modules',
  '/Users/catchme/node_modules',
  '/Users/node_modules',
  '/node_modules' ]
```
由于 a.js 被 b.js 调用，所以 parent 属性指向 b.js 模块，id 属性和 filename 属性一致，都是模块的绝对路径。

`require`方法

```javascript
Module.prototype.require = function(id) {
  validateString(id, 'id'); //判断typeof id === 'string'
  if (id === '') {
    throw new ERR_INVALID_ARG_VALUE('id', id,
                                    'must be a non-empty string');
  }
  return Module._load(id, this, /* isMain */ false); // 因为是引入模块，require.main === module 为 false
};
```

`Module._load` 方法

```javascript
Module._load = function(request, parent, isMain) {
  // 这一段就在打印输出  
  if (parent) {
    debug('Module._load REQUEST %s parent: %s', request, parent.id);
  }
  // 计算文件的绝对路径
  // 如果是核心模块直接返回核心模块的名字
  var filename = Module._resolveFilename(request, parent, isMain);
  
  // 在缓存中取该模块
  var cachedModule = Module._cache[filename];
  if (cachedModule) {
    // 如果取到了就给调用require的模块添加添加(push)children[]
    updateChildren(parent, cachedModule, true);
    return cachedModule.exports;
  }
  // 如果是核心模块就用NativeModule.require方法来导入
  if (NativeModule.nonInternalExists(filename)) {
    debug('load native module %s', request);
    return NativeModule.require(filename);
  }

  // Don't call updateChildren(), Module constructor already does.
  var module = new Module(filename, parent);

  if (isMain) {
    process.mainModule = module;
    module.id = '.';
  }

  Module._cache[filename] = module;
  // 这个方法会调用 Module.prototype.load
    // 通过这个方法确定文件的后缀(.js .json)
    // var extension = findLongestRegisteredExtension(filename);
    // 每一种不同文件的后缀就会使用不同的载入方法
    // Module._extensions[extension](this, filename);
  tryModuleLoad(module, filename);

  return module.exports;
};

```

`Module.__extensions`载入方式
```javascript
// Native extension for .js
Module._extensions['.js'] = function(module, filename) {
  var content = fs.readFileSync(filename, 'utf8');
  module._compile(stripBOM(content), filename);
};


// Native extension for .json
Module._extensions['.json'] = function(module, filename) {
  var content = fs.readFileSync(filename, 'utf8');
  try {
    module.exports = JSON.parse(stripBOM(content));
  } catch (err) {
    err.message = filename + ': ' + err.message;
    throw err;
  }
};


// Native extension for .node
Module._extensions['.node'] = function(module, filename) {
  return process.dlopen(module, path.toNamespacedPath(filename));
};

if (experimentalModules) {
  if (asyncESM === undefined) lazyLoadESM();
  Module._extensions['.mjs'] = function(module, filename) {
    throw new ERR_REQUIRE_ESM(filename);
  };
}
```

**require() 函数同步还是异步?**

正如`Module._extensions['.js']`中`var content = fs.readFileSync(filename, 'utf8');`所以 require 函数是同步的。  
那么为什么设计成同步的呢?  
~~因为 CommonJS 定义了模块同步加载同步执行，所以Node用同步的实现。~~
> [Node的module.js里用了readFileSync而不用readFile？](https://www.zhihu.com/question/38041375)

* *CommonJS 模块是同步加载和同步执行，AMD 模块是异步加载和异步执行，CMD（Sea.js）模块是异步加载和同步执行。ES6 的模块体系最后选择的是异步加载和同步执行。也就是 Sea.js 的行为是最接近 ES6 模块的。不过 Sea.js 这样做是需要付出代价的——需要扫描代码提取依赖，所以它不像 CommonJS/AMD 是纯运行时的模块系统。--贺师俊*
* 更加的简单
* 开销更小
* 历史原因 Node(2009),只有 AMD 和 CommonJS,而与 ES6 更加接近 CMD(Sea.js 2010)
* Node 的模块设计具有缓存,如果异步 require 的时候执行了一个同步的 require 就会出现不可预知的问题

`Module.prototype._compile` 方法

```javascript
Module.prototype._compile = function(content, filename) {
  // 移除
  content = stripShebang(content);
  // 将代码 包裹
  //(function(exports, require, module, __filename, __dirname) {
         // 模块的代码实际上在这里
  // });
  var wrapper = Module.wrap(content);

  var compiledWrapper = vm.runInThisContext(wrapper, {
    filename: filename,
    lineOffset: 0,
    displayErrors: true,
    importModuleDynamically: experimentalModules ? async (specifier) => {
      if (asyncESM === undefined) lazyLoadESM();
      const loader = await asyncESM.loaderPromise;
      return loader.import(specifier, normalizeReferrerURL(filename));
    } : undefined,
  });

  var inspectorWrapper = null;
  if (process._breakFirstLine && process._eval == null) {
    if (!resolvedArgv) {
      // we enter the repl if we're not given a filename argument.
      if (process.argv[1]) {
        resolvedArgv = Module._resolveFilename(process.argv[1], null, false);
      } else {
        resolvedArgv = 'repl';
      }
    }

    // Set breakpoint on module start
    if (filename === resolvedArgv) {
      delete process._breakFirstLine;
      inspectorWrapper = internalBinding('inspector').callAndPauseOnStart;
    }
  }
  var dirname = path.dirname(filename);
  var require = makeRequireFunction(this);
  var depth = requireDepth;
  if (depth === 0) stat.cache = new Map();
  var result;
  if (inspectorWrapper) {
    result = inspectorWrapper(compiledWrapper, this.exports, this.exports,
                              require, this, filename, dirname);
  } else {
    // 入exports、require、module三个全局变量
    result = compiledWrapper.call(this.exports, this.exports, require, this,filename, dirname);
  }
  if (depth === 0) stat.cache = null;
  return result;
};
```

`vm.runInThisContext`方法

这个方法类似与 `eval`







