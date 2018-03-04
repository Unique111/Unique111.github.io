---
title: node模块实现
date: 2017-08-26
categories: ['node']
tags:
---
除了HTML、Webkit布局引擎和显卡这些UI相关技术没有支持外，Node的结构和Chrome十分相似。
Node不处理UI，但用与浏览器相同的机制和原理运行。

### CommonJS的模块规范
CommonJs对模块的定义非常简单，主要分为模块引用（require）、模块定义（module.exports）和模块标识。

<!-- more -->

### Node的模块实现
在Node中引入模块，需要经历3个步骤：路径分析、文件定位、编译执行。

在Node中，模块分为两类：（1）Node提供的模块，称为核心模块；（2）用户编写的模块，称为文件模块。
核心模块：在Node源代码的编译过程中，编译进了二进制执行文件。引用核心模块时，文件定位和编译执行这2个步骤可以省略，并且在路径分析中优先判断，所以它的加载速度是最快的。
文件模块：在运行时动态加载，需要完整的路径分析、文件定位、编译执行过程，速度比核心模块慢。

#### 优先从缓存加载

#### 路径分析和文件定位
##### 模块标识符分析
- 核心模块
- 路径形式的文件模块
- 自定义模块

##### 文件定位
- 文件扩展名分析
.js，.json，.node，分析过程中，会调用fs模块同步阻塞式地判断文件是否存在。
- 目录分析和包
目录 => package.json => 通过JSON.parse()解析出包描述对象，取出main属性指定的文件名进行定位。
若没有package.json文件，Node会将index当作默认文件名。

#### 模块编译
在Node中，每个文件模块都是一个对象，定义如下：
``` javascript
function Module(id, parent) {
  this.id = id;
  this.exports = {};
  this.parent = parent;
  if (parent && parent.children) {
    parent.children.push(this);
  }
  this.filename = null;
  this.loaded = false;
  this.children = [];
}
```
定位到具体的文件后，Node会新建一个模块对象，然后根据路径载入并编译。
- .js文件：通过fs模块同步读取文件后编译执行；
- .node文件：用C/C++编写，通过dlopen()方法加载；
- .json文件：通过fs模块同步读取文件后，用JSON.parse()解析返回结果；
- 其他文件：当作.js文件载入。

每一个编译成功的模块都会将其文件路径作为索引缓存在Module._cache对象上，以提高二次引用的性能。

在确定文件的扩展名后，Node将调用具体的编译方式来将文件执行后返回给调用者。

##### JavaScript模块编译
在编译的过程中，Node对获取的JavaScript文件内容进行了头尾包装。
``` javascript
在头部添加：(function (exports, require, module, __filename, __dirname) {\n
在尾部添加：\n})
```
一个正常的js文件会被包装成如下样子：
``` javascript
(function (exports, require, module, __filename, __dirname) {
  var math = require('math');
  exports.area = function (radius) {
    return Math.PI * radius * radius;
  };
});
```
这样每个模块文件之间都进行了作用域隔离。包装之后的代码会通过vm原生模块的runInThisContext()方法执行，返回一个具体的function对象。

最后，将当前模块对象的exports属性、require()方法、module（模块对象自身）以及在文件定位中得到的完整文件路径和文件目录作为参数传递给这个function()执行。

##### C/C++模块的编译
Node调用process.dlopen()方法进行加载和执行。

##### JSON文件的编译

### 包与NPM
#### NPM常用功能
CommonJS包规范是理论，NPM是其中的一种实践。

##### 常看帮助

##### 安装依赖包

##### NPM钩子命令
package.json 的 scripts 字段，命令：prexxx，xxx，postxxx

##### 发布包
- 编写模块
``` javascript
exports.sayHello = function() {
  return 'Hello, world';
};
```
- 初始化包描述文件
命令：npm init
- 注册包仓库账号
为了维护包，NPM必须要使用仓库账号才允许将包发布到仓库中。
npm adduser
- 上传包
上传包的命令是：npm publish。
在刚刚创建的package.json文件所在的目录下，执行npm publish .开始上传包。
在这个过程中，NPM会将目录打包成一个存档文件，然后上传到官方源仓库中。
- 安装包
- 管理包权限
通常，一个包只有一个人拥有权限进行发布。如果需要多人进行发布，可以使用npm owner命令来管理包的所有者：npm owner ls xxx包名
npm owner ls (package name)
npm owner add (user) (package name)
npm owner rm (user) (package name)

##### 分析包
在使用NPM的过程中，如果不能确认当前目录下能否通过require()顺利引入想要的包，可以执行 npm ls 分析包。
这个命令可以分析出当前路径下能够通过模块路径找到的所有包，并生产依赖树。

### 前后端共用模块

#### 模块的侧重点
前后端JavaScript分别搁置在HTTP的两端，它们扮演的角色并不同。浏览器端的JavaScript需要经历`从同一个服务器端分发到多个客户端执行`，而服务器端JavaScript则是`相同的代码需要多次执行`。

前者的瓶颈在于带宽，后者的瓶颈则在于CPU和内存等资源。前者需要通过网络加载代码，后者从磁盘中加载，两者的加载速度不在一个数量级上。

鉴于网络的原因，CommonJS为后端JavaScript制定的规范并不完全适合前端的应用场景。经过一段争执后，AMD规范最终在前端应用场景中胜出。此外，还有玉伯定义的CMD规范。

#### AMD规范（Asynchronous Module Definition，“异步模块定义”）
AMD规范是CommonJS模块规范的一个延伸，定义：define(id?, dependencies?, factory);
它的模块id和依赖是可选的，与Node模块相似的地方：factory的内容就是实际代码的内容。

``` javascript
define(function() {
  var exports = {};
  exports.sayHello = function() {
    alert('Hello from module: ' + module.id);
  };
  return exports;
});
```
不同之处：AMD模块需要用define来明确定义一个模块，而在Node中是隐式包装的，目的都是进行作用域隔离，仅在需要的时候被引入，避免掉过去那种通过全局变量或者全局命名空间的方式，以免变量污染和不小心被修改。另一个区别是`内容需要通过返回的方式实现导出`。

#### CMD规范
CMD规范与AMD规范的主要区别在于定义模块和依赖引入的部分。

AMD规范需要在声明模块的时候指定所有的依赖，通过形参传递依赖到模块内容中：
``` javascript
define(['dep1', 'dep2'], function(dep1, dep2) {
  return function() {};
});
```
与AMD模块规范相比，CMD模块更接近于Node对CommonJS规范的定义：define(factory);
在依赖部分，CMD支持动态引入：
``` javascript
define(function(require, exports, module) {
  // The module code goes here
});
```
require，exports和module通过形参传递给模块，在需要依赖模块时，随时调用require()引入即可。

#### 兼容多种模块规范
为了保持前后端的一致性，类库开发者需要将类库代码包装在一个闭包内。以下示例能够兼容Node、AMD、CMD以及常见的浏览器环境：
``` javascript
(function(name, definition) {
  // 检测上下文环境是否为AMD或CMD
  var hasDefine = typeof define === 'function';
  // 检查上下文环境是否为Node
  var hasExports = typeof module !== 'undefined' && module.exports;
  if (hasDefine) {
    // AMD环境或CMD环境
    define(definition);
  } else if (hasExports) {
    // 定义为普通Node模块
    module.exports = definition();
  } else {
    // 将模块的执行结果挂在window变量中，在浏览器中this指向window对象
    this[name] = definition();
  }
})('hello', function() {
  var hello = function() {};
  return hello;
});
```


























