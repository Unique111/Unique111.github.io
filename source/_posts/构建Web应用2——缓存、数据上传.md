---
title: 构建Web应用2——缓存、数据上传
date: 2017-09-04
categories: ['node']
tags:
---
### 缓存
为了提高性能，YSlow中提到几条关于缓存的规则：
- 添加Expires或Cache-Control到报文头中
- 配置ETags
- 让Ajax可缓存

通常来说，POST, DELETE, PUT这类带行为性的请求操作一般不做任何缓存，大多数缓存只应用在GET请求中。

<!-- more -->

#### If-Modified-Since/Last-Modifed
本地没有文件时，浏览器必然会请求服务器端的内容，并将这部分内容放置在本地的某个缓存目录中。在第二次请求时，它将对本地文件进行检查，如果不能确定这份本地文件是否可以直接使用，它将会发起一次条件请求。所谓条件请求，就是在普通的GET请求报文中，附带If-Modified-Since字段。

它将询问服务器是否有更新的版本，本地文件的最后修改时间。如果服务器端没有新的版本，只需响应一个304状态码，客户端就使用本地版本。如果有新的版本，就将新的内容发送给客户端，客户端放弃本地版本。
``` javascript
var handle = function(req, res) {
  fs.stat(filename, function(err, stat) {
    var lastModified = stat.mtime.toUTCString();
    if (lastModified === req.headers['if-modified-since']) {
      res.writeHead(304, 'Not Modified');
      res.end();
    } else {
      fs.readFile(filename, function(err, file) {
        var lastModified = stat.mtime.toUTCString();
        res.setHeader('Last-Modified', lastModified);
        res.writeHead(200, 'ok');
        res.end(file);
      });
    }
  });
};
```

这里的条件请求采用时间戳的方式实现，但是时间戳有一些缺陷存在：
- 文件的时间戳改动但内容不一定改动
- 时间戳只能精确到秒级别，更新频繁的内容将无法生效

#### If-None-Match/ETag
为了解决以上问题，HTTP1.1引入了ETag。ETag全称是EntityTag，`由服务器端生成`，服务器端可以决定它的生成规则。如果根据文件内容生成散列值，那么条件请求将不会受到时间戳改动造成的带宽浪费。
``` javascript
var getHash = function(str) {
  var shasum = crypto.createHash('sha1');
  return shasum.update(str).digest('base64');
};
```

与If-Modified-Since/Last-Modifed不同的是，ETag的请求和响应是If-None-Match/ETag：
``` javascript
var handle = function(req, res) {
  fs.readFile(filename, function(err, file) {
    var hash = getHash(file);
    var noneMatch = req.headers['if-none-match'];
    if (hash === noneMatch) {
      res.writeHead(304, 'Not Modified');
      res.end();
    } else {
      res.setHeader('ETag', hash);
      res.writeHead(200, 'ok');
      res.end(file);
    }
  });
};
```

浏览器在收到ETag: ‘xxx’这样的响应后，在下次的请求中，会将其放置在请求头中：If-None-Match: xxx。

#### Expires和Cache-Control
尽管条件请求可以在文件内容没有修改的情况下节省带宽，但是它依然会发起一个HTTP请求，使得客户端依然会花一定时间来等待响应。

可见，最好的方案就是连条件请求都不用发起。那么如何让浏览器知道是否能直接使用本地版本呢？服务器端在响应内容时，让浏览器明确地将内容缓存起来。

YSlow规则提到，在响应里设置Expires或Cache-Control头，浏览器将根据该值进行缓存。
Expires：
``` javascript
var handle = function(req, res) {
  fs.readFile(filename, function(err, file) {
    var expires = new Date();
    expires.setTime(expires.getTime() + 10 * 365 * 24 * 60 * 60 * 1000);
    res.setHeader('Expires', expires.toUTCString());
    res.writeHead(200, 'ok');
    res.end(file);
  });
};
```
Expires是一个GMT格式的时间字符串，浏览器在接收到这个过期值后，只要本地还存在这个缓存文件，在到期时间之前它都不会再发起请求。

但是Expires的缺陷在于浏览器与服务器之间的时间可能不一致。为此，引入Cache-Control：
``` javascript
var handle = function(req, res) {
  fs.readFile(filename, function(err, file) {
    res.setHeader('Cache-Control', 'max-age=' + 10 * 365 * 24 * 60 * 60 * 1000);
    res.writeHead(200, 'ok');
    res.end(file);
  });
};
```
相比于Expires，Cache-Control能够避免浏览器端与服务器端时间不同步带来的不一致性问题，只要进行类似倒计时的方式计算过期时间即可。

