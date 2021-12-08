---
title: vscode 源码解析 - vscode loader
date: 2021/04/17
description: 介绍 vscode 脚本加载器
tag: vscode, source code, Chinese, ipc
author: Wendell
---

# vscode 源码解析 - vscode-loader

![](https://user-images.githubusercontent.com/12122021/70677481-3c5e3580-1cca-11ea-9cc7-de8e131d3cb4.png)

在 vscode 的加载过程中，有这样一行代码值得注意，它位于 main.js 文件中，而这是 Electron App 运行的入口文件：

```tsx
require('./bootstrap-amd').load('vs/code/electron-main/main', () => {
  // ...
})
```

vs/code/election-main/main 是 vscode 应用的主入口，这句代码的意思是加载主入口文件并执行。bootstrap-amd 文件暴露的 load 方法则又调用了 vs/loader 文件暴露的 loader 方法：

```tsx
const loader = require('./vs/loader')

exports.load = function (entrypoint, onLoad, onError) {
  // ...
  loader([entrypoint], onLoad, onError)
}
```

这个 vs/loader 文件中的内容，即是 vscode 自己实现的模块加载系统 vscode-loader，这篇文章我们来学习它的设计和实现。

---

vscode-loader 的源代码不在 vscode 仓库当中，源码地址在 [https://github.com/microsoft/vscode-loader](https://github.com/microsoft/vscode-loader%E3%80%82)，使用 TypeScript 编写，通过 TypeScript compiler 生成最终以下几个文件：

- loader.js，主要文件，负责加载 js 文件，并通过 plugin 机制支持 CSS 和自然语言文件（nls，国际化文件）的加载（这篇文章我们不会讲解 plugin 机制）
- css.build.js，负责在构建时处理 CSS 文件；当 vscode 运行在浏览器当中时，为了减少 js 文件加载的开销，使用构建后的 js 文件，css.build.js 就是在构建过程中处理 CSS 文件用的，下面的 nls.build.js 同
- css.js，以插件的形式加载 CSS 文件
- nls.build.js
- nls.js，以插件的形式加载 nls 文件，即 vscode 的国际化资源文件

## 理解模块化系统

在分析 vscode-loader 的代码之前，我们先从概念上理解一下模块化系统。

现在前端开发基本都不会将所有的代码全部写在一个文件里，而是分散在多个文件当中，这样一个文件就可能会需要引用其他文件中声明的标识符。JavaScript 的灵活性允许我们将这些标识符定义在顶层作用域中然后在另一个文件中访问它，例如：

```ts
// file 'a.js'
var a = 2

// file 'b/js'
console.log(a)
```

只要在 HTML 引用 js 文件的先后顺序是 a.js → b.js，上面这段代码就能正常输出 2。

但是这种方法在开发稍有规模的 JavaScript 项目的时候，显然是不理想的：

1. 很难保证各个文件中声明的全局变量的名称都是不一样的，即很难保证不会发生命名冲突
2. 文件没法创建一个私有的、不能被其他文件所访问的变量
3. 也很难知道一个文件中引用的全局作用域标识符是在哪个文件中定义的。

模块化机制就是为解决以上问题而提出的。在 ECMAScript 6 标准推出 [ESM](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import) 之前，JavaScript 这门语言是没有内置的模块化机制的，社区提出了 AMD [UMD](https://github.com/umdjs/umd/blob/master/templates/amdWeb.js) [CommonJS](https://en.wikipedia.org/wiki/CommonJS) 这三种主要的方案（实际上 UMD 只是对 AMD 和 CommonJS 的一个封装而已），虽然他们的语法上和运行原理上各有不同之处，但是核心概念都是一样的：变量仅能够在一定的**范围**（即作用域）被访问到，这个范围就叫做**模块**，模块要想访问其他模块中的变量，就需要**引入**对方**导出**的变量。我们以 ESM 为例：

```ts
// file 'a.js'
const a = 2

export function getA() {
  return a
}

// file 'b.js'
import { getA } from 'a'

console.log(getA())
```

在 a.js 文件中，有两个变量 `a` 和 `getA`，其中 `getA` 通过 export 语法被**导出**。而 b.js 文件**导入**了 a.js 所导出的 `getA` 并调用，注意：它无法导入 `a`，于是 `a` 就变成 a.js 文件所**私有**的了，我们可以看到，模块化代码不会将变量泄漏到全局环境当中去。

这里一个**模块**的范围就是一个文件。有些规范可以允许一个文件内含有多个模块，比如 AMD 规范，这时模块的范围就是一个函数的作用域。

我们仔细看上面的例子，从实现的角度思考模块化系统，容易发现一个模块化系统就需要做到下面这几点：

- **知道模块之间的依赖关系，并按照依赖关系来顺序执行代码**；例如上面的 b.js 代码的执行调用了 a.js 文件导出的 `getA` 函数，因此 a.js 文件要先于上面的代码先被解析执行；
- **需要支持模块导出标识符给其他模块使用**；例如 a.js 文件能够导出一个 `getA` 函数，就需要通过某种机制传递到 b.js 文件的上下文中去；
- **需要支持将一些标识符封存起来，不暴露给别的模块使用**；例如 a 不能够被 b.js 所访问到。

vscode（大部分的情况下）使用的是 [AMD](https://github.com/amdjs/amdjs-api/blob/master/AMD.md) 规范，vscode 的 src/tsconfig.base.json 里可以看到相关配置， 编译出来的文件像是这样：

```ts
define([
  'require',
  'exports',
  'vs/base/common/lifecycle',
  'vs/base/common/actions',
  'vs/base/browser/dom',
  'vs/base/common/types',
  'vs/base/browser/keyboardEvent',
  'vs/base/common/event',
  'vs/base/browser/ui/actionbar/actionViewItems',
  'vs/css!./actionbar'
], function (
  require,
  exports,
  lifecycle_1,
  actions_1,
  DOM,
  types,
  keyboardEvent_1,
  event_1,
  actionViewItems_1
) {
  'use strict'
  // ...
})
```

我们还是用上面的例子来进行分析。AMD 规范中，声明一个模块的语法如下（注意，这个例子不带有 AMD 对 CommonJS 规范的兼容（即 `"require"`），因此显得比较简单）：

```ts
define('a', [], () => {
  const a = 2

  function getA() {
    return a
  }

  return { getA }
})

define('b', ['a'], (a) => {
  console.log(a.getA())
})
```

其中第一个参数为模块的名称，第二个参数是模块需要引入的其他模块，或者称当前模块的**依赖**（dependency），第三个参数是回调函数，当模块的依赖都已经加载完毕之后就会执行这个回调函数来加载当前模块，回调函数的参数是各个依赖导出的变量，函数的内容则是模块的代码，返回的对象则是这个模块要导出的所有变量。可以看到：通过将变量定义在回调函数的作用域中，可避免把变量泄露到全局环境，而返回的变量（即模块的导出），则可以通过闭包来访问内部变量。

上面的例子中，通过 return 返回了导出变量，除此之外 AMD 还支持一种导出的方式，就是先声明当前模块依赖 `exports`，然后把要导出的变量作为 `exports` 变量的属性，以上面的 a 文件为例：

```ts
define('a', ['exports'], (exports) => {
  const a = 2

  function getA() {
    return a
  }

  exports.getA = getA // 这样导出也是可以的
})
```

至此我们了解了 AMD 规范的基本内容，接下来要了解的就是 define 方法是如何工作的，本文余下的内容即介绍 vscode 中的模块加载工具 vscode-loader 的工作原理。

## vscode-loader 的初始化

打开 vscode-loader 的源代码中 main.ts 文件，文件最后的部分即是 vscode-loader 执行的入口：

```ts
export function init(): void {
  if (typeof global.require !== 'undefined' || typeof require !== 'undefined') {
    // 将原本 node.js 的 require 函数保存在这个局部变量中
    const _nodeRequire = global.require || require
    if (
      typeof _nodeRequire === 'function' &&
      typeof _nodeRequire.resolve === 'function'
    ) {
      // 然后在 RequireFunc 上挂在原来的 require 函数
      const nodeRequire = ensureRecordedNodeRequire(
        moduleManager.getRecorder(),
        _nodeRequire
      )
      global.nodeRequire = nodeRequire
      ;(<any>RequireFunc).nodeRequire = nodeRequire
      ;(<any>RequireFunc).__$__nodeRequire = nodeRequire
    }
  }

  // 在 Node.js 环境中（非 Electron 渲染进程环境）
  if (env.isNode && !env.isElectronRenderer) {
    module.exports = RequireFunc // vscode-loader.js 文件导出 RequireFunc
    require = <any>RequireFunc // patch 全局 require 函数
  } else {
    if (!env.isElectronRenderer) {
      global.define = DefineFunc // patch 全局 define 函数
    }
    global.require = RequireFunc // patch 全局 require 函数
  }
}

if (typeof global.define !== 'function' || !global.define.amd) {
  moduleManager = new ModuleManager(
    env,
    createScriptLoader(env),
    DefineFunc,
    RequireFunc,
    Utilities.getHighPerformanceTimestamp()
  )

  // The global variable require can configure the loader
  if (
    typeof global.require !== 'undefined' &&
    typeof global.require !== 'function'
  ) {
    RequireFunc.config(global.require)
  }

  // This define is for the local closure defined in node in the case that the loader is concatenated
  define = function () {
    return DefineFunc.apply(null, arguments)
  }
  define.amd = DefineFunc.amd

  if (typeof doNotInitLoader === 'undefined') {
    init()
  }
}
```

1. 创建了一个 ScriptLoader，即脚本加载器，脚本加载器因代码运行环境而异，有 `NodeScriptLoader` `WorkScriptLoader` 和 `BrowserScriptLoader` 三种，负责加载 js 文件
2. 定义了 `DefineFunc` 和 `RequireFunc`，即模块化系统中的 define 函数和 require 函数，分别用于定义和加载一个模块
3. 创建了一个 `ModuleManager`，即模块管理器，它用于按照正确的顺序来解析 js 文件并执行
4. 覆盖了全局作用域中的 `define` 以及 `require` 变量（node 原生 `require` 变量的值被保存在 `_nodeRequire` 变量中）

我们接下来分别探究这几项内容，先从 `DefineFunc` 和 `RequireFunc` 开始。

## DefineFunc RequireFunc

### RequireFunc

对 Node.js 有所了解的话，一定会知道 Node.js 的模块规范是 CommonJS，引入资源的时候使用 require 语法：

```ts
const fs = require('fs')
```

而我们之前看到 vscode patch 了 Node.js 的 `require` 函数，那么也需要对原来的模块化系统进行兼容，另外，这个 require 还要能够接入 vscode 自己的 AMD 模块系统：

```ts
const RequireFunc: IRequireFunc = <any>function () {
  if (arguments.length === 1) {
    // 配置
    if (arguments[0] instanceof Object && !Array.isArray(arguments[0])) {
      _requireFunc_config(arguments[0])
      return
    }
    // Node.js 原本的 require() 语法
    if (typeof arguments[0] === 'string') {
      return moduleManager.synchronousRequire(arguments[0])
    }
  }
  // vscode AMD 模块化系统调用方式
  if (arguments.length === 2 || arguments.length === 3) {
    if (Array.isArray(arguments[0])) {
      moduleManager.defineModule(
        Utilities.generateAnonymousModule(),
        arguments[0],
        arguments[1],
        arguments[2],
        null
      )
      return
    }
  }
  throw new Error('Unrecognized require call')
}
```

`RequireFunc` 除了可以接收一个配置对象之外，能够用两种方式进行调用：

1. 第一种，参数是一个字符串，这里直接是 `moduleManager.synchronousRequire` 方法的一个代理，用于同步地调用一个已经加载完毕的依赖；
2. 第二种，要求第一个参数是一个数组，数组里是需要加载的脚本，第二和第三个参数则是加载和失败的回调。这样的调用方式，会调用 `moduleManager.defineModule` 定义一个匿名的模块，然后这些需要加载的脚本就会作为这个匿名模块的依赖而被加载。我们先前看到的 bootstrap-amd 中加载主进程入口的代码即使用了这种调用方式。

```tsx
const loader = require('./vs/loader')

exports.load = function (entrypoint, onLoad, onError) {
  // ...
  loader([entrypoint], onLoad, onError)
}
```

### RequireFunc

```ts
const DefineFunc: IDefineFunc = <any>(
  function (id: any, dependencies: any, callback: any): void {
    if (typeof id !== 'string') {
      callback = dependencies
      dependencies = id
      id = null
    }
    if (typeof dependencies !== 'object' || !Array.isArray(dependencies)) {
      callback = dependencies
      dependencies = null
    }
    if (!dependencies) {
      dependencies = ['require', 'exports', 'module']
    }

    if (id) {
      moduleManager.defineModule(id, dependencies, callback, null, null)
    } else {
      moduleManager.enqueueDefineAnonymousModule(dependencies, callback)
    }
  }
)
```

与 `RequireFunc` 类似，除去一些调整参数的判断之外，`DefineFunc` 的核心就是根据是否有模块 id，来决定是调用 `defineModule` 方法或者是 `enqueueDefineAnonymousModule` 方法。

## ModuleManager

`RequireFunc` 和 `DefineFunc` 的内容都较为简单，主要是调用 `ModuleManager` 的方法，看来它才是所有复杂逻辑发生的地方。接下来我们就来看这个类的实现，代码在 moduleManager.ts 文件当中。

### 模块的表示

我们需要一个数据结构来表示模块，vscode-loader 中即为 `Module` 类型，它的主要属性如下：

```ts
export class Module {
  public readonly id: ModuleId // 字母表示的模块 id
  public readonly strId: string // 字符串形式的模块 id
  public readonly dependencies: Dependency[] | null // 这个模块的依赖

  private readonly _callback: any | null // 即模块的具体内容的代码
  private readonly _errorback:
    | ((err: AnnotatedError) => void)
    | null
    | undefined
  public readonly moduleIdResolver: ModuleIdResolver | null

  public exports: any // 保存这个模块导出的内容
  public error: AnnotatedError | null
  public exportsPassedIn: boolean
  public unresolvedDependenciesCount: number // 尚未加载的依赖的数量
  private _isComplete: boolean // 模块似乎加载完成
}
```

一个模块的依赖即是其他的模块，表示依赖的数据结构很简单，就是一个封装了 `id` 的对象：

```ts
export class RegularDependency {
  public readonly id: ModuleId
}
```

在定义一个模块的时候 vscode-loader 会为每一个 `Module` 分配一个不同的数字 id，这个数字 id 的类型就是 `ModuleId`。vscode 的模块化系统有三个内置的依赖，因此会占据 0 1 2 三个数字。至于这些内置依赖对象有什么用处，我们后面会进行说明。

```ts
export const enum ModuleId {
  EXPORTS = 0,
  MODULE = 1,
  REQUIRE = 2
}
```

### ModuleManager 的数据结构

`ModuleManager` 即是模块加载器，它有以下这些重要的属性：

```ts
export class ModuleManager {
  // 环境信息
  private readonly _env: Environment
  // 脚本加载器，负责加载 js 脚本文件
  private readonly _scriptLoader: IScriptLoader
  private readonly _defineFunc: IDefineFunc
  private readonly _requireFunc: IRequireFunc

  // 管理 Module id
  private _moduleIdProvider: ModuleIdProvider
  private _config: Configuration

  // 是否存在循环依赖
  private _hasDependencyCycle: boolean

  /**
   * 一个从 id 到 Module 的映射
   * 如果一个 id 在该数组中，则说明对应的 Module 已经开始加载过程
   * 但不一定已经加载完成，因为可能有依赖没有加载完成
   */
  private _modules2: Module[]

  /**
   * 一个从 id 到 true 的映射
   * 如果一个 id 在该数组中，说明 script loader 已经开始装载该 module 所需的代码
   * 这个机制可以防止同一份代码被加载两次
   */
  private _knownModules2: boolean[]

  /**
   * 一个从 id 到 ModuleId[] 的映射，这个映射记录了哪些模块依赖编号为 id 的模块
   */
  private _inverseDependencies2: (ModuleId[] | null)[]
}
```

### defineModule 的运行过程

经过上面知识的铺垫，我们可以来学习最为重要的 `defineModule` 方法了，我们这里忽略错误处理和构建情况，专注于其核心机制的实现。

```ts
public defineModule(strModuleId: string, dependencies: string[], callback: any, errorback: ((err: AnnotatedError) => void) | null | undefined, stack: string | null, moduleIdResolver: ModuleIdResolver = new ModuleIdResolver(strModuleId)): void {
	let moduleId = this._moduleIdProvider.getModuleId(strModuleId);

	let m = new Module(moduleId, strModuleId, this._normalizeDependencies(dependencies, moduleIdResolver), callback, errorback, moduleIdResolver);
	this._modules2[moduleId] = m;

	this._resolve(m);
}
```

`defineModule` 方法主要做了下面这几件事情：

1. 调用 `_normalizeDependencies` 对依赖进行处理，这个过程比较简单，实际上就是根据依赖项的标识字符串生成一个 `RegularDependency` 对象
2. 创建一个 `Module` 对象
3. 将 `Module` 记录到 `_moduels2` 数组当中
4. 调用 \_resolve 方法尝试解析该模块

`_resolve` 方法负责解析一个模块。首先它会处理 `Module` 所有的 dependency，查看各个 dependency 是否已经解析完毕。如果有 dependency 没有解析，那么就通过 `_loadModule` 方法加载该 dependency，这个我们下一个小节会详叙述。

```ts
private _resolve(module: Module): void {
	let dependencies = module.dependencies;
	if (dependencies) {
		for (let i = 0, len = dependencies.length; i < len; i++) {
			let dependency = dependencies[i];

			// 如果发现当前模块依赖 exports 则会将 exportsPassedIn 置为 true
			// 这会影响到之后如何处理从模块导出的内容
			if (dependency === RegularDependency.EXPORTS) {
				module.exportsPassedIn = true;
				module.unresolvedDependenciesCount--;
				continue;
			}

			if (dependency === RegularDependency.MODULE) {
				module.unresolvedDependenciesCount--;
				continue;
			}

			if (dependency === RegularDependency.REQUIRE) {
				module.unresolvedDependenciesCount--;
				continue;
			}

			// 依赖已经加载
			let dependencyModule = this._modules2[dependency.id];
			if (dependencyModule && dependencyModule.isComplete()) {
				module.unresolvedDependenciesCount--;
				continue;
			}

			// 检测循环依赖
			if (this._hasDependencyPath(dependency.id, module.id)) {
				this._hasDependencyCycle = true;
				console.warn('There is a dependency cycle between \'' + this._moduleIdProvider.getStrModuleId(dependency.id) + '\' and \'' + this._moduleIdProvider.getStrModuleId(module.id) + '\'. The cyclic path follows:');
				let cyclePath = this._findCyclePath(dependency.id, module.id, 0) || [];
				cyclePath.reverse();
				cyclePath.push(dependency.id);
				console.warn(cyclePath.map(id => this._moduleIdProvider.getStrModuleId(id)).join(' => \n'));

				// Break the cycle
				module.unresolvedDependenciesCount--;
				continue;
			}

			// 记录当前模块需要等待依赖模块加载完成
			// 后面依赖模块加载完成时，会根据这个数组来查看当前模块是否能够可以进入 complete 阶段
			this._inverseDependencies2[dependency.id] = this._inverseDependencies2[dependency.id] || [];
			this._inverseDependencies2[dependency.id]!.push(module.id);

			this._loadModule(dependency.id);
		}
	}

	if (module.unresolvedDependenciesCount === 0) {
		this._onModuleComplete(module);
	}
}
```

这里我们先来看另一个情形，即所有的 dependency 都已经解析完毕的情形，此时 \_resolve 方法会直接调用 `_onModuleComplete` 方法结束一个 `Module` 的加载。这里， `ModuleManager` 需要将依赖的导出内容按照顺序传递给模块的回调函数，然后查看是否有模块依赖当前模块，以及能够将哪些模块也结束加载。

```ts
private _onModuleComplete(module: Module): void {
	let recorder = this.getRecorder();

	let dependencies = module.dependencies;
	let dependenciesValues: any[] = [];
	if (dependencies) {
		for (let i = 0, len = dependencies.length; i < len; i++) {
			let dependency = dependencies[i];

			// 如果依赖项是 exports，说明当前 Module 需要导出到变量 exports 对象上，将 exports 对象传入
			if (dependency === RegularDependency.EXPORTS) {
				dependenciesValues[i] = module.exports;
				continue;
			}

			if (dependency === RegularDependency.MODULE) {
				dependenciesValues[i] = {
					id: module.strId,
					config: () => {
						return this._config.getConfigForModule(module.strId);
					}
				};
				continue;
			}

			// 如果依赖项是 require，构造一个 require 函数并传入
			if (dependency === RegularDependency.REQUIRE) {
				dependenciesValues[i] = this._createRequire(module.moduleIdResolver!);
				continue;
			}

			// 将依赖项导出的变量传入
			let dependencyModule = this._modules2[dependency.id];
			if (dependencyModule) {
				dependenciesValues[i] = dependencyModule.exports;
				continue;
			}

			dependenciesValues[i] = null;
		}
	}

	module.complete(recorder, this._config, dependenciesValues);

	// 找到是否有其他模块依赖当前模块
	let inverseDeps = this._inverseDependencies2[module.id];
	this._inverseDependencies2[module.id] = null; // 移除值，表示其他模块需要的某个依赖（即当前模块）已经解析完成

	if (inverseDeps) {
		// 查看其他模块的依赖是否全部解析完成，如果全部完成了，对其他模块也调用 _onModuleComplete 方法
		for (let i = 0, len = inverseDeps.length; i < len; i++) {
			let inverseDependencyId = inverseDeps[i];
			let inverseDependency = this._modules2[inverseDependencyId];
			inverseDependency.unresolvedDependenciesCount--;
			if (inverseDependency.unresolvedDependenciesCount === 0) {
				this._onModuleComplete(inverseDependency);
			}
		}
	}
}
```

```ts
public complete(recorder: ILoaderEventRecorder, config: Configuration, dependenciesValues: any[]): void {
	this._isComplete = true;

	if (this._callback) {
		if (typeof this._callback === 'function') {
			let r = Module._invokeFactory(config, this.strId, this._callback, dependenciesValues);
		} else {
			this.exports = this._callback;
		}
	}

	(<any>this).dependencies = null;
	(<any>this)._callback = null;
	(<any>this)._errorback = null;
	(<any>this).moduleIdResolver = null;
}

private static _invokeFactory(config: Configuration, strModuleId: string, callback: Function, dependenciesValues: any[]): { returnedValue: any; producedError: any; } {
	return {
		returnedValue: callback.apply(global, dependenciesValues),
		producedError: null
	};
}
```

由上，可以看到 `complete` 的主要过程就是调用 module callback，如果这个模块没有依赖 `exports`，就将模块的返回作为该模块的导出。

![Untitled](https://user-images.githubusercontent.com/12122021/127628471-5ea33233-0d6f-43f4-bfab-75e77096dac4.png)

### 循环依赖的发现和处理

在 `_resolve` 方法的执行过程中，vscode-loader 会检测模块之间是否存在循环依赖，具体方法是通过调用 `_hasDependencyPath` 方法，查看是否存在 dependency 到当前 `Module` 的依赖路径。我们来举一个例子：假设有 a b c d 四个模块，其中 a 依赖 b d 模块，b 依赖 d 模块，d 依赖 c 模块，而 c 又依赖 a 模块， 那么这里显然存在 a - c - d 的循环依赖。我们假设 a 是先加载的模块，那么 `ModuleManager` 应当在加载 c 的环节发现 a 循环依赖了 c。

![Untitled 1](https://user-images.githubusercontent.com/12122021/127628525-afb5a679-adf4-42a2-932e-99550db875da.png)

我们用这个例子来分析 `_hasDependencyPath` 的执行过程。

```ts
private _hasDependencyPath(fromId: ModuleId, toId: ModuleId): boolean {
	// 在加载 a 的时候 b d 不存在 _modules2 中
	// 加载 b 的时候 d 不存在 _modules 2 中
	// 加载 d 的时候 c 不在 _modules2 中
	// 所以直接跳过检查
	// 到 c 的时候，发现它的一个依赖项 a 已经开始加载，这时就需要判断 a 是否也依赖 c
	let from = this._modules2[fromId];
	if (!from) {
		return false;
	}

	// 将当前所有的 module id 都放到这个数组当中来，如果遍历过的话就设置对应的 index 为 true
	let inQueue: boolean[] = [];
	for (let i = 0, len = this._moduleIdProvider.getMaxModuleId(); i < len; i++) {
		inQueue[i] = false;
	}
	let queue: Module[] = []; // 这是一个先进先出队列

	// 将 c 放到 queue 里面来
	queue.push(from);
	inQueue[fromId] = true;

	while (queue.length > 0) {
		// 检查队列首个元素
		let element = queue.shift()!;
		let dependencies = element.dependencies;
		// 如果有依赖的话，检查它的依赖项
		if (dependencies) {
			for (let i = 0, len = dependencies.length; i < len; i++) {
				let dependency = dependencies[i];

				// 如果在依赖项里找到了 toId，则说明发现了循环依赖
				if (dependency.id === toId) {
					// There is a path to 'to'
					return true;
				}

				// 否则将依赖项加入队列，查找依赖项的依赖项
				let dependencyModule = this._modules2[dependency.id];
				if (dependencyModule && !inQueue[dependency.id]) {
					inQueue[dependency.id] = true; // 表示依赖已经查找过了
					queue.push(dependencyModule);
				}
			}
		}
	}

	return false;
}
```

当加载 c 的时候， `_hasDependencyPath` 的调用参数是 a 和 c，即查看是否存在 a 到 c 的依赖关系。

- 第一次循环，发现 a 的依赖 b d，b d 不等于 c，因此将 b d 加入队列
- 第二次循环，发现 b 的依赖项 d，但是 d 已经遍历过了，所以不做处理
- 第三次循环，发现 d 的依赖项 c，找到了循环依赖，返回 `true`

这其实就是一个**广度优先的有向图搜索算法**，在遍历的过程中，我们尝试找到从 `from` 到 `to` 的路径，找得到的话我们就知道有循环依赖了（因为我们已经知道 `from` 是 `to` 的依赖，所以肯定存在一条 `to` 到 `from` 的路径）。

发现循环依赖之后，vscode-loader 会通过 `_findCyclePath` 中的一个深度优先的搜索算法找到该路径并提示开发者，然后直接标记这个模块已经解析完毕，防止出现死锁（当然这种情况下并不能保证能够正常工作，实际上碰到文件的循环依赖，就应该想办法去解开循环依赖）。

讲完了所有依赖项都已经解析完的理想情形，接下来我们看看未加载的依赖是怎么加载的，这里会涉及到 JavaScript 文件的加载和解析过程。

### 依赖的解析

在上面的学习中我们已经知道，依赖的加载过程是从 `_loadModule` 方法开始的，该方法的主要逻辑是：

- 通过 `moduleIdToPaths` 方法，找到文件可能存在的路径
- 按照这些路径去查找文件，主要是调用 `this._scriptLoader.load` 方法，在加载成功的回调里再去调用 `this._onload` 方法

```ts
private _loadModule(moduleId: ModuleId): void {
	if (this._modules2[moduleId] || this._knownModules2[moduleId]) {
		// known module
		return;
	}
	this._knownModules2[moduleId] = true; // 标记为加载中，防止第二次读取文件加载

	let strModuleId = this._moduleIdProvider.getStrModuleId(moduleId);
	let paths = this._config.moduleIdToPaths(strModuleId);

	let scopedPackageRegex = /^@[^\/]+\/[^\/]+$/ // matches @scope/package-name
	if (this._env.isNode && (strModuleId.indexOf('/') === -1 || scopedPackageRegex.test(strModuleId))) {
		paths.push('node|' + strModuleId);
	}

	let lastPathIndex = -1;
	let loadNextPath = (err: any) => {
		lastPathIndex++;

		if (lastPathIndex >= paths.length) {
		} else {
			let currentPath = paths[lastPathIndex];

			this._scriptLoader.load(this, currentPath, () => {
				this._onLoad(moduleId);
			}, (err) => {
			});
		}
	};

	loadNextPath(null);
}
```

我们先来看 `ScriptLoader` 的内容。

### 依赖代码的加载

根据代码运行的平台，有不同的 `ScriptLoader`，均继承 `IScriptLoader` 接口：

- `BrowserScriptLoader`，浏览器主线程环境
- `WorkerScriptLoader`，web worker 环境
- `NodeScriptLoader`，Node.js 环境

另外有一 `OnlyOnceScriptLoader`，它主要起两个作用：

- 封装上面三个 loader，给外部代码一个统一的调用入口，通过判断环境决定实例化哪一个
- 确保一个路径上的代码只会被加载一次，触发多个通过 `load` 方法传进来回调函数

我们这里主要跟大家一起学习 `NodeScriptLoader`，其他三者的代码都比较简单，大家可以自行学习。

### NodeScriptLoader

在 `_load` 方法首次调用的时候，会通过 `_init` 和 `_initNodeRequire` 初始化加载环境。

`_init` 的主要内容就是加载 node 原生的几个 module 并绑定在自己的属性上。而 `_initNodeRequire` 的内容则比较复杂，主要是 patch 了 Node.js 的 Module 模块的 `_compile` 方法，这个方法的主要逻辑是在 v8 的引擎中执行 JavaScript 文件的内容。vscode-loader 所作的 patch，就是在调用 v8 执行 JavaScript 代码时提供了 cacheData，这能够加速 v8 的执行。我们这里就不深入了解 cache 机制了，感兴趣的同学可以自行学习，让我们回到 `_load` 方法。

通过 node 方法加载文件的时候，会有两种情况：

第一种，文件以 node| 开头，位于 node_modules 目录中，此时直接调用原生 require 方法，并将该模块的导出通过 `enqueueDefineAnonymousModule` 方法，创建一个 AMD 模块。注意，这里在加载依赖的时候，就用到了之前对 `Module.prototype._compile` 的 patch，能够通过缓存来加速。

```ts
public load(moduleManager: IModuleManager, scriptSrc: string, callback: () => void, errorback: (err: any) => void): void {
	const opts = moduleManager.getConfig().getOptionsLiteral();
	const nodeRequire = ensureRecordedNodeRequire(moduleManager.getRecorder(), (opts.nodeRequire || global.nodeRequire));
	const nodeInstrumenter = (opts.nodeInstrumenter || function (c) { return c; });

	if (/^node\|/.test(scriptSrc)) {

		let pieces = scriptSrc.split('|');

		let moduleExports = null;
		try {
			moduleExports = nodeRequire(pieces[1]);
		}

		moduleManager.enqueueDefineAnonymousModule([], () => moduleExports);
		callback();

	} else {
		// ...
	}
}
```

```ts
public enqueueDefineAnonymousModule(dependencies: string[], callback: any): void {
	this._currentAnnonymousDefineCall = {
		stack: stack,
		dependencies: dependencies,
		callback: callback
	};
}

// 由 _loadModule 中的一个回调函数调用
private _onLoad(moduleId: ModuleId): void {
	if (this._currentAnnonymousDefineCall !== null) {
		let defineCall = this._currentAnnonymousDefineCall;
		this._currentAnnonymousDefineCall = null;

		// Hit an anonymous define call
		this.defineModule(this._moduleIdProvider.getStrModuleId(moduleId), defineCall.dependencies, defineCall.callback, null, defineCall.stack);
	}
}
```

第二种，则是加载 vscode 自己的代码，这样的情况下，则是通过调用 Node.js vm 模块的运行代码，并通过修改代码运行环境中的 `define` 变量，来检测 `define` 函数是否有被执行（没有执行的话，就认为当前加载的并不是一个模块，并抛出错误）。

```ts
public load(moduleManager: IModuleManager, scriptSrc: string, callback: () => void, errorback: (err: any) => void): void {
	const opts = moduleManager.getConfig().getOptionsLiteral();
	const nodeRequire = ensureRecordedNodeRequire(moduleManager.getRecorder(), (opts.nodeRequire || global.nodeRequire));
	const nodeInstrumenter = (opts.nodeInstrumenter || function (c) { return c; });

	if (/^node\|/.test(scriptSrc)) {
	} else {
		// ...
		scriptSrc = Utilities.fileUriToFilePath(this._env.isWindows, scriptSrc);
		const normalizedScriptSrc = this._path.normalize(scriptSrc);
		const vmScriptPathOrUri = this._getElectronRendererScriptPathOrUri(normalizedScriptSrc);

		this._readSourceAndCachedData(normalizedScriptSrc, cachedDataPath, recorder, (err: any, data: string, cachedData: Buffer, hashData: Buffer) => {
			let scriptSource: string;
			if (data.charCodeAt(0) === NodeScriptLoader._BOM) {
				scriptSource = NodeScriptLoader._PREFIX + data.substring(1) + NodeScriptLoader._SUFFIX;
			} else {
				scriptSource = NodeScriptLoader._PREFIX + data + NodeScriptLoader._SUFFIX;
			}

			scriptSource = nodeInstrumenter(scriptSource, normalizedScriptSrc);
			const scriptOpts: INodeVMScriptOptions = { filename: vmScriptPathOrUri, cachedData };
			const script = this._createAndEvalScript(moduleManager, scriptSource, scriptOpts, callback, errorback);
		});
	}
}

private _createAndEvalScript(moduleManager: IModuleManager, contents: string, options: INodeVMScriptOptions, callback: () => void, errorback: (err: any) => void): INodeVMScript {
	const script = new this._vm.Script(contents, options);
	const ret = script.runInThisContext(options);

	const globalDefineFunc = moduleManager.getGlobalAMDDefineFunc();
	let receivedDefineCall = false;
	const localDefineFunc: IDefineFunc = <any>function () {
		receivedDefineCall = true;
		return globalDefineFunc.apply(null, arguments);
	};
	localDefineFunc.amd = globalDefineFunc.amd;

	ret.call(global, moduleManager.getGlobalAMDRequireFunc(), localDefineFunc, options.filename, this._path.dirname(options.filename));

	if (receivedDefineCall) {
		callback();
	} else {
		errorback(new Error(`Didn't receive define call in ${options.filename}!`));
	}

	return script;
}
```

在执行这个文件的时候，`define` 函数的第一个参数总是空的，即定义的总是一个匿名函数（这点通过查看 vscode 编译后的文件即可知道），所以总是会进入到 `DefineFunc` 的第二个分支情形，之后的运行过程，就跟加载 node_modules 中的文件时一模一样了。

```ts
const DefineFunc: IDefineFunc = <any>(
  function (id: any, dependencies: any, callback: any): void {
    if (id) {
      //
    } else {
      moduleManager.enqueueDefineAnonymousModule(dependencies, callback)
    }
  }
)
```

![Untitled 2](https://user-images.githubusercontent.com/12122021/127628561-d52078dc-6620-48fd-93a5-72fca7ba5351.png)

到这里，vscode-loader 的整个核心过程就基本介绍完毕了。

## 总结

vscode 实现的 loader 有如下特性：

- 基于 AMD 规范，在 node.js 中也支持 node.js 原生的 require
- 支持多种 JavaScript 环境，包括：浏览器、Node.js、Electron 渲染进程、web worker
- 在 node.js 环境中，通过 cacheData 机制来加速文件的加载

由于篇幅的限制，这篇文章没有深入分析以下这些机制，感兴趣的同学在阅读本文之后可以自行学习：

- cacheData 机制的具体实现
- loader 需要对文件相对路径进行 resolve，这一部分是如何工作的
- vscode-loader plugin 机制，vscode-loader 可以通过 plugin 机制加载 CSS 文件和国际化文件（如果之后出国际化机制源码解析的话，我可能会再回过头来讲解 vscode-loader plugin 机制）
- 在构建时 vscode-loader 的额外一些处理逻辑
