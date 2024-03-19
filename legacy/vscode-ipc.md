---
title: vscode 源码解析 - 进程间调用
date: 2021/03/17
description: 介绍 vscode IPC 模块
tag: vscode, source code, Chinese, ipc
author: Wenzhao
---

# vscode 源码解析 - 进程间调用

![](https://user-images.githubusercontent.com/12122021/70677481-3c5e3580-1cca-11ea-9cc7-de8e131d3cb4.png)

---

vscode 的架构中主要有四类进程：主进程、渲染进程、shared 进程和 host 进程，这四个进程之间会发生进程间调用（Inter Process Calling, IPC）。vscode 中有专门的 IPC 模块来实现 IPC 机制，这篇文章将会深入介绍 vscode IPC 模块的设计和原理。

## IPC 原理

在我们开始学习 vscode 的 IPC 机制之前，不妨根据我们已经掌握的关于计算机网络的基本知识，来推演一下 IPC 有何要点：

1. 有**客户端**和**服务端**，客户端向服务端发起请求，请求即是要求调用服务端的某个方法，服务端返回响应，响应即是该方法的返回值
2. 客户端和服务端需要建立**连接**
3. 服务端需要以某种机制**分派请求**，以找到被调用的方法所在的模块
4. 请求需要通过某种**协议**来让双方知道如何解析和生成请求或响应

可以看到 IPC 理念上是比较简单的，而 vscode IPC 模块的优点在于，它清楚地定义了 IPC 模块的各个层次，将客户端的调用过程封装得就像是在调用本地的一个异步方法一样，还让不同的跨进程环境——例如本地进程、基于网络的跨进程、web worker——都能够很容易地实现。

## vscode IPC 机制概述

vscode IPC 分为**基于 Channel 的**和**基于 RpcProtocol 的**两种。

## 基于 Channel 的机制

我们通过一个例子开始对 Channel 机制的介绍。

在渲染进程初始化的时候，会创建一个 `ElectronIPCMainProcessService`，然后以此创建一个 `LoggerChannelClient`，并以 `ILoggerService` 为 key 添加到依赖注入系统当中：

```typescript
// Main Process
const mainProcessService = this._register(new ElectronIPCMainProcessService(this.configuration.windowId));
serviceCollection.set(IMainProcessService, mainProcessService);

// Logger
const loggerService = new LoggerChannelClient(mainProcessService.getChannel('logger'));
serviceCollection.set(ILoggerService, loggerService);
```

我们进一步看 `LoggerChannelClient` 的实现的话，就会发现它会调用 `channel` 的 `call` 方法，这里就就发起了一个 IPC：

```typescript
export class LoggerChannelClient implements ILoggerService {
	constructor(private readonly channel: IChannel) { }

	createConsoleMainLogger(): ILogger {
		return new AdapterLogger({
			log: (level: LogLevel, args: any[]) => {
				this.channel.call('consoleLog', [level, args]);
			}
		});
	}
}
```

就是 **channel IPC**。

channel IPC 主要支持两种类型的调用，通过下面这个枚举类型可以看出：

```typescript
export const enum RequestType {
	Promise = 100,
	PromiseCancel = 101,
	EventListen = 102,
	EventDispose = 103
}
```

* 第一种是基于 Promise 的调用
* 第二种是基于事件的调用
* 而切两种调用方式都有对应的取消的办法

我们这篇文章将会以基于 Promise 的调用为例，基于事件的调用大家可以自行了解。

channel IPC 主要有以下这些参与者，它们之间的关系如下图所示：

![1r4qaFQIf7Vw7xVCw6kCxOl3A26cIaY-So42x1xb7Yw](https://user-images.githubusercontent.com/12122021/133873692-14c212f2-1a8e-431d-9554-8ab0a21dffb1.png)

服务端

*   服务，各种业务逻辑实际发生的地方
*   `IServerChannel`，一个 `IServerChannel` 和一种服务对应，它们提供了 `call` 和 `listen` 两个方法给 `ChannelServer` 调用，然后调用它们对应的服务来具体执行各种业务逻辑，实际上是对服务的一种包裹
*   `IChannelServer`，它负责监听 `IMessagePassingProtocol` 传过来的请求，然后根据请求中指定的 channelName 来找到 `IServerChannel` 并进行调用，还能将执行结果返回给客户端
*   `IPCServer`，它提供了一组注册和获取 `IServerChannel` 的方法，并能够通过路由机制选择要通讯的客户端
*   `IMessagePassingProtocol`，它负责传输 `Uint8Array` 类型的二进制信息，并且在收到消息的时候通过事件通知上层

客户端

*   业务代码，业务代码会调用 `IChannel` 提供的方法来发起一个 IPC
*   `IChannel`，它们提供了 `call` 和 `listen` 两个方法给业务代码调用，用以发起 IPC
*   `IChannelClient`，它们是实际发起请求的地方，会将请求封装成一定的数据格式，在接收到响应的时候返回给业务代码
*   `IPCClient`，它提供一组注册和获取 `IChannel` 的方法
*   `IMessagePassingProtocol`，和它在服务端的对等方的功能一致

下面我们会具体讲解每个模块的机制。

### IChannel 和 IServerChannel

阅读过本系列之前两篇关于依赖注入和服务化的读者，应该已经知道 vscode 中各种功能都是封装在服务当中的，所以 IPC 的执行过程中必须要找到某个能响应特定调用的服务，`IServerChannel` 则是负责和服务一一对应，帮助它们接入 IPC 系统的，我们将称为**实体**。

而在客户端一侧，业务代码不知道 IPC 机制的接口，因此不能直接发起请求，而是将 `IChannel` 作为一个能够帮助它发起请求的**代理**。

`IserverChannel` 和 `IChannel` 分别就是实体和代理的接口：

```typescript
export interface IChannel {
	call<T>(command: string, arg?: any, cancellationToken?: CancellationToken): Promise<T>;
	listen<T>(event: string, arg?: any): Event<T>;
}

/**
 * An `IServerChannel` is the counter part to `IChannel`,
 * on the server-side. You should implement this interface
 * if you'd like to handle remote promises or events.
 */
export interface IServerChannel<TContext = string> {
	call<T>(ctx: TContext, command: string, arg?: any, cancellationToken?: CancellationToken): Promise<T>;
	listen<T>(ctx: TContext, event: string, arg?: any): Event<T>;
}
```

一个 `IChannel` 像就这样（即 return 返回的对象）：

```typescript
getChannel<T extends IChannel>(channelName: string): T {
		const that = this;

		return {
			call(command: string, arg?: any, cancellationToken?: CancellationToken) {
				if (that.isDisposed) {
					return Promise.reject(errors.canceled());
				}
				return that.requestPromise(channelName, command, arg, cancellationToken);
			},
			listen(event: string, arg: any) {
				if (that.isDisposed) {
					return Promise.reject(errors.canceled());
				}
				return that.requestEvent(channelName, event, arg);
			}
		} as T;
	}
```

而一个 `IServerChannel` 会像是这样：

```typescript
export class TestChannel implements IServerChannel {

	constructor(private testService: ITestService) { }

	listen(_: unknown, event: string): Event<any> {
		switch (event) {
			case 'marco': return this.testService.onMarco;
		}

		throw new Error('Event not found');
	}

	call(_: unknown, command: string, ...args: any[]): Promise<any> {
		switch (command) {
			case 'pong': return this.testService.pong(args[0]);
			case 'cancelMe': return this.testService.cancelMe();
			case 'marco': return this.testService.marco();
			default: return Promise.reject(new Error(`command not found: ${command}`));
		}
	}
}
```

在这个例子中 `TestChannel` 就是对 `ITestService` 的一层封装。

创建 IServerChannel 的方式有很多种，除了上面这样的直接实现，还可以借助 `ProxyChannel` namespace 提供的方法。

### ProxyChannel

如果不需要为 service 做一些特殊处理，可以直接使用 `ProxyChannel` namespace 下的 `fromService` 方法将一个 service 包装成一个 `IServerChannel`：

 ```typescript
	export function fromService(service: unknown, options?: ICreateServiceChannelOptions): IServerChannel {
		const handler = service as { [key: string]: unknown };
		const disableMarshalling = options && options.disableMarshalling;

		// Buffer any event that should be supported by
		// iterating over all property keys and finding them
		const mapEventNameToEvent = new Map<string, Event<unknown>>();
		for (const key in handler) {
			if (propertyIsEvent(key)) {
				mapEventNameToEvent.set(key, Event.buffer(handler[key] as Event<unknown>, true));
			}
		}

		return new class implements IServerChannel {

			listen<T>(_: unknown, event: string): Event<T> {
				const eventImpl = mapEventNameToEvent.get(event);
				if (eventImpl) {
					return eventImpl as Event<T>;
				}

				throw new Error(`Event not found: ${event}`);
			}

			call(_: unknown, command: string, args?: any[]): Promise<any> {
				const target = handler[command];
				if (typeof target === 'function') {

					// Revive unless marshalling disabled
					if (!disableMarshalling && Array.isArray(args)) {
						for (let i = 0; i < args.length; i++) {
							args[i] = revive(args[i]);
						}
					}

					return target.apply(handler, args);
				}

				throw new Error(`Method not found: ${command}`);
			}
		};
	}
 ```

同样的，也可以通过 `toService` 将 `IChannel` 封装成服务供业务代码调用，这样业务代码就不用自己去调用 `IChannel` 的 `call` 或者 `listen` 方法。

```typescript
	export function toService<T>(channel: IChannel, options?: ICreateProxyServiceOptions): T {
		const disableMarshalling = options && options.disableMarshalling;

		return new Proxy({}, {
			get(_target: T, propKey: PropertyKey) {
				if (typeof propKey === 'string') {

					// Check for predefined values
					if (options?.properties?.has(propKey)) {
						return options.properties.get(propKey);
					}

					// Event
					if (propertyIsEvent(propKey)) {
						return channel.listen(propKey);
					}

					// Function
					return async function (...args: any[]) {

						// Add context if any
						let methodArgs: any[];
						if (options && !isUndefinedOrNull(options.context)) {
							methodArgs = [options.context, ...args];
						} else {
							methodArgs = args;
						}

						const result = await channel.call(propKey, methodArgs);

						// Revive unless marshalling disabled
						if (!disableMarshalling) {
							return revive(result);
						}

						return result;
					};
				}

				throw new Error(`Property not found: ${String(propKey)}`);
			}
		}) as T;
	}
```

本质上是创建了一个 Proxy，将对 Proxy 属性的访问转换成对 channel 的 `call` `listen` 方法的调用。

### IChannelServer

`IChannelServer` 的主要职责包括：

1. 从 `protocol` 接收消息
2. 根据消息的类型进行处理
3. 调用合适的 `IServerChannel` 来处理请求
4. 将响应发送给客户端
5. 注册 `IServerChannel`

`IChannelServer` 直接监听 `protocol` 的消息，然后调用自己的 `onRawMessage` 方法处理请求。`onRawMessge` 会根据请求的类型来调用其他方法。以基于 Promise 的调用为例，可以看到它的核心逻辑就是调用 `IServerChannel` 的 `call` 方法。

```typescript
  private onRawMessage(message: VSBuffer): void {
		const reader = new BufferReader(message);
		const header = deserialize(reader);
		const body = deserialize(reader);
		const type = header[0] as RequestType;

		switch (type) {
			case RequestType.Promise:
				if (this.logger) {
					this.logger.logIncoming(message.byteLength, header[1], RequestInitiator.OtherSide, `${requestTypeToStr(type)}: ${header[2]}.${header[3]}`, body);
				}
				return this.onPromise({ type, id: header[1], channelName: header[2], name: header[3], arg: body });
        
        // ...
		}
	}

	private onPromise(request: IRawPromiseRequest): void {
		const channel = this.channels.get(request.channelName);

		let promise: Promise<any>;

		try {
			promise = channel.call(this.ctx, request.name, request.arg, cancellationTokenSource.token);
		} catch (err) {
			// ...
		}

		const id = request.id;

		promise.then(data => {
			this.sendResponse(<IRawResponse>{ id, data, type: ResponseType.PromiseSuccess });
			this.activeRequests.delete(request.id);
		}, err => {
			// ...
		});
	}
```

可以看到，这里通过 `request` 的 `channelName` 获取到一个 `IServerChannel`，然后调用了它的 `call` 方法，并将结果通过 `this.sendResponse` 发送给客户端。显然，这里 `this.channels` 需要注册 `IServerChannel`，而 `IChannelServer` 提供了这样的方法：

```typescript
	registerChannel(channelName: string, channel: IServerChannel<TContext>): void {
		this.channels.set(channelName, channel);

		setTimeout(() => this.flushPendingRequests(channelName), 0);
	}
```

### IChannelClient

`IChannelClient` 的逻辑比较简单，它只提供了一个接口，即 `getChannel` ，它返回了一个 `IChannel`，实际上就是通过闭包保存了 `channelName`，然后在业务方调用的时候调用 `requestPromise` 等发起请求。

```typescript
	getChannel<T extends IChannel>(channelName: string): T {
		const that = this;

		return {
			call(command: string, arg?: any, cancellationToken?: CancellationToken) {
				if (that.isDisposed) {
					return Promise.reject(errors.canceled());
				}
				return that.requestPromise(channelName, command, arg, cancellationToken);
			},
			// ...
		} as T;
	}

	private requestPromise(channelName: string, name: string, arg?: any, cancellationToken = CancellationToken.None): Promise<any> {
		const id = this.lastRequestId++;
		const type = RequestType.Promise;
		const request: IRawRequest = { id, type, channelName, name, arg };

		if (cancellationToken.isCancellationRequested) {
			return Promise.reject(errors.canceled());
		}

		let disposable: IDisposable;

		const result = new Promise((c, e) => {
			if (cancellationToken.isCancellationRequested) {
				return e(errors.canceled());
			}

			const doRequest = () => {
				const handler: IHandler = response => {
					switch (response.type) {
						case ResponseType.PromiseSuccess:
							this.handlers.delete(id);
							c(response.data);
							break;

						case ResponseType.PromiseError:
							this.handlers.delete(id);
							const error = new Error(response.data.message);
							(<any>error).stack = response.data.stack;
							error.name = response.data.name;
							e(error);
							break;

						case ResponseType.PromiseErrorObj:
							this.handlers.delete(id);
							e(response.data);
							break;
					}
				};

				this.handlers.set(id, handler);
				this.sendRequest(request);
			};

			let uninitializedPromise: CancelablePromise<void> | null = null;
			if (this.state === State.Idle) {
				doRequest();
			} else {
				// ...
			}

			const cancellationTokenListener = cancellationToken.onCancellationRequested(cancel);
			disposable = combinedDisposable(toDisposable(cancel), cancellationTokenListener);
			this.activeRequests.add(disposable);
		});

		return result.finally(() => { this.activeRequests.delete(disposable); });
	}
```

### 消息传输

我们已经看到了 `IChannelServer` 和 `IChannelClient` 之间会互发数据，这里简单讲解一下消息传输的机制。

首先消息传输需要约定好请求和响应的结构。

IPC 请求的字段如下：

```typescript
type IRawPromiseRequest = { type: RequestType.Promise; id: number; channelName: string; name: string; arg: any; };
type IRawPromiseCancelRequest = { type: RequestType.PromiseCancel, id: number };
type IRawEventListenRequest = { type: RequestType.EventListen; id: number; channelName: string; name: string; arg: any; };
type IRawEventDisposeRequest = { type: RequestType.EventDispose, id: number };

type IRawRequest = IRawPromiseRequest | IRawPromiseCancelRequest | IRawEventListenRequest | IRawEventDisposeRequest;
```

*   `type`，表明这是一种什么类型的调用
*   `id`，请求的唯一标识符，与请求相对应的响应会有相同的 id
*   `channelName`，调用的 channel 的名称
*   `name`，如果是基于 Promise 的调用，就是方法的名称，如果是基于事件的监听，就是事件的名称
*   `arg`，参数

IPC 响应的字段如下：

```typescript
type IRawInitializeResponse = { type: ResponseType.Initialize };
type IRawPromiseSuccessResponse = { type: ResponseType.PromiseSuccess; id: number; data: any };
type IRawPromiseErrorResponse = { type: ResponseType.PromiseError; id: number; data: { message: string, name: string, stack: string[] | undefined } };
type IRawPromiseErrorObjResponse = { type: ResponseType.PromiseErrorObj; id: number; data: any };
type IRawEventFireResponse = { type: ResponseType.EventFire; id: number; data: any };

type IRawResponse = IRawInitializeResponse | IRawPromiseSuccessResponse | IRawPromiseErrorResponse | IRawPromiseErrorObjResponse | IRawEventFireResponse;
```

*   `type`，表明是什么类型的响应
*   `id`，响应的唯一标识符
*   `data`，返回的数据

请求和响应在被发送之前，都会通过 `VSBuffer` 进行序列化，在接收之后则会进行反序列化。

需要一定的机制来将请求和响应对应起来。这在服务端比较容易，因为服务端的处理在顺序上处于 IPC 的中间环节，可以很自然的通过作用域来对应请求和响应。而在客户端，则需要一些机制来匹配请求和响应。

`IChannelClient` 在 `sendRequest` 之前，会通过 `id` 来在自身的 `handlers` Map 上绑定一个 handler

```typescript
this.handlers.set(id, handler);
```

而在收到消息的时候，就会通过这里 `id` 调用相应的 `handler`，从而 resolve 客户端 `IChannel` 的调用。

```typescript
	private onResponse(response: IRawResponse): void {
		if (response.type === ResponseType.Initialize) {
			this.state = State.Idle;
			this._onDidInitialize.fire();
			return;
		}

		const handler = this.handlers.get(response.id);

		if (handler) {
			handler(response);
		}
	}
```

### IMessagePassingProtocol

`IChannelServer` 和 `IChannelClient` 之间会通过 protocol 传输数据。对于上层，它提供二进制数据流传输服务（用 `VSBuffer` 进行了封装），并能够在有新消息到达的时候通知上层。

其接口非常简单：

```tsx
export interface IMessagePassingProtocol {
	send(buffer: VSBuffer): void;
	onMessage: Event<VSBuffer>;
	/**
	 * Wait for the write buffer (if applicable) to become empty.
	 */
	drain?(): Promise<void>;
}
```

* `send` 通过下层信道发送 `Uint8Array` 格式的消息
* `onMessage` 则在下层信道收到消息时触发上层的回调函数

不同的通讯端有不同的信道，因此 `IMessagePassingProtocol` 也有多种实现，大致有以下几种：

- 基于 Electron IPC 模块的实现，通过 webContents 和 ipcRenderer 收发消息；主进程和渲染进程的通讯使用这种方法
- 基于 web worker 的实现，通过 postMessage 和 onMessage 进行通讯；vscode 浏览器端部分插件基于这种实现
- 基于 web socket 的实现或 Node.js net 模块实现 ，通过 WebSocket 或 net 创建的套接字或者 pipe 进行通讯；vscode 浏览器端部分插件，以及渲染进程和 Host 进程的通讯基于这种实现（这是最有趣的一个对 Protocol 的实现，vscode 团队在这里实现了一个翻版的 TCP 协议）

### IPCClient

它用于在客户端管理 `IChannel` ，它同时实现了 `IChannelClient` 和 `IChannelServer`，所以它实际上可以发起也可以响应 IPC：

```tsx
export interface IChannelClient {
	getChannel<T extends IChannel>(channelName: string): T;
}

export interface IChannelServer<TContext = string> {
	registerChannel(channelName: string, channel: IServerChannel<TContext>): void;
}
```

```typescript
export class IPCClient<TContext = string> implements IChannelClient, IChannelServer<TContext>, IDisposable {

	private channelClient: ChannelClient;
	private channelServer: ChannelServer<TContext>;

	constructor(protocol: IMessagePassingProtocol, ctx: TContext, ipcLogger: IIPCLogger | null = null) {
		const writer = new BufferWriter();
		serialize(writer, ctx);
		protocol.send(writer.buffer);

		this.channelClient = new ChannelClient(protocol, ipcLogger);
		this.channelServer = new ChannelServer(protocol, ctx, ipcLogger);
	}

	getChannel<T extends IChannel>(channelName: string): T {
		return this.channelClient.getChannel(channelName) as T;
	}

	registerChannel(channelName: string, channel: IServerChannel<TContext>): void {
		this.channelServer.registerChannel(channelName, channel);
	}

	dispose(): void {
		this.channelClient.dispose();
		this.channelServer.dispose();
	}
}
```

可以看到它仅有一个 `IMessagePassingProtocol`，换句话说，就是只能跟一方进行通讯，这也是它跟 `IPCServer` 最大的区别。

### IPCServer

它一共实现了三个接口：

```tsx
export interface IChannelServer<TContext = string> {
	registerChannel(channelName: string, channel: IServerChannel<TContext>): void;
}

export interface IRoutingChannelClient<TContext = string> {
	getChannel<T extends IChannel>(channelName: string, router?: IClientRouter<TContext>): T;
}

export interface IConnectionHub<TContext> {
	readonly connections: Connection<TContext>[];
	readonly onDidAddConnection: Event<Connection<TContext>>;
	readonly onDidRemoveConnection: Event<Connection<TContext>>;
}
```

* `IRoutingChannelClient` 说明它可以根据一定的条件选择向哪个 `IChannelServer` 发起调用，即对调用进行**路由**
* `IConnectionHub` 则说明它可以管理客户端连接

我们来看 `IPCServer` 的构造方法：

```typescript
export class IPCServer<TContext = string> implements IChannelServer<TContext>, IRoutingChannelClient<TContext>, IConnectionHub<TContext>, IDisposable {
	constructor(onDidClientConnect: Event<ClientConnectionEvent>) {
		onDidClientConnect(({ protocol, onDidClientDisconnect }) => {
			const onFirstMessage = Event.once(protocol.onMessage);

			onFirstMessage(msg => {
				const reader = new BufferReader(msg);
				const ctx = deserialize(reader) as TContext;

				const channelServer = new ChannelServer(protocol, ctx);
				const channelClient = new ChannelClient(protocol);

				this.channels.forEach((channel, name) => channelServer.registerChannel(name, channel));

				const connection: Connection<TContext> = { channelServer, channelClient, ctx };
				this._connections.add(connection);
				this._onDidAddConnection.fire(connection);

				onDidClientDisconnect(() => {
					channelServer.dispose();
					channelClient.dispose();
					this._connections.delete(connection);
					this._onDidRemoveConnection.fire(connection);
				});
			});
		});
	}
}
```

可以看到，在对方发来第一条消息时，`IPCServer` 会创建：

* `ChannelServer`
* `ChannelClient`
* `Connection`，这个 `Connection` 就是来描述连接的，它的接口如下

```typescript
interface Connection<TContext> extends Client<TContext> {
	readonly channelServer: ChannelServer<TContext>;
	readonly channelClient: ChannelClient;
}

export interface Client<TContext> {
	readonly ctx: TContext;
}
```

注意这里的 `ctx` 属性，它是客户端的标识符，将用在请求路由的过程中。它的 `getChannel` 和 `ChannelClient` 的 `getChannel` 有很大不同：

```typescript
	getChannel<T extends IChannel>(channelName: string, router: IClientRouter<TContext>): T;
	getChannel<T extends IChannel>(channelName: string, clientFilter: (client: Client<TContext>) => boolean): T;
	getChannel<T extends IChannel>(channelName: string, routerOrClientFilter: IClientRouter<TContext> | ((client: Client<TContext>) => boolean)): T {
		const that = this;

		return {
			call(command: string, arg?: any, cancellationToken?: CancellationToken): Promise<T> {
				let connectionPromise: Promise<Client<TContext>>;

				if (isFunction(routerOrClientFilter)) {
					// when no router is provided, we go random client picking
					let connection = getRandomElement(that.connections.filter(routerOrClientFilter));

					connectionPromise = connection
						// if we found a client, let's call on it
						? Promise.resolve(connection)
						// else, let's wait for a client to come along
						: Event.toPromise(Event.filter(that.onDidAddConnection, routerOrClientFilter));
				} else {
					connectionPromise = routerOrClientFilter.routeCall(that, command, arg);
				}

				const channelPromise = connectionPromise
					.then(connection => (connection as Connection<TContext>).channelClient.getChannel(channelName));

				return getDelayedChannel(channelPromise)
					.call(command, arg, cancellationToken);
			},
			listen(event: string, arg: any): Event<T> {
				// ...
			}
		} as T;
	}
```

可以看到，在调用 `getChannel` 的时候如果传入了 `routerOrClientFilter`，则会在 `connections` 中选择一个。

### Routing

选择  `Connection` 的方法，可以是一个简单的 filter 函数，也可以是通过 `IClientRouter` 提供的 `routeCall` 或者 `routeEvent` 方法。我们以 `StaticRouter` 为例：

```typescript
export class StaticRouter<TContext = string> implements IClientRouter<TContext> {

	constructor(private fn: (ctx: TContext) => boolean | Promise<boolean>) { }

	routeCall(hub: IConnectionHub<TContext>): Promise<Client<TContext>> {
		return this.route(hub);
	}

	routeEvent(hub: IConnectionHub<TContext>): Promise<Client<TContext>> {
		return this.route(hub);
	}

	private async route(hub: IConnectionHub<TContext>): Promise<Client<TContext>> {
		for (const connection of hub.connections) {
			if (await Promise.resolve(this.fn(connection.ctx))) {
				return Promise.resolve(connection);
			}
		}

		await Event.toPromise(hub.onDidAddConnection);
		return await this.route(hub);
	}
}
```

实际上在 `getChannel` 调用他的时候，会通过 `fn` 来选择一个 `IConnectionHub` 中的 `Connection`。

到这里，整个基于 channel 的 IPC 机制我们就介绍完毕了。

## 基于 RpcProtocol 的机制

vscode IPC 的第二种机制基于 `RpcProtocol`，用于渲染进程和 extension host 进程通讯（如果 vscode 的运行环境是浏览器，那么就是主线程和 extension host web worker 之间进行通讯）。

举个例子，在 host 进程初始化时如果发生了错误，它会告知渲染进程，代码如下：

```typescript
		const mainThreadExtensions = rpcProtocol.getProxy(MainContext.MainThreadExtensionService);
		const mainThreadErrors = rpcProtocol.getProxy(MainContext.MainThreadErrors);
		errors.setUnexpectedErrorHandler(err => {
			const data = errors.transformErrorForSerialization(err);
			const extension = extensionErrors.get(err);
			if (extension) {
				mainThreadExtensions.$onExtensionRuntimeError(extension.identifier, data);
			} else {
				mainThreadErrors.$onUnexpectedError(data);
			}
		});
```

在调用 `mainThreadExtensions` 或 `mainThreadError` 的方法的时候，即发生了 IPC。

该机制如下图所示：

![](https://user-images.githubusercontent.com/12122021/112595602-99d9af80-8e45-11eb-9a4c-a16150537b97.jpeg)

下面介绍其原理。

### shape

客户端怎么知道 `mainThreadExtensions` 上有一个 `$onExtensionRuntimeError` 方法可以调用呢？

显然，这里需要定义一个接口，这个接口就是 `MainThreadExtensionServiceShape` ，定义在 extHost.protocol.ts 文件中。**vscode 对于每一个可以调用的*实体*，都定义了一个以 `Shape` 为后缀的接口，服务端的实体必须要实现该接口，这样客户端在编写代码的时候就知道有哪些方法可以调用了。**

### identifier

客户端如何获取到服务端的实体在本地的*代理*，也就是 `mainThreadExtensions` 呢？换个问法，`mainThreadExtensions` 是如何跟 `mainThreadErrors` 相区别的呢？

代码中我们可以看到  `mainThreadExtensions` 是通过 `rpcProtocol.getProxy(MainContext.MainThreadExtensionService)` 获得的，`MainContext.MainThreadExtensionService` 在这里就起到了一个标识符的作用，它将每一个实体-代理的对子区别开。

`MainContext.MainThreadExtensionService` 定义在 extHost.protocol.ts 当中：

```typescript
export const MainContext = {
  MainThreadExtensionService: createMainId<MainThreadExtensionServiceShape>('MainThreadExtensionService')
}
```

而 `createMainId` 就是用于创建标识符的方法，本质上是创建了一个 `ProxyIdentifier` 对象并存在到一个数组当中：

```typescript
export class ProxyIdentifier<T> {
	public static count = 0;
	_proxyIdentifierBrand: void;

	public readonly isMain: boolean;
	public readonly sid: string;
	public readonly nid: number;

	constructor(isMain: boolean, sid: string) {
		this.isMain = isMain;
		this.sid = sid;
		this.nid = (++ProxyIdentifier.count);
	}
}

const identifiers: ProxyIdentifier<any>[] = [];

export function createMainContextProxyIdentifier<T>(identifier: string): ProxyIdentifier<T> {
	const result = new ProxyIdentifier<T>(true, identifier);
	identifiers[result.nid] = result;
	return result;
}

export function createExtHostContextProxyIdentifier<T>(identifier: string): ProxyIdentifier<T> {
	const result = new ProxyIdentifier<T>(false, identifier);
	identifiers[result.nid] = result;
	return result;
}
```

每个标识符有三个字段：

* isMain 标识实体是否是在渲染进程当中
* sid 标识字符串 id
* nid 标识数字 id

### context

我们如何知道另外一个进程中，有哪些实体可以被调用？

extHost.protocol.ts 文件中定义了 MainContext 和 ExtHostContext 两个文件。前者定义了渲染进程中可被调用的实体，后者定义了 host 进程中可被调用的实体。这里也可以看出，在 RpcProtocol 机制下，渲染进程和 host 进程是可以互相调用的。

### customer

可被调用的实体是如何注册的？

host 进程调用 `mainThreadExtensions` 方法的时候，渲染进程必须要有类提供这个方法，而且它还需要注册到这个 RpcProtocol 的机制上。通过查找实现了 `MainThreadExtensionServiceShape` 的类，不难发现 mainThreadExtensionService.ts 中存在这样一段代码：

 ```typescript
@extHostNamedCustomer(MainContext.MainThreadExtensionService)
export class MainThreadExtensionService implements MainThreadExtensionServiceShape {
	// ...
}
 ```

注意这里装饰器的调用，我们探究其实现：

```typescript
export function extHostNamedCustomer<T extends IDisposable>(id: ProxyIdentifier<T>) {
	return function <Services extends BrandedService[]>(ctor: { new(context: IExtHostContext, ...services: Services): T }): void {
		ExtHostCustomersRegistryImpl.INSTANCE.registerNamedCustomer(id, ctor as IExtHostCustomerCtor<T>);
	};
}
```

可以发现它是将 `id`，也就是 `MainContext.MainThreadExtensionService` 和 `MainThreadExtensionService` 绑定起来，而在 extension host 初始化的时候实例化它：

```typescript
		const namedCustomers = ExtHostCustomersRegistry.getNamedCustomers();
		for (let i = 0, len = namedCustomers.length; i < len; i++) {
			const [id, ctor] = namedCustomers[i];
			const instance = this._instantiationService.createInstance(ctor, extHostContext);
			this._customers.push(instance);
			this._rpcProtocol.set(id, instance);
		}
```

注册的最后一步就是调用 `RpcProtocol.set` 方法注册可被调用的实体。

### RpcProtocol 的通讯原理

到这里我们基本了解了 RpcProtocol 的接口了，下面来了解一下它的内部逻辑。

首先来看 `getProxy`，我们知道客户端要通过这个方法获取可调用的代理：

```typescript
	public getProxy<T>(identifier: ProxyIdentifier<T>): T {
		const { nid: rpcId, sid } = identifier;
		if (!this._proxies[rpcId]) {
			this._proxies[rpcId] = this._createProxy(rpcId, sid);
		}
		return this._proxies[rpcId];
	}

	private _createProxy<T>(rpcId: number, debugName: string): T {
		let handler = {
			get: (target: any, name: PropertyKey) => {
				if (typeof name === 'string' && !target[name] && name.charCodeAt(0) === CharCode.DollarSign) {
					target[name] = (...myArgs: any[]) => {
						return this._remoteCall(rpcId, name, myArgs);
					};
				}
				if (name === _RPCProxySymbol) {
					return debugName;
				}
				return target[name];
			}
		};
		return new Proxy(Object.create(null), handler);
	}
```

可以看到它的核心逻辑就是创建一个 Proxy 对象，当对象上的属性被访问时，所有以 $ 开头的属性都会被包装为一个对 `this._remoteCall` 进行调用的方法。

`_remoteCall` 的核心逻辑则主要是下面几行（这里主要省略了取消请求相关的逻辑）：

```typescript
	private _remoteCall(rpcId: number, methodName: string, args: any[]): Promise<any> {
		const serializedRequestArguments = MessageIO.serializeRequestArguments(args, this._uriReplacer);

		const req = ++this._lastMessageId;
		const callId = String(req);
		const result = new LazyPromise();

		this._pendingRPCReplies[callId] = result;
		this._onWillSendRequest(req);
		const msg = MessageIO.serializeRequest(req, rpcId, methodName, serializedRequestArguments, !!cancellationToken);

		this._protocol.send(msg);
		return result;
	}
  
  // MessageIO
  public static serializeRequest(req: number, rpcId: number, method: string, serializedArgs: SerializedRequestArguments, usesCancellationToken: boolean): VSBuffer {
		if (serializedArgs.type === 'mixed') {
			return this._requestMixedArgs(req, rpcId, method, serializedArgs.args, serializedArgs.argsType, usesCancellationToken);
		}
		return this._requestJSONArgs(req, rpcId, method, serializedArgs.args, usesCancellationToken);
	}

	private static _requestJSONArgs(req: number, rpcId: number, method: string, args: string, usesCancellationToken: boolean): VSBuffer {
		const methodBuff = VSBuffer.fromString(method);
		const argsBuff = VSBuffer.fromString(args);

		let len = 0;
		len += MessageBuffer.sizeUInt8();
		len += MessageBuffer.sizeShortString(methodBuff);
		len += MessageBuffer.sizeLongString(argsBuff);

		let result = MessageBuffer.alloc(usesCancellationToken ? MessageType.RequestJSONArgsWithCancellation : MessageType.RequestJSONArgs, req, len);
		result.writeUInt8(rpcId);
		result.writeShortString(methodBuff);
		result.writeLongString(argsBuff);
		return result.buffer;
	}

  // MessageBuffer
	public static alloc(type: MessageType, req: number, messageSize: number): MessageBuffer {
		let result = new MessageBuffer(VSBuffer.alloc(messageSize + 1 /* type */ + 4 /* req */), 0);
		result.writeUInt8(type);
		result.writeUInt32(req);
		return result;
	}
```

可以看到一个请求主要有以下这些信息：

1. `type`，请求的类型，由一个枚举 MessageType 所定义
2. `req`，请求的序号，是一个自增的数字
3. `rpcId`，identifier 的字符串 id，表明是哪个实体-代理之间的请求
4. `method`，指定要调用实体的哪个方法
5. `argsBuff`，序列化的参数

最终这些参数都会被封装为一个 `VSBuffer` 并通过 protocol 发送，而这些而这里的  protocol，这是我们的老朋友 `IMessagePassingProtocol` 。 所以我们可以看到 RpcProtocol 机制也是分层的设计，可以在不同的环境中使用。

当服务端接收到一个请求时，会回调 `_receiveOneMessage` 方法进行处理：

```typescript
	private _receiveOneMessage(rawmsg: VSBuffer): void {
		if (this._isDisposed) {
			return;
		}

		const msgLength = rawmsg.byteLength;
		const buff = MessageBuffer.read(rawmsg, 0);
		const messageType = <MessageType>buff.readUInt8();
		const req = buff.readUInt32();

		switch (messageType) {
			case MessageType.RequestJSONArgs:
			case MessageType.RequestJSONArgsWithCancellation: {
				let { rpcId, method, args } = MessageIO.deserializeRequestJSONArgs(buff);
				if (this._uriTransformer) {
					args = transformIncomingURIs(args, this._uriTransformer);
				}
				this._receiveRequest(msgLength, req, rpcId, method, args, (messageType === MessageType.RequestJSONArgsWithCancellation));
				break;
			}
			
			// ...
	}
```

即根据 `type` 来调用不同的方法对请求进行处理，这里来看 `_receiveRequest` 方法：

```typescript
	private _receiveRequest(msgLength: number, req: number, rpcId: number, method: string, args: any[], usesCancellationToken: boolean): void {
		const callId = String(req);

		let promise: Promise<any>;
		let cancel: () => void;
		if (usesCancellationToken) {
      // ...
		} else {
			// cannot be cancelled
			promise = this._invokeHandler(rpcId, method, args);
			cancel = noop;
		}

		// Acknowledge the request
		const msg = MessageIO.serializeAcknowledged(req);
		this._protocol.send(msg);

		promise.then((r) => {
			delete this._cancelInvokedHandlers[callId];
			const msg = MessageIO.serializeReplyOK(req, r, this._uriReplacer);
			this._protocol.send(msg);
		}, (err) => {
      // ...
		});
	}
  
  private _invokeHandler(rpcId: number, methodName: string, args: any[]): Promise<any> {
		try {
			return Promise.resolve(this._doInvokeHandler(rpcId, methodName, args));
		} catch (err) {
			return Promise.reject(err);
		}
	}

	private _doInvokeHandler(rpcId: number, methodName: string, args: any[]): any {
		const actor = this._locals[rpcId];
		if (!actor) {
			throw new Error('Unknown actor ' + getStringIdentifierForProxy(rpcId));
		}
		let method = actor[methodName];
		if (typeof method !== 'function') {
			throw new Error('Unknown method ' + methodName + ' on actor ' + getStringIdentifierForProxy(rpcId));
		}
		return method.apply(actor, args);
	}
```

核心就是调用 `_invokeHandler` 然后将结果发送回去。

注意到这一行 `const actor = this._locals[rpcId];` 获取了可被调用的实体，记得之前注册实体时调用的 set 方法吗：

```typescript
	public set<T, R extends T>(identifier: ProxyIdentifier<T>, value: R): R {
		this._locals[identifier.nid] = value;
		return value;
	}
```

到这里，我们就了解了 RpcProtocol 的原理了。