#### 清除缓存
缓存一旦设定，当服务器端意外更新内容时，却无法通知客户端更新。这使得我们在使用缓存时也要为其设定版本号，所幸浏览器是根据URL进行缓存，那么，一旦内容有所更新时，我们就让浏览器发起新的URL请求，使得新内容能够被客户端更新。一般的更新机制有如下两种：

- 每次发布，路径中跟随Web应用的版本号：http://url.com?v=xxx
- 每次发布，路径中跟随该文件内容的hash值：http://url.com/?hash=xxx
根据文件内容的hash值进行缓存淘汰会更加高效，因为文件内容不一定随着Web应用的版本而更新，而内容没有更新时，版本号的改动导致的更新毫无意义，因此以文件内容形成的hash值更精准。

### Basic认证
Basic认证是当客户端与服务器端进行请求时，允许通过用户名和密码实现的一种身份认证方式。
如果一个页面需要Basic认证，它会检查请求报文头中的Authorization字段的内容，该字段的值由认证方式和加密值构成。eg：
``` javascript
curl -v 'http://user:pass@www.baidu.com'
```

在Basic认证中，它会将用户和密码部分组合：username + ‘:’ + password。然后进行Base64编码：
``` javascript
var encode = function(username, password) {
  return new Buffer(username + ':' + password).toString('base64');
};
```

如果用户首次访问该网页，URL地址中也没携带认证内容，那么浏览器会响应一个401未授权的状态码。
``` javascript
function(req, res) {
  var auth = req.headers['authorization'] || '';
  var parts = auth.split(' ');
  var method = parts[0] || '';  // Basic
  var encoded = parts[1] || ''; // xxx
  var decoded = new Buffer(encoded, 'base64').toString('utf-8').split(':');
  var user = decoded[0];
  var pass = decoded[1];
  if (!checkUser(user, pass)) {
    res.setHeader('WWW-Authenticate', 'Basic realm="Secure Area"');
    res.writeHead(401);
    res.end();
  } else {
    handle(req, res);
  }
}
```

响应中的WWW-Authenticate字段告知浏览器采用什么样的认证和加密方式。
当认证通过后，服务器端响应200状态码后，浏览器会保存用户名和密码口令，在后续的请求中都携带上Authorization信息。

### 数据上传
头部报文中的内容已经能够让服务器端进行大多数业务逻辑操作了，但是单纯的头部报文无法携带大量的数据，在业务中，我们往往需要接收一些数据，比如表单提交、文件提交、JSON上传、XML上传等。

Node的http模块只对HTTP报文的头部进行了解析，然后出发request事件。如果请求中还带有内容部分，内容部分需要用户自己接收和解析。通过报头的Transfer-Encoding或Content-Length即可判断请求中是否带有内容。
``` javascript
var hasBody = function(req) {
  return 'transfer-encoding' in req.headers || 'content-length' in req.headers;
};
```

在HTTP_Parser解析报头结束后，报文内容部分会通过data事件触发，我们只需以流的方式处理即可：
``` javascript
function(req, res) {
  if (hasBody(req)) {
    var buffers = [];
    req.on('data', function(chunk) {
      buffers.push(chunk);
    });
    req.on('end', function() {
      req.rawBody = Buffer.concat(buffers).toString();
      handle(req, res);
    });
  } else {
    handle(req, res);
  }
}
```

将接收到的Buffer列表转化为一个Buffer对象后，再转换为没有乱码的字符串，暂时挂载在req.rawBody处。

#### 表单数据
默认的表单提交，请求头中的Content-Type字段值为application/x-www-form-urlencoded。
由于它的报文体内容跟查询字符串相同：foo=bar&baz=val
因此，解析它很容易：
``` javascript
var handle = function(req, res) {
  if (req.headers['content-type'] === 'application/x-www-form-urlencoded') {
    req.body = querystring.parse(req.rawBody);
  }
  todo(req, res);
};
```

#### 其他格式
JSON类型：application/json，XML：application/xml。
注意：在Content-Type中可能附带其他信息：Content-Type: application/json; charset=utf-8
所以，在判断时，要注意区分：
``` javascript
var mime = function(req) {
  var str = req.headers['content-type'] || '';
  return str.split(';')[0];
};
```

##### JSON文件
``` javascript
var handle = function(req, res) {
  if (mime(req) === 'application/json') {
    try {
      req.body = JSON.parse(req.rawBody);
    } catch(e) {
      res.writeHead(400);
      res.end('Invalid JSON');
      return;
    }
  }
  todo(req, res);
};
```

##### XML文件
``` javascript
var xml2js = require('xml2js');
var handle = function(req, res) {
  if (mime(req) === 'application/xml') {
    xml2js.parseString(req.rawBody, function(err, xml) {
      if (err) {
        res.writeHead(400);
        res.end('Invalid XML');
        return;
      }
      req.body = xml;
      todo(req, res);
    });
  }
};
```

