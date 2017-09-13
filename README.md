# Koa中文文档 - [点击查看](https://www.w3cways.com/doc/koa/)

# 简介

koa 是由 Express 原班人马打造的，致力于成为一个更小、更富有表现力、更健壮的 Web 框架。使用 koa 编写 web 应用，通过组合不同的 generator，可以免除重复繁琐的回调函数嵌套，并极大地提升错误处理的效率。koa 不在内核方法中绑定任何中间件，它仅仅提供了一个轻量优雅的函数库，使得编写 Web 应用变得得心应手。



## 安装

Koa需要 node v7.6.0或更高版本来支持ES2015、异步方法

你可以安装自己支持的node版本。
```javascript
$ nvm install 7
$ npm i koa
$ node my-koa-app.js
```

### Babel异步函数

在node < 7.6的版本中使用async 函数, 我们推荐使用[babel's require hook](http://babeljs.io/docs/usage/require/).
```javascript
require('babel-core/register');
// require the rest of the app that needs to be transpiled after the hook
const app = require('./app');
```


为了解析和转译异步函数，你应该至少有[transform-async-to-generator](http://babeljs.io/docs/plugins/transform-async-to-generator/) or [transform-async-to-module-method](http://babeljs.io/docs/plugins/transform-async-to-module-method/)这2个插件。例如，在你的.babelrc文件中，应该有如下代码
```javascript
{
  "plugins": ["transform-async-to-generator"]
}
```
也可以使用[env preset](http://babeljs.io/docs/plugins/preset-env/)并设置"node": "current"来替代.



## 应用
Koa 应用是一个包含一系列中间件 generator 函数的对象。 这些中间件函数基于 request 请求以一个类似于栈的结构组成并依次执行。 Koa 类似于其他中间件系统（比如 Ruby's Rack 、Connect 等）， 然而 Koa 的核心设计思路是为中间件层提供高级语法糖封装，以增强其互用性和健壮性，并使得编写中间件变得相当有趣。

Koa 包含了像 content-negotiation（内容协商）、cache freshness（缓存刷新）、proxy support（代理支持）和 redirection（重定向）等常用任务方法。 与提供庞大的函数支持不同，Koa只包含很小的一部分，因为Koa并不绑定任何中间件。

任何教程都是从 hello world 开始的，Koa也不例外^_^:
```javascript
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);
```

### 级联

Koa 的中间件通过一种更加传统（您也许会很熟悉）的方式进行级联，摒弃了以往 node 频繁的回调函数造成的复杂代码逻辑。 然而，使用异步函数，我们可以实现"真正" 的中间件。与之不同，当执行到 yield next 语句时，Koa 暂停了该中间件，继续执行下一个符合请求的中间件('downstrem')，然后控制权再逐级返回给上层中间件('upstream')。

下面的例子在页面中返回 "Hello World"，然而当请求开始时，请求先经过 x-response-time 和 logging 中间件，并记录中间件执行起始时间。 然后将控制权交给 reponse 中间件。当一个中间件调用next()函数时，函数挂起并控件传递给定义的下一个中间件。在没有更多的中间件执行下游之后，堆栈将退出，并且每个中间件被恢复以执行其上游行为。

```javascript
const Koa = require('koa');
const app = new Koa();

// x-response-time

app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  ctx.set('X-Response-Time', `${ms}ms`);
});

// logger

app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}`);
});

