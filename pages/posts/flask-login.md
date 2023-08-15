---
title: flask-login 源码解析
date: 2019/10/16
description: 这篇文章介绍了 flask-login 是如何实现一个不需要使用数据库的用户认证组件的
tag: Flask, python, source code, Chinese
author: Wenzhao
---

# flask-login 源码解析

## flask-login 的基本使用

在介绍 flask-login 的工作原理之前先来简要回顾一下 flask-login 的使用方法。

首先要创建一个 `LoginManager` 的实例并注册在 `Flask` 实例上，然后提供一个 `user_loader` 回调函数来根据会话中存储的用户 ID 来加载用户对象。

```py
login_manager = LoginManager()
login_manager.init_app(app)

@login_manager.user_loader
def load_user(user_id):
    return User.get(user_id)
```

flask-login 还要求你对数据对象做一些改动，添加以下属性和方法：

```py
@property
def is_active(self):
   return True

@property
def is_authenticated(self):
   return True

@property
def is_anonymous(self):
   return False

#: 这个方法返回一个能够识别唯一用户的 ID
def get_id(self):
   try:
       return text_type(self.id)
   except AttributeError:
       raise NotImplementedError('No `id` attribute - override `get_id`')
```

完成这些设置工作之后，就可以使用 flask-login 了，一些典型的用法包括登录和登出用户：`login_user(user)` 及 `logout_user()`，及使用 `@login_required` 保护一些视图函数，检测当前用户是否有访问的权限（根据是否认证进行区别）：

```py
@app.route("/settings")
@login_required
def settings():
    pass
```

以及通过 `current_user` 对象来访问当前用户。

## flask-login 源码解析

我们按照使用过程中调用 flask-login 的顺序来解析其源码。

### LoginManager 对象

先来看 `LoginManager` 对象，它用于记录所有的配置信息，其 `__init__` 方法中初始化了这些配置信息。一个 `LoginManager` 对象通过 init_app 方法注册到 Flask 实例上：

```py
def init_app(self, app, add_context_processor=True):
   app.login_manager = self
   app.after_request(self._update_remember_cookie)

   self._login_disabled = app.config.get('LOGIN_DISABLED', False)

   if add_context_processor:
       app.context_processor(_user_context_processor)
```

这个方法的主要工作是在 Flask 实例的 after_request 钩子上添加了一个用户更新 remember_me cookie 的函数，并在 Flask 的上下文处理器中添加了一个用户上下文处理器。

```py
def _user_context_processor():
    return dict(current_user=_get_user())
```

这个上下文处理器设置了一个全局可访问的变量 `current_user`，这样我们就可以在视图函数或者模板文件中访问这个变量了。

### user_loader 修饰器

然后就到了这个方法，它是 LoginManager 的实例方法，把 user_callback 设置成我们传入的函数，在实际的使用过程中，我们是通过修饰器传入这个函数的，就是 `load_user(user_id)` 函数。

```py
def user_loader(self, callback):
   self.user_callback = callback
   return callback
```

该方法要求你的回调函数必须能够接收一个 unicode 编码的 ID 并返回一个用户对象，如果用户不存在就返回 None。

### login_user 方法

我们跳过对 User 类的修改，直接来看这个方法。

```py
def login_user(user, remember=False, force=False, fresh=True):
    if not force and not user.is_active:
        return False

    user_id = getattr(user, current_app.login_manager.id_attribute)()
    session['user_id'] = user_id
    session['_fresh'] = fresh
    session['_id'] = current_app.login_manager._session_identifier_generator()

    if remember:
        session['remember'] = 'set'

    _request_ctx_stack.top.user = user
    user_logged_in.send(current_app._get_current_object(), user=_get_user())
    return True
```

如果用户不活跃 `not.is_active` 而且不要求强制登录 `force`，就返回失败。否则，先得到 `user_id`，它是通过 `getattr` 函数访问 `user` 的 `login_manager.id_attribute` 属性得到的。追根溯源，最终 `getattr` 访问的是 `user` 的 `get_id` 方法，这就是为什么 flask-login 要求我们在 `User` 类中添加该方法。

然后在 Flask 提供的 session 中添加以下三个 session：`user_id` `_fresh` `_id`，其中 `_id` 是通过 `LoginManager` 的 `_session_identifier_generator` 方法获取到的，而这个方法默认绑定在这个方法上：

```py
def _create_identifier():
    user_agent = request.headers.get('User-Agent')
    if user_agent is not None:
        user_agent = user_agent.encode('utf-8')
    base = '{0}|{1}'.format(_get_remote_addr(), user_agent)
    if str is bytes:
        base = text_type(base, 'utf-8', errors='replace')  # pragma: no cover
    h = sha512()
    h.update(base.encode('utf8'))
    return h.hexdigest()
```

不用太深究，知道这个方法最终根据放着用户代理和 IP 信息生成了一个加盐的 ID 就行了，它的作用是防止有人伪造 cookie。