#### 附件上传
浏览器在遇到multipart/form-data表单提交时，构造的请求报文与普通表单完全不同，报文头：
``` javascript
Content-Type: multipart/form-data; boundary=AaB03x
Content-Length: 18231
```

它代表本次提交的内容是由多部分构成的，其中boundary-AaB03x指定的是每部分内容的分界符，AaB03x是随机生成的一段字符串，报文体的内容将通过在它前面添加–进行分割，报文结束时在它前后都加上–表示结束。另外，Content-Length的值必须确保是报文体的长度。

一旦我们知晓报文是如何构成的，那么解析它就变得十分容易。注意，由于是文件上传，那么像普通表单、JSON或XML那样先接收内容再解析的方式将变得不可接受。接收大小未知的数据量时，我们需要十分谨慎：
``` javascript
function(req, res) {
  if (hasBody(req)) {
    var done = function() {
      handle(req, res);
    };
    if (mime(req) === 'application/json') {
      parseJSON(req, done);
    } else if (mime(req) === 'application/xml') {
      parseXML(req, done);
    } else if (mime(req) === 'multipart/form-data') {
      parseMultipart(req, done);
    }
  } else {
    handle(req, res);
  }
}
```

formidable模块：基于流式处理解析报文，将接收到的文件写入到系统的临时文件夹中，并返回对应的路径：
``` javascript
function(req, res) {
  if (hasBody(req)) {
    if (mime(req) === 'multipart/form-data') {
      var form = new formidable.IncomingForm();
      form.parse(req, function(err, fields, files) {
        req.body = fields;
        req.files = files;
        handle(req, res);
      });
    }
  } else {
    handle(req, res);
  }
}
```

因此，在业务逻辑中，只要检查req.body和req.files的内容即可。

### 数据上传与安全
#### 内存限制
在解析表单、JSON和XML部分，我们采取的策略是先保存用户提交的所有数据，然后再解析处理，最后才传递给业务逻辑。这种方式仅仅适合数据量小的提交请求，一旦数据量过大，将发生内存被占光的情况。

解决方案：
- 限制上传内容的大小，一旦超出限制，停止接收数据，并响应400状态码
- 通过流式解析，将数据流导向到磁盘中，Node只保留文件路径等小数据

流式处理在上文的文件上传已有体现，下面介绍Connect中采用的上传数据量的限制方式：
``` javascript
var bytes = 1024;
function(req, res) {
  var received = 0;
  var len = req.headers['content-length'] ? parseInt(req.headers['content-length'], 10) : null;
  if (len && len > bytes) {
    res.writeHead(413);
    res.end();
    return;
  }
  // 限制
  req.on('data', function(chunk) {
    received += chunk.length;
    if (received > bytes) {
      // 停止接收数据，触发end()
      req.destroy();
    }
  });
  handle(req, res);
}
```

#### CSRF（Cross-Site Request Forgery，跨站请求伪造）
服务器端与客户端通过Cookie来标识和认证用户，通常来说，用户通过浏览器访问服务器端的SessionID是无法被第三方知道的，但是CSRF的攻击者并不需要知道Session ID就能让用户中招。

例子：网站A有一个留言程序，提交留言的接口：http://domain_a.com/guestbook
用户通过POST提交content字段就能成功留言。服务器端会自动从Session数据中判断是谁提交的数据，补足username和UpdateAt两个字段后向数据库中写入数据。
``` javascript
function(req, res) {
  var content = req.body.content || '';
  var username = req.session.username;
  var feedback = {
    username,
    content,
    updateAt: Date.now(),
  };
  db.save(feedback, function(err) {
    res.writeHead(200);
    res.end('Ok');
  });
}
```

如果某个攻击者发现这里有CSRF漏洞，那么他就可以在另一个网站上构造了一个表单。攻击者只要引诱某个domain_a的登录用户访问这个网站，就会自动提交一个留言。

解决方案：添加随机值。
``` javascript
var generateRandom = function(len) {
  return crypto.randomBytes(Math.ceil(len * 3 / 4))
    .toString('base64')
    .slice(0, len);
};
```

也就是说，为每个请求的用户，在Session中赋予一个随机值。
``` javascript
var token = req.session._csrf || (req.session._csrf = generateRandom(24));
```

在做页面渲染时，将这个_csrf值告知前端。
由于该值是一个随机值，攻击者构造出相同的随机值的难度相当大，所以我们只需要再接收端做一次校验即可。

``` javascript
function(req, res) {
  var token = req.session._csrf || (req.session._csrf = generateRandom(24));
  var _csrf = req.body._csrf;
  if (token !== _csrf) {
    res.writeHead(403);
    res.end('禁止访问');
  } else {
    handle(req, res);
  }
}
```



























