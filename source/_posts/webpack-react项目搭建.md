---
title: webpack react项目搭建
date: 2017-11-08
categories: ['构建工具', 'webpack']
tags:
---
### 创建项目
``` javascript
mkdir webpack-react-demo
cd webpack-react-demo
npm init
```

<!-- more -->

目录结构：
webpack-react-deom
- build
    - main.js
- client
    - src
        - main.js
    - index.html

### 添加入口文件
index.html：
``` html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <title>React Demo</title>
  </head>
  <body>
    <div id="app"></div>
  </body>
</html>
```

创建一个 js 入口文件，client/src/main.js：
``` javascript
ReactDOM.render(<h1>哈罗，哈哈哈哈</h1>, document.getElementById('app'));
```

将 client/src/main.js 引入到 index.html 中：
``` javascript
<!-- ... -->
<body>
  <div id="app"></div>
  <script src="client/src/main.js"></script>
</body>
<!-- ... -->
```

毫无疑问，使用浏览器打开index.html是跑不起来的。一是因为我们没有引入react和react-dom包，二是因为这里用到了jsx语法，需要通过babel进行转义。

### 安装相关依赖
为了方便，直接将react和react-dom通过cdn的方式引入。
``` javascript
<!-- ... -->
<body>
  <div id="app"></div>
  <script src="https://unpkg.com/react@15/dist/react.js"></script>
  <script src="https://unpkg.com/react-dom@15/dist/react-dom.js"></script>
  <script src="client/src/main.js"></script>
</body>
<!-- ... -->
```

安装babel-cli和babel-preset-react：
``` javascript
npm install babel-cli babel-preset-react --save-dev
```

我们将babel-cli安装到了本地依赖，运行babel需要借助npm script，将运行脚本添加到package.json的scripts中：
``` javascript
// ...
"scripts": {
  "build": "babel --presets react client/src --out-dir build --watch"
},
// ...
```

接着运行npm run build，我们可以看到新产生的build目录，以及该目录里面的main.js文件。

最后，修改 index.html 里面的引用路径：
``` javascript
<!-- ... -->
<body>
  <div id="app"></div>
  <script src="https://unpkg.com/react@15/dist/react.js"></script>
  <script src="https://unpkg.com/react-dom@15/dist/react-dom.js"></script>
  <script src="build/main.js"></script><!-- 修改位置 -->
</body>
<!-- ... -->
```

总结：这里用了最最简单的方式将react跑起来，即在index.html里引入经过babel转化的入口js文件。

存在的问题：
- 没有使用模块化管理，简单地使用了script标签引入模块
- 没有热加载，每次修改都要刷新页面
- 没有使用ES6语法
- 没有引入环境变量，项目缺少适用性
- …