// response

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);
```
### 配置
应用配置是 app 实例属性，目前支持的配置项如下：
- app.env 默认为 NODE_ENV or "development"
- app.proxy 如果为 true，则解析 "Host" 的 header 域，并支持 X-Forwarded-Host
- app.subdomainOffset 默认为2，表示 .subdomains 所忽略的字符偏移量。

### app.listen(...)
Koa 应用并非是一个 1-to-1 表征关系的 HTTP 服务器。 一个或多个Koa应用可以被挂载到一起组成一个包含单一 HTTP 服务器的大型应用群。

如下为一个绑定3000端口的简单 Koa 应用，其创建并返回了一个 HTTP 服务器，为 Server#listen() 传递指定参数（参数的详细文档请查看[nodejs.org](http://nodejs.org/api/http.html#http_server_listen_port_hostname_backlog_callback)）。

```javascript
const Koa = require('koa');
const app = new Koa();
app.listen(3000);
```
The app.listen(...) 实际上是以下代码的语法糖:
```javascript
const http = require('http');
const Koa = require('koa');
const app = new Koa();
http.createServer(app.callback()).listen(3000);
```

这意味着您可以同时支持 HTTPS 和 HTTPS，或者在多个端口监听同一个应用。
```javascript
const http = require('http');
const https = require('https');
const Koa = require('koa');
const app = new Koa();
http.createServer(app.callback()).listen(3000);
https.createServer(app.callback()).listen(3001);
```

### app.callback()

返回一个适合 http.createServer() 方法的回调函数用来处理请求。 您也可以使用这个回调函数将您的app挂载在 Connect/Express 应用上。

### app.use(function)

为应用添加指定的中间件，详情请看 [Middleware](https://github.com/koajs/koa/wiki#middleware)

### app.keys=

设置签名cookie密钥。

该密钥会被传递给KeyGrip, 当然，您也可以自己生成 KeyGrip. 例如:
```javascript
app.keys = ['im a newer secret', 'i like turtle'];
app.keys = new KeyGrip(['im a newer secret', 'i like turtle'], 'sha256');
```

在进行cookie签名时，只有设置 signed 为 true 的时候，才会使用密钥进行加密：
```javascript
ctx.cookies.set('name', 'tobi', { signed: true });
```

### app.context

app.context是从中创建ctx的原型。 可以通过编辑app.context向ctx添加其他属性。当需要将ctx添加到整个应用程序中使用的属性或方法时，这将会非常有用。这可能会更加有效（不需要中间件）和/或更简单（更少的require()），而不必担心更多的依赖于ctx，这可以被看作是一种反向模式。

例如，从ctx中添加对数据库的引用：
```javascript
app.context.db = db();

