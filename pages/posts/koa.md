---
title: Koa 源码解析
date: 2019/10/16
description: 这篇文章简要分析 Koa 的源码
tag: koa, source code, Chinese
author: Wenzhao
---

# Koa 源码解析

这篇文章介绍一个应用服务器框架的主要两个过程：app init 过程和 request handle 过程。一些有趣的细节问题看看以后再写，包括 context, request, response 三个对象，错误处理，egg.js 等等。

---

## init 过程

通过一个简单的 demo（实际上就是官网的例子）来讲解 app init 过程。对于 Koa 来说，init 过程是比较简单的。

```js
const Koa = require('koa')
const app = new Koa() // Koa 对象实例化

// use 增加 middleware
app.use(async (ctx, next) => {
  await next()
  const rt = ctx.response.get('X-Response-Time')
  console.log(`${ctx.method} ${ctx.url} - ${rt}`)
})

app.use(async (ctx, next) => {
  const start = Date.now()
  await next()
  const ms = Date.now() - start
  ctx.set('X-Response-Time', `${ms}ms`)
})

app.use(async (ctx) => {
  ctx.body = 'Hello World'
})

app.listen(3000) // 监听端口
```

### Koa 对象实例化

`lib/application.js`

```js
  constructor() {
    super();

    this.proxy = false;
    this.middleware = [];
    this.subdomainOffset = 2;
    this.env = process.env.NODE_ENV || 'development';

    // 在应用程序实例上绑定 context request repsonse 的原型，实际上这三个对象都没有任何属性
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);

    if (util.inspect.custom) {
      this[util.inspect.custom] = this.inspect;
    }
  }
```

### use 增加 middleware

```js
  use(fn) {
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');

    // 如果是一个生成器函数要转换成 async 函数，细节问题
    if (isGeneratorFunction(fn)) {
      deprecate('Support for generators will be removed in v3. ' +
                'See the documentation for examples of how to convert old middleware ' +
                'https://github.com/koajs/koa/blob/master/docs/migration.md');
      fn = convert(fn);
    }
    debug('use %s', fn._name || fn.name || '-');

    // 直接将回调函数存储到 this.middleware 当中
    this.middleware.push(fn);
    return this; // 通过返回自己可以进行链式调用
  }
```

### 监听端口

```js
  listen(...args) {
    debug('listen');
    const server = http.createServer(this.callback());
    return server.listen(...args); // 调用 Node.js 原生的方法监听端口
  }
```

`this.callback()` 方法返回一个回调函数，它符合 Node.js 原生 http.createServer 的要求，被当作 request handler.

```js
  callback() {
    // 将自己绑定的中间件封装起来
    const fn = compose(this.middleware);

    // 进行错误处理的回调函数
    // 由于 Koa 继承了 Emitter, 所以用户可以在上面绑定 error 方法，如果用户没用绑定，就绑定自带的 onerror 方法
    // 错误处理暂时不讲
    if (!this.listenerCount('error')) this.on('error', this.onerror);

    // http server 的回调函数
    const handleRequest = (req, res) => {
      // 将 request response 对象封装为 context 对象，然后开始对 request 的处理过程，这个放到第二节再讲
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn); // 返回对 request 的处理结果
    };

    return handleRequest;
  }
```

#### compose

这是个很重要的方法，其返回的 `fn`, 将会在请求到达的时候实际负责 context 在 middleware 中的传递。

```js
function compose(middleware) {
  // 类型检查
  if (!Array.isArray(middleware))
    throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function')
      throw new TypeError('Middleware must be composed of functions!')
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  // 这个函数签名就是 koa middleware 常见的函数签名
  // 它作为 this.handleRequest 的参数，this.handleRequest 会调用它
  // 注意! 下面的代码及注释请在阅读 request handler 的过程阅读
  return function (context, next) {
    // last called middleware #
    // 指示 context 在 middleware 链上的位置
    // context 刚来的时候没有进入链，所以 index === -1
    let index = -1

    // 从第 1 个 middleware 开始 context 之旅，index === 0
    return dispatch(0)

    // 这个 dispatch 串接 context 在 middleware 中的流动
    function dispatch(i) {
      // 如果 i 到了起点之前，说明 next 被用了太多次
      if (i <= index)
        return Promise.reject(new Error('next() called multiple times'))
      index = i // 压栈阶段，记录自己所在的位置
      let fn = middleware[i]
      if (i === middleware.length) fn = next // 如果走到了 middleware 的最后一站，那么就用传入的 next 当作 next
      if (!fn) return Promise.resolve() // 如果都没有 middleware, 直接返回，然后层层 resolve 返回
      try {
        // 进入 middleware 函数的执行过程，middleware 中访问的 next 被定义在这里，
        // 而当这个 middleware 调用 next 的时候，就等于调用 dispatch, 同时进入 middleware 的下一层
        // 如果当前 middleware 是最后一个，上面的 if (i === middleware.length) fn = next 逻辑就会被激活，顶层调用 next
        // 可以看到我们 await 的东西就是一个 resolved 的 Promise!
        // 根据 async 函数的定义，默认返回的就是一个 resolved 的 Promise<undefined>
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)))
      } catch (err) {
        // 如果抛出了异常，就会被捕获，层层 reject 回来
        return Promise.reject(err)
      }
    }
  }
}
```

## request handle 过程

还是用上面的例子来讲解 request handle 过程。

当有 http 请求过来的时候，如下的方法最先被调用：

