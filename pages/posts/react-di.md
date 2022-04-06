---
title: React 的新范式 - DI, RxJS & Hooks
date: 2020/1/17
description: 介绍一种开发 React 应用的新思路
tag: di, react, Chinese
author: Wendell
---

# React 的新范式 - DI, RxJS & Hooks

## Introduction

从去年 12 月起我一直在做在线文档的开发工作，工作中遇到了如下问题：

第一个问题是：如何**处理不同平台之间的差异**。

我们的产品需要兼容多个平台，而且同一个功能在不同的平台上要调用不同的 API，之前为了处理这些平台差异性，写了很多这样的代码：

```ts
if (isMobile) {
  // ... mobile 的特殊逻辑
} else if (isDesktop) {
  // ... desktop 的特殊逻辑
} else {
  // ... browser 的逻辑
}
```

这样的写法不仅很难维护，而且由于无法 tree shake，会导致仅在 B 平台上运行的代码被 ship 到 A 平台用户的设备上，无端增加了包大小。

_有一种比较 hacky 的方案是通过 uglify 来消除不会被执行的分支，但这仍然无法解决可维护性低的问题。_

第二个问题是：如何**在多个产品之间复用代码**。

我们的项目有两个文档与表格两个子产品，这两个产品在 UI、逻辑上既有相同之处又有不同之处。例如，两个产品标题栏下拉框的行为一致，但是下拉框内的菜单项不一致，比如文档有页面设置的菜单项，但是表格没有。又例如，两个产品鉴权、网络方面的逻辑一致，但是对文档模型处理方法不一致。

第三个问题是前端开发的老大难问题，即如何优雅地做**状态管理和逻辑复用**。

目前对于这个问题，社区已经提出了很多方案：