app.use(async ctx => {
  console.log(ctx.db);
});
```
注:

- ctx上的很多属性是被限制的，在app.context只能通过使用Object.defineProperty()来编辑这些属性（不推荐）。可以在 https://github.com/koajs/koa/issues/652 上查阅
- 已安装的APP沿用父级的ctx和配置. Thus, mounted apps are really just groups of middleware.

### 错误处理

默认情况下Koa会将所有错误信息输出到 stderr， 除非 app.silent 是 true.当err.status是404或者err.expose时，默认错误处理程序也不会输出错误。要执行自定义错误处理逻辑，如集中式日志记录，您可以添加一个"错误"事件侦听器：
```javascript
app.on('error', err => {
  log.error('server error', err)
});
```
如果错误发生在 请求/响应 环节，并且其不能够响应客户端时，Contenxt 实例也会被传递到 error 事件监听器的回调函数里。
```javascript
app.on('error', (err, ctx) => {
  log.error('server error', err, ctx)
});
```
当发生错误但仍能够响应客户端时（比如没有数据写到socket中），Koa会返回一个500错误(Internal Server Error)。 无论哪种情况，Koa都会生成一个应用级别的错误信息，以便实现日志记录等目的。

## Context(上下文)
Koa Context 将 node 的 request 和 response 对象封装在一个单独的对象里面，其为编写 web 应用和 API 提供了很多有用的方法。 这些操作在 HTTP 服务器开发中经常使用，因此其被添加在上下文这一层，而不是更高层框架中，因此将迫使中间件需要重新实现这些常用方法。

context 在每个 request 请求中被创建，在中间件中作为接收器(receiver)来引用，或者通过 this 标识符来引用：
```javascript
app.use(async ctx => {
  ctx; // is the Context
  ctx.request; // is a koa Request
  ctx.response; // is a koa Response
});
```
许多 context 的访问器和方法为了便于访问和调用，简单的委托给他们的 ctx.request 和 ctx.response 所对应的等价方法， 比如说 ctx.type 和 ctx.length 代理了 response 对象中对应的方法，ctx.path 和 ctx.method 代理了 request 对象中对应的方法。

### API

Context 详细的方法和访问器。

#### ctx.req

Node 的 request 对象。

#### ctx.res

Node 的 response 对象。

Koa 不支持 直接调用底层 res 进行响应处理。请避免使用以下 node 属性：

- res.statusCode
- res.writeHead()
- res.write()
- res.end()

#### ctx.request

Koa 的 Request 对象。

#### ctx.response

Koa 的 Response 对象。

#### ctx.state

推荐的命名空间，用于通过中间件传递信息到前端视图
```javascript
ctx.state.user = await User.find(id);
```

#### ctx.app

应用实例引用。

#### ctx.cookies.get(name, [options])

获得 cookie 中名为 name 的值，options 为可选参数：

- signed 如果为 true，表示请求时 cookie 需要进行签名。
注意：Koa 使用了 Express 的 [cookies](https://github.com/jed/cookies) 模块，options 参数只是简单地直接进行传递。

#### ctx.cookies.set(name, value, [options])

设置 cookie 中名为 name 的值，options 为可选参数：

- maxAge 一个数字，表示 Date.now()到期的毫秒数
- signed 是否要做签名
- expires cookie有效期
- pathcookie 的路径，默认为 /'
- domain cookie 的域
- secure false 表示 cookie 通过 HTTP 协议发送，true 表示 cookie 通过 HTTPS 发送。
- httpOnly true 表示 cookie 只能通过 HTTP 协议发送
- overwrite 一个布尔值，表示是否覆盖以前设置的同名的Cookie（默认为false）。 如果为true，在设置此cookie时，将在同一请求中使用相同名称（不管路径或域）设置的所有Cookie将从Set-Cookie头部中过滤掉。
注意：Koa 使用了 Express 的 [cookies](https://github.com/jed/cookies) 模块，options 参数只是简单地直接进行传递。

#### ctx.throw([status], [msg], [properties])

抛出包含 .status 属性的错误，默认为 500。该方法可以让 Koa 准确的响应处理状态。 Koa支持以下组合：
```javascript
ctx.throw(400);
ctx.throw(400, 'name required');
ctx.throw(400, 'name required', { user: user });
```
this.throw('name required', 400) 等价于：
```javascript
const err = new Error('name required');
err.status = 400;
err.expose = true;
throw err;
```
注意：这些用户级错误被标记为 err.expose，其意味着这些消息被准确描述为对客户端的响应，而并非使用在您不想泄露失败细节的场景中。

您可以选择传递一个属性对象，该对象被合并到错误中，这对装饰机器友好错误非常有用，并且这些错误会被报给上层请求。
```javascript
ctx.throw(401, 'access_denied', { user: user });
```

koa用 [http-errors](https://github.com/jshttp/http-errors)来创建错误。

#### ctx.assert(value, [status], [msg], [properties])

当!value时， Helper 方法抛出一个类似.throw()的错误。 类似node's [assert()](http://nodejs.org/api/assert.html) 方法。
```javascript
ctx.assert(ctx.state.user, 401, 'User not found. Please login!');
```
koa 使用 [http-assert](https://github.com/jshttp/http-assert) 来断言。

#### ctx.respond

为了避免使用 Koa 的内置响应处理功能，您可以直接赋值 this.repond = false;。如果您不想让 Koa 来帮助您处理 reponse，而是直接操作原生 res 对象，那么请使用这种方法。

注意： 这种方式是不被 Koa 支持的。其可能会破坏 Koa 中间件和 Koa 本身的一些功能。其只作为一种 hack 的方式，并只对那些想要在 Koa 方法和中间件中使用传统 fn(req, res) 方法的人来说会带来便利。


#### Request aliases

以下访问器和别名与 Request 等价：

- ctx.header
- ctx.headers
- ctx.method
- ctx.method=
- ctx.url
- ctx.url=
- ctx.originalUrl
- ctx.origin
- ctx.href
- ctx.path
- ctx.path=
- ctx.query
- ctx.query=
- ctx.querystring
- ctx.querystring=
- ctx.host
- ctx.hostname
- ctx.fresh
- ctx.stale
- ctx.socket
- ctx.protocol
- ctx.secure
- ctx.ip
- ctx.ips
- ctx.subdomains
- ctx.is()
- ctx.accepts()
- ctx.acceptsEncodings()
- ctx.acceptsCharsets()
- ctx.acceptsLanguages()
- ctx.get()

#### Response aliases

以下访问器和别名与 Response 等价：

- ctx.body
- ctx.body=
- ctx.status
- ctx.status=
- ctx.message
- ctx.message=
- ctx.length=
- ctx.length
- ctx.type=
- ctx.type
- ctx.headerSent
- ctx.redirect()
- ctx.attachment()
- ctx.set()
- ctx.append()
- ctx.remove()
- ctx.lastModified=
- ctx.etag=


## 请求(Request)

Koa Request 对象是对 node 的 request 进一步抽象和封装，提供了日常 HTTP 服务器开发中一些有用的功能。

### API

#### request.header

请求头对象


#### request.header=

设置请求头对象

#### request.headers

请求头对象。等价于 request.header.

#### request.headers=

设置请求头对象。 等价于request.header=.

#### request.method

请求方法

#### request.method=

设置请求方法, 在实现中间件时非常有用，比如 methodOverride()

#### request.length

以数字的形式返回 request 的内容长度(Content-Length)，或者返回 undefined。

#### request.url

获得请求url地址.

#### request.url=

设置请求地址，用于重写url地址时

#### request.originalUrl

获取请求原始地址

#### request.origin

获取URL原始地址, 包含 protocol 和 host
```javascript
ctx.request.origin
// => http://example.com
```
#### request.href

获取完整的请求URL, 包含 protocol, host 和 url
```javascript
ctx.request.href;
// => http://example.com/foo/bar?q=1
```
#### request.path

获取请求路径名

#### request.path=

设置请求路径名并保留当前查询字符串

#### request.querystring

获取查询参数字符串(url中?后面的部分)，不包含?

#### request.querystring=

设置原始查询字符串

#### request.search

获取查询参数字符串，包含?

#### request.search=

设置原始查询字符串

#### request.host

获取 host (hostname:port)。 当 app.proxy 设置为 true 时，支持 X-Forwarded-Host

#### request.hostname

获取 hostname。当 app.proxy 设置为 true 时，支持 X-Forwarded-Host。

如果主机是IPv6, Koa 将解析转换为 [WHATWG URL API](https://nodejs.org/dist/latest-v8.x/docs/api/url.html#url_the_whatwg_url_api), 注意 这可能会影响性能

#### request.URL

获取 WHATWG 解析的对象.

#### request.type

获取请求 Content-Type，不包含像 "charset" 这样的参数。
```javascript
const ct = ctx.request.type;
// => "image/png"
```
#### request.charset

获取请求 charset，没有则返回 undefined:
```javascript
ctx.request.charset;
// => "utf-8"
```
#### request.query

将查询参数字符串进行解析并以对象的形式返回，如果没有查询参数字字符串则返回一个空对象。注意：该方法不支持嵌套解析。

例如 "color=blue&size=small":
```javascript
{
  color: 'blue',
  size: 'small'
}
```
#### request.query=

根据给定的对象设置查询参数字符串。 注意：该方法不 支持嵌套对象。
```javascript
ctx.query = { next: '/login' };
```

#### request.fresh

检查请求缓存是否 "fresh"(内容没有发生变化)。该方法用于在 If-None-Match / ETag, If-Modified-Since 和 Last-Modified 中进行缓存协调。当在 response headers 中设置一个或多个上述参数后，该方法应该被使用。
```javascript
// freshness check requires status 20x or 304
ctx.status = 200;
ctx.set('ETag', '123');

