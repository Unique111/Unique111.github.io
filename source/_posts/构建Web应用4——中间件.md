---
title: 构建Web应用4——中间件
date: 2017-10-05
categories: ['node']
tags:
---
在实际项目中，有多琐碎的工作需要完成。对于Web应用而言，我们希望不用接触到这么多细节性的处理，由此引入中间件（middleware）来简化和隔离这些基础设施与业务逻辑之间的细节，让开发者能够关注在业务的开发上，提升开发效率。

从HTTP请求到具体业务逻辑之间，其实有很多的细节要处理。Node的http模块提供了应用层协议网络的封装，对具体业务并没有支持，在业务逻辑之下，必须有开发框架对业务提供支持。这里我们通过中间件的形式搭建开发框架，这个开发框架用来组织各个中间件。对于Web应用的各种基础功能，我们通过中间件来完成，每个中间件处理掉相对简单的逻辑，最终汇成强大的基础框架。

<!-- more -->

中间件的上下文就是请求对象和响应对象：req和res。由于Node异步的原因，我们需要提供一种机制，在当前中间件处理完成后，通知下一个中间件执行。
``` javascript
var middleware = function(req, res, next) {
  // TODO
  next();
};
```

在业务逻辑中添加中间件：
``` javascript
app.use('/user/:username', querystring, cookie, session, function(req, rs) {
  // TODO
});
// querystring解析中间件
var querystring = function(req, res, next) {
  req.query = url.parse(req.url, true).query;
  next();
};
```

组织好中间件，将路由分离开来，将中间件和具体业务逻辑都看成业务处理单元，改进use()方法：
``` javascript
app.use = function(path) {
  var handle = {
    // 第一个参数作为路径
    path: pathRegexp(path),
    // 其他的都是处理单元
    stack: Array.prototype.slice.call(arguments, 1)
  };
  routes.all.push(handle);
};
```

改进后的use()方法将中间件都存进了stack数组中，等待匹配后触发执行。由于结构变化了，匹配部分也需要修改：
``` javascript
var match = function (pathname, routes) {
  for (var i = 0; i < routes.length; i++) {
    var route = routes[i];
    // 正则匹配
    var reg = route.path.regexp;
    var matched = reg.exec(pathname);
    if (matched) {
      // 抽取具体值
      // 代码省略
      // 将中间件数组交给handle()方法处理
      handle(req, res, route.stack);
      return true;
    }
  }
  return false;
};
```

一旦匹配成功，中间件具体如何调动都交给了handle()方法处理，该方法封装后，递归性地执行数组中的中间件，每个中间件执行完成后，按照约定调用传入next()方法以触发下一个中间件执行（或者直接响应），直到最后的业务逻辑。

``` javascript
var handle = function(req, res, stack) {
  var next = function() {
    // 从stack数组中取出中间件并执行
    var middleware = stack.shift();
    if (middleware) {
      // 传入next()函数自身，使中间件能够执行结束后递归
      middleware(req, res, next);
    }
  };
  // 启动执行
  next();
};
```

中间件以上组织方式可进行优化，参考《深入浅出Node.js》p213。

### 异常处理
为next()方法添加err参数，并捕获中间件直接抛出的同步异常：
``` javascript
var handle = function(req, res, stack) {
  var next = function(err) {
    if (err) {
      return handle500(err, req, res, stack);
    }
    // 从stack数组中取出中间件并执行
    var middleware = stack.shift();
    if (middleware) {
      // 传入next()函数自身，使中间件能够执行结束后递归
      try {
        middleware(req, res, next);
      } catch(err) {
        next(err);
      }
    }
  };
  // 启动执行
  next();
};
```

由于异步方法的异常不能直接捕获，中间件异步产生的异常需要自己传递出来：
``` javascript
var session = function(req, res, next) {
  var id = req.cookie.sessionid;
  store.get(id, function(err, session) {
    if (err) {
      // 将异常通过next()传递
      return next(err);
    }
    req.session = session;
    next();
  });
};
```

处理异常的中间件的设计与普通中间件略有差别，它的参数有4个：
``` javascript
var middleware = function(err, req, res, next) {
  // TODO
  next();
};
```

通过use()可以将异常处理的中间件注册起来：
``` javascript
app.use(function(err, req, res, next) {
  // TODO
});
```

为了区分普通中间件和异常处理中间件，handle500()方法将会对中间件按参数进行选取，然后递归执行：
``` javascript
var handle500 = function(err, req, res, stack) {
  // 选取异常处理中间件
  stack = stack.filter(function(middleware) {
    return middleware.length === 4;
  });
  var next = function() {
    // 从stack数组中取出中间件并执行
    var middleware = stack.shift();
    if (middleware) {
      // 传递异常对象
      middleware(err, req, res, next);
    }
  };
  // 启动执行
  next();
};
```

### 中间件与性能
- 编写高效的中间件
    - 使用高效的方法
    - 缓存需要重复计算的结果
    - 避免不必要的计算
- 合理利用路由，避免不必要的中间件执行
在拥有一堆高效的中间件后，并不意味着每个中间件我们都使用，合理的路由使得不必要的中间件不参与请求处理的过程。

例如有一个静态文件的中间件，它会对请求进行判断，如果磁盘上存在对应文件，就响应对应的静态文件，否则就交由下游中间件处理：
``` javascript
var staticFile = function(req, res, next) {
  var pathname = url.parse(req.url).pathname;
  fs.readFile(path.join(ROOT, pathname), function(err, file) {
    if (err) {
      return next();
    }
    res.writeHead(200);
    res.end(file);
  });
};
```

如果以如下方式注册路由：
``` javascript
app.use(staticFile);
```

那么意味着对/路径下所有的URL请求都会进行判断。十分浪费性能。
提升匹配成功率，添加一个更好的路由路径：
``` javascript
app.use('/public', staticFile);
```








