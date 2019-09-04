# koajs/compose 源码解析

源码地址 [koajs/compose](https://github.com/koajs/compose)

`compose` 方法代码很少，但很关键，负责将 middlewares 构建成层级执行的结构。

- 判断参数，需要的类型是 `Array<Function>`
- 返回匿名函数，接收 `context`，`next` 参数，加上声明的 `index` 变量，这个闭包持有了3个变量。
  - `context: Any` 直接传给 middleware，没做任何操作
  - `next: Function?` 在最深一级中执行，相当于向 middlewares push
  - `index: Number` 记录当前执行到的最深的层级，避免重复执行，这个实现十分巧妙

  执行时，只执行了第一个 middleware，同时传入下一个 middleware 的 dispatch (触发方法)，由前一个 middleware 决定 下一个 middleware 的执行时机，所以形成了类似洋葱的结构。

  需要注意的是，middleware 中需要执行 `next`，否则不会执行下一层。
  `next` 不需要传参，`context` 一直保持在闭包中。

- 做了 `catch`，返回 `Promise.reject(err)`


```javascript
'use strict'

/**
 * Expose compositor.
 */

module.exports = compose

/**
 * Compose `middleware` returning
 * a fully valid middleware comprised
 * of all those which are passed.
 *
 * @param {Array} middleware
 * @return {Function}
 * @api public
 */

function compose (middleware) {
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```