// cache is ok
if (ctx.fresh) {
  ctx.status = 304;
  return;
}

// cache is stale
// fetch new data
ctx.body = await db.find('something');
```

#### request.stale

与 req.fresh 相反。

#### request.protocol

返回请求协议，"https" 或者 "http"。 当 app.proxy 设置为 true 时，支持 X-Forwarded-Host。

#### request.secure

简化版 this.protocol == "https"，用来检查请求是否通过 TLS 发送。

#### request.ip

请求远程地址。 当 app.proxy 设置为 true 时，支持 X-Forwarded-Host。

#### request.ips

当 X-Forwarded-For 存在并且 app.proxy 有效，将会返回一个有序（从 upstream 到 downstream）ip 数组。 否则返回一个空数组。

#### request.subdomains

以数组形式返回子域名。

子域名是在host中逗号分隔的主域名前面的部分。默认情况下，应用的域名假设为host中最后两部分。其可通过设置 app.subdomainOffset 进行更改。

举例来说，如果域名是 "tobi.ferrets.example.com":

如果没有设置 app.subdomainOffset，其 subdomains 为 ["ferrets", "tobi"]。 如果设置 app.subdomainOffset 为3，其 subdomains 为 ["tobi"]。

#### request.is(types...)

检查请求所包含的 "Content-Type" 是否为给定的 type 值。 如果没有 request body，返回 undefined。 如果没有 content type，或者匹配失败，返回 false。 否则返回匹配的 content-type。
```javascript
// With Content-Type: text/html; charset=utf-8
ctx.is('html'); // => 'html'
ctx.is('text/html'); // => 'text/html'
ctx.is('text/*', 'text/html'); // => 'text/html'

