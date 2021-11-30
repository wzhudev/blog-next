---
title: Flask 源码解析
date: 2019/10/16
description: 简单分析 Flask 源代码
tag: Flask, python, source code, Chinese
author: Wendell
---

# Flask 源码解析

本文简单的分析了 Flask 的源码，主要关注 WSGI、Flask 对象的数据结构、Flask 应用启动过程、请求处理过程、视图函数、URL 的映射、应用上下文和请求上下文。

---

这是 Flask 官方钦定的 Demo 代码：

```python
from flask import Flask
app = Flask(__name__)

@app.route(‘/‘)
def index():
    return ‘Hello, world!’

if __name__ == ‘__main__’:
    app.run()
```

这篇文章从这个简单的代码开始，简要介绍了 WSGI、Flask 对象的数据结构、Flask 应用启动过程、请求处理过程、视图函数、URL 的映射、request 和 response 类（应用上下文和请求上下文），这些主题涵盖了一个 web 框架的核心。

## WSGI

在用户发起的请求到达服务器之后，会被一个 HTTP 服务器所接收，然后交给 web 应用程序做业务处理，这样 HTTP 服务器和 web 应用之间就需要一个接口，在 Python web 开发的世界里，Python 官方**钦定**了这个接口并命名为 WSGI，由 PEP333 所规定。只要服务器和框架都遵守这个约定，那么就能实现服务器和框架的任意组合。按照这个规定，一个面向 WSGI 的框架必须要实现这个方法：

```py
def application(environ, start_response)
```

在工作过程中，HTTP 服务器会调用上面这个方法，传入请求信息，即名为 `environ` 的字典和 `start_response` 函数，应用从 `environ` 中获取请求信息，在进行业务处理后调用 `start_response` 设置响应头，并返回响应体（必须是一个可遍历的对象，比如列表、字典）给 HTTP 服务器，HTTP 服务器再返回响应给用户。

所以 Flask 作为一个开发 web 应用的 web 框架，负责解决的问题就是：

1. 作为一个应用，能够被 HTTP 服务器所调用，必须要有 `__call__` 方法
2. 通过传入的请求信息（URL、HTTP 方法等），找到正确的业务处理逻辑，即正确的视图函数
3. 处理业务逻辑，这些逻辑可能包括表单检查、数据库 CRUD 等（这个在这篇文章里不会涉及）
4. 返回正确的响应
5. 在同时处理多个请求时，还需要保护这些请求，知道应该用哪个响应去匹配哪个请求，即线程保护

下面就来看看 Flask 是如何解决这些问题的。

