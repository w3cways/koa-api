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

### ctx.req

Node 的 request 对象。

### ctx.res

Node 的 response 对象。

Koa 不支持 直接调用底层 res 进行响应处理。请避免使用以下 node 属性：

- res.statusCode
- res.writeHead()
- res.write()
- res.end()

### ctx.request

Koa 的 Request 对象。

### ctx.response

Koa 的 Response 对象。

### ctx.state

推荐的命名空间，用于通过中间件传递信息到前端视图
```javascript
ctx.state.user = await User.find(id);
```

### ctx.app

应用实例引用。

### ctx.cookies.get(name, [options])

获得 cookie 中名为 name 的值，options 为可选参数：

- signed 如果为 true，表示请求时 cookie 需要进行签名。
注意：Koa 使用了 Express 的 [cookies](https://github.com/jed/cookies) 模块，options 参数只是简单地直接进行传递。

### ctx.cookies.set(name, value, [options])

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

### ctx.throw([status], [msg], [properties])

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

### ctx.assert(value, [status], [msg], [properties])

当!value时， Helper 方法抛出一个类似.throw()的错误。 类似node's [assert()](http://nodejs.org/api/assert.html) 方法。
```javascript
ctx.assert(ctx.state.user, 401, 'User not found. Please login!');
```
koa 使用 [http-assert](https://github.com/jshttp/http-assert) 来断言。

### ctx.respond

为了避免使用 Koa 的内置响应处理功能，您可以直接赋值 this.repond = false;。如果您不想让 Koa 来帮助您处理 reponse，而是直接操作原生 res 对象，那么请使用这种方法。

注意： 这种方式是不被 Koa 支持的。其可能会破坏 Koa 中间件和 Koa 本身的一些功能。其只作为一种 hack 的方式，并只对那些想要在 Koa 方法和中间件中使用传统 fn(req, res) 方法的人来说会带来便利。


### Request aliases

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

### Response aliases

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
























