---
title: Koa 生成器源码解析
date: 2019/10/16
description: 这篇文章简要分析 Koa middleware 的源码
tag: koa, source code, Chinese, Zone.js
author: Wendell
---

# Koa 生成器源码解析

简要分析了 Koa middleware 的生成器写法。

---

虽然 Koa 要在下一个 major 版本里移除对生成器 generator 的支持，但是看一看它对生成器的处理还是能够加深我们对生成器的理解的。

Koa 源码中和生成器有关的代码就以下几行，判断 `use` 方法添加的函数是否是生成器函数，是的话，将它转换成异步函。其中调用的两个函数都是由周边库提供的。

```js
if (isGeneratorFunction(fn)) {
  deprecate(
    'Support for generators will be removed in v3. ' +
      'See the documentation for examples of how to convert old middleware ' +
      'https://github.com/koajs/koa/blob/master/docs/migration.md'
  )
  fn = convert(fn)
}
```

## `isGeneratorFunction`

> 这些依赖都是很短小的单文件，不如全部粘贴过来。

判断函数是否是一个生成器函数。

```js
'use strict'

var toStr = Object.prototype.toString
var fnToStr = Function.prototype.toString
var isFnRegex = /^\s*(?:function)?\*/
// 这个似乎是用来检测当前执行环境有没有引入生成器函数，还要再看看
var hasToStringTag =
  typeof Symbol === 'function' && typeof Symbol.toStringTag === 'symbol'
var getProto = Object.getPrototypeOf
var getGeneratorFunc = function () {
  // eslint-disable-line consistent-return
  // 如果没有 hasToStringTag，直接返回 false 表示无法生成生成器函数
  if (!hasToStringTag) {
    return false
  }
  // 否则尝试利用 Function 生成一个生成器函数并返回
  try {
    return Function('return function*() {}')()
  } catch (e) {}
}
var generatorFunc = getGeneratorFunc()
// 如果没有返回生成器函数，返回一个空对象，这样最后的判定就会失败
// 如果返回了一个生成器函数，得到生成器函数的原型对象
var GeneratorFunction = generatorFunc ? getProto(generatorFunc) : {}

module.exports = function isGeneratorFunction(fn) {
  // 不是函数的话肯定也不是生成器函数
  if (typeof fn !== 'function') {
    return false
  }
  // 将这个函数转换成 string，然后查看函数字面中是否包含 function*，有则是一个生成器函数
  // 但是这个判断是很不严谨的，因为它强制要求写法为 function*
  // 而 function *boo 就没有办法识别了
  if (isFnRegex.test(fnToStr.call(fn))) {
    return true
  }
  // 如果上面的方法不行，就尝试利用 toString 的方法
  if (!hasToStringTag) {
    var str = toStr.call(fn)
    return str === '[object GeneratorFunction]'
  }
  // 最后的方法，通过原型对象判别
  return getProto(fn) === GeneratorFunction
}
```

## `convert`

并不是将生成器函数转换成异步函数，而是让它能融入到 Koa 2.0 的工作流程中。

```js
'use strict'

const co = require('co')
const compose = require('koa-compose')

module.exports = convert

function convert(mw) {
  if (typeof mw !== 'function') {
    throw new TypeError('middleware must be a function')
  }
  if (mw.constructor.name !== 'GeneratorFunction') {
    // assume it's Promise-based middleware
    return mw
  }
  // 真正核心的代码就这三行
  // 返回了一个符合 koa 中间件函数签名要求的函数，这个函数内部调用了 co
  const converted = function (ctx, next) {
    // co 函数和中间件在执行的时候，绑定上下文到 ctx，也就是 koa 的 context
    // mw.call 的时候，返回了一个迭代器，然后 co 去执行这个迭代器，最终返回一个 Promise
    // 到这里我们有必要知道 koa 要求如何写一个生成器
    return co.call(ctx, mw.call(ctx, createGenerator(next)))
  }
  converted._name = mw._name || mw.name
  return converted
}

// 这里的生成器返回了迭代器，当在用户的生成器函数中调用
function* createGenerator(next) {
  return yield next()
}

// 后面两个方法没有用到，省略
```

### Koa 对生成器的写法要求

在 Koa 1.x 版本中，中间件要求是生成器函数，写法如下：

```js
function* legacyMiddleWare(next) {
  yield next
}
```

可以看到, `createGenerator(next)` 返回的迭代器就是这里的 next.

### `co`

co 是一个迭代器的执行器，返回一个 Promise.

