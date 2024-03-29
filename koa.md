# Koa 源码解析

## 依赖

与许多框架一样，Koa 并非凭空写成的，也基于了一系列的第三方包。

所以整个解析的第一步，让我们简单看一下第三方包的依赖情况。

- accepts
- cache-content-type
- content-disposition
- content-type
- cookies
- debug
- delegates
- depd
- destroy
- error-inject
- escape-html
- fresh
- http-assert
- http-errors
- is-generator-function
- koa-compose
- koa-convert
- koa-is-json
- on-finished
- only
- parseurl
- statuses
- type-is
- vary

## Application

`lib/application` 是 Koa 的入口文件

festures:

- 初始化 context、request、response
- 基于 `http` 模块，创建 http 服务，监听事件
- 加载 middleware

methods：
- constructor
- listen
- toJSON
- inspect
- use
- callback
- handleRequest
- createContext
- onerror

functions:
- respond


**constructor**

```javascript
const Emitter = require('events');
const context = require('./context');
const request = require('./request');
const response = require('./response');

/**
 * Expose `Application` class.
 * Inherits from `Emitter.prototype`.
 */

class Application extends Emitter {

  /**
    *
    * @param {object} [options] Application options
    * @param {string} [options.env='development'] Environment
    * @param {string[]} [options.keys] Signed cookie keys
    * @param {boolean} [options.proxy] Trust proxy headers
    * @param {number} [options.subdomainOffset] Subdomain offset
    *
    */

    constructor(options) {
        super();
        options = options || {};
        this.proxy = options.proxy || false;
        this.subdomainOffset = options.subdomainOffset || 2;
        this.env = options.env || process.env.NODE_ENV || 'development';
        if (options.keys) this.keys = options.keys;
        this.middleware = [];
        this.context = Object.create(context);
        this.request = Object.create(request);
        this.response = Object.create(response);
        if (util.inspect.custom) {
            this[util.inspect.custom] = this.inspect;
        }
    }
}
```

Application 继承于 Emitter，方便后面使用事件机制实现观察者模式。

构造函数对传入参数 `options` 进行消费，同时初始化 context、request、response、middleware

`options` 参数解析

| Name | Default | Description |
|---|---|---|
| env | process.env.NODE_ENV \|\| 'development' | 运行时的 env |
| proxy | `false` | 是否信任代理的 headers |
| keys | `undefined` | cookie 生成使用的 keys |
| subdomainOffset | 2 | Subdomain offset |




**listen**

listen 方法很简单，创建了 http server，然后直接暴露 server 的 listen 方法。

这有一个很特别的做法，application 并不持有创建出来的 server。即使后续不是再使用到，一般做法也会加到上下文里。
在这里就是 `this.server = ...`

```javascript
/**
   * Shorthand for:
   *
   *    http.createServer(app.callback()).listen(...)
   *
   * @param {Mixed} ...
   * @return {Server}
   * @api public
   */

  listen(...args) {
    debug('listen');
    const server = http.createServer(this.callback());
    return server.listen(...args);
  }
```

**toJSON/inspect**

这两个方法用于类似，就放在一起看。

inspect 就是暴露给外部 debug 时转换为简单数据的方法，这里的做法是直接调用了 toJSON。

toJSON 则是将 application 中有效信息制作成简单数据，包括 this 及一些配置，这里使用了 `only` 包

```javascript
/**
   * Return JSON representation.
   * We only bother showing settings.
   *
   * @return {Object}
   * @api public
   */

  toJSON() {
    return only(this, [
      'subdomainOffset',
      'proxy',
      'env'
    ]);
  }

  /**
   * Inspect implementation.
   *
   * @return {Object}
   * @api public
   */

  inspect() {
    return this.toJSON();
  }
```

**use**

这是最常用的方法，将一个 function 作为 middleware 加入 application 的执行队列里。

这个方法很简单，直接将 `fn` push 到 middleware 数组。

```javascript
/**
   * Use the given middleware `fn`.
   *
   * Old-style middleware will be converted.
   *
   * @param {Function} fn
   * @return {Application} self
   * @api public
   */

  use(fn) {
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
    if (isGeneratorFunction(fn)) {
      deprecate('Support for generators will be removed in v3. ' +
                'See the documentation for examples of how to convert old middleware ' +
                'https://github.com/koajs/koa/blob/master/docs/migration.md');
      fn = convert(fn);
    }
    debug('use %s', fn._name || fn.name || '-');
    this.middleware.push(fn);
    return this;
  }
```

**callback**

创建 http server 时执行，生成 request handler，执行过程：
- 使用 compose 包，将 middleware array 打成嵌套执行结构，得到 `fn: (ctx: Context) => Promise`
- 通过 http server 运行时的 req，res 构造得到 context
- 调用原型方法 handleRequest，将 context，fn 传入

最后返回 handler。

