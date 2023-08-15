---
title: vscode 源码解析 - 依赖注入
date: 2019/12/11
description: 介绍 vscode 依赖注入系统的实现
tag: di, vscode, source code, Chinese
author: Wenzhao
---

# vscode 源码解析 - 依赖注入

![cover](https://user-images.githubusercontent.com/12122021/70677481-3c5e3580-1cca-11ea-9cc7-de8e131d3cb4.png)

_最近在阅读 [vscode](https://github.com/Microsoft/vscode) 的源代码。我将会在 blog 中逐步公开一些整理过的学习笔记和大家交流。_

在阅读这篇文章之前，你需要[依赖注入模式](https://en.wikipedia.org/wiki/Dependency_injection)有基本的了解。和依赖注入相关的项目包括 [Angular](https://angular.io/)、[NestJS](https://nestjs.com/) 和 [InversifyJS](https://github.com/inversify/InversifyJS)。

---

## Introduction

vscode 项目中，对象基本都是通过依赖注入模式构造的。比如编辑器实例 [CodeApplication](https://github.com/microsoft/vscode/blob/efc8667b4c6d3021c0fdbfbce9495ae925ca2e28/src/vs/code/electron-main/app.ts#L84) 的 constructor 如下，所有被装饰的参数都是依赖注入项：

_如果你还不了解 TypeScript 装饰器，你可以先阅读[官方文档](https://www.typescriptlang.org/docs/handbook/decorators.html)。_

```ts
export class CodeApplication extends Disposable {
  constructor(
    private readonly mainIpcServer: Server,
    private readonly userEnv: IProcessEnvironment,
    @IInstantiationService
    private readonly instantiationService: IInstantiationService,
    @ILogService private readonly logService: ILogService,
    @IEnvironmentService
    private readonly environmentService: IEnvironmentService,
    @ILifecycleMainService
    private readonly lifecycleMainService: ILifecycleMainService,
    @IConfigurationService
    private readonly configurationService: IConfigurationService,
    @IStateService private readonly stateService: IStateService
  ) {
    // ...
  }
}
```

`CodeMain` 类将会在应用初始化的时候[实例化该类](https://github.com/microsoft/vscode/blob/68db7fc3f79e4fa8438343d9040d6b8aff6020d1/src/vs/code/electron-main/main.ts#L128)：

```ts
await instantiationService.invokeFunction(async (accessor) => {
  // ...

  // 进行实例化，可以看到除了要被构造的类 CodeApplication 之外
  // 剩下参数的数目和 constructor 中未被装饰的参数的数目一致
  return instantiationService
    .createInstance(CodeApplication, mainIpcServer, instanceEnvironment)
    .startup()
})
```

我们提炼出在 vsocde 中使用依赖注入模式的三个要素：

- 一个将要被实例化的**类**，在其**构造函数**中使用装饰器声明了需要注入的参数（依赖注入项）
- **装饰器**，是注入的参数的**类型标识**（indentifier）
- `InstantiationService`，提供方法实例化类，并且也是依赖注入项所存放的位置

下面介绍一些实现细节。

---

## 实现

### 装饰器

所有 identifier 均由 [createDecorator]() 方法创建，比如 [IInstantiationService]()。

```ts
export const IInstantiationService = createDecorator<IInstantiationService>(
  'instantiationService'
)
```

_TypeScript 允许同名的类型声明和变量声明，这就是为什么 `IInstantiation` 同时可以作为装饰器函数的名称和接口的名称。_

`createDecorator` 方法的内容如下：

```ts
export function createDecorator<T>(serviceId: string): ServiceIdentifier<T> {
  // 装饰器会被缓存
  if (_util.serviceIds.has(serviceId)) {
    return _util.serviceIds.get(serviceId)!
  }

  // 装饰器
  const id = <any>function (target: Function, key: string, index: number): any {
    if (arguments.length !== 3) {
      throw new Error(
        '@IServiceName-decorator can only be used to decorate a parameter'
      )
    }
    storeServiceDependency(id, target, index, false)
  }

  id.toString = () => serviceId

  _util.serviceIds.set(serviceId, id)
  return id
}

function storeServiceDependency(
  id: Function,
  target: Function,
  index: number,
  optional: boolean
): void {
  // 在被装饰的类上记录一个依赖项
  if ((target as any)[_util.DI_TARGET] === target) {
    ;(target as any)[_util.DI_DEPENDENCIES].push({ id, index, optional })
  } else {
    ;(target as any)[_util.DI_DEPENDENCIES] = [{ id, index, optional }]
    ;(target as any)[_util.DI_TARGET] = target
  }
}
```

可以看到代码的核心是实现了一个装饰器函数 `id`，在装饰器被应用的时候，它就会调用 `storeServiceDependency` 方法在被装饰的类（比如 `CodeApplication`）上记录依赖项，包括装饰器本身（`id`），参数的下标（`index`）以及是否可选（`optional`）。

当类声明的时候，装饰器就会被应用（我做了一个简单的[证明](https://stackblitz.com/edit/typescript-decorator-when?file=index.ts)），所以在有类被实例化之前依赖关系就已经确定好了。

### InstantiationService

[InstantiantionService](https://github.com/microsoft/vscode/blob/efc8667b4c6d3021c0fdbfbce9495ae925ca2e28/src/vs/platform/instantiation/common/instantiationService.ts#L29) 用于提供依赖注入项，也就是起到依赖注入框架中的**注入器**（Injector）的功能，它以 `identifier` 为 key 在自身的 [\_services](https://github.com/microsoft/vscode/blob/efc8667b4c6d3021c0fdbfbce9495ae925ca2e28/src/vs/platform/instantiation/common/instantiationService.ts#L33) 属性中保存了各个依赖项的实例。

它暴露了三个公开方法：

1. `createInstance`，该方法接受一个类以及构造该类的非依赖注入参数，然后创建该类的实例。
2. `invokeFunction`，该方法接受一个回调函数，该回调函数通过 acessor 参数可以访问该 InstantiationService 中的所有依赖注入项。
3. `createChild`，该方法接受一个依赖项集合，并创造一个新的 InstantiationService，说明 vscode 的依赖注入机制也是有层次的。

`_createInstance` 方法是实例化的核心方法：

```ts
	private _createInstance<T>(ctor: any, args: any[] = [], _trace: Trace): T {

		// arguments defined by service decorators
		let serviceDependencies = _util.getServiceDependencies(ctor).sort((a, b) => a.index - b.index);
		let serviceArgs: any[] = [];
		for (const dependency of serviceDependencies) {
			let service = this._getOrCreateServiceInstance(dependency.id, _trace);
			if (!service && this._strict && !dependency.optional) {
				throw new Error(`[createInstance] ${ctor.name} depends on UNKNOWN service ${dependency.id}.`);
			}
			serviceArgs.push(service);
		}

		let firstServiceArgPos = serviceDependencies.length > 0 ? serviceDependencies[0].index : args.length;

		// check for argument mismatches, adjust static args if needed
		if (args.length !== firstServiceArgPos) {
			console.warn(`[createInstance] First service dependency of ${ctor.name} at position ${
				firstServiceArgPos + 1} conflicts with ${args.length} static arguments`);

			let delta = firstServiceArgPos - args.length;
			if (delta > 0) {
				args = args.concat(new Array(delta));
			} else {
				args = args.slice(0, firstServiceArgPos);
			}
		}

		// now create the instance
		return <T>new ctor(...[...args, ...serviceArgs]);
	}
```

这个方法首先通过 [getServiceDependencies](https://github.com/microsoft/vscode/blob/efc8667b4c6d3021c0fdbfbce9495ae925ca2e28/src/vs/platform/instantiation/common/instantiation.ts#L18) 获取被构造类的依赖，这里获取到的依赖就是我们在声明该类的时候就已经通过 `storeServiceDependency` 所注册的（见上文）。

然后通过 `_getOrCreateServiceInstance` 根据方法 `indentifer` 拿到 `_services` 中注册的依赖项，如果拿不到的话就构建一个（我们先假设我们总是能拿到所需要的依赖注册项，需要现场构建的情形我们会在后面的小节中说明），拿到的依赖项会被 push 到 `serviceArgs` 数组当中。

然后会进行 constructor 参数处理。总而言之，args 数组的长度应该满足被构造的类声明的非注入参数的数量，这样才能确保依赖注入的参数和非依赖注入的参数都能被送到构造函数中正确的顺序上。

最后用实例化目标类。

### 依赖项不存在的情形

我们先前提到在调用 `_getOrCreateServiceInstance` 时可能会拿不到依赖注入项而需要现场构建一个，下面是具体的实现过程。

首先会调用 `_getServiceInstanceOrDescriptor` 尝试拿到已经注册的实例，或者是一个 [SyncDescriptor](https://github.com/microsoft/vscode/blob/ba2524065b199509f40eaff1cfe20ed9f9725751/src/vs/platform/instantiation/common/descriptors.ts#L8) 对象。`SyncDescriptor` 是什么东西呢，它其实就是封装了实例构造参数的一个数据对象，包括以下属性：

- `ctor` 将要被构造的类
- `staticArguments` 被传入这个类的参数，和上文中的 args 意义相同
- `supportsDelayedInstantiation` 是否支持延迟实例化

使用起来就像这样：

```ts
services.set(ILifecycleMainService, new SyncDescriptor(LifecycleMainService))
```

表示的其实就是 **“不立刻实例化这个类，而当需要被注入的时候再进行实例化”**。

拿到了 `SyncDescriptor` 之后，会通过 [\_createAndCacheServiceInstance](https://github.com/microsoft/vscode/blob/efc8667b4c6d3021c0fdbfbce9495ae925ca2e28/src/vs/platform/instantiation/common/instantiationService.ts#L149) 方法先实例化这个依赖项，它的代码如下：

```ts
	private _createAndCacheServiceInstance<T>(id: ServiceIdentifier<T>, desc: SyncDescriptor<T>, _trace: Trace): T {
		type Triple = { id: ServiceIdentifier<any>, desc: SyncDescriptor<any>, _trace: Trace };
		const graph = new Graph<Triple>(data => data.id.toString());

		let cycleCount = 0;
		const stack = [{ id, desc, _trace }];
		while (stack.length) {
			const item = stack.pop()!;
			graph.lookupOrInsertNode(item);

			// a weak but working heuristic for cycle checks
			if (cycleCount++ > 150) {
				throw new CyclicDependencyError(graph);
			}

			// check all dependencies for existence and if they need to be created first
			for (let dependency of _util.getServiceDependencies(item.desc.ctor)) {

				let instanceOrDesc = this._getServiceInstanceOrDescriptor(dependency.id);
				if (!instanceOrDesc && !dependency.optional) {
					console.warn(`[createInstance] ${id} depends on ${dependency.id} which is NOT registered.`);
				}

				if (instanceOrDesc instanceof SyncDescriptor) {
					const d = { id: dependency.id, desc: instanceOrDesc, _trace: item._trace.branch(dependency.id, true) };
					graph.insertEdge(item, d);
					stack.push(d);
				}
			}
		}

		while (true) {
			const roots = graph.roots();

			// if there is no more roots but still
			// nodes in the graph we have a cycle
			if (roots.length === 0) {
				if (!graph.isEmpty()) {
					throw new CyclicDependencyError(graph);
				}
				break;
			}

			for (const { data } of roots) {
				// create instance and overwrite the service collections
				const instance = this._createServiceInstanceWithOwner(data.id, data.desc.ctor, data.desc.staticArguments, data.desc.supportsDelayedInstantiation, data._trace);
				this._setServiceInstance(data.id, instance);
				graph.removeNode(data);
			}
		}

		return <T>this._getServiceInstanceOrDescriptor(id);
	}
```

这里有两个 `while`，分别做了以下这几件事情：

1. 第一个 `while` 是利用 DFS 的方法，找到一个类的所有未实例化的依赖（还是基于 `SyncDescriptor`），以及依赖的未实例化的依赖……最终得到一个依赖图
2. 第二个 `while` 根据前一步得到的依赖图，从根节点开始构造实例

最后我们就得到了我们最初想要的依赖。

_其实利用递归也可以实现。_

### 依赖图

![](https://user-images.githubusercontent.com/12122021/94639446-c38aa180-030e-11eb-9cfc-49fada58147a.png)

vscode 在实例化类及其依赖（以及依赖的依赖）的时候并没有使用简单的递归方法，而是利用 Graph 来管理类之间的依赖关系。

假设我们要实例化 `A` 类，vscode 会监测到它需要依赖 `B` 和 `C` 类，并将依赖关系记录到图当中，图中的一个 `Node` 的 outgoing 属性，就代表了依赖关系。如果这两类都没有实例化，就会重复对这两个类进行此步操作，直到所有类的依赖项都被实例化（或者是没有依赖项为止），比如 `D` 类，它没有依赖项。

之后，vscode 会从根结点开始将这些类一一实例化，所谓根节点，也就是没有 outgoing 的节点，即没有依赖项的节点。

### 全局单例依赖注入

在 vscode 中，有的依赖是全局唯一、单例的，即在 JavaScript 线程中该类最多只有一个实例（这在 render process 中用得非常多，我们以后会讲解什么是 render process）。vscode 提供了一个简单的机制实现全局单例依赖。

比方说我们想要创建一个单例的生命周期依赖，[就这样做](https://github.com/microsoft/vscode/blob/ef4e7cfda38e9ce00e076e4f69b3ce053cb7ac84/src/vs/workbench/services/lifecycle/browser/lifecycleService.ts#L66)：

```ts
registerSingleton(ILifecycleService, BrowserLifecycleService)
```

即调用 `registerSingleton` 方法，将 `identifier` 和具体的实现类绑定即可。

而 [registerSingleton](https://github.com/microsoft/vscode/blob/ef4e7cfda38e9ce00e076e4f69b3ce053cb7ac84/src/vs/platform/instantiation/common/extensions.ts#L11) 的实现也异常简单，仅仅是在一个数组中保存一条记录。

```ts
const _registry: [ServiceIdentifier<any>, SyncDescriptor<any>][] = []

export function registerSingleton<T, Services extends BrandedService[]>(
  id: ServiceIdentifier<T>,
  ctor: { new (...services: Services): T },
  supportsDelayedInstantiation?: boolean
): void {
  _registry.push([
    id,
    new SyncDescriptor<T>(ctor, [], supportsDelayedInstantiation)
  ])
}

export function getSingletonServiceDescriptors(): [
  ServiceIdentifier<any>,
  SyncDescriptor<any>
][] {
  return _registry
}
```

在[需要用到这些依赖注入项的时候](https://github.com/microsoft/vscode/blob/ef4e7cfda38e9ce00e076e4f69b3ce053cb7ac84/src/vs/workbench/browser/workbench.ts#L192)，调用 `getSingletonServiceDescriptor` 获取这个数组就好。

所以从本质上来说，全局单例依赖注入其实就是把所有的依赖注入项保存在一个全局变量里。

### 可选依赖

有时候我们想让一个依赖是可选的，即允许依赖不存在。对此 vscode 提供了 optional 方法用于标记可选依赖。

```ts
export function optional<T>(serviceIdentifier: ServiceIdentifier<T>) {
  return function (target: Function, key: string, index: number) {
    if (arguments.length !== 3) {
      throw new Error(
        '@optional-decorator can only be used to decorate a parameter'
      )
    }
    storeServiceDependency(serviceIdentifier, target, index, true)
  }
}
```

可见它与 `createDecorator` 方法的主要区别在于在调用 `storeServiceDependency` 的时候第四个参数为 `true`。这样当获取不到 `serviceIdentifier` 所对应的依赖项时，`InstantiationService` 能够允许这样的情况而不是抛出错误。

## 延迟实例化

在上文中，我们看到 `SyncDecriptor` 可以被当作依赖项实例的占位符使用，从而做到在需要依赖它的类被实例化的时候，再进行自身的实例化，即延迟实例化。另外，它还能**把实例化过程进一步延迟到访问实例的属性和方法的时候**！我们来看看这是如何实现的。

当创建一个 `SyncDescriptor` 的时候我们可以传参 `supportsDelayedInstantiation = true`，比如[这里](https://github.com/microsoft/vscode/blob/efbcd7f563320982ac133ddd72d45c286ead48c4/src/vs/workbench/workbench.common.main.ts#L113)：

```ts
registerSingleton(IExtensionGalleryService, ExtensionGalleryService, true)
```

这样在调用 `_createServiceInstance` 的时候就会进入 [else 分支](https://github.com/microsoft/vscode/blob/efbcd7f563320982ac133ddd72d45c286ead48c4/src/vs/platform/instantiation/common/instantiationService.ts#L218-L243)。

```ts
		} else {
			// Return a proxy object that's backed by an idle value. That
			// strategy is to instantiate services in our idle time or when actually
			// needed but not when injected into a consumer
			const idle = new IdleValue<any>(() => this._createInstance<T>(ctor, args, _trace));
			return <T>new Proxy(Object.create(null), {
				get(target: any, key: PropertyKey): any {
					if (key in target) {
						return target[key];
					}
					let obj = idle.getValue();
					let prop = obj[key];
					if (typeof prop !== 'function') {
						return prop;
					}
					prop = prop.bind(obj);
					target[key] = prop;
					return prop;
				},
				set(_target: T, p: PropertyKey, value: any): boolean {
					idle.getValue()[p] = value;
					return true;
				}
			});
		}
	}
```

原理是用一个 Proxy 代替实例返回，当需要用到实例上的属性或方法时，再调用 this.\_createInstance 方法（通过闭包来保存参数）。这里用到了一种被称为 [Idle Until Urgent](https://www.notion.so/wendzhue/vscode-6302a2030d264d5bbb9bb24fbd1092fc#d92374d275004bccb7d33ecfa930eecc) 的模式。

## InstantiationService 的方法

`InstatiationService` 这个类有很多方法，而且名字都很接近，这里列一个梗概，方便大家阅读源码：

- `createChild` 创建一个子 `InstantiationService`
- `invokeFunction` 执行一个函数，该函数可以通过 accessor 访问 `InstantiationService` 里存储的服务
- `createInstance` 创建一个服务
- `_createInstance` 实例化的最终方法，`new` 调用的位置
- `_setServiceInstance` 将一个创建好的 service set 到保存了对应的 identifier 的 `InstantiationService` 当中
- `_getServiceInstaneOrDescriptor` 根据 identifier 从某个 `InstantiationService` 中拿到服务实例或者 `SyncDescriptor`
- `_getOrCreateServiceInstance` 被 `invokeFunction` 所调用，会尝试调用 `_getServiceInstanceOrDescriptor` 拿到服务实例，如果拿到的是一个 `SyncDescriptor`，则走 `_createAndCacheServiceInstance`
- `_createAndCacheServiceInstance` 这里根据“要被创建的服务”的“未被实例化的依赖”来构建依赖树，然后依次构建这些未被实例化的依赖
- `_createServiceInstanceWithOwner` 寻找保存了对应的 identifier 的 `InstantiationService` ，调用它的 `_createServiceInstance` 方法进行实例化
- `_createServiceInstance` 这里处理延迟实例化逻辑，调用 `_createInstance` 的时候，所有依赖应该都已经被实例化，而不是 `SyncDescriptor`

## Conclusion

- vscode 自己实现了一套依赖注入机制，并没有依赖 [reflect-metadata](https://www.npmjs.com/package/reflect-metadata)（知乎上的[这篇文章](https://zhuanlan.zhihu.com/p/96041706)对此有误）
- `InstantiationService` 是实现依赖注入的核心
- 用装饰器来声明依赖关系
- 允许可选依赖
- 允许延迟实例化
- 支持多层依赖注入

这篇文章 cover 了 vscode 依赖注入模式的主干部分，能帮助我们理解 vscode 是如何管理各种各样的服务的，也对自行实现依赖注入提供了一些有益的参考。