```js
const handleRequest = (req, res) => {
  const ctx = this.createContext(req, res) // 创建 context
  return this.handleRequest(ctx, fn) // 过程处理 === context 在 middleware 中的传递
}
```

### 创建 context

```js
  createContext(req, res) {
    // 创建三个对象，将它们的 prototype 分别指向 this.context, this.request, this.repsonse, 实际上这三个对象 hasOwnProperties 为空
    // 然后就是各种引用，比较简单
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

### 处理过程

对 request 的实际处理过程。

```js
  handleRequest(ctx, fnMiddleware) {
    // 这里的 fnMiddleware 即是 compose() 返回的 fn 的函数，可以看到并没有给第二参数传递值，所以在那里 next === undefined, 直接 resolve
    const res = ctx.res;
    res.statusCode = 404;

    // 准备两个 Promise 的回调函数
    const onerror = err => ctx.onerror(err);
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);

    // 开始 middleware 的传递过程
    // 如果执行过程成功，并没有从 Promise 里拿任何的参数，是利用闭包访问的 ctx 来生成响应的
    // 但执行失败则要从 Promise 链条里拿到错误信息
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }
```

#### context 在 middleware 中间的传递

当 `fnMiddleware` 被调用的时候，即这个函数被调用：

```js
function (context, next) {
  // last called middleware #
  // 指示 context 在 middleware 链上的位置
  // context 刚来的时候没有进入链，所以 index === -1
  let index = -1

  // 从第 1 个 middleware 开始 context 之旅，index === 0
  return dispatch(0)

  // 这个 dispatch 串接 context 在 middleware 中的流动
  // 注意! 下面的代码及注释在阅读 request handler 的过程阅读
  function dispatch (i) {
    // 如果 i 到了起点之前，说明 next 被用了太多次
    if (i <= index) return Promise.reject(new Error('next() called multiple times'))
    index = i // 压栈阶段，记录自己所在的位置
    let fn = middleware[i]
    if (i === middleware.length) fn = next // 如果走到了 middleware 的最后一站，那么就用传入的 next 当作 next
    if (!fn) return Promise.resolve() // 如果都没有 middleware, 直接返回，然后层层 resolve 返回
    try {
      // 进入 middleware 函数的执行过程，middleware 中访问的 next 被定义在这里，
      // 而当这个 middleware 调用 next 的时候，就等于调用 dispatch, 同时进入 middleware 的下一层
      // 如果当前 middleware 是最后一个，上面的 if (i === middleware.length) fn = next 逻辑就会被激活，顶层调用 next
      // 可以看到我们 await 的东西就是一个 resolved 的 Promise!
      // 根据 async 函数的定义，默认返回的就是一个 resolved 的 Promise<undefined>
      return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
    } catch (err) {
      // 如果抛出了异常，就会被捕获，层层 reject 回来
      return Promise.reject(err)
    }
  }
}
```

可以看到这个函数是递归的：

用我们的例子：

1. 执行 `dispatch(0)`, 我们注册的第一个异步函数被当成 `fn`, 然后 `fn(context, dispatch.bind(null, 1))` 调用了这第一个异步函数
2. 第一个异步函数执行 `next()`, 实际上执行了 `dispatch(1)`, 然后调用了第二个异步函数。
3. 同理，调用了第三个异步函数，middleware 到这里已经全部执行过了
4. 第三个函数执行的时候没用再调用 `next()`, 所以异步函数返回了状态为 resolved 的 `Promise<undefined>`
5. `return Promise.resolve()` 把异步函数返回的 Promise 接着 resolved 下去
6. 直到 `dispatch(0)` 中的 resolved 的 Promise 被 return 出去
7. `handleRequest` 进入 `fnMiddleware(ctx).then(handleResponse)`, 执行 `handleResponse`

##### 例外情形

###### 如果最后一个中间件也调用了 `next`

此时 `fn === undefined`, 并且 `next === undefined`, 所以就会直接返回已 resolved 的 Promise, 开始回溯。

###### 如果有一个中间件调用了两次 `next`

我们已经知道每次调用 `next` 实际是调用了一次 `dispatch(i)`, 如果我们调用了同一个 `next` 两次，那么第二次调用的时候，`i === index` 的条件就会成立。我们说过 `index` 是指示 context 在 middleware 中的位置的。

#### 创建响应

```js
function respond(ctx) {
  // allow bypassing koa
  // 允许 bypass 直通 koa
  if (false === ctx.respond) return

  const res = ctx.res
  if (!ctx.writable) return

  let body = ctx.body
  const code = ctx.status

  // ignore body
  if (statuses.empty[code]) {
    // strip headers
    ctx.body = null
    return res.end()
  }

  if ('HEAD' == ctx.method) {
    if (!res.headersSent && isJSON(body)) {
      ctx.length = Buffer.byteLength(JSON.stringify(body))
    }
    return res.end()
  }

  // status body
  if (null == body) {
    body = ctx.message || String(code)
    if (!res.headersSent) {
      ctx.type = 'text'
      ctx.length = Buffer.byteLength(body)
    }
    return res.end(body)
  }

  // responses
  if (Buffer.isBuffer(body)) return res.end(body)
  if ('string' == typeof body) return res.end(body)
  if (body instanceof Stream) return body.pipe(res)

  // body: json
  body = JSON.stringify(body)
  if (!res.headersSent) {
    ctx.length = Buffer.byteLength(body)
  }
  res.end(body)
}
```

这个方法和 Koa 的关系不大了，其实就是在处理 response 的各种可能情况，然后调用 http 模块 res 的方法返回响应。
