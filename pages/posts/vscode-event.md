---
title: vscode 源码解析 - 事件模块
date: 2021/03/17
description: 介绍 vscode 事件模块
tag: vscode, source code, Chinese
author: Wendell
---

# vscode 源码解析 - 事件模块

![](https://user-images.githubusercontent.com/12122021/70677481-3c5e3580-1cca-11ea-9cc7-de8e131d3cb4.png)

---

在进一步深入学习 vscode 的各种机制之前，我们先对 vscode 当中的一些基础工具做一些探索，因为核心机制大量地用到了这些基础模块，这篇文章将会介绍事件（event）模块，相关代码在 vs/base/common/event.ts 文件中。

## Event 模块实现

### Event 接口

Event 接口规定了一个函数，当调用了这个函数，就表示监听了这个函数所对应的事件流。

- `listener` 参数是事件派发时将会被调用的回调函数，参数 `e` 为单个事件，换句话说， `listener` 就是事件的消费者
- `thisArgs` 参数是回调函数中 `this` 所指向的对象
- `disposables`

```tsx
export interface Event<T> {
  (
    listener: (e: T) => any,
    thisArgs?: any,
    disposables?: IDisposable[] | DisposableStore
  ): IDisposable
}
```

返回的 `IDisposable` 对象用于解除这个监听的（通过调用它的 `dispose` 方法）。

另外一种解除监听的方式就是 `disposable` 了，Event 函数在执行的过程中会将 `IDisposable` 插入 `disposables`，方便调用方决定在什么时候解除监听。

### Emitter

有了 `Event` 接口，我们可以规定事件如何被消费，那么事件是如何产生的呢？一种方法就是通过 `Emitter` 。

`Emitter` 类型暴露了两个重要方法：

`fire`，从这个方法的函数签名就能看出它就是用来派发一个事件的，该方法的主要逻辑就是将 `this._listeners` 当中的保存的 `listener` 全部调用一遍（省略了部分分支逻辑和性能监控相关代码）

```tsx
	fire(event: T): void {
		if (this._listeners) {
			for (let listener of this._listeners) {
				this._deliveryQueue.push([listener, event]);
			}

			while (this._deliveryQueue.size > 0) {
				const [listener, event] = this._deliveryQueue.shift()!;
				try {
					if (typeof listener === 'function') {
						listener.call(undefined, event);
					} else {
						listener[0].call(listener[1], event);
					}
				} catch (e) {
					onUnexpectedError(e);
				}
			}
		}
	}
```

`get event()`，这个方法会在 `Emitter` 中创建一个 `Event`，其主要逻辑就是将 `listener` 添加到 `this._listeners` 当中

```tsx
const remove = this._listeners.push(!thisArgs ? listener : [listener, thisArgs])
```

`Emitter` 类型还提供了一些特殊的回调接口：

```tsx
export interface EmitterOptions {
  onFirstListenerAdd?: Function
  onFirstListenerDidAdd?: Function
  onListenerDidAdd?: Function
  onLastListenerRemove?: Function
}
```

这使得 `Emitter` 在注册消费者的时候执行一些额外的逻辑。我们将会在下文中看到其中一些回调所扮演的重要角色。

### Event 辅助方法

vscode 还提供了一系列工具方法用于组合 `Event` ，得到更加丰富的事件处理能力。下面我们一一进行说明。

#### once

```tsx
export function once<T>(event: Event<T>): Event<T> {
  return (listener, thisArgs = null, disposables?) => {
    // we need this, in case the event fires during the listener call
    let didFire = false
    let result: IDisposable
    result = event(
      /* A */ (e) => {
        if (didFire) {
          return
        } else if (result) {
          result.dispose()
        } else {
          didFire = true
        }

        return listener.call(thisArgs, e)
      },
      null,
      disposables
    )

    if (didFire) {
      result.dispose()
    }

    return result
  }
}
```

这个方法用于将一个 `Event` 变为只能派发一次的，事件类型相同的 `Event` 。

每一个事件到达时，会从 A 处开始执行，可以看到这段代码通过 `didFire` 作为锁，保证 `listener.call(thisArgs, e)` 只会被执行一次。

很明显， `once` 的执行过程中有两个 `Event` ，那么消息如何在 Event 之间传递的呢？我们注意到 A 处的匿名函数调用了一个 `Event` 的 `listener`，而 A 本身又是另一个 Event 的 `listener`，所以答案是很明显的：**消息沿着 `Event` 链传递的过程，就是 `Event` 的 `listener` 们递归调用的过程。**

#### snapshot

```tsx
export function snapshot<T>(event: Event<T> /* B */): Event<T> {
  let listener: IDisposable
  const emitter = new Emitter<T>({
    onFirstListenerAdd() {
      listener = event(emitter.fire, emitter)
    },
    onLastListenerRemove() {
      listener.dispose()
    }
  })

  /* C */
  return emitter.event
}
```

这个工具方法用于生成 `map` 等操作，我们把它和 `map` 一起分析。

#### map

将一种类型的事件转换成另一种类型的事件，看起来和 `Array` 的 `map` 非常相似。

```tsx
export function map<I, O>(event: Event<I> /* A */, map: (i: I) => O): Event<O> {
  return snapshot(
    /* B */
    (listener, thisArgs = null, disposables?) =>
      event((i) => listener.call(thisArgs, map(i)), null, disposables)
  )
}
```

从代码可以看出：这里的 `Event` 链的顺序是

1. 被 `map` 装饰的 `Event` A
2. `snapshot` 的参数，匿名的 `Event` B
3. `Emitter` 暴露出的 `Event` C

当用户调用这个 `map` 转换出的 `Event` 的时候，实际上订阅的是 C，然后 C 在第一次被订阅时，会调用 B，而 B 又去订阅了 A。这里我们看到了 `Emitter` 的参数钩子起到了什么作用：B 是一个很特殊的 `Event` 它在 `onFirstListenerAdd` 中被订阅了 ，并且之后它并不会参与到 listener 的调用链中来，而是帮助 A 和 C 的 listener 之间创建了调用链，同时调用 `map` 对事件做了处理。

当有事件传递过来的时候，则是调用 A 的 listener `i => listener.call(thisArgs, map(i))` ，而这里的 `listener` 很明显可以看出是 C 的 listener，也就是 `Emitter` 的 `fire` 方法，通过上文中对 Emitter 的学习，我们知道，fire 方法会触发用户调用 C 时所传递来的 listener，这样整个传递链条就完整了。

#### forEach / filter / reduce

了解了 `map` 的工作原理之后，这三个函数就容易理解了，大家可以自行阅读代码。

#### signal

这个函数仅仅是做了一下类型转换，让订阅者忽略事件所携带的数据，比较简单。

#### any

```tsx
export function any<T>(...events: Event<T>[]): Event<T> {
  return (listener, thisArgs = null, disposables?) =>
    combinedDisposable(
      ...events.map((event) =>
        event((e) => listener.call(thisArgs, e), null, disposables)
      )
    )
}
```

这个方法会在 events 中任意一个 `Event` 派发事件的时候派发一个事件。

#### debounce

对 Event 链条上的事件做防抖处理。

```tsx
export function debounce<T>(
  event: Event<T>,
  merge: (last: T | undefined, event: T) => T,
  delay?: number,
  leading?: boolean,
  leakWarningThreshold?: number
): Event<T>
export function debounce<I, O>(
  event: Event<I>,
  merge: (last: O | undefined, event: I) => O,
  delay?: number,
  leading?: boolean,
  leakWarningThreshold?: number
): Event<O>
export function debounce<I, O>(
  event: Event<I>,
  merge: (last: O | undefined, event: I) => O,
  delay: number = 100,
  leading = false,
  leakWarningThreshold?: number
): Event<O> {
  let subscription: IDisposable
  let output: O | undefined = undefined
  let handle: any = undefined
  let numDebouncedCalls = 0

  const emitter = new Emitter<O>({
    leakWarningThreshold,
    onFirstListenerAdd() {
      subscription = event(
        /* A */ (cur) => {
          numDebouncedCalls++
          output = merge(output, cur)

          if (leading && !handle) {
            emitter.fire(output)
            output = undefined
          }

          clearTimeout(handle)
          handle = setTimeout(() => {
            const _output = output
            output = undefined
            handle = undefined
            if (!leading || numDebouncedCalls > 1) {
              emitter.fire(_output!)
            }

            numDebouncedCalls = 0
          }, delay)
        }
      )
    },
    onLastListenerRemove() {
      subscription.dispose()
    }
  })

  return emitter.event
}
```

不难看出这段代码的核心逻辑就是 A 处的 listener，它会对 debounce 时间内对数据做归并处理，并设置定时器，当收到新事件时就取消定时器，而定时器到期时就调用 [emitter.fire](http://emitter.fire) 向下游继续发送事件。

#### stopWatch

这是一个记录耗时的 `Event`，当它收到第一个事件时，会把这个事件转换为它从创建到收到该事件的耗时。

#### latch

这个 `Event` 仅有当事件确实发生变化时，才会向下游发送事件。原理也很简单，就是在 `filter` 的基础上，利用闭包来缓存上一次事件的数据，然后用新数据和它做比较，新老数据不同或者是第一次接收数据才放通。

#### buffer

这个 `Event` 在没有人订阅它时，会缓存所有收到的事件，并在收到订阅时将已经缓存的事件全部发送出去。

```tsx
export function buffer<T>(
  event: Event<T>,
  nextTick = false,
  _buffer: T[] = []
): Event<T> {
  let buffer: T[] | null = _buffer.slice()

  let listener: IDisposable | null = event((e) => {
    if (buffer) {
      buffer.push(e)
    } else {
      emitter.fire(e)
    }
  })

  const flush = () => {
    if (buffer) {
      buffer.forEach((e) => emitter.fire(e))
    }
    buffer = null
  }

  const emitter = new Emitter<T>({
    onFirstListenerAdd() {
      if (!listener) {
        listener = event((e) => emitter.fire(e))
      }
    },

    onFirstListenerDidAdd() {
      if (buffer) {
        if (nextTick) {
          setTimeout(flush)
        } else {
          flush()
        }
      }
    },

    onLastListenerRemove() {
      if (listener) {
        listener.dispose()
      }
      listener = null
    }
  })

  return emitter.event
}
```

如果调用 buffer 时传入了 `nextTick = True` ，则发送缓存事件的操作会易步进行，所以如果你第一次订阅时同步添加了很多 listener，则它们都会收到这些缓存的事件。

#### ChainableEvent

如果要多次使用 `map`, `filter` 等函数，一个比较优雅的写法是链式调用，例如 `Event.map.filter.xxx`， `ChainableEvent` 就是为此准备的，通过调用 `chain` 方法，一个 `Event` 会转换成 `ChainableEvent`，然后就可以进行链式调用：

```tsx
export function chain<T>(event: Event<T>): IChainableEvent<T> {
  return new ChainableEvent(event)
}
```

`ChainableEvent` 的实现很简单，就是对上面的方法进行了一次包裹，这里就不再赘述了。

除了上面提到的对 `Event` 的转换方法之外，还有一些生成 `Event` 的方法。

#### fromNodeEventEmitter

```tsx
export function fromNodeEventEmitter<T>(
  emitter: NodeEventEmitter,
  eventName: string,
  map: (...args: any[]) => T = (id) => id
): Event<T> {
  const fn = (...args: any[]) => result.fire(map(...args))
  const onFirstListenerAdd = () => emitter.on(eventName, fn)
  const onLastListenerRemove = () => emitter.removeListener(eventName, fn)
  const result = new Emitter<T>({ onFirstListenerAdd, onLastListenerRemove })

  return result.event
}
```

该方法是对 node.js 原生事件的包裹，在原生事件的回调中调用 [`Emitter.fire`](http://emitter.fire)。

#### fromDOMEventEmitter

对 DOM 事件进行包装，和上面的非常相似，这里就不赘述了。

#### fromPromise

```tsx
export function fromPromise<T = any>(promise: Promise<T>): Event<undefined> {
  const emitter = new Emitter<undefined>()
  let shouldEmit = false

  promise
    .then(undefined, () => null)
    .then(() => {
      if (!shouldEmit) {
        setTimeout(() => emitter.fire(undefined), 0)
      } else {
        emitter.fire(undefined)
      }
    })

  shouldEmit = true
  return emitter.event
}
```

将 Promise 转换为事件。通过 `shouldEmit` 确保 Promise 不会因为已经 resolve 而在订阅发生之前就开始派发事件（这样会导致错过事件）。

奇怪的是这里丢失了 Promise 返回的结果，不知道为什么这么设计，可能是 vscode 自己用不着吧。

#### toPromise

将事件转换为 Event，这个也比较简单。

另外，还有一些工具类提供更多的事件管理能力。

### PauseableEmitter

类似于 `Emitter`，但是能通过 `pause` 和 `resume` 方法暂停一条 Event 链上事件的传播，比较简单。

### EventMultiplexer

这个类可以订阅多个事件，并在任意一个事件派发的时候，将该事件转发给自己所有的订阅者，它的核心就是它的 `hook` 方法：

```tsx
	private hook(e: { event: Event<T>; listener: IDisposable | null; }): void {
		e.listener = e.event(r => this.emitter.fire(r));
	}
```

也较为简单，这里不再赘述。

### EventBufferer

这是一个非常有趣的类，它提供了一个 `wrapEvent` 方法包裹一个 `Event`，并提供了一个 `bufferEvents` 方法，在这个方法的回调内所有经过它 `wrapEvent` 包裹的 `Event`，都先不会被传播给订阅者。

````tsx
/**
 * The EventBufferer is useful in situations in which you want
 * to delay firing your events during some code.
 * You can wrap that code and be sure that the event will not
 * be fired during that wrap.
 *
 * ```
 * const emitter: Emitter;
 * const delayer = new EventDelayer();
 * const delayedEvent = delayer.wrapEvent(emitter.event);
 *
 * delayedEvent(console.log);
 *
 * delayer.bufferEvents(() => {
 *   emitter.fire(); // event will not be fired yet
 * });
 *
 * // event will only be fired at this point
 * ```
 */
export class EventBufferer {
  private buffers: Function[][] = []

  wrapEvent<T>(event: Event<T>): Event<T> {
    return (listener, thisArgs?, disposables?) => {
      return event(
        (i) => {
          const buffer = this.buffers[this.buffers.length - 1]

          if (buffer) {
            buffer.push(() => listener.call(thisArgs, i))
          } else {
            listener.call(thisArgs, i)
          }
        },
        undefined,
        disposables
      )
    }
  }

  bufferEvents<R = void>(fn: () => R): R {
    const buffer: Array<() => R> = []
    this.buffers.push(buffer)
    const r = fn()
    this.buffers.pop()
    buffer.forEach((flush) => flush())
    return r
  }
}
````

当 `bufferEvents` 被调用的时候，会往 `this.buffers` 中压入一个新 buffer，在 fn 执行过程中派发的事件，就会因为 `if (buffer)` 判断为 `true` 而被缓存， `fn` 执行完毕之后， `buffer` 被弹出，其中包含的事件全部被派发。

### Relay

这个类提供了切换上游 `Event` 的方法。当设置 `Relay` 的 `input` 属性时，就会切换监听的 `Event`，而下游的 `Event` 监听的是 `Relay` 的 `Emitter`，因此无需重新设置监听。

```tsx
export class Relay<T> implements IDisposable {
  // ...

  set input(event: Event<T>) {
    this.inputEvent = event

    if (this.listening) {
      this.nputEventListener.dispose()
      this.inputEventListener = event(this.emitter.fire, this.emitter)
    }
  }

  // ...
}
```

## 总结

至此我们已经学习了 vscode 事件模块的主要内容（其他的性能分析和泄漏监测等这篇文章就不分析了，感兴趣的读者可以自行阅读）。

vscode 事件模块是所谓响应式编程的一种实现，如果想要继续学习响应式编程，非常推荐以下两个项目：

- [RxJS](https://rxjs-dev.firebaseapp.com/guide/overview)，前端最强大的响应式编程库，很多 RxJS 的概念都可以在 vscode 的事件模块中找到对应，例如 Emitter 十分类似于 Subject， `Event` 类似于 `Observable`， `map` `reduce` `filter` 等函数，在 RxJS 中都有同名的操作符，但是 RxJS 更加强大，除了传递事件外，还能够传递异常以及事件流结束信息、支持事件调度等等，操作符更是要多得多
- [callbag](https://github.com/callbag/callbag)，是一套响应式编程规范（注意，不是库），vscode 和 RxJS 的事件链都是“推”机制的，而 callbag 同时支持推拉机制，而且实现上完全基于 JavaScript 强大的闭包机制，喜欢闭包体操的读者可以好好研究作者对几个操作符的实现
