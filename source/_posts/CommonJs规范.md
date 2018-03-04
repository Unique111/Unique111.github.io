---
title: CommonJs规范
date: 2017/08/22
categories: ['javascript']
banner:
---
### 概述
Node应用由模块组成，采用CommonJS模块规范。
根据这个规范，每个文件就是一个模块，有自己的作用域。

如果想在多个文件分享变量，必须定义为global对象的属性。
``` javascript
global.warning = true;
```

<!-- more -->

CommonJS规范规定，每个模块内部，`module`变量代表当前模块。这个变量是一个对象，它的`exports`属性（即module.exports）是对外的接口。加载某个模块，其实是加载该模块的`module.exports`属性。

``` javascript
var x = 5;
var addX = function (value) {
  return value + x;
};
module.exports.x = x;
module.exports.addX = addX;
```
上面代码通过module.exports输出变量 x 和函数 addX。

require方法用于加载模块。
``` javascript
var example = require('./example.js');
console.log(example.x); // 5
console.log(example.addX(1)); // 6
```

CommonJs模块的特点：
- 所有代码都运行在模块作用域，不会污染全局作用域。
- 模块可以多次加载，但是只会在第一次加载时运行一次，然后运行结果就被缓存了，以后再加载，就直接读取缓存结果。要想让模块再次运行，必须清除缓存。
- 模块加载的顺序，按照其在代码中出现的顺序。

### module对象
Node内部提供一个Module构建函数，所有模块都是Module的实例。
``` javascript
function Module(id, parent) {
  this.id = id;
  this.exports = {};
  this.parent = parent;
  // ...
}
```

每个模块内部，都有一个module对象，代表当前模块。它有以下属性。
- module.id 模块的识别符，通常是带有绝对路径的模块文件名。
- module.filename 模块的文件名，带有绝对路径。
- module.loaded 返回一个布尔值，表示模块是否已经完成加载。
- module.parent 返回一个对象，表示调用该模块的模块。
- module.children 返回一个数组，表示该模块要用到的其他模块。
- module.exports 表示模块对外输出的值。

例子：
``` javascript
// example.js
var jquery = require('jquery');
exports.$ = jquery;
console.log(module);
```
运行结果：
``` javascript
{
  "id": ".",
  "exports": {
    "$": [Function]
  },
  "parent": null,
  "filename": "/Users/moyongxin/github/node-learning/demo02/commonjs.js",
  "loaded": false,
  "children": [
    {
      "id": "/Users/moyongxin/github/node-learning/demo02/node_modules/jquery/dist/jquery.js",
      "exports": [Function],
      "parent": [Circular],
      "filename": "/Users/moyongxin/github/node-learning/demo02/node_modules/jquery/dist/jquery.js",
      "loaded": true,
      "children": [],
      "paths": [Object]
    }
  ],
  "paths": [
    "/Users/moyongxin/github/node-learning/demo02/node_modules",
    "/Users/moyongxin/github/node-learning/node_modules",
    "/Users/moyongxin/github/node_modules",
    "/Users/moyongxin/node_modules",
    "/Users/node_modules"
  ]
}
```

以上可知，可以通过 module.parent 是否为 null 来判断当前模块是否为入口脚本。

#### module.exports属性
module.exports属性表示当前模块对外输出的接口，其他文件加载该模块，实际上就是读取module.exports变量。

#### exports变量
为了方便，Node为每个模块提供一个exports变量，指向module.exports。这等同在每个模块头部，有一行这样的命令。
``` javascript
var exports = module.exports;
```
所以，可以在exports变量上添加方法。
`注意，不能直接将exports变量指向一个值，因为这样等于切断了exports与module.exports的联系。`

下面的写法是无效的：
``` javascript
exports.hello = function() {
  return 'hello';
};
module.exports = 'Hello world';

```
上面代码中，hello函数是无法对外输出的，因为module.exports被重新赋值了。
这意味着，`如果一个模块的对外接口，就是一个单一的值，不能使用exports输出，只能使用module.exports输出`。
``` javascript
module.exports = function() {};
```

### AMD规范与CommonJS规范的兼容性
CommonJS规范加载模块是同步的，也就是说，只有加载完成，才能执行后面的操作。AMD规范则是非同步加载模块，允许指定回调函数。由于Node.js主要用于服务器编程，模块文件一般都已经存在于本地硬盘，所以加载起来比较快，不用考虑非同步加载的方式，所以CommonJS规范比较适用。但是，如果是浏览器环境，要从服务器端加载模块，这时就必须采用非同步模式，因此浏览器端一般采用AMD规范。









