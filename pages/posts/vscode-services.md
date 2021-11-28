---
title: vscode 源码解析 - 服务化
date: 2019/12/15
description: 介绍 vscode 服务化
tag: di, vscode, source code, Chinese
author: Wendell
---

# vscode 源码解析 - 服务化

![](https://user-images.githubusercontent.com/12122021/70677481-3c5e3580-1cca-11ea-9cc7-de8e131d3cb4.png)

---

## Introduction

[上一篇文章](https://wendell.fun/posts/vscode-di)介绍了 vscode 的依赖注入机制。

在 vscode 中，依赖注入主要用于**将服务注入到消费者对象当中**，**将一些基础能力提供给业务代码使用**。

我们来拆解一下这句话：

* **服务**。熟悉 Angular 的同学肯定也很熟悉这个概念。简单来说，服务就是对逻辑上相关联的数据和方法（函数）的封装，让这部分逻辑可复用。比如，如果我们通过将处理 HTTP 请求的发送、处理、中间件、拦截、取消、去重等等的逻辑封装到一个名为 `HTTPService` 的服务当中，这样在使用 HTTP 的时候，我们就不需要再去担心具体的实现问题了。从这个例子也可以看出，服务实际上也是一种解耦（decoupling）和关注分离（SOC）的手段。
* **消费者对象**。既然有服务，那么肯定有其他对象来使用这些服务，这些对象我们就称为消费者对象，它们往往和业务直接相关，在 vscode 中可以理解为是用户能够使用到的功能。
* **注入**。消费者对象在使用服务的时候，并不是自己构造一个服务对象（仔细品味：如果它知道要具体要构造哪个类，它实际上也就知道了这个类的实现细节），而是从某个服务注册机构（通常叫做注入器）得到一个服务对象，只要这个对象满足它的接口的要求就可以了，这个过程往往是在消费者对象构造时完成的，所以被称为**注入**。这实际上是一种控制反转（IOC）。

*如果你对上面的某些概念不是很理解，你可以稍后去学习它们，这里的铺垫已经足够你理解本文的全部内容。*

到这里我们了解了**服务**的含义，接下来就来看看 vscode 中用到了哪些服务吧！

*我们之后才会讲到 Electron 的双进程架构。这里画了一个简图来方便你理解下面文章的内容，仅仅用于表达各个对象间的层次关系。*

```pla
-------------------------------------    ------------------------------------------
                                                         *Workbench*
-------------------------------------    ------------------------------------------
      Window ----electron.BrowserWindow.load()----> *Desktop Main / BrowserMain*
-------------------------------------    ------------------------------------------
           *CodeApplication*
-------------------------------------    ------------------------------------------
              *CodeMain*
-------------------------------------    ------------------------------------------
             Main Process                              Renderer Process
-----------------------------------------------------------------------------------
                                    Electron
-----------------------------------------------------------------------------------
```

## 主线程中的服务

### CodeMain

`CodeMain` 是 vscode 主线程中最底层的类，[它在初始化时会创建以下这些服务](https://github.com/microsoft/vscode/blob/3dea65ba1bb14e3df6e9d10e2c667ef4bd2b4180/src/vs/code/electron-main/main.ts#L135)：

* `InstantiationService` 我们在上一篇文章中就知道了这个类是负责实现依赖注入的
* `EnvironmentService` 环境变量服务，保存了诸如根目录、用户目录、插件目录等信息
* `MultiplexLogService` 多级日志服务 
* `ConfigurationService` 
* `LifecycleMainSercice` 生命周期服务，封装了 Electron 的一写生命周期事件，使得消费者能够在这些生命周期钩子里做一些事情
* `StateService` 状态服务，它负责对 vscode 的 db 的读写
* `RequestMainService` 请求服务，负责发起 http 请求，背后调用的是 node 的 https 和 http 等模块
* `ThemeMainService` 负责编辑器主题相关
* `SignService` 应用签名服务

等。

`CodeApplication` [会创建以下服务](https://github.com/microsoft/vscode/blob/3dea65ba1bb14e3df6e9d10e2c667ef4bd2b4180/src/vs/code/electron-main/app.ts#L426)：

* `FileService` 文件存取
* `WindowMainService` 用于管理 vscode 的所有的窗口（打开、关闭、激活等等）
* `UpdateService` 根据运行平台的不同分别注入 `Win32UpdateService` `DarwinUpdateService` 等，负责应用程序的更新
* `DialogMainService` 对话框管理
* `ShareProcessMainService` 用于跨进程通讯
* `DiagnosticsService` 应用运行性能诊断
* `LaunchMainService`
* `IssueMainService`
* `ElectronMainService`
* `WorkspacesService` 工作区管理服务
* `MenubarMainService` 菜单栏管理服务
* `StorageMainService` 存储
* `BackupMainService` 备份
* `WorkspacesHistoryMainService` 工作区历史
* `URLService` URL 解析
* `TelementryService`

等。

## 渲染进程中的服务

### DesktopMain

`DesktopMain` 是 vscode 渲染进程中最底层的类，它虽然自己并没有创建 `InitailizationService`，但是它[创建了一个 `service` 集合](https://github.com/microsoft/vscode/blob/46a15a90549fda7d60597e275aed2134068f0edb/src/vs/workbench/electron-browser/desktop.main.ts#L171)并将这个集合传递给 `Workbench`，由 `Workbench` 创建了 `InitializationService`。

在这一层次上提供的服务有：

* `MainProcessService` 用于和主进程进行通讯
* `ElectronEnvironmentService` Electron 环境变量
* `WorkbenchEnvironmentService` Workbench 环境变量
* `ProductService`
* `LogService` 日志
* `RemoteAuthorityService`
* `SignService`
* `RemoteAgentService`
* `FileService` 文件存储服务

等。

### Workbench

Workbench 实际上就是我们能看到的 vscode 工作区的 UI。它会创建一个 `InstantiationService`，除了将从 `DesktopMain` 传递来的依赖注入项保存起来之外，它还要将**全局单例注入项**保存到 `InstantiationService` 当中，[代码](https://github.com/microsoft/vscode/blob/ef4e7cfda38e9ce00e076e4f69b3ce053cb7ac84/src/vs/workbench/browser/workbench.ts#L192-L197)如下：

```ts
const contributedServices = getSingletonServiceDescriptors();
for (let [id, descriptor] of contributedServices) {
    serviceCollection.set(id, descriptor);
}

const instantiationService = new InstantiationService(serviceCollection, true);
```

*我们在[上一篇文章](https://github.com/wendellhu95/blog/issues/25)讲过 vscode 的全局单例注入。*

那么究竟有哪些服务会被注入进来呢？这其实是在入口文件中确定的。

在桌面端的 vscode 中，入口文件为 [workbench.js](https://github.com/microsoft/vscode/blob/3dea65ba1b/src/vs/code/electron-browser/workbench/workbench.js)，从中可以看到引入了脚本 [vs/workbench/workbench.desktop.main](vs/workbench/workbench.desktop.main)，而这个脚本在全局注册了很多服务（即 `#region --- workbench services` 里面的内容），另外通过引入 [workbench.common.main.ts](https://github.com/microsoft/vscode/blob/3dea65ba1b/src/vs/workbench/workbench.common.main.ts)，还引入了很多服务（注意 `#region --- workbench parts` 里面的内容也是依赖注入项且和 UI 相关）。而在浏览器端的 vscode 中，入口文件则为 [workbench.html](https://github.com/microsoft/vscode/blob/46a15a9054/src/vs/code/browser/workbench/workbench.html)，引入的主要脚本则是 [vs/workbench/workbench.web.main](vs/workbench/workbench.web.main)。

由于 `Workbench` 引入的全局单例服务实在是太多了，这里我们仅仅列举几个，感兴趣的话可以到入口文件中去查看：

* `NativeLifeCycleService` 这个服务封装了 Electron 窗口 `onBeforeUnload` 和 `onWillUnload` 的回调，让 vscode 的其余部分可以在窗口即将关闭之前做一些 clean up 的工作。
* TextMateService 用于代码高亮
* NativeKeymapService 用于处理不同语言键盘布局 keycode 不同的问题
* ExtensionService 管理拓展
* ContextMenuService 上下文菜单

等等。

### 为什么是依赖注入？

到这里，我们就对 vscode 中常用到的服务有哪些，它们是如何注入的，以及它们被注入的位置等问题有了一个大致上的认识。接下来的问题是，**为什么 vscode 要使用依赖注入的方式来组织代码呢**？

对于 vscode 来说，使用依赖注入模式有以下这些好处：

一、**繁杂的功能点借助依赖注入被合理划分到不同的服务中**，在横向上降低了模块间的耦合度，提高了模块内聚性。如果想要修改某些功能，很容易就能知道去哪里查找相关代码；对某个模块的修改，不会影响其他模块的开发。

二、**消费者和服务通过接口解耦**，对于服务消费者来说，它只要求被注入的类符合它的接口要求就可以了，并用不关心注入项究竟是如何实现的，即在纵向上降低了耦合度（其实就是依赖反转 DIP），这使得 vscode 的架构十分灵活，能够通过提供不同的服务来做到一些神奇的事情。

如果你有关注 vscode 的动态，那么你肯定知道今年他们搞的一个大动作就是推出了在完全在浏览器环境中运行的 [Visual Studio Code Online](https://code.visualstudio.com/docs/remote/vsonline)（你可以通过在 vscode 项目中执行 `yarn web` 脚本启动它）。

vscode 基于 Electron，所以可以访问一些桌面端才有的 module，但是在浏览器环境下并没有这样的模块。以 `FileService` 为例，[在 Electron 中](https://github.com/microsoft/vscode/blob/3dea65ba1bb14e3df6e9d10e2c667ef4bd2b4180/src/vs/workbench/electron-browser/desktop.main.ts#L212 )它需要 [fs module](https://github.com/microsoft/vscode/blob/3dea65ba1bb14e3df6e9d10e2c667ef4bd2b4180/src/vs/platform/files/node/diskFileSystemProvider.ts#L6)，因此它注册的是一个 `diskFileSystemProvider`，

```ts
const diskFileSystemProvider = this._register(new DiskFileSystemProvider(logService));
fileService.registerProvider(Schemas.file, diskFileSystemProvider);
```

但是在浏览器中我们不能使用模块，所以，[在 vscode online 中](https://github.com/microsoft/vscode/blob/46a15a90549fda7d60597e275aed2134068f0edb/src/vs/workbench/browser/web.main.ts#L249)，`FileService` 注册的是一个 `remoteFileSystemProvider`。

```ts
const channel = connection.getChannel<IChannel>(REMOTE_FILE_SYSTEM_CHANNEL_NAME);
const remoteFileSystemProvider = this._register(new RemoteFileSystemProvider(channel, remoteAgentService.getEnvironment()));
fileService.registerProvider(Schemas.vscodeRemote, remoteFileSystemProvider);
```

但对于 `FileService` 的消费者即 `DesktopMain` 来说，它并不需要（也不应该）知道这种差别，它只要按照自己的需要调用符合 `IFileService` 接口的服务就好了。

再以“拖动文件位置前弹出对话框”功能为例，它在 Electron 和浏览器中展现出不同的样式：

![在 Electron 中](https://user-images.githubusercontent.com/12122021/72893435-f29a4d80-3d53-11ea-9603-570aed2a5949.png)

![在浏览器中](https://user-images.githubusercontent.com/12122021/72893405-e0b8aa80-3d53-11ea-9506-890628234467.png)

但是业务层不需要了解各个平台上如何创建 dialog，只需要调用 IDialogService 提供的方法就可以了：

```ts
const confirmation = await this.dialogService.confirm({
  message: items.length > 1 && items.every(s => s.isRoot) ? localize('confirmRootsMove', "Are you sure you want to change the order of multiple root folders in your workspace?")
    : items.length > 1 ? getConfirmMessage(localize('confirmMultiMove', "Are you sure you want to move the following {0} files into '{1}'?", items.length, target.name), items.map(s => s.resource))
      : items[0].isRoot ? localize('confirmRootMove', "Are you sure you want to change the order of root folder '{0}' in your workspace?", items[0].name)
        : localize('confirmMove', "Are you sure you want to move '{0}' into '{1}'?", items[0].name, target.name),
  checkbox: {
    label: localize('doNotAskAgain', "Do not ask me again")
  },
  type: 'question',
  primaryButton: localize({ key: 'moveButtonLabel', comment: ['&& denotes a mnemonic'] }, "&&Move")
});
```

在打包不同平台上的 vscode 时，注入不同的 IDialogService：

```ts
import 'vs/workbench/services/dialogs/electron-browser/dialogService';
```

```ts
import 'vs/workbench/services/dialogs/browser/dialogService';
```

总的来说，想要让 vscode 在浏览器中运行，只需要修改被注入的服务，然后通过不同的打包入口（已在上文中介绍）引入这些服务，无须修改上层代码。

三、依赖注入模式也带来了**软件工程方面的一些好处**。

* 依赖注入是一种得到时间锤炼、十分成熟的技术，开发者们很容易就能理解它，这使得架构清晰易懂，上手速度快。
* 方便分工，开发团队成员可以进行明确的分工，只要在编码时严格遵守接口的要求，就无需担心会搞坏队友的代码。
* 能够充分利用 TypeScript 提供的类型信息在代码之间快速跳转。

## Conclusion

* vscode 在主进程中和渲染进程中都通过依赖注入的方式给业务代码注入了很多基础服务
* 通过在不同的代码入口中引入符合同一接口的不同服务，vscode 实现了跨平台（桌面端、web）运行

这篇文章展示了 vscode 如何利用依赖注入系统提供各种基础功能来服务业务代码，对需要支持多平台的大型应用提供了一个优秀的模板。