*这里 compose 包是关键*

```javascript
/**
   * Return a request handler callback
   * for node's native http server.
   *
   * @return {Function}
   * @api public
   */

  callback() {
    const fn = compose(this.middleware);

    if (!this.listenerCount('error')) this.on('error', this.onerror);

    const handleRequest = (req, res) => {
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);
    };

    return handleRequest;
  }
```

**handleRequest**
根据传入的 context 及封装好的 middleware，真正处理 http 请求。
- 初始状态为 404。如果后续没有程序对其改变时，认为没匹配到对应资源(router/file)，即为404。
- 传入 context 执行 middleware，成功时执行独立方法 `respond`, 失败执行 `onerror`。
- `onerror` 直接使用 `context.onerror`，这个错误处理方法还用于监听请求完成事件。

*疑问：`onerror` 为什么要二次监听？会重复执行吗？*

*可能：`catch(onerror)` 处理逻辑上的错误，`onFinished(res, onerror)` 处理代码执行错误*

其中：

`respond` 将 `ctx.status` 及 `ctx.body` 写入到 http response 中。

`onFinished` 来自 [on-finished](https://github.com/jshttp/on-finished) 包，监听请求完成事件。

    Execute a callback when a HTTP request closes, finishes, or errors.
    Usage: onFinished(res, listener)

```javascript
  /**
   * Handle request in callback.
   *
   * @api private
   */

  handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;
    const onerror = err => ctx.onerror(err);
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }
```

**createContext**

创建 context，context 是连接整个请求执行过程的线。

context 主要包含：
- `req`：封装过的 request
- `res`：封装过的 response
- `app`：application 示例，即 `this`
- `originalUrl`：原始 Url
- `state`：一个对象，*疑问：可能用于存储数据*

其中 `ctx` `req` `res` 进行了相互引用。

创建 context，request，response 的方法很特别，使用 Object.create 复制一个对象的方法。

所以，所有方法都是对象中属性，不是原型中的方法。

*疑问：不知道这样做的目的。可能是为了避免不可预知的注入，特殊情况时，可放心修改属性方法，不用担心影响下次请求（修改原型会影响？）*

```javascript
/**
   * Initialize a new context.
   *
   * @api private
   */

  createContext(req, res) {
    const context = Object.create(this.context);
    const request = context.request = Object.create(this.request);
    const response = context.response = Object.create(this.response);
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    context.originalUrl = request.originalUrl = req.url;
    context.state = {};
    return context;
  }
```

**onerror**

application 错误的回调，不是 http 请求的错误。
- 不是错误时，抛出类型错误
- 404 / expose / silent 时不执行
- 正常情况，使用 `console.error` 输出错误信息

*疑问：执行时机不明*

```javascript
/**
   * Default error handler.
   *
   * @param {Error} err
   * @api private
   */

  onerror(err) {
    if (!(err instanceof Error)) throw new TypeError(util.format('non-error thrown: %j', err));

    if (404 == err.status || err.expose) return;
    if (this.silent) return;

    const msg = err.stack || err.toString();
    console.error();
    console.error(msg.replace(/^/gm, '  '));
    console.error();
  }
```

**respond**

将 context 接收到的数据，写入 response。

设置 `ctx.respond` 为 `false`，可以自己处理这个过程。

- `code` 为会忽略 `body` 时，清空 `body`
- `method` 为 `HEAD` 时，且 `body` 为 JSON 时，重新计算 `length`
- `body` 为 `null` 时，判断 http 版本，构成一个 `body` 
- 判断 `body` 类型，使用不同方法返回 response，类型有 buffer/string/steam/json

```javascript
/**
 * Response helper.
 */

function respond(ctx) {
  // allow bypassing koa
  if (false === ctx.respond) return;

  if (!ctx.writable) return;

  const res = ctx.res;
  let body = ctx.body;
  const code = ctx.status;

  // ignore body
  if (statuses.empty[code]) {
    // strip headers
    ctx.body = null;
    return res.end();
  }

  if ('HEAD' == ctx.method) {
    if (!res.headersSent && isJSON(body)) {
      ctx.length = Buffer.byteLength(JSON.stringify(body));
    }
    return res.end();
  }

  // status body
  if (null == body) {
    if (ctx.req.httpVersionMajor >= 2) {
      body = String(code);
    } else {
      body = ctx.message || String(code);
    }
    if (!res.headersSent) {
      ctx.type = 'text';
      ctx.length = Buffer.byteLength(body);
    }
    return res.end(body);
  }

  // responses
  if (Buffer.isBuffer(body)) return res.end(body);
  if ('string' == typeof body) return res.end(body);
  if (body instanceof Stream) return body.pipe(res);

  // body: json
  body = JSON.stringify(body);
  if (!res.headersSent) {
    ctx.length = Buffer.byteLength(body);
  }
  res.end(body);
}
```