```js
/**
 * slice() reference.
 */

var slice = Array.prototype.slice

/**
 * Execute the generator function or a generator
 * and return a Promise.
 *
 * @param {Function} fn
 * @return {Promise}
 * @api public
 */

function co(gen) {
  var ctx = this
  var args = slice.call(arguments, 1)

  // we wrap everything in a Promise to avoid Promise chaining,
  // which leads to memory leak errors.
  // see https://github.com/tj/co/issues/180
  return new Promise(function (resolve, reject) {
    // 如果传入的是一个生成器，那么调用这个生成器以得到一个迭代器
    if (typeof gen === 'function') gen = gen.apply(ctx, args)
    // 如果不存在迭代器或者迭代器没有 next，那么直接返回一个 resolved 状态的 Promise
    if (!gen || typeof gen.next !== 'function') return resolve(gen)
    // 执行这个迭代器
    onFulfilled()

    /**
     * 对迭代器进行一次 next 调用
     *
     * @param {Mixed} res
     * @return {Promise}
     * @api private
     */

    function onFulfilled(res) {
      var ret
      try {
        // 利用上次得到的结果对迭代器进行 next 调用，得到 yield 出的返回值
        ret = gen.next(res)
      } catch (e) {
        // 有错直接返回出一个拒绝态的 Promise
        return reject(e)
      }
      // 如果没出错就通过 next 进行处理
      next(ret)
    }

    /**
     * @param {Error} err
     * @return {Promise}
     * @api private
     */

    function onRejected(err) {
      var ret
      try {
        // 如果出错的话会调用迭代器的 throw 尝试解决错误
        ret = gen.throw(err)
      } catch (e) {
        return reject(e)
      }
      next(ret)
    }

    /**
     * Get the next value in the generator,
     * return a Promise.
     *
     * 得到迭代器的下一个值，并返回一个 Promise
     *
     * @param {Object} ret
     * @return {Promise}
     * @api private
     */

    function next(ret) {
      // 如果迭代已执行完成，返回一个 resolved 状态的 Promise, resolved undefined
      if (ret.done) return resolve(ret.value)
      // 否则将 value 包装成一个 Promise
      var value = toPromise.call(ctx, ret.value)
      // 如果是包装了 truthy 值的 Promise，那么通过 then 来后处理
      // 这里的 value 实际是 createGenerator 返回的迭代器封装好的 Promise
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected)
      // 如果不能封装为 Promise 则抛出错误
      return onRejected(
        new TypeError(
          'You may only yield a function, Promise, generator, array, or object, ' +
            'but the following object was passed: "' +
            String(ret.value) +
            '"'
        )
      )
    }
  })
}

/**
 * Convert a `yield`ed value into a Promise.
 *
 * 针对 yield 的 value 可能具有的不同情形来封装成 Promise
 *
 * @param {Mixed} obj
 * @return {Promise}
 * @api private
 */

function toPromise(obj) {
  // 如果为 falsy 直接返回
  if (!obj) return obj
  // 如果是 Promise 直接返回
  if (isPromise(obj)) return obj
  // 如果是迭代器或者是生成器就用 co 再执行
  // 实际上 koa 走的是这个分支，它会再用 co 执行这个迭代器，返回 Promise
  // 迭代器在执行的时候，就往 koa middleware 的下游走
  if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj)
  // 如果是一个 function 那么就通过 thunkToPromise 封装，不展开了
  if ('function' == typeof obj) return thunkToPromise.call(this, obj)
  if (Array.isArray(obj)) return arrayToPromise.call(this, obj)
  if (isObject(obj)) return objectToPromise.call(this, obj)
  return obj
}

/**
 * Convert a thunk to a Promise.
 *
 * 其他的辅助方法从略，只看看这个。
 *
 * @param {Function}
 * @return {Promise}
 * @api private
 */

function thunkToPromise(fn) {
  var ctx = this
  return new Promise(function (resolve, reject) {
    fn.call(ctx, function (err, res) {
      if (err) return reject(err)
      if (arguments.length > 2) res = slice.call(arguments, 1)
      resolve(res)
    })
  })
}
```

### `convert` 的执行过程

1. 当 Koa 要运行生成器函数转换成的中间件的时候，即调用 `return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));` 时，执行 `return co.call(ctx, mw.call(ctx, createGenerator(next)))`，它返回一个 Promise
2. 其中，用户提供的生成器 `mw` 被调用，同时调用 `createGenerator(next)` 返回一个迭代器
3. co 调用自己的 `onFulfilled` 方法执行用户的迭代。用户会写 `yield next` 这一句，将控制权交还给 co, co 调用 `next` 方。此时，由于 `ret.value` 是 `createGenerator(next)` 返回的迭代器，所以 `next` 方法进入 `if (value && isPromise(value)) return value.then(onFulfilled, onRejected);` 的分支
4. `value` 被封装成一个 Promise，其实内部又用了一次 co 对 `return yield next` 进行执行
5. `return yield next` 被执行，进入下游 middleware 并最终回溯到当前的 middleware
6. co 第二次执行 `onFulfilled`，然后调用 `next` 方法，此时 `ret.done` 为真，返回一个解决态的 Promise
7. 这里就回到了 `return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));`，继续往上游回溯

到这里，我们就梳理清楚了 Koa 1.x 时代所采用的生成器函数是如何被 Koa 2.x 所采用的异步函数兼容的。
