---
title: 时隔一年回顾 Icon 组件库的开发
date: 2019/12/13
description: 对 ng-zorro-antd icon 升级 SVG 的总结
tag: icon, ng-zorro-antd, Chinese
author: Wendell
---

# 时隔一年回顾 Icon 组件库的开发

> 图标是 UI 设计中必不可少的组成部分。通常我们理解图标设计的含义，是将某个概念转换成清晰易读的图形，从而降低用户的理解成本，提升界面的美观度。

Ant Design 有一套[成熟的图标设计规范](https://ant.design/docs/spec/icon)。作为它的 Angular 实现，[ng-zorro-antd](https://ng.ant.design/) 提供了数以百计的图标给开发者们使用。

之前这些图标都被封装在一个字体文件中，但在 1.7.0 版本之后，ng-zorro-antd 开始在它的底层图标库 [@ant-design/icons-angular](https://www.npmjs.com/package/@ant-design/icons-angular) 中使用 SVG 技术。

相对于基于字体文件的图标，例如 [Font Awesome](https://fontawesome.com/)（当然它现在也支持 SVG 图标了），SVG 具有许多优点，比如在低分辨率屏幕上显示效果更好，支持多种颜色（ng-zorro-antd 提供了对[双色图标](https://ng.ant.design/components/icon/en## ##  components-icon-demo-twotone)的支持），并且它可以被打包在 JavaScript 文件中让用户无须发起额外请求去获得字体文件，等等，但是它也有一个不容忽视的致命缺陷：

**SVG 太特喵的大了。**

![https://miro.medium.com/max/3840/0*YFupRTSUmB8qV3n_.png](https://miro.medium.com/max/3840/0*YFupRTSUmB8qV3n_.png)

*如果不谨慎对待 SVG 的体积问题，结果可能会很糟糕。这个例子来自[这里](https://github.com/ant-design/ant-design/issues/12011## issuecomment-418021842)。*

字体文件实际上是二进制位文件，所以它的体积很小。但是 SVG 实际上就是纯文本文件（只是后缀名是 .svg），尽管你可以使用 gzip 压缩 SVG（并且主流的 web 服务器也会帮你这么做），但是它还是很大，会占用很多网络传输带宽。所以我们面临的挑战就是：

**如何在不造成破坏性更新的前提下以最低的开销把这些 icon 送到用户的浏览器上。**


这篇文章介绍了我们当时是如何解决这一问题的，以及作为它的主要开发者，我本人在做这个项目中得到的一些感想。如果你也在打造一个类似的库，或者仅仅是对我们的工作感兴趣，请继续阅读 ;)

##  两种方案

你可能想到下面两种方案：

第一种方案是：**我们把所有的图标都打包到组件库里**，就像以前用字体图标时那样。但是 [antd](https://ant.design/) 的这个 [issue](https://github.com/ant-design/ant-design/issues/12011) 警醒我们千万不要这么做：用户并不想在他们的包里多出 500KB 无法被 tree shake 的，当然最主要的是自己用不着的代码。React 社区提出了一些替代方案，比如[这个](https://github.com/ant-design/ant-design/issues/12011## issuecomment-549652300)，但是在我们看来，使用这些方法甚至比我接下来要介绍的第二种方案还要麻烦。

第二种方案是：**我们让用户自己决定要打包哪些图标**，这样的话就不会把用不着的图标打包进去了。这种实现非常简洁，逻辑上也是最说得通的。但是，我们也不能采纳这种方案，因为我们的用户都已经习惯了使用图标而无须事先引用它们（这多亏了字体文件的体积很小我们才能这么做）。这种方案将会迫使用户添加大量 `import XXXIcon` 这样的代码，才能让项目重新运行，即引入了破坏性更新，这也是我们无法接受的。（顺便推荐一下最近看到的[一篇文章](https://medium.com/angular-in-depth/how-to-create-an-icon-library-in-angular-4f8863d95a)，介绍了方案二的具体实现。）

情况愈发复杂了，我们似乎走进了死胡同：

- 一方面，把所有图标都打包进去的代价实在太大，用户不希望我们这么做
- 另一方面，我们必须把所有图标都打包进去，因为我们不知道用户会用哪些图标

所以，真的无路可走了吗？（显然不是，毕竟我们的新图标组件都 release 一年多了 😄）

如果你听说过 [Idle while Urgent](https://philipwalton.com/articles/idle-until-urgent/) 或者 iOS 社区的 [Lazy Initialization](http://andyarvanitis.com/lazy-initialization-for-objective-c) 的话，你可能会想到这里蕴含了一个更通用的模式，即在你需要使用某项资源的时候，再去创建这项资源。同理，我们可以在需要渲染图标的时候，再去加载图标资源。

我们将这种方案称为“动态加载”。

##  第三种方案

*你可以在[这里](https://github.com/hullis/ant-design-icons/tree/master/packages/icons-angular)找到 @ant-design/icons-angular 的源代码。*

动态加载的意思就是：当我们需要渲染一个图标，但是这个图标还未加载的时候，我们就从服务器端获取这个图标，缓存它，再进行渲染。

接下来我会通过带你阅读源码来了解这个机制是如何实现的。别担心，我们不会去看那些繁杂的细节，而是专注于主要流程。

当我们需要[渲染图标](https://github.com/hullis/ant-design-icons/blob/585354361dc80ba24a974492ce13a645e749a538/packages/icons-angular/lib/component/icon.service.ts## L168)的时候，会先去缓存里查找这个图标是否加载了，如果没有的话，我们就通过调用 `_loadIconDynamically` 方法加载该图标。

```ts
// If `icon` is a `IconDefinition` of successfully fetch, wrap it in an `Observable`.
// Otherwise try to fetch it from remote.
const $iconDefinition = definitionOrNull
  ? rxof(definitionOrNull)
  : this._loadIconDynamically(icon as string);
```

发起的[请求](https://github.com/hullis/ant-design-icons/blob/585354361dc80ba24a974492ce13a645e749a538/packages/icons-angular/lib/component/icon.service.ts## L235-L237)仅仅是向服务器请求图标的内容（其实就是一个字符串），然后将图标名称和其内容[封装](https://github.com/hullis/ant-design-icons/blob/585354361dc80ba24a974492ce13a645e749a538/packages/icons-angular/lib/component/icon.service.ts## L237)成 icon 对象，并[缓存](https://github.com/hullis/ant-design-icons/blob/585354361dc80ba24a974492ce13a645e749a538/packages/icons-angular/lib/component/icon.service.ts## L241)。

```ts
const source = !this._enableJsonpLoading
  ? this._http
      .get(safeUrl, { responseType: 'text' })
      .pipe(map(literal => ({ ...icon, icon: literal }))) // assemble an icon object, type as IconDefinition
  : this._loadIconDynamicallyWithJsonp(icon, safeUrl); // jsonp-like loading

inProgress = source.pipe(
  tap(definition => this.addIcon(definition)),
  finalize(() => this._inProgressFetches.delete(type)), // delete the request object
  catchError(() => rxof(null)),
  share() // share with other subscribers
);
```

这样我们就得到了图标，可以继续之前的渲染过程了！ 🎊🎊🎊

**但这还远远不够，还有很多细节问题需要注意。**

如果我们要同时渲染很多相同的图标，为每一个图标发起 HTTP 请求的话开销就太大了，所以我们应该在每个渲染流程之间[分享](https://github.com/hullis/ant-design-icons/blob/585354361dc80ba24a974492ce13a645e749a538/packages/icons-angular/lib/component/icon.service.ts## L212)请求（这个请求就是个名为 `inProgress` 的流）。有一个 [share 操作符](https://github.com/hullis/ant-design-icons/blob/585354361dc80ba24a974492ce13a645e749a538/packages/icons-angular/lib/component/icon.service.ts## L244)将会负责把图标对象广播给所有的[订阅者](https://github.com/hullis/ant-design-icons/blob/585354361dc80ba24a974492ce13a645e749a538/packages/icons-angular/lib/component/icon.directive.ts## L41)。并且在请求完成之后我们要[移除](https://github.com/hullis/ant-design-icons/blob/585354361dc80ba24a974492ce13a645e749a538/packages/icons-angular/lib/component/icon.service.ts## L242)这个请求流。

![https://miro.medium.com/max/3048/1*b4lnigsma-eueVGsup6WvA.png](https://miro.medium.com/max/3048/1*b4lnigsma-eueVGsup6WvA.png)

我们怎么知道从哪里去加载图标呢？多亏了 [Angular/CLI](https://cli.angular.io/)，我们可以通过提供一个 schematic 来帮助用户添加图标资源到他们的网站静态资源文件夹中。当用户通过命令 `ng add ng-zorro-antd` 安装 zorro 时，zorro 会询问用户是否需要修改 angular.json 文件以便添加图标资源。

如果用户想从 CDN 加载图标呢？我们必须让加载图标的 URL 变成可配置的。所以我们提供了一个名为 `changeAssetsSource` 的[方法](https://github.com/hullis/ant-design-icons/blob/585354361dc80ba24a974492ce13a645e749a538/packages/icons-angular/lib/component/icon.service.ts## L128-L130)用于修改 URL 前缀。

如果 CDN 禁用了跨域 XML 请求呢？我们提供了一种[类似于 JSONP 的加载机制](https://github.com/hullis/ant-design-icons/blob/585354361dc80ba24a974492ce13a645e749a538/packages/icons-angular/lib/component/icon.service.ts## L253)帮助开发者们绕过跨域问题，可以通过调用 `useJsonpLoading` 方法开启。

等等。所以你可以看出，即使核心概念非常简单，我们也必须考虑用户可能面临的多种场景，即使部分场景我们自己可能都不会遇到。

除了动态加载，还有很多工作要做：

1. 上面说到的第二种方案很棒，我们也想支持它，所以我们提供了[静态加载](https://ng.ant.design/components/icon/en## static-loading-and-dynamic-loading)方案。
2. 需要一些[脚本](https://github.com/hullis/ant-design-icons/blob/master/packages/icons-angular/build/generate.ts)来生成图标资源。
3. 我们之前的图标 API 就是没有 API，真的。用户们想要渲染图标的时候仅需要写一个有特定 class 的 `i` 标签，比如 `<i class="anticon anticon-clock">`，所以我们还要兼容这种基于类名的 API，我们使用了 [MutationObserver](https://github.com/NG-ZORRO/ng-zorro-antd/blob/97eb720afdce9606fcea95ad74d70c9fa1b7c5fa/components/icon/nz-icon.directive.ts## L233)。
4. 需要支持[旋转、自定义图标、命名空间和 iconfont 等](https://ng.ant.design/components/icon/zh## %5Bnz-icon%5D)多种功能。
5. 撰写文档（**十分重要**）。

最终，我写作了 @ant-design/icons-angular 和 ng-zorro-antd 的新 Icon 组件。

![https://miro.medium.com/max/2560/0*sF7medQ3PrHC_76y.png](https://miro.medium.com/max/2560/0*sF7medQ3PrHC_76y.png)

@ant-design/icons-angular 作为 ng-zorro-antd 的底层依赖，提供了图标资源，以及静态加载、动态加载、jsonp 加载和命名空间等基础功能。

ng-zorro-antd 的 Icon 组件则负责脏活（比如适配 API），以及提供旋转等扩展功能。

![https://miro.medium.com/max/5712/1*c2W1ILxsKRI0kcoucpmrCQ.png](https://miro.medium.com/max/5712/1*c2W1ILxsKRI0kcoucpmrCQ.png)


##  结论

在 2018 年 10 月发布的 1.7.0 版本中，我们发布了全新的图标系统，并且我写作了[升级指南](https://github.com/NG-ZORRO/ng-zorro-antd/issues/2304)来解释为什么我们要用新图标替换旧图标，以及用户们要迁移到新图标应当做什么。令人高兴的是绝大部分用户接受了这一次的重大变更并顺利完成了升级（感谢！）。

总结一下，我们做到了什么？

- 我们使用了 SVG 来渲染图标，并尽可能少地向浏览器传输 SVG 代码
- 我们帮助用户平滑升级，避免了破坏式更新

好极了！我们以一种优雅的方式解决了开头提到的问题。

“**性能**呢？”你可能会发出这样的疑问。

实际上，我们的 Icon 的文档页会一次渲染 300 多个图标，证明了动态加载并不会造成严重的性能问题。事实上，当你只使用少数几个图标的时候，访问者几乎感觉不到图标是动态加载的。如果你的网页是一个 PWA 的话，你就完全无须担心这个问题！因为 PWA 会将这样的请求进行本地缓存（就和我们的官网一样）。

作为开源项目的维护者，以下是我在这个项目中学到的一些东西：

1. **谨慎思考你做的变更会给用户带来怎样的影响**，避免破坏性变更，适配旧的 API 让用户有充足的时间去升级，严格遵守[语义化版本规范](https://semver.org/)。
2. **避免把状况搞得难以收拾，更不要替你的用户做决定**，相反，让用户们有得选择。引入所有的 icon 就是一种不太明智的做法，这会让不想这么做的用户大伤脑筋。
3. **在做出最终决定前要三思**，多问自己“还有更好的解决方案吗”。

以上就是本文的全部内容，感谢阅读！
