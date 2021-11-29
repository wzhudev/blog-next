---
title: Angular 源码解析 - Zone.js
date: 2019/10/16
description: 介绍 Angular 的 Zone.js 机制
tag: Angular, source code, Chinese, Zone.js
author: Wendell
---

# Angular 源码解析 - Zone.js

Angular 源码解析系列。这篇文章有关于 Zone.js 的用途，实现和 NgZone 的实现，以及 Angular 如何使用 Zone.js 实现自动变更检测。

---

这篇文章是 Angular 源码解析系列的第一篇，分为以下三个小节：

- 为什么 Angular 需要 Zone.js？
- Zone.js 如何工作？
  - Zone.js 核心的实现
  - 举一个例子介绍 Zone.js 如何打补丁
- Angular 如何使用 Zone.js？

阅读这篇文章你需要熟悉 JavaScript 的[事件循环 (Event Loop) 机制](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/EventLoop)。

现在 Zone.js 的源码已经被放到 Angular 底下以 mono repo 的形式管理，你可以通过下面的链接阅读 Zone.js 的源码。

[angular/angular](https://github.com/angular/angular/tree/master/packages/zone.js)

## 为什么 Angular 需要 Zone.js？

所有前端框架需要解决的一个共同问题就是：**应该何时将应用状态的变化反映到视图中，即变更检测**。React 的方案是交给用户自行决定，即让用户通过 `setState` 方法告诉 React 应用的状态发生了改变；Vue 通过拦截对象的赋值操作来监测状态改变（即所谓响应式）；而 Angular 的方案就是 Zone.js。Zone.js 通过给一些会触发异步事件 API 打补丁（monkey patch），比如 XHR、DOM event、定时器等来监听异步事件的编排（比如调用 `setTimeout`）和触发（比如 `setTimeout` 到时），而应用状态的变化一定是某个异步事件的结果，这样 Angular 就可以借助 Zone.js 实现变更检测。

体现在代码中：[Angular 应用会在 zone 的 `onMicrotaskEmpty` 回调中调用 `tick` 方法](https://github.com/angular/angular/blob/82b97280f3e4826ba396930e5805b9053f550500/packages/core/src/application_ref.ts#L522-L523)，而 `tick` 方法会调用顶层组件的 `detectChanges` 方法执行变更检测，就是下面这行代码：

```typescript
this._zone.onMicrotaskEmpty.subscribe({
  next: () => {
    this._zone.run(() => {
      this.tick()
    })
  }
})
```

可以通过看[代码注释](https://github.com/angular/angular/blob/37de490e2342b9a64bb33944d13f0e26659288bd/packages/zone.js/lib/zone.ts#L15-L35)来了解官方对 Zone.js 作用的描述。

## Zone.js 如何工作？

### 编程模型

把 Zone.js 中的 zone 想象成 JavaScript VM 线程里的 mini 线程。

JavaScript 是单线程，基于事件循环的，而通过 Zone.js，我们可以把处理事件的回调函数放到不同的 zone 里面执行，而且在当前回调函数内触发的异步事件也会在当前 zone 里面得到处理，即我们给事件的回调函数提供了执行环境。而且，Zone.js 还提供了钩子，允许我们在回调函数执行前后执行额外一些代码（还有其他的一些钩子）。

总而言之——

> zone 提供了 JavaScript 异步函数的执行环境。

### 核心代码

核心部分实现了 Zone.js 的机制，而不关心各种 patch 该如何实现，代码都在 [zone.ts](https://github.com/angular/angular/blob/master/packages/zone.js/lib/zone.ts) 当中，前面几百行都是接口声明，请自行阅读，本文主要聚焦于其[实现](https://github.com/angular/angular/blob/82b97280f3e4826ba396930e5805b9053f550500/packages/zone.js/lib/zone.ts#L684)。

#### 重要类型和方法

这个文件主要声明和实现了如下几个类：

- `Zone`，JavaScript 事件的执行环境，和线程一样，它们可以带一些数据，并且可能拥有父子 zone。
- `ZoneTask`，包装后的异步事件，这些 task 有三种子类：
  - `MicroTask`，由 `Promise` 创建，我们知道 native 的 `Promise` 是在当前事件循环结束前就要执行的，所以打过补丁的 Promise 也应该在事件循环结束前执行。
  - `MacroTask`，由 `setTimeout` 等创建，native 的 `setTimeout` 会在将来某个时间被处理，而且会被处理一到多次。
  - `EventTask`，由 `addEventListener` 等创建，这些 task 可能被触发多次，也可能一直不会被触发。
- `ZoneSpec`，创建一个 zone 时给它提供的参数，除了 `name` 是必须的外，还可以传入如下的钩子：
  - `onFork`，创建新 zone 的钩子。
  - `onIntercept`，包装某个回调函数时触发的钩子。
  - `onInvoke`，调用某个回调函数时触发的钩子。
  - `onHandleError`，调用某个回调函数出错时触发的钩子。
  - `onScheduleTask`，`ZoneTask` 被安排时触发的钩子，比如调用了 `setTimeout`。
  - `onInvokeTask`，`ZoneTask` 被触发时触发的钩子，比如 `setTimeout` 到时。
  - `onCancelTask`，`ZoneTask` 被取消时触发的钩子，比如用 `clearTimeout` 取消了计时器。
  - `onHasTask`，检测到有或无 `ZoneTask` 时触发的钩子（即对第一个 schedule 的 zone 和最后一个 invoke 或 cancel 的 task 触发）。
- `ZoneDelegate`，负责调用钩子。

> 官方文档中对这些类所实现的接口有非常详细的注释，可以比较下面的内容阅读。

#### `Zone` 类

[这些代码是对 Zone 的实现](https://github.com/angular/angular/blob/82b97280f3e4826ba396930e5805b9053f550500/packages/zone.js/lib/zone.ts#L718-L975)。

有几个值得关注的静态方法：

- `get root()`，该方法返回根 zone，所有其他 zone 都是该 zone 的子孙，类似于操作系统的第一个进程。根 zone 是 Zone.js 初始化时自行创建的，相关代码[在这里](https://github.com/angular/angular/blob/82b97280f3e4826ba396930e5805b9053f550500/packages/zone.js/lib/zone.ts#L1426)。根 zone 确保了所有的异步函数都在 Zone.js 的机制内运行。
- `get current()`，返回当前 zone，类似于单线程 CPU 中正在占用 CPU 的进程，它本质上是返回闭包内的一个变量 `_currentZoneFrame` 的引用。
- `get currentTask`，返回当前正被 invoke 的 task。
- `__load_patch`，这是 Zone.js 加载补丁的方法，后面讲解 patch 的加载时会详细说明。

这个类的[构造函数](https://github.com/angular/angular/blob/82b97280f3e4826ba396930e5805b9053f550500/packages/zone.js/lib/zone.ts#L769-L775)，要点如下：

- 必须要有名字作为 zone 的标识符。
- `parent` 变量保存了父 zone，这样 zone 就可以形成一个树型结构。
- 可以挂一些变量到 `_properties` 上作为函数运行的上下文，而 `get()` 和 `getZoneWith()` 方法分别用于取得某个 key 所对应的变量和上下文中有某个 key 的 zone。
- 构建了一个 `ZoneDelegate` 并赋值给 `_zoneDelegate` 属性。

`Zone` 类还有如下的实例方法：

- `fork`，创建一个子 zone，相当于 fork 一个进程。
- `wrap`，包裹一个函数，这个被包裹的函数执行时，**首先会通过 `runGurad` 把该函数运行的上下文置换为原来包裹它的 zone，然后通过 `ZoneSpec` 去执行钩子**。
- `run`，**立即在当前 zone 执行函数，并调用 `ZoneSpec` 执行 `invoke` 钩子**。
- `runGuarded`，和 run 做的事情基本相同，不同点在于如果执行过程出错，错误会被 `ZoneSpec` 注册的 error 钩子先处理，如果 `ZoneSpec` error 钩子不能处理，再抛出。
- `runTask`，运行一个 `ZoneTask`：
  - 看到这里请先去阅读 `ZoneTask` 类的实现。
  - 未被安排的 `MacroTask` 和 `EventTask` 两种类型的 task 是不需要执行的。
  - 调用 `ZoneDelegate` 的方法来执行 task。
  - 执行完毕之后调整 task 的状态，对于周期性触发的 task 来说，需要通过 `reEntryGuard` 进行状态保护。
- `scheduleTask`，用来安排一个 task 的执行环境。

> 在阅读代码的时候我们经常能看到 `_currentZoneFrame` 这个变量，这实际上上记录了一个 zone 的栈（以链表的形式），如果某个 zone 中执行了函数调用，该 zone 就进入这个栈中，这样就将函数的调用栈与进入和离开 zone 的先后顺序对应起来了。
>
> 该变量被声明在创建 Zone.js 全局对象的函数里（即在一个闭包中），外部没有办法直接访问。

#### `ZoneDelegate` 类

[代码在这里](https://github.com/angular/angular/blob/82b97280f3e4826ba396930e5805b9053f550500/packages/zone.js/lib/zone.ts#L990-L1220)。

要点：

- 它的构造函数中前面一坨代码实际上干的事都可以概括为：我有没有这个回调，没有我就用父 `ZoneDelegate` 的（当然父级也有可能是空的）。这样父级 zone 可以通过 delegate 干预子 zone 的行为。
- 大部分方法都是在尝试调用钩子，不成的话再进行磨人的简单处理。
- 特别注意 [`scheduleTask()`](https://github.com/angular/angular/blob/82b97280f3e4826ba396930e5805b9053f550500/packages/zone.js/lib/zone.ts#L1145-L1166)，该方法对于 `MicroTask` 会调用 `scheduleMicroTask`。

#### `ZoneTask`

[这个类的代码在这里](https://github.com/angular/angular/blob/82b97280f3e4826ba396930e5805b9053f550500/packages/zone.js/lib/zone.ts#L1222-L1316)。

要点：

- `_state`，记录了这个 task 的状态。
- 在构造函数的最后要为当前 task [设置 invoke 方法](https://github.com/angular/angular/blob/82b97280f3e4826ba396930e5805b9053f550500/packages/zone.js/lib/zone.ts#L1252-L1258)，通常走的是 else 的分支，可以看到这里绑定了 invoke 函数中的 `task` 为当前 task。
- `static invokeTask()`，该函数执行一个 task：
  1. 执行 task 的时候会增加 `_numberOfNestedTaskFrames` 计数器的值。
  2. 之后通过 zone 执行 task。
  3. 当执行完毕的时候调用 `drainMicroTaskQueue` 尝试清空所有的 `MicroTask`。
  4. 然后减少计数器的值。
- `_transitionTo`，改变 task 的状态，task 可以从两个源状态转移到一个目标状态，当 zone 的状态不属于两个源状态中的任何一个时，这个方法会抛出错误。

除了上面几个重要的类，下面两个方法也值得关注：

[`scheduleMicroTask`](https://github.com/angular/angular/blob/82b97280f3e4826ba396930e5805b9053f550500/packages/zone.js/lib/zone.ts#L1331-L1354)，这个方法是对 Promise 这样的所谓 micro task 的处理。

- 我们知道根据 JavaScript 的规范，Promise 是要在当前 VM 时间循环结束前触发的，所以 Zone.js 会尝试在恰当的时机释放所有的 Promise，即通过调用 `drainMicroTaskQueue`。

[`drainMicroTaskQueue`](https://github.com/angular/angular/blob/37de490e2342b9a64bb33944d13f0e26659288bd/packages/zone.js/lib/zone.ts#L1356)。该方法内容十分简单，即尝试对 `_microTaskQueue` 中的每一个 MicroTask run 一下。

- 除了 schedule micro task 的时候有可能调用该方法，还有一种可能就是 [`_numberOfNestedTaskFrames === 1` 时](https://github.com/angular/angular/blob/master/packages/zone.js/lib/zone.ts#L1270)。

### 补丁的实现

我们之前提到了 `__load_patch` 方法是 Zone.js 用来加载补丁的，这一小节我们将以 `setTimeout` 为例介绍 Zone.js 如何加载补丁。同时还会结合上一小节的内容，讲解 `setTimeout` 在 Zone.js 执行的全过程。因为各个 JavaScript runtime 对异步函数的支持情况不尽相同（比如在 Node.js 环境里不可能有 DOM 事件相关的异步函数，如 `addEventListener`），所以 Zone.js 会给不同的 runtime 提供不同的 dist 包，patch 不同的异步函数。好在不管在任何环境中，`setTimeout` 都是存在的。

patch 的具体实现[在这里](https://github.com/angular/angular/blob/2a6e6c02eddf1bf58ee5be15d4439f7532ddefa0/packages/zone.js/lib/common/timers.ts#L22-L133)，下面我们将会仔细讲解这部分代码。

首先准备好要 patch 的函数的名称：

```typescript
setName += nameSuffix // setTimeout
cancelName += nameSuffix // clearTimetout
```

接下来，调用 `patchMethod` 方法，传入的三个参数分别是目标对象（被 patch 后的函数应当挂载在目标对象上，因为 `setTimeout` 其实是 `window` 的一个属性，所以这里的形参叫做 `window`），被 patch 函数的名字，以及一个回调函数，请注意这个回调函数直接返回了另外一个函数(如果你了解什么叫做柯里化，应该很容易理解，其实就是让该函数的执行时能够访问到 `delegate` 参数）：

```typescript
patchMethod(window, setNmae, (delegate: Function) => function(self: any, args: any[]): void);
```

来看 [`patchMethod`](https://github.com/angular/angular/blob/2a6e6c02eddf1bf58ee5be15d4439f7532ddefa0/packages/zone.js/lib/common/utils.ts#L386) 方法:

```typescript
export function patchMethod(
  target: any,
  name: string,
  patchFn: (
    delegate: Function,
    delegateName: string,
    name: string
  ) => (self: any, args: any[]) => any
): Function | null
```

[首先](https://github.com/angular/angular/blob/2a6e6c02eddf1bf58ee5be15d4439f7532ddefa0/packages/zone.js/lib/common/utils.ts#L389-L396)，该方法从 `target` 的原型链上找到 name 代表的方法的具体位置（不要忘记 JavaScript 访问对象属性是通过原型链机制进行的），如果找不到这个方法，就直接在 `target` 上创建 patch 过的方法.

[然后](https://github.com/angular/angular/blob/2a6e6c02eddf1bf58ee5be15d4439f7532ddefa0/packages/zone.js/lib/common/utils.ts#L398-L400)，检查 patch 过的方法是否存在，不存在才进行 patch。

在进行 patch 时：

- 首先，让变量 `delegate` 先指向原生方法。
- 然后，确保该方法是可复写的 （通过检查 `PropertyDescriptor` 的 `writable` 字段），不然就用原来的，相当于没有 patch。
- 然后，调用 `patchFn` （这里是一个类似于柯里化的过程），将 `patchDelegate` 变量指向 patch 过后的函数，然后再将 `proto[name]` 指向一个新的函数，这里同时确保了 `this` 能够绑定在正确的对象上（对于 `setTimeout` 的例子就是 `window`）。
- 到这里 patch 过程就已经完成了，函数返回的是 patch 之前的方法，即原生方法。

**用户执行 `setTimeout` 后会发生什么？**

我在这里给读者准备了[一个简单的例子](https://github.com/wendzhue/ng-source-tour/tree/master/zone.js)，通过给 `console.log('z')` 这一行打断点，就能够看到整个调用栈。

![debugger](https://user-images.githubusercontent.com/12122021/66885890-4c2e0600-f008-11e9-9098-4c21d6a07f1d.png)

- 用户执行 `setTimeout` 时，实际上执行的是[这一行的匿名函数](https://github.com/angular/angular/blob/2a6e6c02eddf1bf58ee5be15d4439f7532ddefa0/packages/zone.js/lib/common/utils.ts#L407)，最终执行了[这一行](https://github.com/angular/angular/blob/2a6e6c02eddf1bf58ee5be15d4439f7532ddefa0/packages/zone.js/lib/common/timers.ts#L60)的 `function(self: any, args: any[])`
- 准备 task 的参数
- 调用 [`scheduleMacroTaskWithCurrentZone`](https://github.com/angular/angular/blob/2a6e6c02eddf1bf58ee5be15d4439f7532ddefa0/packages/zone.js/lib/common/utils.ts#L46) 方法，这个函数直接将参数透传给 `Zone.current.scheduleTaskMacroTask`
- 回到我们先前讲过的 `Zone.current.scheduleTaskMacroTask`，最终是调用了 [`ZoneDelegate` 的 `scheduleTask` 方法](https://github.com/angular/angular/blob/2a6e6c02eddf1bf58ee5be15d4439f7532ddefa0/packages/zone.js/lib/zone.ts#L1145-L1166)。
- [`scheduleTask` 方法](https://github.com/angular/angular/blob/2a6e6c02eddf1bf58ee5be15d4439f7532ddefa0/packages/zone.js/lib/common/timers.ts#L30-L55)将被调用，而这一句 `task.scheduleFn(task);` 实际上调用了 [`scheduleTask` 函数](https://github.com/angular/angular/blob/2a6e6c02eddf1bf58ee5be15d4439f7532ddefa0/packages/zone.js/lib/common/timers.ts#L30)，最终调用原生的 `setTimeout`。
- 返回 `handle`，即原生 `setTimeout` 的返回值。
- 当原生的 `setTimeout` 触发时，[`task.invoke`](https://github.com/angular/angular/blob/2a6e6c02eddf1bf58ee5be15d4439f7532ddefa0/packages/zone.js/lib/common/timers.ts#L34) 被调用，最终 `ZoneDelegate.invokeTask` 被调用。

## Angular 如何使用 Zone.js？

Angular 应用初始化过程中，[实例化了一个 NgZone 然后将所有逻辑都跑在该对象的 `_inner` 对象中](https://github.com/angular/angular/blob/2a6e6c02eddf1bf58ee5be15d4439f7532ddefa0/packages/core/src/application_ref.ts#L248-L285)，[该 `_inner` 即为 Angular zone](https://github.com/angular/angular/blob/fc6f48185c3a546b130296d3d5dce86fdf334115/packages/core/src/zone/ng_zone.ts#L254-L292)。

Angular 创建该 zone 的过程中传入的 `ZoneSpec` 的部分如下所示：

```typescript
onHasTask:
        (delegate: ZoneDelegate, current: Zone, target: Zone, hasTaskState: HasTaskState) => {
          delegate.hasTask(target, hasTaskState);
          if (current === target) {
            // We are only interested in hasTask events which originate from our zone
            // (A child hasTask event is not interesting to us)
            if (hasTaskState.change == 'microTask') {
              zone.hasPendingMicrotasks = hasTaskState.microTask;
              checkStable(zone);
            } else if (hasTaskState.change == 'macroTask') {
              zone.hasPendingMacrotasks = hasTaskState.macroTask;
            }
          }
        },
```

在 `checkStable` 这个方法中你可以看到这样一行：

```typescript
zone.onMicrotaskEmpty.emit(null)
```

这就联系到了开篇提到的代码：

```typescript
this._zone.onMicrotaskEmpty.subscribe({
  next: () => {
    this._zone.run(() => {
      this.tick()
    })
  }
})
```

当然 [`checkStable` 方法](https://github.com/angular/angular/blob/fc6f48185c3a546b130296d3d5dce86fdf334115/packages/core/src/zone/ng_zone.ts#L236-L252)还有可能在其他时机被调用，主要是通过 Zone.js 的 `onInvokeTask` 和 `onInvoke` 两个钩子，即在异步事件触发时，交给读者自行验证，这里就不赘述了。

## 结论

这篇文章介绍了 Zone.js 的实现，包括 Zone.js 核心和 patch 的实现，还讲解了 Angular 对 Zone.js 的使用。

- Zone.js 提供的 zone 是 JavaScript 函数的执行环境，Zone.js patch 了几乎所有的异步函数。
- Angular 应用在初始化时创建了一个 Angular zone，Angular 的代码都运行在该 Zone 当中。
- 经过 patch 的异步函数所触发的异步事件会被 Zone.js 所捕捉，在事件回调函数执行后触发 Zone.js 的钩子。
- Angular 在部分钩子中进行整个应用的变更检测。
- Angular 的变更检测默认依赖于 Zone.js，但如果开发者完全了解应该在什么时候做变更检测，就可以抛弃 Zone.js。实际上，开发者可以通过给 `bootstrapModule` 方法的第二个参数传参 `{ ngZone: 'noop' }` 来使用一个[在钩子里不做任何事情的 `ZoneSpec`](https://github.com/angular/angular/blob/fc6f48185c3a546b130296d3d5dce86fdf334115/packages/core/src/zone/ng_zone.ts#L319-L335)。

## 参考资料

- [I reverse-engineered Zones (zone.js) and here is what I've found](https://blog.angularindepth.com/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found-1f48dc87659b)