然后根据是否需要记住用户添加 `remember` session。最后，在 `_request_ctx_stack.top` 中添加该用户，发出一个用户登录信号后返回成功。在这个登录信号中，调用了 `_get_user` 方法，`_get_user` 方法的细节是先检测在 `_request_ctx_stack.top` 中有没有用户信息，如果没有，就通过 `_load_user` 方法在栈顶添加用户信息，如果有就返回这个用户对象。`_load_user` 方法很重要，但是在这里不会被调用，很明显 `_request_ctx_stack.top` 中肯定有 `user` 值，我们待会再来看这个方法。

```py
def _get_user():
    if has_request_context() and not hasattr(_request_ctx_stack.top, 'user'):
        current_app.login_manager._load_user()

    return getattr(_request_ctx_stack.top, 'user', None)
```

### login_required 修饰器

这个修饰器常被用来保护只有登录用户才能访问的视图函数，它会在实际调用视图函数之前先检查当前用户是否已经登录并认证，如果没有，就调用 `LoginManager.unauthorized` 这个回调函数，它还对一些 HTTP 方法和测试情况提供了例外处理。

```py
def login_required(func):
    @wraps(func)
    def decorated_view(*args, **kwargs):
        if request.method in EXEMPT_METHODS:
            return func(*args, **kwargs)
        elif current_app.login_manager._login_disabled:
            return func(*args, **kwargs)
        elif not current_user.is_authenticated:
            return current_app.login_manager.unauthorized()
        return func(*args, **kwargs)
    return decorated_view
```

### current_user 对象

在之前的分析中，可以看到这个变量经常出现并大有用途，开发者可以通过访问这个变量来获取到当前用户，如果用户未登录，获取到的就是一个匿名用户，它的定义：

```py
current_user = LocalProxy(lambda: _get_user())
```

`_get_user()` 方法之前已经讲过，我们直接跳到 `_load_user` 方法。显然，如果用户登录后再次发出了请求，我们就要从 cookie，或者说，Flask 在此之上封装的 session 中获取用户信息才能正确地进行后续处理，`_load_user` 方法的作用就是这个，该方法如下：

```py
def _load_user(self):
   user_accessed.send(current_app._get_current_object())

   config = current_app.config
   if config.get('SESSION_PROTECTION', self.session_protection):
       deleted = self._session_protection()
       if deleted:
           return self.reload_user()

   is_missing_user_id = 'user_id' not in session
   if is_missing_user_id:
       cookie_name = config.get('REMEMBER_COOKIE_NAME', COOKIE_NAME)
       header_name = config.get('AUTH_HEADER_NAME', AUTH_HEADER_NAME)
       has_cookie = (cookie_name in request.cookies and
                     session.get('remember') != 'clear')
       if has_cookie:
           return self._load_from_cookie(request.cookies[cookie_name])
       elif self.request_callback:
           return self._load_from_request(request)
       elif header_name in request.headers:
           return self._load_from_header(request.headers[header_name])

   return self.reload_user()

def _session_protection(self):
   sess = session._get_current_object()
   ident = self._session_identifier_generator()

   app = current_app._get_current_object()
   mode = app.config.get('SESSION_PROTECTION', self.session_protection)

   if sess and ident != sess.get('_id', None):
       if mode == 'basic' or sess.permanent:
           sess['_fresh'] = False
           session_protected.send(app)
           return False
       elif mode == 'strong':
           for k in SESSION_KEYS:
               sess.pop(k, None)

           sess['remember'] = 'clear'
           session_protected.send(app)
           return True

   return False
```

该方法首先保证 session 的安全，如果 session 通过了安全验证，就通过 `reload_user` 方法重载用户，否则检查 session 中是否没有 `user_id` 来重载用户，如果没有，通过三种不同的方式重载用户。

```py
def reload_user(self, user=None):
    ctx = _request_ctx_stack.top

    if user is None:
        user_id = session.get('user_id')
        if user_id is None:
            ctx.user = self.anonymous_user()
        else:
            if self.user_callback is None:
                raise Exception(
                    "No user_loader has been installed for this "
                    "LoginManager. Add one with the "
                    "'LoginManager.user_loader' decorator.")
            user = self.user_callback(user_id)
            if user is None:
                ctx.user = self.anonymous_user()
            else:
                ctx.user = user
    else:
        ctx.user = user
```

在这个重载方法中，如果 `user_id` 不存在，就把匿名用户加载到 `_request_ctx_stack.top`，否则根据 `user_id` 加载用户，若该用户不存在，仍加载匿用户。

之后，`current_user` 就能获取到用户对象，或者是一个匿名用户对象了。

```py
current_user = LocalProxy(lambda: _get_user())
```

### logout_user 方法

这个方法先获取当前用户，然后移除 `user_id` `_fresh` 等 session，然后移除 `remember`，最后重载当前用户，很明显，重载之后会是一个匿名用户。

```py
def logout_user():
    user = _get_user()

    if 'user_id' in session:
        session.pop('user_id')

    if '_fresh' in session:
        session.pop('_fresh')

    cookie_name = current_app.config.get('REMEMBER_COOKIE_NAME', COOKIE_NAME)
    if cookie_name in request.cookies:
        session['remember'] = 'clear'

    user_logged_out.send(current_app._get_current_object(), user=user)

    current_app.login_manager.reload_user()
    return True
```