// When Content-Type is application/json
ctx.is('json', 'urlencoded'); // => 'json'
ctx.is('application/json'); // => 'application/json'
ctx.is('html', 'application/*'); // => 'application/json'

ctx.is('html'); // => false
```
比如说您希望保证只有图片发送给指定路由
```javascript
if (ctx.is('image/*')) {
  // process
} else {
  ctx.throw(415, 'images only!');
}
```
#### Content Negotiation

Koa request 对象包含 content negotiation 功能（由 [accepts](http://github.com/expressjs/accepts) 和 [negotiator](http://github.com/expressjs/accepts) 提供）：

- request.accepts(types)
- request.acceptsEncodings(types)
- request.acceptsCharsets(charsets)
- request.acceptsLanguages(langs)
如果没有提供 types，将会返回所有的可接受类型。

如果提供多种 types，将会返回最佳匹配类型。如果没有匹配上，则返回 false，您应该给客户端返回 406 "Not Acceptable"。

为了防止缺少 accept headers 而导致可以接受任意类型，将会返回第一种类型。因此，您提供的类型顺序非常重要。
#### request.accepts(types)

检查给定的类型 types(s) 是否可被接受，当为 true 时返回最佳匹配，否则返回 false。type 的值可以是一个或者多个 mime 类型字符串。 比如 "application/json" 扩展名为 "json"，或者数组 ["json", "html", "text/plain"]。
```javascript
// Accept: text/html
ctx.accepts('html');
// => "html"

// Accept: text/*, application/json
ctx.accepts('html');
// => "html"
ctx.accepts('text/html');
// => "text/html"
ctx.accepts('json', 'text');
// => "json"
ctx.accepts('application/json');
// => "application/json"

// Accept: text/*, application/json
ctx.accepts('image/png');
ctx.accepts('png');
// => false

// Accept: text/*;q=.5, application/json
ctx.accepts(['html', 'json']);
ctx.accepts('html', 'json');
// => "json"

// No Accept header
ctx.accepts('html', 'json');
// => "html"
ctx.accepts('json', 'html');
// => "json"
```
this.accepts() 可以被调用多次，或者使用 switch:
```javascript
switch (ctx.accepts('json', 'html', 'text')) {
  case 'json': break;
  case 'html': break;
  case 'text': break;
  default: ctx.throw(406, 'json, html, or text only');
}
```
#### request.acceptsEncodings(encodings)

检查 encodings 是否可以被接受，当为 true 时返回最佳匹配，否则返回 false。 注意：您应该在 encodings 中包含 identity。
```javascript
// Accept-Encoding: gzip
ctx.acceptsEncodings('gzip', 'deflate', 'identity');
// => "gzip"

ctx.acceptsEncodings(['gzip', 'deflate', 'identity']);
// => "gzip"
```
当没有传递参数时，返回包含所有可接受的 encodings 的数组：
```javascript
// Accept-Encoding: gzip, deflate
ctx.acceptsEncodings();
// => ["gzip", "deflate", "identity"]
```
注意：如果客户端直接发送 identity;q=0 时，identity encoding（表示no encoding） 可以不被接受。当这个方法返回false时，虽然这是一个边界情况，您仍然应该处理这种情况。

#### request.acceptsCharsets(charsets)

检查 charsets 是否可以被接受，如果为 true 则返回最佳匹配， 否则返回 false。
```javascript
// Accept-Charset: utf-8, iso-8859-1;q=0.2, utf-7;q=0.5
ctx.acceptsCharsets('utf-8', 'utf-7');
// => "utf-8"

ctx.acceptsCharsets(['utf-7', 'utf-8']);
// => "utf-8"
```
当没有传递参数时， 返回包含所有可接受的 charsets 的数组：
```javascript
// Accept-Charset: utf-8, iso-8859-1;q=0.2, utf-7;q=0.5
ctx.acceptsCharsets();
// => ["utf-8", "utf-7", "iso-8859-1"]
```
#### request.acceptsLanguages(langs)

检查 langs 是否可以被接受，如果为 true 则返回最佳匹配，否则返回 false。
```javascript
// Accept-Language: en;q=0.8, es, pt
ctx.acceptsLanguages('es', 'en');
// => "es"