> 参考阅读：[一起写一个 web 服务器](http://python.jobbole.com/81524/)，该系列文章能够让你基本理解 web 服务器和框架是如何通过 WSGI 协同工作的。

## 应用的创建

> 源码阅读：`app.py` 中 `Flask` 类的代码。

Demo 代码的第二行创建了一个 Flask 类的实例，传入的参数是当前模块的名字。我们先来看看 Flask 应用到底是什么，它的数据结构是怎样的。

`Flask` 是这样一个类：

> The flask object implements a WSGI application and acts as the central
> object. It is passed the name of the module or package of the
> application. Once it is created it will act as a central registry for
> the view functions, the URL rules, template configuration and much more.

> The name of the package is used to resolve resources from inside the
> package or the folder the module is contained in depending on if the
> package parameter resolves to an actual python package (a folder with
> an `__init__.py` file inside) or a standard module (just a `.py` file).

> 一个 Flask 对象实际上是一个 WSGI 应用。它接收一个模块或包的名字作为参数。它被创建之后，所有的视图函数、URL 规则、模板设置等都会被注册到它上面。之所以要传入模块或包的名字，是为了定位一些资源。

Flask 类有这样一些属性：

- `request_class = Request` 设置请求的类型
- `response_class = Response` 设置响应的类型

这两个类型都来源于它的依赖库 `werkzeug` 并做了简单的拓展。

Flask 对象的 `__init__` 方法如下：

```py
def __init__(self, package_name):
    #: Flask 对象有这样一个字典来保存所有的视图函数
    self.view_functions = {}

    #: 这个字典用来保存所有的错误处理视图函数
    #: 字典的 key 是错误类型码
    self.error_handlers = {}

    #: 这个列表用来保存在请求被分派之前应当执行的函数
    self.before_request_funcs = []

    #: 在接收到第一个请求的时候应当执行的函数
    self.before_first_request_funcs = []

    #: 这个列表中的函数在请求完成之后被调用，响应对象会被传给这些函数
    self.after_request_funcs = []

    #: 这里设置了一个 url_map 属性，并把它设置为一个 Map 对象
    self.url_map = Map()
```

到这里一个 Flask 对象创建完毕并被变量 `app` 所指向，其实它就是一个保存了一些配置信息，绑定了一些视图函数并且有个 URL 映射对象（`url_map`）的对象。但我们还不知道这个 Map 对象是什么，有什么作用，从名字上看，似乎其作用是映射 URL 到视图函数。源代码第 21 行有 `from werkzeug.routing import Map, Rule`，那我们就来看看 `werkzeug` 这个库中对 Map 的定义：

> The map class stores all the URL rules and some configuration
> parameters. Some of the configuration values are only stored on the
> `Map` instance since those affect all rules, others are just defaults
> and can be overridden for each rule. Note that you have to specify all
> arguments besides the `rules` as keyword arguments!

可以看到这个类的对象储存了所有的 URL 规则和一些配置信息。由于 `werkzeug` 的映射机制比较复杂，我们下文中讲到映射机制的时候再深入了解，现在只要记住 Flask 应用（即一个 `Flask` 类的实例）存储了视图函数，并通过 `url_map` 这个变量存储了一个 URL 映射机构就可以了。

## 应用启动过程

> 源码阅读：`app.py` 中 `Flask` 类的代码和 `werkzeug.serving` 的代码，特别注意 `run_simple` `BaseWSGIServer` `WSGIRequestHandler`。

Demo 代码的第 6 行是一个限制，表示如果 Python 解释器是直接运行该文件或包的，则运行 Flask 程序：在 Python 中，如果直接执行一个模块或包，那么解释器就会把当前模块或包的 `__name__` 设置为为 `__main_`。

第 7 行中的 `run` 方法启动了 Flask 应用：

```py
def run(self, host=None, port=None, debug=None, **options):
    from werkzeug.serving import run_simple
    if host is None:
        host = '127.0.0.1'
    if port is None:
        server_name = self.config['SERVER_NAME']
        if server_name and ':' in server_name:
            port = int(server_name.rsplit(':', 1)[1])
        else:
            port = 5000
    if debug is not None:
        self.debug = bool(debug)
    options.setdefault('use_reloader', self.debug)
    options.setdefault('use_debugger', self.debug)
    try:
        run_simple(host, port, self, **options)
    finally:
        # reset the first request information if the development server
        # reset normally.  This makes it possible to restart the server
        # without reloader and that stuff from an interactive shell.
        self._got_first_request = False
```

可以看到这个方法基本上是在配置参数，实际上启动服务器的是 `werkzeug` 的 `run_simple` 方法，该方法在默认情况下启动了服务器 `BaseWSGIServer`，继承自 Python 标准库中的 `HTTPServer.TCPServer`。注意在调用 `run_simple` 时，Flask 对象把自己 `self` 作为参数传进去了，这是正确的，因为服务器在收到请求的时候，必须要知道应该去调用谁的 `__call__` 方法。

按照标准库中 `HTTPServer.TCPServer` 的模式，服务器必须有一个类来作为 request handler 来处理收到的请求，而不是由 `HTTPServer.TCPServer` 本身的实例来处理，`werkzeug` 提供了 `WSGIRequestHandler` 类来作为 request handler，这个类在被 `BaseWSGIServer` 调用时，会执行这个函数：

```py
def execute(app):
    application_iter = app(environ, start_response)
    try:
        for data in application_iter:
            write(data)
        if not headers_sent:
            write(b'')
    finally:
        if hasattr(application_iter, 'close'):
            application_iter.close()
        application_iter = None
```

函数的第一行就是按照 WSGI 要求的，调用了 app 并把 `environ` 和 `start_response` 传入。我们再看看 flask 中是如何按照 WSGI 要求对服务器的调用进行呼应的。

```py
def __call__(self, environ, start_response):
    return self.wsgi_app(environ, start_response)

def wsgi_app(self, environ, start_response):
    ctx = self.request_context(environ)
    ctx.push()
    error = None
    try:
        try:
            response = self.full_dispatch_request()
        except Exception as e:
            error = e
            response = self.handle_exception(e)
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        ctx.auto_pop(error)
```

可以看到 Flask 按照 WSGI 的要求实现了 `__call__` 方法，因此成为了一个可调用的对象。但它不是在直接在 `__call__` 里写逻辑的，而是调用了 `wsgi_app` 方法，这是为了中间件的考虑，不展开谈了。这个方法返回的 `response(environ, start_response)` 中，`response` 是 `werkzueg.response` 类的一个实例，它也是个可以调用的对象，这个对象会负责生成最终的可遍历的响应体，并调用 `start_response` 形成响应头。

## 请求处理过程

> 源码阅读：`app.Flask` 的代码。

```py
def wsgi_app(self, environ, start_response):
    ctx = self.request_context(environ)
    ctx.push()
    error = None
    try:
        try:
            response = self.full_dispatch_request()
        except Exception as e:
            error = e
            response = self.handle_exception(e)
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        ctx.auto_pop(error)
```

`wsgi_app` 方法中里面的内容就是对请求处理过程的一个高度抽象。

首先，在接收到服务器传过来的请求时，Flask 调用 `request_context` 函数建立了一个 `RequestContext` 请求上下文对象，并把它压入 `_request_ctx_stack` 栈。关于上下文和栈的内容下文会再讲到，你现在需要知道的是，这些操作是为了 flask 在处理多个请求的时候不会混淆。之后，Flask 会调用 `full_dispatch_request` 方法对这个请求进行分发，开始实际的请求处理过程，这个过程中会生成一个响应对象并最终通过调用 `response` 对象来返回给服务器。如果当中出错，就声称相应的错误信息。不管是否出错，最终 Flask 都会把请求上下文推出栈。

`full_dispatch_request` 是请求分发的入口，我们再来看它的实现：

```py
def full_dispatch_request(self):
    self.try_trigger_before_first_request_functions()
    try:
        rv = self.preprocess_request()
        if rv is None:
            rv = self.dispatch_request()
    except Exception as e:
        rv = self.handle_user_exception(e)
    return self.finalize_request(rv)
```

首先调用 `try_trigger_before_first_request_functions` 方法来尝试调用 `before_first_request` 列表中的函数，如果 `Flask` 的 `_got_first_request` 属性为 `False`，`before_first_request` 中的函数就会被执行，执行一次之后，`_got_first_request` 就会被设置为 `True` 从而不再执行这些函数。

然后调用 `preprocess_request` 方法，这个方法调用 `before_request_funcs` 列表中所有的方法，如果这些 `before_request_funcs` 方法中返回了某种东西，那么就不会真的去分发这个请求。比如说，一个 `before_request_funcs` 方法是用来检测用户是否登录的，如果用户没有登录，那么这个方法就会调用 `abort` 方法从而返回一个错误，Flask 就不会分发这个请求而是直接报 401 错误。

如果 `before_request_funcs` 中的函数没有返回，那么再调用 `dispatch_request` 方法进行请求分发。这个方法首先会查看 URL 规则中有没有相应的 `endpoint` 和 `value` 值，如果有，那么就调用 `view_functions` 中相应的视图函数（`endpoint` 作为键值）并把参数值传入（`**req.view_args`），如果没有就由 `raise_routing_exception` 进行处理。视图函数的返回值或者错误处理视图函数的返回值会返回给 `wsgi_app` 方法中的 `rv` 变量。

```py
def dispatch_request(self):
        req = _request_ctx_stack.top.request
        if req.routing_exception is not None:
            self.raise_routing_exception(req)
        rule = req.url_rule
        if getattr(rule, 'provide_automatic_options', False) \
           and req.method == 'OPTIONS':
            return self.make_default_options_response()
        return self.view_functions[rule.endpoint](**req.view_args)

def finalize_request(self, rv, from_error_handler=False):
    response = self.make_response(rv)
    try:
        response = self.process_response(response)
        request_finished.send(self, response=response)
    except Exception:
        if not from_error_handler:
            raise
        self.logger.exception('Request finalizing failed with an '
                              'error while handling an error')
    return response

def make_response(self, rv):
    if isinstance(rv, self.response_class):
        return rv
    if isinstance(rv, basestring):
        return self.response_class(rv)
    if isinstance(rv, tuple):
        return self.response_class(*rv)
    return self.response_class.force_type(rv, request.environ)
```

然后 Flask 就会根据 `rv` 生成响应，这个 `make_response` 方法会查看 rv 是否是要求的返回值类型，否则生成正确的返回类型。比如 Demo 中返回值是字符串，就会满足 `isinstance(rv, basestring)` 判断并从字符串生成响应。这一步完成之后，Flask 查看是否有后处理视图函数需要执行（在 `process_response` 方法中），并最终返回一个完全处理好的 `response` 对象。

## 视图函数注册

在请求处理过程一节中，我们已经看到了 Flask 是如何调用试图函数的，这一节我们要关注 Flask 如何构建和请求分派相关的数据结构。我们将主要关注 `view_functions`，因为其他的数据结构如 `before_request_funcs` 的构建过程大同小异，甚至更为简单。我们也将仔细讲解在应用的创建一节中遗留的问题，即 `url_map` 到底是什么。

Demo 代码的第 4 行用修饰器 `route` 注册一个视图函数，这是 Flask 中受到广泛称赞的一个设计。在 Flask 类的 `route` 方法中，可以看到它调用了 `add_url_rule` 方法。

```py
def route(self, rule, **options):
    def decorator(f):
        endpoint = options.pop('endpoint', None)
        self.add_url_rule(rule, endpoint, f, **options)
        return f
    return decorator

def add_url_rule(self, rule, endpoint, **options):
    if endpoint is None:
        endpoint = _endpoint_from_view_func(view_func)
    options['endpoint'] = endpoint
    methods = options.pop('methods', None)
    if methods is None:
        methods = getattr(view_func, 'methods', None) or ('GET',)
    if isinstance(methods, string_types):
        raise TypeError('Allowed methods have to be iterables of strings, '
                        'for example: @app.route(..., methods=["POST"])')
    methods = set(item.upper() for item in methods)

    required_methods = set(getattr(view_func, 'required_methods', ()))

    provide_automatic_options = getattr(view_func,
        'provide_automatic_options', None)

    if provide_automatic_options is None:
        if 'OPTIONS' not in methods:
            provide_automatic_options = True
            required_methods.add('OPTIONS')
        else:
            provide_automatic_options = False

    methods |= required_methods

    rule = self.url_rule_class(rule, methods=methods, **options)
    rule.provide_automatic_options = provide_automatic_options

    self.url_map.add(rule)
    if view_func is not None:
        old_func = self.view_functions.get(endpoint)
        if old_func is not None and old_func != view_func:
            raise AssertionError('View function mapping is overwriting an '
                                 'existing endpoint function: %s' % endpoint)
        self.view_functions[endpoint] = view_func
```

这个方法负责注册视图函数，并实现 URL 到视图函数的映射。首先，它要准备好一个视图函数所支持的 HTTP 方法（基本上一半多的代码都是在做这个），然后通过 `url_rule_class` 创建一个 `rule` 对象，并把这个对象添加到自己的 `url_map` 里。我们那个遗留问题在这里就得到解答了：`rule` 对象是一个保存合法的（Flask 应用所支持的） URL、方法、`endpoint`（在 `**options` 中） 及它们的对应关系的数据结构，而 `url_map` 是保存这些对象的集合。然后，这个方法将视图函数添加到 `view_functions` 当中，`endpoint` 作为它的键，其值默认是函数名。

我们再来深入了解一下 `rule` ，它被定义在 `werkzeug.routing.Rule` 中：

> A Rule represents one URL pattern. There are some options for `Rule` that change the way it behaves and are passed to the `Rule` constructor.
> 一个 Rule 对象代表了一种 URL 模式，可以通过传入参数来改变它的许多行为。

Rule 的 `__init__` 方法为：

```py
def __init__(self, string, defaults=None, subdomain=None, methods=None,
                 build_only=False, endpoint=None, strict_slashes=None,
                 redirect_to=None, alias=False, host=None):
    if not string.startswith('/'):
        raise ValueError('urls must start with a leading slash')
    self.rule = string
    self.is_leaf = not string.endswith('/')

    self.map = None
    self.strict_slashes = strict_slashes
    self.subdomain = subdomain
    self.host = host
    self.defaults = defaults
    self.build_only = build_only
    self.alias = alias
    if methods is None:
        self.methods = None
    else:
        if isinstance(methods, str):
            raise TypeError('param `methods` should be `Iterable[str]`, not `str`')
        self.methods = set([x.upper() for x in methods])
        if 'HEAD' not in self.methods and 'GET' in self.methods:
            self.methods.add('HEAD')
    self.endpoint = endpoint
    self.redirect_to = redirect_to

    if defaults:
        self.arguments = set(map(str, defaults))
    else:
        self.arguments = set()
    self._trace = self._converters = self._regex = self._weights = None
```

一个 Rule 被创建后，通过 `Map` 的 `add` 方法被绑定到 `Map` 对象上，我们之前说过 `flask.url_map` 就是一个 `Map` 对象。

```py
def add(self, rulefactory):
    for rule in rulefactory.get_rules(self):
        rule.bind(self)
        self._rules.append(rule)
        self._rules_by_endpoint.setdefault(rule.endpoint, []).append(rule)
    self._remap = True
```

而 `Rule` 的 `bind` 方法的内容，就是添加 `Rule` 对应的 `Map`，然后调用 `compile` 方法生成一个正则表达式，`compile` 方法比较复杂，就不展开了。

```py
def bind(self, map, rebind=False):
    """Bind the url to a map and create a regular expression based on
    the information from the rule itself and the defaults from the map.

    :internal:
    """
    if self.map is not None and not rebind:
        raise RuntimeError('url rule %r already bound to map %r' %
                           (self, self.map))
    self.map = map
    if self.strict_slashes is None:
        self.strict_slashes = map.strict_slashes
    if self.subdomain is None:
        self.subdomain = map.default_subdomain
    self.compile()
```

在 Flask 应用收到请求时，这些被绑定到 `url_map` 上的 `Rule` 会被查看，来找到它们对应的视图函数。这是在请求上下文中实现的，我们先前在 `dispatch_request` 方法中就见过——我们是从 `_request_ctx_stack.top.request` 得到 `rule` 并从这个 `rule` 找到 `endpoint`，最终找到用来处理该请求的正确的视图函数的。所以，接下来我们需要看请求上下的具体实现，并且看一看 Flask 是如何从 `url_map` 中找到这个 `rule` 的。

```py
def dispatch_request(self):
    req = _request_ctx_stack.top.request
    if req.routing_exception is not None:
        self.raise_routing_exception(req)
    rule = req.url_rule
    if getattr(rule, 'provide_automatic_options', False) \
       and req.method == 'OPTIONS':
        return self.make_default_options_response()
    return self.view_functions[rule.endpoint](**req.view_args)
```

## 请求上下文

> 源码阅读：`ctx.RequestContext` 的代码。

请求上下文是如何、在何时被创建的呢？我们先前也见过，在服务器调用应用的时候，Flask 的 `wsgi_app` 中有这样的语句，就是创建了请求上下文并压栈。

```py
def wsgi_app(self, environ, start_response):
    ctx = self.request_context(environ)
    ctx.push()
```

`request_context` 方法非常简单，就是创建了 `RequestContext` 类的一个实例，这个类被定义在 `flask.ctx` 文件中，它包含了一系列关于请求的信息，**最重要的是**它自身的 `request` 属性指向了一个 `Request` 类的实例，这个类继承自 `werkzeug.Request`，在 `RequestContext` 的创建过程中，它会根据传入的 `environ` 创建一个 `werkzeug.Request` 的实例。

接着 `RequestContext` 的 `push` 方法被调用，这个方法将自己推到 `_request_ctx_stack` 的栈顶。

`_request_ctx_stack` 被定义在 `flask.global` 文件中，它是一个 `LocalStack` 类的实例，是 `werkzeug.local` 所实现的，如果你对 Python 的 threading 熟悉的话，就会发现这里实现了线程隔离，就是说，在 Python 解释器运行到 `_request_ctx_stack` 相关代码的时候，解释器会根据当前进程来选择正确的实例。

但是，在整个分析 Flask 源码的过程中，我们也没发现 Flask 在被调用之后创建过线程啊，那么为什么要做线程隔离呢？看我们开头提到的 `run` 函数，其实它可以传一个 `threaded` 参数。当不传这个函数的时候，我们启动的是 `BasicWSGIServer`，这个服务器是单线程单进程的，Flask 的线程安全自然没有意义，但是当我们传入这个参数的时候，我们启动的是 `ThreadedWSGIServer`，这时 Flask 的线程安全就是有意义的了，在其他多线程的服务器中也是一样。

## 总结

### 一个请求的旅程

这里，我们通过追踪一个请求到达服务器并返回（当然是通过“变成”一个相应）的旅程，串讲本文的内容。

1. 在请求发出之前，Flask 注册好了所有的视图函数和 URL 映射，服务器在自己身上注册了 Flask 应用。
2. 请求到达服务器，服务器准备好 `environ` 和 `make_response` 函数，然后调用了自己身上注册的 Flask 应用。
3. 应用实现了 WSGI 要求的 `application(environ, make_response)` 方法。在 Flask 中，这个方法是个被 `__call__` 中转的叫做 `wsgi_app` 的方法。它首先通过 `environ` 创建了请求上下文，并将它推入栈，使得 flask 在处理当前请求的过程中都可以访问到这个请求上下文。
4. 然后 Flask 开始处理这个请求，依次调用 `before_first_request_funcs` `before_request_funcs` `view_functions` 中的函数，并最终通过 `finalize_request` 生成一个 `response` 对象，当中只要有函数返回值，后面的函数组就不会再执行，`after_request_funcs` 进行 `response` 生成后的后处理。
5. Flask 调用这个 `response` 对象，最终调用了 `make_response` 函数，并返回了一个可遍历的响应内容。
6. 服务器发送响应。

### Flask 和 werkzeug

在分析过程中，可以很明显地看出 Flask 和 `werkzeug` 是强耦合的，实际上 `werkzeug` 是 Flask 唯一不可或缺的依赖，一些非常细节的工作，其实都是 `werkzeug` 库完成的，在本文的例子中，它至少做了这些事情：

1. 封装 `Response` 和 `Request` 类型供 Flask 使用，在实际开发中，我们在请求和响应对象上的操作，调用的其实是 `werkzeug` 的方法。
2. 实现 URL 到视图函数的映射，并且能把 URL 中的参数传给该视图函数。我们看到了 Flask 的 url_map 属性并且看到了它如何绑定视图函数和错误处理函数，但是具体的映射规则的实践，和在响应过程中的 URL 解析，都是由 werkzeug 完成的。
3. 通过 LocalProxy 类生成的 `_request_ctx_stack` 对 Flask 实现线程保护。

对于 Flask 的源码解析先暂时到这里。有时间的话，我会分析 Flask 中的模板渲染、`import request`、蓝图和一些好用的变量及函数，或者深入分析 `werkzeug` 库。

## 参考阅读

1. [flask 源码解析系列文章](http://cizixs.com/2017/01/10/flask-insight-introduction)，你可以在读完本文了解主线之后，再看这系列文章了解更加细节的东西。
2. [一起写一个 web 服务器](http://python.jobbole.com/81524/)。