### remember_me cookie

记得我们之前提到 flask-login 在 Flask 实例的 after_request 钩子上添加了一个用户更新 remember_me cookie 的函数吗，我们显然需要在请求的最后对 `remember` 进行处理。

```py
def _update_remember_cookie(self, response):
   # Don't modify the session unless there's something to do.
   if 'remember' in session:
       operation = session.pop('remember', None)
       if operation == 'set' and 'user_id' in session:
           self._set_cookie(response)
       elif operation == 'clear':
           self._clear_cookie(response)
    return response
```

这个函数根据是否要设置 `remember` 来调用不同的函数

```py
def _set_cookie(self, response):
    config = current_app.config
    cookie_name = config.get('REMEMBER_COOKIE_NAME', COOKIE_NAME)
    duration = config.get('REMEMBER_COOKIE_DURATION', COOKIE_DURATION)
    domain = config.get('REMEMBER_COOKIE_DOMAIN')
    path = config.get('REMEMBER_COOKIE_PATH', '/')

    secure = config.get('REMEMBER_COOKIE_SECURE', COOKIE_SECURE)
    httponly = config.get('REMEMBER_COOKIE_HTTPONLY', COOKIE_HTTPONLY)

    data = encode_cookie(text_type(session['user_id']))

    try:
        expires = datetime.utcnow() + duration
    except TypeError:
        raise Exception('REMEMBER_COOKIE_DURATION must be a ' +
                        'datetime.timedelta, instead got: {0}'.format(
                            duration))

    response.set_cookie(cookie_name,
                        value=data,
                        expires=expires,
                        domain=domain,
                        path=path,
                        secure=secure,
                        httponly=httponly)

def _clear_cookie(self, response):
    config = current_app.config
    cookie_name = config.get('REMEMBER_COOKIE_NAME', COOKIE_NAME)
    domain = config.get('REMEMBER_COOKIE_DOMAIN')
    path = config.get('REMEMBER_COOKIE_PATH', '/')
    response.delete_cookie(cookie_name, domain=domain, path=path)
```

## 总结

- flask-login 使用 Flask 提供的 session 来保存用户信息，通过 `user_id` 来记录用户身份，`_id` 来防止攻击者对 session 的伪造。
- 通过 `_request_ctx_stack.top.user`，flask-login 实现了线程安全。
- 通过 cookie 来实现 remember 功能。

其他功能如 fresh login 请自行查看源码了解。

## 仿造 flask-login 写一个基于 token 的身份认证模块

flask-login 虽然好用，但由于其是基于 session 的，对于无状态的 RESTful API 应用无能为力。我在一个最近的项目模仿了它的接口，实现了一个简单但是好用的身份认证模块。

```py
from functools import wraps
from flask import (_request_ctx_stack, has_request_context, request,
                   current_app)
from flask_restful import abort
from werkzeug.local import LocalProxy
from app.models.user import User

#: a proxy for the current user
#: it would be an anonymous user if no user is logged in
current_user = LocalProxy(lambda: _get_user())


class AnonymousUserMixin(object):
    @property
    def is_active(self):
        return False

    @property
    def is_authenticated(self):
        return False

    @property
    def is_anonymous(self):
        return True

    def __repr__(self):
        return ''


class Manager(object):
    def __init__(self, app=None):
        if app:
            self.init_app(app)

    def init_app(self, app):
        app.login_manager = self
        app.context_processor(_user_context_processor)

        self._anonymous_user = AnonymousUserMixin
        self._login_disabled = app.config['LOGIN_DISABLED'] or False

    @staticmethod
    def _load_user():
        """Try to load user from request.json.token and set it to
        `_request_ctx_stack.top.user`. If None, set current user as an anonymous
        user.
        """
        ctx = _request_ctx_stack.top
        json = request.json
        user = AnonymousUserMixin()

        if json and json.get('token'):
            real_user = User.load_user_from_auth_token(json.get('token'))
            if real_user:
                user = real_user

        ctx.user = user


def _get_user():
    """Get current user from request context."""
    if has_request_context() and not hasattr(_request_ctx_stack.top, 'user'):
        current_app.login_manager._load_user()

    return getattr(_request_ctx_stack.top, 'user', None)


def _user_context_processor():
    """A context processor to prepare current user."""
    return dict(current_user=_get_user())


def login_user(user):
    """Login a user and return a token."""
    _request_ctx_stack.top.user = user
    return user.generate_auth_token()


def logout_user(user):
    """For a restful API there shouldn't be a `logout` method because the
    server is stateless.
    """
    pass


def login_required(func):
    """Decorator to protect view functions that should only be accessed
    by authenticated users.
    """

    @wraps(func)
    def decorated_view(*args, **kwargs):
        if current_app.login_manager._login_disabled:
            return func(*args, **kwargs)
        elif not current_user.is_authenticated:
            abort(403, err='40300',
                  message='Please login before carrying out this action.')
        return func(*args, **kwargs)

    return decorated_view
```