ctx.acceptsLanguages(['en', 'es']);
// => "es"
```
当没有传递参数时，返回包含所有可接受的 langs 的数组：
```javascript
// Accept-Language: en;q=0.8, es, pt
ctx.acceptsLanguages();
// => ["es", "pt", "en"]
```
#### request.idempotent

检查请求是否为幂等(idempotent)

#### request.socket

返回请求的socket。

#### request.get(field)

返回请求头

## Response

Koa Response 对象是对 node 的 response 进一步抽象和封装，提供了日常 HTTP 服务器开发中一些有用的功能。
### API

#### response.header

Response header 对象。

#### response.headers

Response header 对象。等价于 response.header.

#### response.socket

Request socket.

#### response.status

获取响应状态。 默认情况下，response.status设置为404，而不像node's res.statusCode默认为200。

#### response.status=

通过数字设置响应状态:

- 100 "continue"
- 101 "switching protocols"
- 102 "processing"
- 200 "ok"
- 201 "created"
- 202 "accepted"
- 203 "non-authoritative information"
- 204 "no content"
- 205 "reset content"
- 206 "partial content"
- 207 "multi-status"
- 208 "already reported"
- 226 "im used"
- 300 "multiple choices"
- 301 "moved permanently"
- 302 "found"
- 303 "see other"
- 304 "not modified"
- 305 "use proxy"
- 307 "temporary redirect"
- 308 "permanent redirect"
- 400 "bad request"
- 401 "unauthorized"
- 402 "payment required"
- 403 "forbidden"
- 404 "not found"
- 405 "method not allowed"
- 406 "not acceptable"
- 407 "proxy authentication required"
- 408 "request timeout"
- 409 "conflict"
- 410 "gone"
- 411 "length required"
- 412 "precondition failed"
- 413 "payload too large"
- 414 "uri too long"
- 415 "unsupported media type"
- 416 "range not satisfiable"
- 417 "expectation failed"
- 418 "I'm a teapot"
- 422 "unprocessable entity"
- 423 "locked"
- 424 "failed dependency"
- 426 "upgrade required"
- 428 "precondition required"
- 429 "too many requests"
- 431 "request header fields too large"
- 500 "internal server error"
- 501 "not implemented"
- 502 "bad gateway"
- 503 "service unavailable"
- 504 "gateway timeout"
- 505 "http version not supported"
- 506 "variant also negotiates"
- 507 "insufficient storage"
- 508 "loop detected"
- 510 "not extended"
- 511 "network authentication required"
注意：不用担心记不住这些字符串，如果您设置错误，会有异常抛出，并列出该状态码表来帮助您进行更正。

#### response.message

获取响应状态消息。默认情况下, response.message关联response.status。

#### response.message=

将响应状态消息设置为给定值。

#### response.length=

将响应Content-Length设置为给定值。

#### response.length

如果 Content-Length 作为数值存在，或者可以通过 ctx.body 来进行计算，则返回相应数值，否则返回 undefined。

#### response.body

获取响应体。

#### response.body=

设置响应体为如下值:

- string written
- Buffer written
- Stream piped
- Object || Array json-stringified
- null no content response
如果 res.status 没有赋值，Koa会自动设置为 200 或 204。

#### String

Content-Type 默认为 text/html 或者 text/plain，两种默认 charset 均为 utf-8。 Content-Length 同时会被设置。

#### Buffer

Content-Type 默认为 application/octet-stream，Content-Length同时被设置。

#### Stream

Content-Type 默认为 application/octet-stream。

当stream被设置为响应体时， .onerror将作为监听器自动添加到错误事件中以捕获任何错误。此外，每当请求被关闭（甚至更早）时，stream都将被销毁。如果不想要这两个功能，请不要直接将stream设置为响应体。例如，当将响应体设置为代理中的HTTP stream时，会破坏底层连接。

请查阅: https://github.com/koajs/koa/pull/612 来获取更多信息。

以下是stream error处理的示例，并且不会自动销毁stream：
```javascript
const PassThrough = require('stream').PassThrough;

