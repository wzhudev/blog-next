---
title: swr 源码解析
date: 2019/11/17
description: 这篇文章简要分析 swr 的源码
tag: swr, vercel, source code, Chinese
author: Wendell
---

# swr 源码解析

不久前知名前端团队 ZEIT 在 [twitter](https://twitter.com/zeithq/status/1188911002626097157) 上公布了他们的 **data fetching** 工具库 [swr](https://swr.now.sh/) ，目前这个库已经在 [GitHub](https://github.com/zeit/swr) 上收获了 4,500+ star 。它有着一些非常亦可赛艇的功能，最主要的是实现了 [RFC 5861 草案](https://tools.ietf.org/html/rfc5861)，即**发起网络请求之前，利用本地缓存数据进行渲染，待请求的响应返回后再重新渲染并更新缓存**，从而提高用户体验。其他功能包括在窗口 focus 时更新、定时更新、支持任意网络请求库、支持 Suspense 等。

本文将会分析其源码以探究它是如何工作的。

## 代码结构

```plaintext
src
├── config.ts // 配置，同时也定义了一些重要的全局变量
├── index.ts
├── libs // 一些工具函数
│   ├── hash.ts
│   ├── is-document-visible.ts
│   ├── is-online.ts
│   ├── throttle.ts
│   └── use-hydration.ts
├── swr-config-context.ts // 实现 context 支持
├── types.ts // 类型定义
├── use-swr-pages.tsx // 分页
└── use-swr.ts // 主文件
```

## 主要流程

我们从官网举的最简单的例子开始：

```jsx
import useSWR from 'swr'

function Profile () {
  const { data, error } = useSWR('/api/user', fetch)

  if (error) return <div>failed to load</div>
  if (!data) return <div>loading...</div>
  return <div>hello {data.name}!</div>
}
```

我们所使用的 useSWR 函数定义[在这里](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L116-L130)。

[参数处理阶段](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L131-L164)。`useSWR` 有多种 function overload，但无论如何都需要传入一个符合 `keyInterface` 的 `key` 才行。swr 会对开发者传入的参数进行处理，最终得到以下四个重要变量：

* `key`，这是请求的唯一标识符
  * 有时候也包括请求函数的参数 fnArgs
* `keyErr`，对应请求标识符的错误标识符
* `config`，由默认配置，上下文配置和函数参数合并而成的配置对象
* `fn`，请求函数

然后，swr 会[准备一些变量](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L166-L183)：

* `initialData` 和 `initialError`，swr 会通过 [cacheGet](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/config.ts#L13) 方法从一个全局的 [__cache map](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/config.ts#L11) 中按照 key 存取缓存值，这其实就是 swr 请求缓存的实现原理，**也就是对 RFC 5861 的实现**，并没有任何黑魔法！
* `data` 和 `dispatch`，swr 是通过调用 `useReducer` 来生成的，它们记录了 `key` 所对应的请求的状态：请求的数据，请求的错误信息以及请求是否正在发送。
* `unmountedRef` `keyRef` `dataRef` `errorRef `等工具变量。

然后 swr 定义了一个非常重要的函数 [revalidate](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L186) ，这个函数内部定义了发起请求、处理响应和错误的主要过程。

> 注意生成这个函数的时候调用了 useCallback，依赖项为 key，即只有在 key 发生变化的时候才会重新生成 revalidate 函数。

我们先聚焦于主要流程，此时 `shouldDeduping === false`。

[首先会在 CONCURRENT_PROMISES 这个全局变量上缓存 fn 调用后的返回值（一个 Promise）](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L221-L225)，其实这里调用 `fn` 就已经发起了网络请求。[CONCURRENT_PROMISES](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/config.ts#L26) 这个变量是一个 Map，实际上建立了 `key` 和网络请求之间的映射，swr 利用这个 Map 来实现去重和超时报错等功能。

> 很明显能够看出 `fn` 必须返回一个 Promise，这个简单的约定也使得 swr 能够支持任意的网络请求库，不管是 REST 还是 GraphQL，只要返回 Promise 就行！

然后 revalidate 会等待网络请求完毕，获取到请求数据：

```js
newData = await CONCURRENT_PROMISES[key]
```

并触发 `onSuccess` 事件。

接着，[更新缓存](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L249-L250)，并通过 [dispatch 方法更新 state](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L273) ，此时就会触发 React 的重新渲染，重新渲染时就能[从 state 里拿到请求数据](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L508-L516)了。

以上就是 `revalidate` 函数的主要过程，那么这个函数是在什么时候被调用的呢？我们接着看 [useIsomorphicLayoutEffect 的回调函数](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L325-L456)。

> [useIsomorphicLayoutEffect 函数](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L45)在服务端就是 `useEffect` ，在浏览器端就是 `useLayoutEffect` 。

首先要判断本次调用时的 key 和上次调用的 key 是否相等。考虑下面这个组件：

```js
const Profile = (props) => {
  const { userData, userErr } = useSWR(() => `/${props.userId}`)
}
```

可以看到即使函数调用的位置相同（Hooks 的正确工作依赖各个 hook 的调用顺序），`key` 的值也可能不同，所以 swr 必须做这个检验。另外也要判断 `data` 是否相同，有可能别处更新了 `key` 所对应的缓存值。总之，当 swr 检查到 key 或者 data 不同，就会执行更新当前的 `key` 和 `data`，并调用 `dispatch` 进行重绘等操作。

然后，在 `revalidate` 的基础上定义了 `softRevalidate` 函数，在 `revalidate` 执行时执行去重逻辑。

```js
const softRevalidate = () => revalidate({ dedupe: true })
```

然后 swr 就会调用 [softRevalidate](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L352-L361)，如果当前有缓存值且浏览器支持 `requestIdleCallback` 的话，就作为 `requestIdleCallback` 的回调执行，避免打断 React 的渲染过程，否则就立即执行。

### 错误处理

如果数据请求的过程中发生了错误该怎么办呢？

注意到 `revalidate` 的函数，有很大一部分都在一个 try catch 块中，如果请求出错就会进入 catch 块。

主要做如下几件事情：

* [删除请求缓存](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L280)
* 更新 `state`，[更新错误内容](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L292-L295)
* [派发 onError 事件](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L304)
* 如果配置了重试的话，就[执行重试](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L305-L315)

[默认的重试方法](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/config.ts#L33)会在当前文档 visible 时执行重试，使用了一个指数退避策略，在一定的时间后重新调用 `revalidate` 。

## 请求去重 (dedupe)

> 我们在讲解主流程的过程中忽略了很多代码，而这些代码实现了 swr 的一些重要的实用特性，从这个小节开始我会一一讲解。

swr 提供了请求去重的功能，避免某个时间段内重复发起的请求过多。

实现的原理也非常简单。每次 `revalidate` 函数执行的时候，[都会判断是否需要去重](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L195-L196)：

```js
let shouldDeduping =
  typeof CONCURRENT_PROMISES[key] !== 'undefined' && revalidateOpts.dedupe
```

即检验 `CONCURRENT_PROMISES` 里有没有 `key` 所对应的进行中的请求。

如果 `shouldDeduping` 为 `true`，[直接等待请求完成](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L211)，如果为 `false`，就按照上文所述进行处理。

而 `revalidateOpts` 的 `dedupe` 属性何时为 `true` 呢？可以看到声明 `softRevalidate` 的时候传入了参数：

```js
const softRevalidate = () => revalidate({ dedupe: true })
```

而调用 `useSWR` 时返回的 `revalidate` 就是原本的 `revalidate` ，不带 `dedupe` 属性。

## 请求依赖 (dependent fetching)

还是举[官网的例子]( https://swr.now.sh/#dependent-fetching )：

```js
function MyProjects () {
  const { data: user } = useSWR('/api/user')
  const { data: projects } = useSWR(
    () => '/api/projects?uid=' + user.id
  )

  if (!projects) return 'loading...'
  return 'You have ' + projects.length + ' projects'
}
```

可见第二个请求依赖于第一个请求，调用 useSWR 时 `key` 为一个函数，函数体中访问 `user` 的 `id` 属性。

swr 通过 [getKeyArgs 函数](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L49)处理 key 为函数的情况，并将调用过程包裹在函数里：

```js
  if (typeof _key === 'function') {
    try {
      key = _key()
    } catch (err) {
      // dependencies not ready
      key = ''
    }
  }
```

当 `user` 为 `undefined` 时，获取 `undefined.id` 出错，`key` 为空字符串，而 [revalidate 函数在 key 为假值时直接返回](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L190)：

```js
if (!key) return false
```

因此在第一次渲染（`MyProjects` 函数第一次被调用）时，第二个请求实际上并未发出。而当第一个请求得到响应时，`dispatch` 会导致组件重绘（`MyProjects` 函数再次被调用），此时 `user` 不是 `undefined` ，第二个请求就能发出了。

所以 swr 所支持的“最大并行请求”的原理非常简单，就是判断能不能获得 `key`，如果不能获得 `key` 就用 try catch 语句捕获错误，不发出请求，等待其他请求得到响应后在下次重绘时再试。 

## 请求广播

当请求成功或失败时，都需要调用 [broadcastState](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L84-L91) 函数，这个函数本身非常简单，根据 `key` 从 `CACHE_REVALIDATORS` 中获取一组函数，挨个调用而已：

```js
const broadcastState: broadcastStateInterface = (key, data, error) => {
  const updaters = CACHE_REVALIDATORS[key]
  if (key && updaters) {
    for (let i = 0; i < updaters.length; ++i) {
      updaters[i](false, data, error)
    }
  }
}
```

这些 updater 是什么？追踪源码可以看到是 [onUpdate 函数](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L377-L407)，可以看到核心就在于下面几行代码：

```js
dispatch(newState)

if (shouldRevalidate) {
  return softRevalidate()
}
return false
```

即更新 `state` 触发重新渲染，并调用 `softRevalidate` 。

所以这个机制的目的是在一个 `useSWR` 发起的请求得到响应时，刷新所有使用相同 `key` 的 `useSWR` 。

## Mutate

[mutate](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L93-L110) 是 swr 暴露给用户操作本地缓存的方法，其他部分经过上面的介绍理解起来应该很容易了，关键是如下这行：

```js
MUTATION_TS[key] = Date.now() - 1
```

这其实是为了抛弃过时的请求用的。

revalidate 函数在执行的时候会[记录发起请求的时间](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L227)：

```js
CONCURRENT_PROMISES_TS[key] = startAt = Date.now()
```

而当请求得到响应时，会[判断 mutate 函数调用的时间和发起请求的时间的前后关系](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L244)：

```js
if (MUTATION_TS[key] && startAt <= MUTATION_TS[key]) {
  dispatch({ isValidating: false })
  return false
}
```

当发起请求的时间早于 mutate 调用的时间，说明请求已经过期，就抛弃掉这个请求不做任何后处理。

## 自动轮询 (refetch on interval)

只需要[设置一个 timeout 定时调用 softRevalidate 就可以了](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L416-L434)。

## Config Context

在 swr 执行的开始，准备 `config` 对象时[调用 useContext 获取 SWRConfigContext](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L154-L159) ：

```js
  config = Object.assign(
    {},
    defaultConfig,
    useContext(SWRConfigContext),
    config
  )
```

## Suspense

想要支持 Suspense 很容易，[仅需要把数据请求的 Promise 抛出就可以了](https://github.com/zeit/swr/blob/481177080a395b7f3f08e53dd6904bf84fad97ed/src/use-swr.ts#L486)：

```js
throw CONCURRENT_PROMISES[key]
```

但是和通常情况下不同：当抛出的 Promise 未 resolve 时，React 并不会渲染这部分组件，因此返回值里也无需判断 `keyRef.current` 是否和 `key` 相同。