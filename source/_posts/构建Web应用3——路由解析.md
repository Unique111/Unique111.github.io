---
title: 构建Web应用3——路由解析
date: 2017-09-09
categories: ['node']
tags:
---
### 文件路径型
- 静态文件
这种方式的路由很简单，URL的路径与网站目录的路径一致，无须转换，非常直观。这种路由的处理方式也很简单，将请求路径对应的文件发送给客户端即可。
- 动态文件

<!-- more -->

### MVC
在MVC流行之前，主流的处理方式都是通过文件路径进行处理的，甚至以为是常态。直到有一天开发者发现用户请求的URL路径原来可以跟具体脚本所在的路径没有任何关系。

MVC模型的主要思想是将业务逻辑按照职责分离：
- 控制器（Controller）：一组行为的组合
- 模型（Model）：数据相关的操作和封装
- 视图（View）：视图的渲染

这是目前最经典的分层模式，工作模式如下：
- 路由解析，根据URL寻找到对应的控制器和行为
- 行为调用相关的模型，进行数据操作
- 数据操作结束后，调用视图和相关数据进行页面渲染，输出到客户端

控制器如何调用模型和如何渲染页面，各种实现都大同小异。
如何根据URL做路由映射，这里有2个分支实现：
- 手工关联映射：有一个路由文件来将URL映射到对应的控制器
- 自然关联映射：没有这样的文件

User => Controller(Action) => Model => View => User

#### 手工映射
手工映射除了需要手工配置路由较为原始外，它对URL的要求十分灵活，几乎没有格式上的限制。
假设有一个处理设置用户信息的控制器：
``` javascript
exports.setting = function(req, res) {
  // TODO
}
```

再添加一个映射的方法即可：
``` javascript
var routes = [];
var use = function(path, action) {
  routes.push([path, action]);
};
```

我们在入口程序中判断URL，然后执行对应的逻辑，于是就完成了基本的路由映射过程了：
``` javascript
function(req, res) {
  var pathname = url.parse(req.url).pathname;
  for(var i = 0; i < routes.length; i++) {
    var route = routes[i];
    if (pathname === route[0]) {
      var action = route[1];
      action(req, res);
      return;
    }
  }
  // 处理404请求
  handle404(req, res);
}
```

- 正则匹配
``` javascript
use('/profile/:username', function(req, res) {
  // TODO
});
```

于是改进我们的匹配方式，在通过use注册路由时需要将路径转换为一个正则表达式，然后通过它来进行匹配：
``` javascript
var pathRegexp = function(path) {
  ... // 十分复杂正则表达式转换
  return new RegExp('^' + path + '$');
};
// /profile/:username => /profiel/aaa, /profile/bbb
```

现在，重新改进注册部分：
``` javascript
var use = function(path, action) {
  routes.push([pathRegexp(path), action]);
};
```

以及匹配部分：
``` javascript
function(req, res) {
  var pathname = url.parse(req.url).pathname;
  for(var i = 0; i < routes.length; i++) {
    var route = routes[i];
    // 正则匹配
    if (route[0].exec(pathname)) {
      var action = route[1];
      action(req, res);
      return;
    }
  }
  // 处理404请求
  handle404(req, res);
}
```

- 参数解析
尽管完成了正则匹配，可以实现相似URL的匹配，但是:username到底匹配了啥，还没有解决。为此我们还需要进一步将匹配到的内容抽取出来，希望在业务中能这样调用：
``` javascript
use('/profile/:username', function(req, res) {
  var username = req.params.username;
  // TODO
});
```

这里的目标是将抽取的内容设置到req.params处。那么第一步就是将键值抽取出来：
``` javascript
var pathRegexp = function(path) {
  var keys = [];
  ...
  return {
    keys: keys,
    regexp: new RegExp('^' + path + '$');
  }
};
```

我们将根据抽取的键值和实际的URL得到键值匹配到的实际值，并设置到req.params处：
``` javascript
function(req, res) {
  var pathname = url.parse(req.url).pathname;
  for(var i = 0; i < routes.length; i++) {
    var route = routes[i];
    // 正则匹配
    var reg = route[0].regexp;
    var keys = route[0].keys;
    var matched = reg.exec(pathname);
    if (matched) {
      // 抽取具体值
      var params = {};
      for(var i = 0; i < keys.length; i++) {
        var value = matched[i+1];
        if (value) {
          params[keys[i]] = value;
        }
      }
      req.params = params;
      var action = route[1];
      action(req, res);
      return;
    }
  }
  // 处理404请求
  handle404(req, res);
}
```

