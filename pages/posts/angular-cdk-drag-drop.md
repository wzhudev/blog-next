---
title: Angular CDK Drag Drop 源码解析
date: 2019/10/16
description: 探究 Angular CDK 拖拽的实现
tag: Angular, Angular CDK, drag-drop, source code, Chinese
author: Wendell
---

# Angular CDK Drag Drop 源码解析

什么是 drag drop？来自[官方网站的描述](https://link.zhihu.com/?target=https%3A//material.angular.io/cdk/drag-drop/overview)：

> The @angular/cdk/drag-drop module provides you with a way to easily and declaratively create drag-and-drop interfaces, with support for free dragging, sorting within a list, transferring items between lists, animations, touch devices, custom drag handles, previews, and placeholders, in addition to horizontal lists and locking along an axis。

简单来说，drag drop 能够帮助你声明式地，方便地创建可拖拽元素。

这篇会探究以下四个 demo 所体现的 drag drop 一些 feature 是如何实现的:

- [Basic Drag & Drop](https://link.zhihu.com/?target=https%3A//material.angular.io/cdk/drag-drop/overview%23getting-started)
- [Drag & Drop with a handle](https://link.zhihu.com/?target=https%3A//material.angular.io/cdk/drag-drop/overview%23customizing-the-drag-area-using-a-handle)
- [Drag & Drop boundary](https://link.zhihu.com/?target=https%3A//material.angular.io/cdk/drag-drop/overview%23restricting-movement-within-an-element)
- [Drag & Drop position locking](https://link.zhihu.com/?target=https%3A//material.angular.io/cdk/drag-drop/overview%23restricting-movement-along-an-axis)

drag drop 最简单的用法是在组件或 HTML 元素上声明一个 `cdkDrag` 指令，我们将通过追踪该指令的生命周期与其所创建的一些 listener 来探究 drag drop 的运行机制。具体而言，这篇文章的内容会包含以下几个主题:

- `cdkDrag` 的初始化，以及相关类的初始化
- 监听用户按下光标 (可能是按下鼠标左键，或者是按下触屏)，并准备好处理后续的拖拽事件
- 根据用户光标的移动来调整组件或元素的位置
- 当用户松开光标时结束这次拖拽

有关于 container 的内容将会在下一篇文章中叙述。

## 第一阶段：初始化

### `cdkDrag`

你可以在这个 [directives/drag.ts](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/master/src/cdk/drag-drop/directives/drag.ts) 中找到该类的实现。

> 为了方便阅读源码来更好地理解文章中介绍的内容，我给一些文件，类和方法添加了链接。

当你在一个 HTML 元素上声明了该指令时，它在初始化过程中做了如下几件事情，参考它的 [`constructor`](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/e7b0e40bfa1b91b7ab05c1375f0b0d85c7e18d69/src/cdk/drag-drop/directives/drag.ts%23L162-L195) 和 [`afterViewInit`](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/e7b0e40bfa1b91b7ab05c1375f0b0d85c7e18d69/src/cdk/drag-drop/directives/drag.ts%23L215-L247) 钩子：

```ts
if (dragDrop) {
  this._dragRef = dragDrop.createDrag(element, config)
} else {
  this._dragRef = new DragRef(
    element,
    config,
    _document,
    _ngZone,
    viewportRuler,
    dragDropRegistry
  )
}
this._dragRef.data = this
this._syncInputs(this._dragRef)
this._proxyEvents(this._dragRef)
```

1. 实例化 `DragRef` 对象，拖拽逻辑主要由该对象负责完成。这个对象十分重要，我们待会儿再详细讨论它，先看下其他代码做了什么。
2. **`_syncInputs` 方法会将指令的一些参数同步到 `DragRef` 对象上**。在该方法中，`cdkDrag` 订阅了 `DragRef` 的 `beforeStarted` 事件，在收到该事件时调用 `DragRef` 的一系列以 with 的开头的方法来同步 inputs。记住这一点，后面我们会反复提及。
3. **`_proxyEvents` 会将自己作为 `DragRef` 的代理**。在 `_proxyEvents` 方法中，订阅了 `DragRef` 的一系列事件并转发出去，类似于 `DragRef` 的代理。

> 你或许注意到注入了一个 `DragDrop` 对象，这是一个用于创建 `DragRef` 的简单服务，如果对它感兴趣你可以自行查看它的代码。

在该指令的 `AfterViewInit` 钩子 中，它对 handle 做了处理：当页面第一次渲染（通过 `startWith` 操作符来实现）或者 handle 发生改变时，订阅所有 handle 的状态改变事件，并且调用 `DragRef` 的方法来禁用或启用它们。我们会在讲解完主要逻辑之后再介绍 handle 是如何工作的。

### `DragRef`

这个类执行了拖拽最主要的逻辑，它的代码在 [drag-ref.ts](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/master/src/cdk/drag-drop/drag-ref.ts) 这个文件里。

它的 [`constructor`](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/e7b0e40bfa1b91b7ab05c1375f0b0d85c7e18d69/src/cdk/drag-drop/drag-ref.ts%23L267-L277) 做了如下几件事情:

- [`withRootElement`](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/e7b0e40bfa1b91b7ab05c1375f0b0d85c7e18d69/src/cdk/drag-drop/drag-ref.ts%23L323-L338) 方法在 `rootElement`（即绑定了 `cdkDrag` 指令的元素）上绑定了 drag start 事件的 listener `_pointerDown`。当用户在该元素上按下鼠标左键或者是按下该元素时，`_pointerDown` 就会被调用。
- 然后，它在 `DragDropRegistry` 上注册了自己。

### `DragDropRegistry`

[`DragDropRegistry`](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/master/src/cdk/drag-drop/drag-drop-registry.ts) 服务注册在根注入器上，并被 `DragRef` 所依赖。**它负责监听 `mousemove` 和 `touchmove` 事件，并确保同一时刻只有一个 `DragRef` 能够响应这些事件**。`registerDragItem` 被调用时，它会注册调用它的 `DragRef` 对象，然后在某个 `DragRef` 拖拽时，把光标移动的事件发送给它。

到这里，整个 drag drop 机制就做好了准备来响应用户的操作。

## 第二阶段：开始拖拽

当用户在 `rootElement` 上按下光标时，会派发一个 `mousedown` 或 `touchdown` 事件，此时 `DragRef` 的 [`_pointerDown`](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/e7b0e40bfa1b91b7ab05c1375f0b0d85c7e18d69/src/cdk/drag-drop/drag-ref.ts%23L453-L469) 方法就会被调用。

我们先考虑没有 handle 的最简单的情形。

1. `beforeStart` 事件被派发出去，此时，就像我们之前提到的那样，所有的 inputs 会从 `cdkDrag` 同步到 `DragRef` 上。
2. 之后，[`_initializeDragSequance`](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/e7b0e40bfa1b91b7ab05c1375f0b0d85c7e18d69/src/cdk/drag-drop/drag-ref.ts%23L617-L679) 就会实例化一个 drag sequence，会确保整个机制已经做好准备接收 move 事件的准备。

> 一个 drag sequance 就是一次拖拽过程。

具体而言，实例化 drag sequence 的过程会做如下的事情：

1. 首先，它会缓存元素已有的 (往往是用户设置的) transform。在拖拽结束之后，缓存的 transform 会被设置回去。
2. 然后它订阅了来自 registry 的 move 和 up 事件。

其他的代码和设置状态相关，例如:

- `_hasStartedDragging` `_hasMoved`，字面意思。
- `_pickupPositionInElement`。以 `rootElement` 的左上角为原点，鼠标相对于 `rootElement` 的位置。
- `_pointerDirectionDelta`，一个记录光标移动的矢量。

最后它调用了 registry 的 [`startDragging`](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/e7b0e40bfa1b91b7ab05c1375f0b0d85c7e18d69/src/cdk/drag-drop/drag-drop-registry.ts%23L114-L158) 方法，这个方法会绑定 move 和 up 事件的 listener，这样当用户移动光标时，就会触发这两个 listener，然后把事件发送给 `dragRef`。

## 第三阶段：拖动

现在用户就可以移动元素了，[`_pointerMove`](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/master/src/cdk/drag-drop/drag-ref.ts%23L472-L540) 方法会在用户移动光标时被调用。这个方法做了如下几件事情:

1. 首先，它会检查光标移动的距离是否超过了设定的阈值。如果超过了，它会将 `_hasStartedDragging` 标记为 true。
2. 然后，它会计算 transform，然后将它赋给 `rootElement`。当我们讨论拖动边界的时候我们会回来讨论 `_getConstrainedPointerPosition`，现在先假设 `constrainedPointerPosition` 就是光标此时所在的位置，看看 `rootElement` 的新位置是如何计算的。

```ts
const activeTransform = this._activeTransform
activeTransform.x =
  constrainedPointerPosition.x -
  this._pickupPositionOnPage.x +
  this._passiveTransform.x
activeTransform.y =
  constrainedPointerPosition.y -
  this._pickupPositionOnPage.y +
  this._passiveTransform.y
const transform = getTransform(activeTransform.x, activeTransform.y)

this._rootElement.style.transform = this._initialTransform
  ? transform + ' ' + this._initialTransform
  : transform
```

在我们开始讨论之前，让我们先来弄清楚这段代码中几个变量的含义。

- `activeTransform` 表示在这一次的拖拽过程中，相对于 `rootElement` 最初的位置，`rootElement` 沿着 x 和 y 轴移动了多少像素。
- `passiveTransform` 表示在这一次拖拽过程开始之前，相对于 `rootElement` 最初的位置，`rootElement` 沿着 x 和 y 轴移动了多少像素。很显然，在当前这次拖拽结束过后，`activatedTransform` 会被赋值到 `passiveTransform` 上。
- `pickUpPositionOnPage` 表示指的是在这一次拖拽过程开始鼠标相对于页面原点的位置。
  知道了这些变量的含义，我们现在就能很轻易地明白 `activatedTransform` 是如何计算的了：就是一个简单的向量加法，上次移动的向量，加上这次移动的向量 (当前鼠标位置减去开始拖拽时鼠标的位置)。

计算完成之后，我们会通过一个辅助方法生成 transform 字符串，然后将它设置到 `rootElement` 的 style 上。记得之前提到过的缓存的 `initialTransform` 吗？它会被添加到 transform 属性后面。

## 第四阶段：停止拖拽

现在元素已经能够随着光标移动了，现在我们来看看如何停止这个拖拽过程。

当用户释放鼠标左键或者抬起手指的时候，[`pointerUp`](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/64010d785b2cd1a95c03564b43fce99194f57d3c/src/cdk/drag-drop/drag-ref.ts%23L543-L580) 方法就会被调用。它会取消订阅来自 registry 的 move 和 up 事件，`DragDropService` 所绑定的全局事件 listener 也会被移除。然后它将 `passiveTransform` 赋值给 `activatedTransform`，这样下一次拖拽开始时就会有一个正确的起始点。

Bingo！现在整个拖拽过程就完成了。

## 其他

这里我们会谈论一些之前跳过的高级内容。

### 拖拽边界 boundary

当我们在第三阶段中讨论定位问题时我们略过了 `_getConstrainedPointerPosition` 方法，现在我们来谈一谈边界机制。

当拖拽开始时 `beforeStarted` 事件派发，`withBoundaryElement` 方法会被调用。此时，`getClosestMatchingAncestor` 方法使用一个 CSS 选择器，来选择 `rootElement` 最近的一个满足选择器的祖先元素作为边界元素。

当 drag sequence 被初始化的时候，该元素的位置信息被赋值给 `this._boundaryRect`。

```ts
if (this._boundaryElement) {
  this._boundaryRect = this._boundaryElement.getBoundingClientRect()
}
```

> `getBoundingClientRect` 会触发 reflow，所以必须要在每次拖动开始的时候进行计算并缓存。
> 在 `_getConstrainedPointerPosition` 中，光标的位置会被修改已确保 `rootElement` 不会超出边界元素。

```ts
const point = this._getPointerPositionOnPage(event)
if (this._boundaryRect) {
  const { x: pickupX, y: pickupY } = this._pickupPositionInElement
  const boundaryRect = this._boundaryRect
  const previewRect = this._previewRect!
  const minY = boundaryRect.top + pickupY
  const maxY = boundaryRect.bottom - (previewRect.height - pickupY)
  const minX = boundaryRect.left + pickupX
  const maxX = boundaryRect.right - (previewRect.width - pickupX)
  point.x = clamp(point.x, minX, maxX)
  point.y = clamp(point.y, minY, maxY)
}
```

### handle

> 对 handle 比较准确的翻译是“把手”。

早前我们就提到 `cdkDrag` 会负责 handle 的处理。当 handle 改变时，[`withHandles`](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/c8ddaa80e7bd622ee43573e23d0eea0ad112b797/src/cdk/drag-drop/drag-ref.ts%23L310-L315) 方法会被调用。

```ts
tap((handles: QueryList<CdkDragHandle>) => {
  const childHandleElements = handles
    .filter(handle => handle._parentDrag === this)
    .map(handle => handle.element);
  this._dragRef.withHandles(childHandleElements);
}),
```

这个 filter 是如何工作的，它怎么知道哪些 handler 属于当前的 `cdkDrag` 呢？原来 `cdkDrag` 会在 `DragHandle` 的 [constructor](https://link.zhihu.com/?target=https%3A//github.com/angular/material2/blob/c8ddaa80e7bd622ee43573e23d0eea0ad112b797/src/cdk/drag-drop/directives/drag-handle.ts%23L38-L44) 中被注入：

```ta
constructor(
  public element: ElementRef<HTMLElement>,
  @Inject(CDK_DRAG_PARENT) @Optional() parentDrag?: any) {
  this._parentDrag = parentDrag;
  toggleNativeDragInteractions(element.nativeElement, false);
}
```

`cdkDrag` 的 metadata 中也体现了 `cdkDrag` 会以 `CDK_DRAG_PARENT` 这个令牌注入到它的 handle 子元素中：

```ts
@Directive({
  selector: '[cdkDrag]',
  exportAs: 'cdkDrag',
  host: {
    'class': 'cdk-drag',
    '[class.cdk-drag-dragging]': '_dragRef.isDragging()',
  },
  providers: [{provide: CDK_DRAG_PARENT, useExisting: CdkDrag}]
})
```

至于 `withHandles`，它仅仅是注册这些 handle 的 HTML 元素。

在 `_pointerDown` 方法中，如果有 handle 的话，handle 就会代替 `rootElement` 作为是否应该开始一个 drag sequence 的依据。

```ts
if (this._handles.length) {
  const targetHandle = this._handles.find((handle) => {
    const target = event.target
    return (
      !!target && (target === handle || handle.contains(target as HTMLElement))
    )
  })
  if (
    targetHandle &&
    !this._disabledHandles.has(targetHandle) &&
    !this.disabled
  ) {
    this._initializeDragSequence(targetHandle, event)
  }
} else if (!this.disabled) {
  this._initializeDragSequence(this._rootElement, event)
}
```

> `contains` 这个 API 可以判断某个元素是否是另一个元素的子孙元素。

我们如何知道一个 handle 是启用还是禁用呢？还记得在 `cdkDrag` 中 `_pointerDown` 事件的 subscriber 吗？这行代码

```ts
handleInstance.disabled ？dragRef.disableHandle(handle) : dragRef.enableHandle(handle);
```

会启用或禁用 handle。

### 在某个方向上锁定 axis locking

我们知道了如何将 `rootElement` 限定在边界元素之内，锁定拖动的方向就更简单了，只需要重新赋 x 或 y 的值就可以了。

```ts
if (this.lockAxis === 'x' || dropContainerLock === 'x') {
  point.y = this._pickupPositionOnPage.y
} else if (this.lockAxis === 'y' || dropContainerLock === 'y') {
  point.x = this._pickupPositionOnPage.x
}
```

这样 `constrainedPointerPosition.x - this._pickupPositionOnPage.x 或者 constrainedPointerPosition.y - this._pickupPositionOnPage.y` 就会等于 0，在某个方向上的 transform 也就不会变化了。

## 结论

这篇文章详细的解释了在没有 container 的情况下，drag drop 是如何工作的，以及 handle，axis locking 和 boundary 是如何生效的。

- 声明了 `cdkDrag` 指令的元素会变成可拖拽的，但 `DragRef` 是拖拽逻辑的主要执行者
- DragDropRegister 负责监听全局的 move 事件并确保界面上同一时刻只有一个元素是可拖拽的
- drag sequence 是一次拖拽过程的抽象