- mixin，Vue 社区比较多地采用这种方案，但是现在看来 mixin 并不是一种好的方案，它会导致隐式依赖、命名冲突等问题，React 官方已不推荐它，详细请看 Dan Abramov 的这篇文章 [Mixins Considered Harmful](https://zh-hans.reactjs.org/blog/2016/07/13/mixins-considered-harmful.html)。
- HOC，这是 React 之前推荐的方案，但这种方案同样不够理想，它会导致过多的标签嵌套，同样也会导致命名冲突。
- Hooks，这是现在 React 社区的主流方案，它解决了 mixin 和 HOC 的问题，但也有其局限性，例如只能用于函数式组件、一不留神就会导致多余的重复渲染等等。

当然，并没有银弹可以完美地解决这个问题，但是我们仍然需要量体裁衣，针对我们项目的情况和需求来探索新的模式。

第四个问题是**代码组织**。

产品逐渐复杂、代码量自然水涨船高，项目也随之腐化——比如大量的复制粘贴的代码、模块边界不清晰、文件和方法过长等等——结果导致维护成本剧增。

总结来说，我们需要一种机制：

- 分离平台相关代码，并以统一的接口给业务代码调用；
- **尽可能多地复用相同的 UI 和逻辑**，并且能够方便地处理不一致的部分；
- 提供一种新的状态管理和逻辑复用方式；
- 组织代码，让各个模块之间尽量解耦，提升代码可维护性。

在寻找解决方案的过程中，我从 vscode 和 Angular 这两个项目中获得了许多灵感，它们的共性是都使用了**依赖注入** (DI) 。

### vscode

之所以要学习 vscode 是因为它和我们的项目存在很多相似之处：都是编辑器应用、需要处理复杂的数据和状态、架构复杂、需要支持多平台等等。

vscode 的代码组织和运行机制都明显突出了依赖注入在这个项目中的核心位置：

- platform 目录下包含了 vscode 中的数十种服务（即依赖项）
- vscode 基于依赖注入模式构建，从第一个类 `CodeMain` 开始，DI 就被引入，所有功能都被划分到数十个 service 当中，以 DI 的方式给相关方使用。
- 平台差异也是通过 DI 处理的。（下面会有简单的讲解）

_想要详细了解可阅读我在阅读 vscode 源码时写的两篇笔记。[一](https://github.com/hullis/blog/issues/25)、[二](https://github.com/hullis/blog/issues/27)。_

### Angular

Angular 框架本身和使用 Angular 开发的应用都基于依赖注入：

- 依赖注入可以被用于装态管理和逻辑复用。逻辑上相关联的状态和方法被划分在各个类中，称为 service，service 可被注入到组件或其他 service 中。组件可以订阅 service 当中的状态（结合 RxJS），这样当 service 中的状态变更时，组件就会响应式地重渲染；当需要执行业务逻辑的时候，组件就可以调用 service 的方法。
- 组件可以通过依赖注入访问父组件或者同元素上其他指令的属性和方法。
- 框架提供的 HTTP 拦截器、路由鉴权接口也基于依赖注入。

那么，在 vscode 和 Angular 中大放异彩的依赖注入究竟是什么，为什么依赖注入可以解决文章开头提到的四个问题？

## 依赖注入

> 在[软件工程](https://zh.wikipedia.org/wiki/軟件工程)中，依赖注入是种实现[控制反转](https://zh.wikipedia.org/wiki/控制反转)用于解决依赖性设计模式。一个依赖关系指的是可被利用的一种对象（即服务提供端） 。依赖注入是将所依赖的传递给将使用的从属对象（即客户端）。该服务是将会变成客户端的状态的一部分。 传递服务给客户端，而非允许客户端来建立或寻找服务，是本设计模式的基本要求。

以上定义来自维基百科，而维基百科的特点就是不爱说人话。让我们换一种简单的表达：

**依赖注入就是不自行构造想要的东西（即依赖），而是声明自己想要的东西，让别人来构造。当这个构造过程发生在自己的构造阶段时，就叫做依赖注入。**

_如果你深入挖掘的话，你会发现依赖注入和依赖倒置、控制反转等概念相关。可以参考[这个](https://www.zhihu.com/question/265433666/answer/337599960)知乎回答。_

在依赖注入系统中，有三种主要角色：

- **依赖项 **(dependency)：任何能被消费者——主要是类（在前端框架中要加上组件）——所使用的东西，这些*东西*可能意味着类、值、函数、组件等等，依赖项通常会有一个**标识符**以和其他依赖项相区别，这个标识符可能是接口、类，也可能是某些特定的数据结构。和一个标识符绑定的依赖项应该具有相同的接口，这样消费者才能无差别（无感知）地使用。
- **提供者** (provider)：或称注入器 (injector)，它们会根据消费者的需要实例化和提供依赖项。
- **消费者** (consumer)：它们通过标识符从提供者获得依赖项，然后使用它们。一个消费者可能同时也是别的消费者的依赖项。

现在，你应该对依赖注入模式有一个大致的理解了，让我们来看看依赖注入如何解决文章开头提到的那些问题。

### 如何解决平台差异

要解决平台差异问题，就要将依赖于平台的代码和其他代码区别开来，让其他代码不需要知道当前在什么平台上。

这里以 vscode 的“拖拽文件到新目录前进行提示”功能为例，这一功能在应用中和在浏览器中的 UI 是不同的。

在 vscode 应用内它看起来是这样：

![Snipaste_2020-01-21_11-13-01](https://user-images.githubusercontent.com/12122021/72880052-1ef4a080-3d39-11ea-91a1-68e1db4b2cb3.png)

在浏览器端看起来像是这样：

![Snipaste_2020-01-21_11-14-27](https://user-images.githubusercontent.com/12122021/72880420-d8537600-3d39-11ea-9ac9-eb18579163e9.png)

实现方式就是将“弹出对话框”功能抽象成一个依赖项，在不同的平台注入不同的依赖项，这样调用者就不用关心当前的平台是什么了。

对于[这段代码](https://github.com/microsoft/vscode/blob/7ac39e13bd0501a152bb858ad353b08a340d4ca1/src/vs/workbench/contrib/files/browser/views/explorerViewer.ts#L902-L910)中的调用者来说，它并不知道也不需要知道目前的环境：

```ts
const confirmation = await this.dialogService.confirm({
  message,
  detail
  // ...
})
```

因为 `dialogService` 是被注入的，而[桌面端](https://github.com/microsoft/vscode/blob/master/src/vs/workbench/services/dialogs/electron-browser/dialogService.ts)和[浏览器端](https://github.com/microsoft/vscode/blob/master/src/vs/workbench/services/dialogs/browser/dialogService.ts)分别注入了不同的 service。

_详情请看第二篇关于 vscode 的博客。_

### 如何实现代码复用

解决第二个问题的思路其实和解决第一个的是一致的。我们只需要将只要将不一样的部分抽象成依赖项，然后让其余代码依赖它就可以了。

### 如何解决状态管理

依赖注入可以管理共享状态。将多个组件共享的状态提取到依赖项中并结合发布-订阅模式，就能实现直观的单项数据流；将能改变状态的方法放到依赖项中，就能直观地知道这些状态会如何改变；另外还可以方便地结合 RxJS 管理更复杂的数据流。这种方案

- 和 mixin 方案相比：
  - 它的依赖是显式的；
  - 不会导致命名冲突问题。
- 和 HOC 相比：
  - 不存在多重嵌套导致的 wrapper hell 问题；
  - 很容易追踪状态的位置和变更。
- 和 Hooks 相比：
  - 不用考虑 memorize 问题；
  - 用类来保存状态比 `setState` 等 API 更符合人类的思维直觉。
- 实现的状态管理是 scoped 的，如果你在界面上有很多个相似的模块（比如 Trello 的看板），依赖注入模式可以让你方便地管理各个模块的状态，确保它们之间不会错误地共享某些状态。

### 如何解决代码组织问题

依赖注入模式中的“依赖项”概念会强迫开发者思考哪些代码在逻辑上是相关联的，应该放到同一个类当中，从而让各个功能模块解耦；也会强迫开发者思考哪些是 UI 代码，哪些是业务代码，让 UI 和业务分开。并且，由于在依赖注入系统中，类的实例化过程（甚至包括销毁过程）是依赖注入框架完成的，因此开发者只需要关心功能应该划分到哪些模块中、模块之间的依赖关系如何，无需自己实例化一个个类，这样就降低了编码时的心智负担。最后，由于依赖注入使得类不用再负责构造自己的依赖，这样就能很方便地进行单元测试。

## wedi

为了能在 React 中方便地使用依赖注入模式，在重构的过程中，我实现了一个轻量的依赖注入库以及一组 React binding，现已开源。

[GitHub](https://github.com/hullis/wedi) / [npm](https://www.npmjs.com/package/wedi)

wedi 具有如下特性：

- 非侵入式：不像 Angular 那样一切都基于 DI，wedi 完全是 opt-in 的，你可以自己决定何时何处使用 DI。
- 简单易用：没有引入任何新概念。
- 同时支持 React 类组件和函数式组件。
- 支持层次化依赖注入
- 支持注入类、值（实例）、工厂函数三种类型的依赖项。
- 支持延迟实例化。
- 基于 TypeScript ，提供了良好的类型支持。

接下来我们结合几个具体的例子来讲解如何使用 wedi 。

### 在函数式组件中使用

当你需要提供依赖项的时候，只需要调用 `useCollection` 生成 collection，然后塞给 `Provider` 组件即可，`Provider` 的 children 就可以访问它们。

```tsx
import { useCollection } from 'wedi'

function FunctionProvider() {
  const collection = useCollection([FileService])

  return (
    <Provider collection={collection}>
      {/* children 可访问 collection 中的依赖项 */}
    </Provider>
  )
}
```

```tsx
import { useDependency } from 'wedi';

function FunctionConsumer() {
  const fileService = useDependency(FileService);

  return (
    /* 从这里开始可以调用 FileService 上的属性和方法 */
  );
}
```

_wedi 保证在函数式组件的 `Provider` 重渲染时不会重新构建依赖项，这样你就不会丢失依赖项里保存的状态。_

### 可选依赖

可以通过给 `useDependency` 传入第二个参数 `true` 来声明该依赖是可选的，TypeScript 会推断出返回值可能为 `null` 。如果未声明依赖项可选且获取不到该依赖项，wedi 会抛出错误。

```tsx
import { useDependency } from 'wedi'

function FunctionConsumer() {
  const nullable: NullableService | null = useDependency(NullableService, true)
  const required: NullableService = useDependency(NullableService) // Error!
}
```

### 在类组件中使用

当然 wedi 也支持传统的类组件。

当需要在某个组件及其子组件中注入依赖项时，使用 `Provide` 装饰器传递这些依赖项。

```ts
import { Inject, InjectionContext, Provide } from 'wedi';
import { IPlatformService } from 'services/platform';

@Provide([
  FileService,
  IPlatformService, { useClass: MobilePlatformService });
])
class ClassComponent extends Component {
  static contextType = InjectionContext;

  @Inject(IPlatformService) platformService!: IPlatformService;

  @Inject(NullableService, true) nullableService?: NullableService;
}
```

当需要使用这些依赖项时，需要将组件的默认 context 设置为 `InjectionContext`，然后就可以通过 `Inject` 装饰器获取依赖项了。同样，可以传入 `true` 给 `Inject` 声明依赖是可选的。

### 多种多样的依赖项

wedi 支持各种各样的依赖项，包括类，值、实例和工厂函数。

#### 类

有两种方法将一个类声明为依赖项，一是传递类本身，二是使用 `useClass` API 结合 identifier 。

```ts
const classDepItems = [
  FileService, // 直接传递类
  [IPlatformService, { useClass: MobilePlatformService }] // 结合 identifier
]
```

#### 值、实例

使用 `useValue` 注入值或实例。

```tsx
const valueDepItem = [IConfig, { useValue: '2020' }]
```

#### 工厂函数

使用 `useFactory` 注入工厂方法。

```ts
const factorDepItem = [
  IUserService,
  {
    useFactory: (http: IHTTPService): IUserService => new TimeSerialUserService(http, TIME)，
  deps: [IHTTPService]
  }
]
```

#### 注入组件

wedi 甚至可以注入组件：

```tsx
const IDropdown = createIdentifier<any>('dropdown')
const IConfig = createIdentifier<any>('config')

const WebDropdown = function () {
  const dep = useDependency(IConfig)
  return <div>WeDropdown, {dep}</div>
}

@Provide([
  [IDropdown, { useValue: WebDropdown }],
  [IConfig, { useValue: 'wedi' }]
])
class Header extends Component {
  static contextType = InjectionContext

  @Inject(IDropdown) private dropdown: any

  render() {
    const Dropdown = this.dropdown
    return <Dropdown></Dropdown> // WeDropdown, wedi
  }
}
```

这种方式可以满足在不同平台展现不同的 UI 的需求。

### 层次化的依赖注入

wedi 能够构建起层次化的依赖注入体系，wedi 在获取依赖项时，采取“就近原则”：

```tsx
@Provide([
  [IConfig, { useValue: 'A' }],
  [IConfigRoot, { useValue: 'inRoot' }]
])
class ParentProvider extends Component {
  render() {
    return <ChildProvider />
  }
}

@Provide([[IConfig, { useValue: 'B' }]])
class ChildProvider extends Component {
  render() {
    return <Consumer />
  }
}

function Consumer() {
  const config = useDependency(IConfig)
  const rootConfig = useDependency(IConfigRoot)

  return (
    <div>
      {config}, {rootConfig}
    </div> // <div>B, inRoot</div>
  )
}
```

这样你就可以使某些依赖项全局可用，而使另外一些依赖项范围可用，你可以利用这个特性方便地管理全局状态和局部状态。这对于界面上存在大量相同组件的应用特别合适。

你甚至可以通过 React Dev Tools 可视化地查看依赖项的可用范围（也就意味着状态的范围）。

![Snipaste_2020-01-22_17-44-12](https://user-images.githubusercontent.com/12122021/72883180-d3dd8c00-3d3e-11ea-921b-3814bc55f1c0.png)

_这个截图来自[用 wedi 构建的 TodoMVC](https://hullis.github.io/wedi-demo/)（开发环境下）。_

### 结合 RxJS

依赖注入模式可以很方便地与响应式编程相结合，用于状态管理。当一些组件之间需要共享状态时，你就可以把状态提取到各个组件都能访问到的依赖项当中，然后去订阅该状态的改变。

下面是一个计时器的例子。

```tsx
import { Provide, useDependency, useDependencyValue, Disposable } from 'wedi'

class CounterService implements Disposable {
  counter$ = interval(1000).pipe(
    startWith(0),
    scan((acc) => acc + 1)
  )

  // 如果有 dispose 函数，wedi 就会在组件销毁的时候调用它，这里你可以做一些 clean up 的工作
  dispose(): void {
    this.counter$.complete()
  }
}

function App() {
  const collection = useCollection([CounterService])

  return (
    <Provide collection={collection}>
      <Display />
    </Provide>
  )
}

function Display() {
  const counter = useDependency(CounterService)
  const count = useDependencyValue(counter.counter$)

  return <div>{count}</div> // 0, 1, 2, 3 ...
}
```

_更多关于 wedi 的 API 可关注 [README](https://github.com/hullis/wedi)。_