#### 自然映射
手工映射的优点在于路径可以很灵活，但是如果项目较大，路由映射的数量也会很多。从前端路径到具体的控制器文件，需要进行查阅才能定位到实际代码的位置，为此有人提出，尽是路由不如无路由。实际上并非没有路由，而是路由按一种约定的方式自然而然地实现了路由，而无需去维护路由映射。
``` javascript
/controller/action/param1/param2/param3
```

例子：/user/setting/12/1987，它会按约定去找controllers目录下的user文件，将其require出来后，调用这个文件模块的setting()方法，而其余的值作为参数直接传递给这个方法。

``` javascript
function(req, res) {
  var pathname = url.parse(req.url).pathname;
  var paths = pathname.split('/');
  var controller = paths[1] || 'index';
  var action = paths[2] || 'index';
  var args = paths.slice(3);
  var module;
  try {
    // require的缓存机制使得只有第一次是阻塞的
    module = require('./controllers/' + controller);
  } catch (ex) {
    handle500(req, res);
    return;
  }
  var method = module[action];
  if (method) {
    method.apply(null, [req, res].concat(args));
  } else {
    handle500(req, res);
  }
}
```

由于这种自然映射的方式没有指明参数的名称，所以无法采用req.params的方式提取，但是直接通过参数获取更简洁：
``` javascript
exports.setting = function(req, res, month, year) {
  // TODO
};
```

### RESTful（Representational State Transfer，表现层状态转化）
MVC模式大行其道多年，知道RESTful的流行，大家才意识到URL也可以设计得很规范，请求方法也能作为逻辑分发的单元。

RESTful的设计哲学主要将服务器端提供的内容实体看作一个资源，并表现在URL上。

例子：/users/jack
这个地址代表一个资源，对这个资源的操作，主要体现在HTTP请求方法上，不是体现在URL上。
过去的增删查改（操作行为主要体现在行为上，主要使用的请求方法时POST和GET）：
``` javascript
POST /user/add?username=jack
GET /user/remove?username=jack
POST /user/update?usernaem=jack
GET /user/get?username=jack
```

RESTful设计：
``` javascript
POST /user/jack
DELETE /user/jack
PUT /user/jack
GET /user/jack
```

它将DELETE和PUT请求方法引入设计中，参与资源的操作和更改资源的状态。

另外，在RESTful设计中，资源的具体格式由请求报头中的Accept字段和服务器端的支持情况来决定。

所以，RESTful的设计就是，通过URL设计资源、请求方法定义资源的操作，通过Accept决定资源的表现形式。

RESTful与MVC设计并不冲突，而且是更好的改进。相比MVC，RESTful只是将HTTP请求方法也加入了路由的过程，以及在URL路径上体现得更资源化。

#### 请求方法
为了让Node能够支持RESTful需求，改进下我们的设计。如果use是对所有请求方法的处理，那么在RESTful的场景下，我们需要区分请求方法设计：
``` javascript
var routes = {'all': []};
var app = {};
app.use = function(path, action) {
  routes.all.push([pathRegexp(path), action]);
};
['get', 'put', 'delete', 'post'].forEach(function(method) {
  routes[method] = [];
  app[method] = function(path, action) {
    routes[method].push([pathRegexp(path), action]);
  };
});
```

通过以下方式完成路由映射：
``` javascript
// 增加用户
app.post('/user/:username', addUser);
// 删除用户
app.delete('/user/:username', removeUser);
// 修改用户
app.put('/user/:username', updateUser);
// 查询用户
app.get('/user/:username', getUser);
```

这样的路由能够识别请求方法，并将业务进行分发。为了让分发部分更简洁，我们先将匹配的部分抽取为match()方法：
``` javascript
var match = function(pathname, routes) {
  for (var i = 0; i < routes.length; i++) {
    var route = routes[i];
    // 正则匹配
    var reg = route[0].regexp;
    var keys = route[0].keys;
    var matched = reg.exec(pathname);
    if (matched) {
      // 抽取具体值
      var params = {};
      for (var i = 0, l = keys.length; i < l; i++) {
        var value = matched[i + 1];
        if (value) {
          params[keys[i]] = value;
        }
      }
      req.params = params;
      var action = route[1];
      action(req, res);
      return true;
    }
  }
  return false;
};
```

然后改进我们的分发部分：
``` javascript
function(req, res) {
  var pathname = url.parse(req.url).pathname;
  // 将请求方法变为小写
  var method = req.method.toLowerCase();
  if (routes.hasOwnProperty(method)) {
    // 根据请求方法分发
    if (match(pathname, routes[method])) {
      return;
    } else {
      // 如果路径没有匹配成功，尝试让all()来处理
      if (match(pathname, routes.all)) {
        return;
      }
    }
  } else {
    // 直接让all()来处理
    if (match(pathname, routes.all)) {
      return;
    }
  }
  // 处理404请求
  handle404(req, res);
}
```




























