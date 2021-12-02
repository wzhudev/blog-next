---
title: Angular CDK Portal 源码解析
date: 2019/10/16
description: 探究 Angular CDK Portal 的实现
tag: Angular, Angular CDK, portal, source code, Chinese
author: Wendell
---

# Angular CDK Portal 源码解析

[Portal](https://github.com/angular/components/blob/master/src/cdk/portal/portal.md) 用于在任意位置动态渲染模板或组件。

> Portal 指的是需要动态创建的内容（模板或者组件），而 PortalOutlet 指的是渲染这些内容的位置。

## 目录结构

```sh
portal
├── BUILD.bazel
├── dom-portal-outlet.ts // DomPortalOutlet
├── index.ts
├── portal-directives.ts // 包含所有的指令，使得你能够以声明式的使用方式来使用 Portal
├── portal-errors.ts // 包含所有的错误信息
├── portal-injector.ts // 你可以临时创建一个 Injector 给 Portal，从而干预依赖注入
├── portal.md
├── portal.spec.ts
├── portal.ts // 定义了核心部分的几个类
├── public-api.ts
└── tsconfig-build.json
```

最重要的文件有以下三个：

1. portal.ts，这个文件定义了抽象类 `Portal` 和 `BasePortalOutlet`，还定义了类 `ComponentPortal` 和 `TemplatePortal`。
2. portal-directive.ts，这个文件里定义了指令 `CdkPortal` 和 `CdkPortalOutlet`，让我们可以以声明式的方式使用 portal，后者还定义了动态创建内容的机制。
3. dom-portal-outlet.ts，这个文件里定义了 `DomPortalOutlet`，使得 portal 可以被渲染到 Angular 的组件树之外（这一点被 overlay 模块所使用，之后会 cover 到这部分内容）。

## 核心机制

以 README 中的示例来讲解这部分代码是如何工作的：

```ts
this.userSettingsPortal = new ComponentPortal(UserSettingsComponent)
```

```html
<!-- Attaches the `userSettingsPortal` from the previous example. -->
<ng-template [cdkPortalOutlet]="userSettingsPortal"></ng-template>
```

示例中的代码首先创建了一个 `ComponentPortal`，那我们就先来看 `ComponentPoral` 和其父类 `Portal`。

[Portal](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/portal.ts#L36-L77) 这个类其实非常简单，用于挂载和卸载 portal 的几个所做的事情基本都是：检查边界情况，然后调用 `PortalOutlet` 的对应方法。

[ComponentPortal](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/portal.ts#L83-L114) 这个类也非常简单，它仅仅是对动态创建组件所需要的数据结构的一个封装。这些数据结构包括：

- `component`
- `viewContainerRef`
- `injector`
- `componentFactory`

后面三个参数在构造一个 `ComponentPortal` 的时候都是可选的，注意这一点，之后在讲解动态渲染过程中就会了解到为啥是可选的。

示例中的代码到这里，就会给 `cdkPortalOutlet` 赋值这个新创建的 `ComponentPortal`。

```html
<ng-template [cdkPortalOutlet]="userSettingsPortal"></ng-template>
```

我们再来看 `cdkPortalOutlet` 和其父类 `BasePortalOutlet` 的代码。

[BasePortalOutlet](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/portal.ts#L182-L261) 有以下要点：

- 定义了 `attach` 方法，该方法会根据需要挂载的不同类型的 portal，来调用`attachComponentPortal` 和 `attachTemplatePortal` 两个方法。
- 声明了 `attachComponentPortal` 和 `attachTemplatePortal` 两个方法。为什么这里没有做实现？是因为对于在 Angular 组件树中的 outlet 和不在组件树中的 outlet，实现是不同的，我们将会看到这一点。
- 支持了 portal 卸载回调，通过 `setDisposeFn`，子类可以设置 portal 卸载的时候需要触发的回调函数。

[cdkPortalOutlet](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/portal-directives.ts#L66-L173) 有以下要点：

- 首先这是一个结构型指令，依赖项里有 `ComponentFactoryResolver` 和 `ViewContainerRef`。
- 有一个 input `cdkPortalOutlet`，被赋值到 `portal` 属性上。
- `portal` [被赋值的时候](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/portal-directives.ts#L89-L107)，调用父类的 `attach` 方法。

在例子里，我们的 portal 是一个 `ComponentPortal`，这里我们就只分析该情形，即 `attachComponentPortal` [被调用](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/portal-directives.ts#L134-L155)的情形。

- 准备调用 `createComponent` 函数所需要的参数，总而言是就是如果 Portal 没有携带这些参数，就用 PortalOutlet 的（组件树中的）位置所能 get 到的参数（呼应了我们前面讲过的要点）。
- 调用 `ViewContainerRef` 的 `createComponent` 方法动态创建一个组件。
- [设置了一个 portal 卸载回调](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/portal-directives.ts#L149)，在 portal 卸载的时候，动态创建的组件也会被销毁。

到这里，就是 Portal 机制的主要运行过程了。

## 其他

接下来我们 cover 一些之前没有 cover 到的要点。

### cdkPortal

[cdkPortal](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/portal-directives.ts#L25-L37) 允许使用者以声明式的方式创建一个 `TemplatePortal`，它的代码非常简单，仅仅是把 `TemplatePortal` 变成了一个结构型指令。

### DomPortalOutlet

我们看到 `cdkPortalOutlet` 是以声明式方式使用的，这意味着它必须在 Angular 的组件树当中，如果我们想把 portal 挂载到组件树之外的位置，就需要用 `DomPortalOutlet`。

可以看到 `DomPortalOutlet` 和 `cdkPortalOutlet` 有以下不同：

- 它无法以声明方式使用，必须指令式地构造，[构造函数的第一个参数](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/dom-portal-outlet.ts#L26)，是要挂载的 DOM 元素。
- [在动态创建组件的时候](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/dom-portal-outlet.ts#L38-L67)，如果 portal 没有带 `ViewContainerRef`，就调用 `ComponentFactory` 的 `create` 方法直接得到组件的 view，挂载在应用的根 view 上，然后手动 append DOM 元素。
- 当 portal 卸载的时候，也需要[手动 detach view](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/dom-portal-outlet.ts#L58)。

以上不同的根源都是挂载到组件树之外的 portal 可能会没有 `ViewContaienerRef`。

### PortalInjector

在创建 ComponentPortal 的时候[我们可以传入一个 Injector](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/portal.ts#L106)，而 [PortalInjector](https://github.com/angular/components/blob/master/src/cdk/portal/portal-injector.ts) 允许我们对这个 Injector 作出干预，增加新的 provider。

可以看到它所做的全部事情，其实就是在返回依赖项的时候，[检查用户而外提供的 provider 里有没有符合依赖注入令牌的](https://github.com/angular/components/blob/c8ceae5b2fe10cf3fedbe7ac81ed7d2dc19efab5/src/cdk/portal/portal-injector.ts#L21-L29)。

## 总结

我们都知道通过 Angular 的 `ViewContrainerRef` 提供的 `createEmbedeView` 和 `createComponent` 方法就可以动态创建界面内容，为什么还需要 CDK 提供的 Portal 呢？通过上文的分析，我们可以得出使用 Portal 的一些好处：

- 除了可以方便的创建动态内容，以及移除这部分内容。
- 可以声明式地使用。
- 可以指定 `ViewContainerRef` 和 `Injector` 等，从而改变变更检测的顺序，依赖注入项的获取等等，更加灵活。
- 可以在 Angular 的组件树之外的位置创建内容。

Portal 被 Angular CDK 的其他很多模块所采用。
