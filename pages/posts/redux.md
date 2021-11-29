---
title: Redux 源码解析
date: 2019/10/16
description: 对 Redux 运行原理和源代码进行简单分析
tag: redux, react, source code, Chinese
author: Wendell
---

# Redux 源码解析

## createStore

有 middleware 的情形后面再讨论，这里先讨论最简单的情形。

首先设置了[初始状态](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/createStore.js#L61)，还有其他一些属性，通过闭包封装。

```js
let currentReducer = reducer
let currentState = preloadedState
let currentListeners = []
let nextListeners = currentListeners
let isDispatching = false
```

另外这个作用域内声明了这些函数：

- [getState](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/createStore.js#L79-L94)：只要当前没有 `reducer` 在执行就返回当前状态树。
  - 在 `dispatch` 函数调用的时候会设置标志位 `isDispatching` 为 `true`
- [dispatch](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/createStore.js#L184-L217)：触发这个方法是改变状态的唯一途径。
  - 通过 `isPlainObject` 方法确保传入的 action 是一个 PlainObject（原理是判断 `obj` 的原型链上只有 `Object`），其他类型的 action 都交给中间件处理。
  - [在调用 reducer 的过程中不允许派发新的事件](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/createStore.js#L199-L201)。
  - 调用 `currentReducer` 来变更状态，变更后的状态会被赋值给 `currentState` 属性。
  - 通知所有监听者。
- [subscribe](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/createStore.js#L96-L157)：注册状态变化监听者，返回一个取消监听的函数。
- [replaceReducer](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/createStore.js#L219-L241)：用于在运行时切换 reducer。
- [observable](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/createStore.js#L243-L280)：这个用来实现和 RxJS 类似的 subscribe API，这里就不赘述了，总之它会在状态变更之后 next 最新的状态。

最终被封装在[对象](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/createStore.js#L287-L293)上返回，就是我们通常说的 `Store` 对象。

```js
return {
  dispatch,
  subscribe,
  getState,
  replaceReducer,
  [$$observable]: observable
}
```

## Middleware

如果要使用中间件的话，在 `createStore` 的过程中，就会调用[这一行代码](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/createStore.js#L53)：

```js
// [A]
return enhancer(createStore)(reducer, preloadedState)
```

这里的 enhancer 实际上是 `applyMiddleware` 函数的返回值，接下来就看一看这个函数。

```js
export default function applyMiddleware(...middlewares) {
  return /* B */ (createStore) =>
    /* C */ (...args) => {
      const store = createStore(...args)
      let dispatch = () => {
        throw new Error(
          'Dispatching while constructing your middleware is not allowed. ' +
            'Other middleware would not be applied to this dispatch.'
        )
      }

      const middlewareAPI = {
        getState: store.getState,
        dispatch: (...args) => dispatch(...args)
      }
      const chain = middlewares.map((middleware) => middleware(middlewareAPI))
      dispatch = compose(...chain)(store.dispatch)

      return {
        ...store,
        dispatch
      }
    }
}
```

[redux-thunk 也算是官配了](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/applyMiddleware.js#L8)，而且它还非常简短，这里我们就拿[它](https://github.com/reduxjs/redux-thunk/blob/master/src/index.js)来举例子方便大家更好地理解：

```js
function createThunkMiddleware(extraArgument) {
  return
  // [1]
  ;({ dispatch, getState }) =>
    /* [2] */ (next) =>
    /* [3] */ (action) => {
      if (typeof action === 'function') {
        return action(dispatch, getState, extraArgument)
      }

      return next(action)
    }
}

const thunk = createThunkMiddleware()
thunk.withExtraArgument = createThunkMiddleware

export default thunk
```

使用这个 middleware 的时候如下所示：

```js
createStore(reducer, applyMiddleware(thunk))
```

我们现在开始仔细地分析调用过程：

1. `applyMiddleware` 函数接收多个 middleware，然后返回了一个函数 `B`
2. `A` 中第一次调用的时候，`createStore` 方法被传递给函数 `B`，再次返回了一个函数 `C`，`C` 再被 `[A]` 中的第二次调用所调用
3. `createStore` 方法创建了一个真正的 `Store` 对象。
4. [这一行代码](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/applyMiddleware.js#L33)注册这些 middleware（当然在这里我们只有 thunk 这一个中间件），可以看到 `getState` 和 `dispatch` 这两个方法就被传给了函数 `1`，返回的是函数 `2`
5. 这个函数 `2` 被 `compose` 函数所用，生成了一个新的 `dispatch` 方法 `3`，注意，这个方法是用户调用的 `dispatch`（**它才是真身**！）
6. 最后的返回看起来像是个 `Store` 对象，其实是个“套壳”的 `Store`

我们先讲解 `compose` 方法，然后再来讲解用户调用 `dispatch` 时会发生什么。

### compose

[`compose` 函数](https://github.com/wendzhue/redux/blob/master/src/compose.js)负责将 dispatch 过程串起来。通过上面的分析我们已经知道了被送给 `compose` 方法的是形如这样的一组函数：

```js
;(next) => (action) => {
  // ... 中间件对 action 进行处理
}
```

而 `compose` 的全部代码如下，它对 middleware 的数目分为三种情况来处理：

```js
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return (arg) => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce(
    (a, b) =>
      (...args) =>
        a(b(...args))
  )
}
```

我们先来考虑没有函数的情形，这种情况下返回一个透传函数 `arg ⇒ arg`，即不对原始的 dispatch 做任何改变：

```js
dispatch = compose(...chain)(store.dispatch) // dispatch === store.dispatch
```

接下来考虑有一个中间件的情形，这种情况下直接返回中间件：

```js
;(next = store.dispatch) =>
  (action) => {
    // next 就是 store.dispatch
    // ... 中间件对 action 进行处理
    // 如果要交给原始的 dispatch 就调用 next
  }
```

下面我们来考虑最复杂的情形，即有多个中间件，这里 `compose` 将它们用 `reduce` 串接起来：

```js
return funcs.reduce(
  (a, b) =>
    (...args) =>
      a(b(...args))
)
```

这样的写法可能具有误导性（让读者误以为 `a` 和 `b` 都是中间件），我们将形参改个名字理解起来会更容易：

```js
return funcs.reduce(
  (alreadyChained, nextMiddleware) =>
    (...args) =>
      alreadyChained(nextMiddleware(...args))
)
```

可以看到最终形成的是一种链式调用，如果我们这样调用 `compose` 方法：

```js
compose(a, b, c)
```

最终就会得到：

```js
;(...args) => a(b(c(...args)))
```

这样用户在调用 `dispatch` 的时候，中间件只要调用 `next` 就能将 action 抛给下一个中间件处理。

### 用户调用 dispatch 时会发生什么

现在我们可以来探究用户调用 `dispatch` 时会发生什么了。我们已经知道了用户实际调用的是 `compose` 函数的返回值，所以实际上执行的是函数 `[4]`，而我们知道在用 redux-thunk 的时候，action 里面会调用 disaptch，而这个 dispatch 就是 `[4]` 本身！只不过 `[4]` 第二次被调用的时候，走的是 `next(action)`，而我们在上面分析过，这里 `next === store.dispatch`，这样就会到真正 `Store` 对象的 `dispatch` 方法啦。

下面这张图能够帮助你理解变量之间的关系：

![call](https://user-images.githubusercontent.com/12122021/66885678-9bc00200-f007-11e9-97e5-91324d037262.png)

## combineReducers

`combineReduces` 也是个十分重要的函数，随着应用规模的扩大，你会希望不同的 reducer 能够处理状态树的局部而非一个 reducer 管理整个状态树。这个函数的[代码在这里](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/combineReducers.js#L97-L180)，其名十分贴切，工作原理就像 mapReduce。

首先这个方法会校验参数的正确性，然后返回一个名为 `combination` 的大 reducer。

`combination` 一开始仍然是校验参数的正确性，真正执行 mapReduce 过程的只有[这几行](https://github.com/wendzhue/redux/blob/9c9a4d2a1c62c9dbddcbb05488f8bd77d24c81de/src/combineReducers.js#L162-L179)。可以看到它会分把先前状态的一部分摘取下来放到名为 `previousStateForKey` 的变量里，然后通过对应的 reducer 来产生局部的新状态，赋值到新的整体状态上，然后，判断局部的状态是否发生变化，从而判断整体的状态是否发生变化。

```js
let hasChanged = false
const nextState = {}
for (let i = 0; i < finalReducerKeys.length; i++) {
  const key = finalReducerKeys[i]
  const reducer = finalReducers[key]
  const previousStateForKey = state[key]
  const nextStateForKey = reducer(previousStateForKey, action)
  nextState[key] = nextStateForKey
  hasChanged = hasChanged || nextStateForKey !== previousStateForKey
}
hasChanged = hasChanged || finalReducerKeys.length !== Object.keys(state).length
return hasChanged ? nextState : state
```

## 总结

以上就是对 redux 源码的解读了。可以看到除了串联 middleware 的部分，都非常清晰易懂。对于 Redux 这样的库来说，其思想远比其实现精妙的多。

另外还有一个函数 `bindActionCreators` 这里就不介绍了，感兴趣的话可以自行阅读[其源码](https://github.com/wendzhue/redux/blob/9c9a4d2a1c/src/bindActionCreators.js)。
