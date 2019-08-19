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

构造函数对传入参数 `options` 进行消费，同时初始化 context、request、response

`options` 参数解析

| Name | Default | Description |
|---|---|---|
| env | process.env.NODE_ENV \|\| 'development' | 运行时的 env |
| proxy | `false` | 是否信任代理的 headers |
| keys | `undefined` | cookie 生成使用的 keys |
| subdomainOffset | 2 | Subdomain offset |