app.use(async ctx => {
  ctx.body = someHTTPStream.on('error', ctx.onerror).pipe(PassThrough());
});
```
#### Object

Content-Type默认为application/json。 这包括普通对象{ foo: 'bar' }和数组['foo', 'bar']。
#### response.get(field)

获取 response header 中字段值，field 不区分大小写。
```javascript
const etag = ctx.response.get('ETag');
```
#### response.set(field, value)

设置 response header 字段 field 的值为 value。
```javascript
ctx.set('Cache-Control', 'no-cache');
```

#### response.append(field, value)

添加额外的字段field 的值为 val
```javascript
ctx.append('Link', '<http://127.0.0.1/>');
```
#### response.set(fields)

使用对象同时设置 response header 中多个字段的值。
```javascript
ctx.set({
  'Etag': '1234',
  'Last-Modified': date
});
```
#### response.remove(field)

移除 response header 中字段 filed。

#### response.type

获取 response Content-Type，不包含像"charset"这样的参数。
```javascript
const ct = ctx.type;
// => "image/png"
```
#### response.type=

通过 mime 类型的字符串或者文件扩展名设置 response Content-Type
```javascript
ctx.type = 'text/plain; charset=utf-8';
ctx.type = 'image/png';
ctx.type = '.png';
ctx.type = 'png';
```
注意：当为你选择一个合适的charset时，例如response.type = 'html'将默认为"utf-8"。 如果需要覆盖charset，请使用ctx.set('Content-Type', 'text/html')直接设置响应头字段值。

#### response.is(types...)

跟ctx.request.is()非常类似。用来检查响应类型是否是所提供的类型之一。这对于创建操作响应的中间件特别有用。

例如，这是一个中间件，它可以缩小除stream以外的所有HTML响应。
```javascript
const minify = require('html-minifier');

app.use(async (ctx, next) => {
  await next();

  if (!ctx.response.is('html')) return;

  let body = ctx.body;
  if (!body || body.pipe) return;

  if (Buffer.isBuffer(body)) body = body.toString();
  ctx.body = minify(body);
});
```
#### response.redirect(url, [alt])

执行 [302] 重定向到对应 url。

字符串 "back" 是一个特殊参数，其提供了 Referrer 支持。当没有Referrer时，使用 alt 或者 / 代替。
```javascript
ctx.redirect('back');
ctx.redirect('back', '/index.html');
ctx.redirect('/login');
ctx.redirect('http://google.com');
```
如果想要修改默认的 [302] 状态，直接在重定向之前或者之后执行即可。如果要修改 body，需要在重定向之前执行。
```javascript
ctx.status = 301;
ctx.redirect('/cart');
ctx.body = 'Redirecting to shopping cart';
```
#### response.attachment([filename])

设置 "attachment" 的 Content-Disposition，用于给客户端发送信号来提示下载。filename 为可选参数，用于指定下载文件名。

#### response.headerSent

检查 response header 是否已经发送，用于在发生错误时检查客户端是否被通知。

#### response.lastModified

如果存在 Last-Modified，则以 Date 的形式返回。

#### response.lastModified=

以 UTC 格式设置 Last-Modified。您可以使用 Date 或 date 字符串来进行设置。
```javascript
ctx.response.lastModified = new Date();
```

#### response.etag=

设置 包含 "s 的 ETag。注意没有对应的 res.etag 来获取其值。
```javascript
ctx.response.etag = crypto.createHash('md5').update(ctx.body).digest('hex');
```
#### response.vary(field)

不同于field.

#### response.flushHeaders()

刷新任何设置的响应头，并开始响应体。

## 相关资源

以下列出了更多第三方提供的 koa 中间件、完整实例、全面的帮助文档等。如果有问题，请加入我们的 IRC！

- [GitHub repository](https://github.com/koajs/koa)
- [Examples](https://github.com/koajs/examples)
- [Middleware](https://github.com/koajs/koa/wiki#middleware)
- [Wiki](https://github.com/koajs/koa/wiki)
- [G+ Community](https://plus.google.com/communities/101845768320796750641)
- [Mailing list](https://groups.google.com/forum/#!forum/koajs)
- [Guide](https://github.com/koajs/koa/blob/master/docs/guide.md)
- [FAQ](https://github.com/koajs/koa/blob/master/docs/faq.md)
- #koajs on freenode
