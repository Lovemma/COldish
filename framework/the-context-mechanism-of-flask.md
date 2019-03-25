# Flask 的 Context 机制
使用过 Flask 进行 Web 开发的同学应该知道 App Context 和 Request Context 这两个非常有特色的设计。
PS: 当然, Flask 的官方文档也对 [App Context](http://flask.pocoo.org/docs/1.0/appcontext/) 和 [Request Context](http://flask.pocoo.org/docs/1.0/reqcontext/) 作出了详细的解释。

从一个 Flask App 读入配置并启动开始, 就进入了 App Context, 在其中我们可以访问配置文件、打开资源文件、通过路由规则反向构造 URL。当一个请求进入开始被处理时, 就进入了 Request Context, 在其中我们可以访问请求携带的信息, 比如 HTTP Method、表单域等。

最近闲着没什么事做, 研究了一番这两个 Context 的具体实现, 同时还解决了一些自己之前“知道结论不知道过程”的疑惑, 所以撰写本文记录下来。

## Thread Local 的概念
在面向对象的设计中, 对象是保存"状态"的地方。Python 也是如此, 一个对象的状态都被保存在对象携带的一个特殊字典(\_\_dict\_\_)中, 可以通过`vars`函数拿到它。

Thread Local 则是一种特殊的对象, 它的"状态"对线程隔离——也就是说每个线程对 Thread Local 对象的修改都不会影响到其他线程。这种对象的实现原理也非常简单, 只要以线程的 ID 来保存多份状态字典即可, 就像按照门牌号隔开的信箱。

在 Python 中获得一个这样的 Thread Local 最简单的方法是`threading.local`:
```python
import threading

storage = threading.local()
storage.foo = 1
print(storage.foo)

class AnotherThread(threading.Thread):
    def run(self):
        storage.foo = 2
        # storage.foo 在这个线程中已经发生改变
        print(storage.foo)

another = AnotherThread()
another.start()

# 但是在主线程里并没有改变
print(storage.foo)

```
这样来说, 只要能构造出 Thread Local 对象, 就能够让同一个对象在多个线程下做到状态隔离。这个"线程"不一定要是系统线程, 也可以是用户代码中的其他调度单元, 例如 Greenlet。

## Werkzeug 实现的 Local Stack 和 Local Proxy
Werkzeug 没有直接使用`threading.local`, 而是自己实现了`werkzeug.local.Local`类。后者和前者有一些区别:
1. Werkzeug 实现的 Local 类会在 Greenlet 可用的情况下优先使用 Greenlet 的 ID 而不是线程 ID 以支持 Gevent 或者 Eventlet 的调度, 而`threading.local`只支持多线程调度;
2. Werkzeug 的 Local 类实现了 Werkzeug 自定义的协议方法`__release_local__`, 可以被 Werkzeug 自己的`release_local`函数释放掉当前线程下的状态, 而`threading.local`没有这个能力。

除 Local 外, Werkzeug 还实现了两种数据结构：LocalStack 和 LocalProxy。

LocalStack 是用 Local 实现的栈结构, 可以将对象推入、弹出, 也可以快速拿到栈顶对象。当然, 所有的修改都只在本线程可见。和 Local 一样, LocalStack 也同样实现了支持 release_local 的接口。

LocalProxy 则是一个典型的代理模式实现, 它在构造时接受一个 callable 的参数（比如一个函数）, 这个参数被调用后的返回值本身应该是一个 Thread Local 对象。对一个 LocalProxy 对象的所有操作, 包括属性访问、方法调用（当然方法调用就是属性访问）甚至是二元操作都会转发到那个 callable 参数返回的 Thread Local 对象上。

LocalProxy 的一个使用场景是 LocalStack 的`__call__`方法。例如, `my_local_stack`是一个 LocalStack 实例, 那么调用`my_local_stack()`会返回一个 LocalProxy 对象, 这个对象始终指向`my_local_stack`的栈顶元素。如果栈顶元素不存在, 访问这个 LocalProxy 对象会抛出`RuntimeError`。

## Flask 基于 LocalStack 的 Context
Flask 是一个基于 Werkzeug 实现的框架, 所以 Flask 的 App Context 和 Request Context 也理所当然地基于 Werkzeug 的 Local Stack 实现。

在概念上, App Context 代表了"应用级别的上下文", 比如配置文件中数据库的连接信息; Request Context 代表了"请求级别的上下文", 比如当前访问的 URL。

这两个上下文对象的类定义在`flask.ctx`中, 它们的用法是将应用上下文和请求上下文推入推出到`flask.globals`中创建的`_app_ctx_stack`和`_request_ctx_stack`这两个 LocalStack 单例中。

因为 LocalStack 的状态是线程隔离的, 而 Web 应用中每个线程(或者 Greenlet)同时只能处理一个请求, 所以 App Context 对象和 Request Context 对象在请求中也是隔离的。

当`app = Flask(__name__)`构造出一个 Flask App 时, App Context 并不会自动被推入 Stack 中。所以此时 Local Stack 的栈顶是空的, `current_app`也是 unbound 状态。

```python
>>> from flask import Flask
>>> from flask.globals import _app_ctx_stack, _request_ctx_stack
>>>
>>> app = Flask(__name__)
>>> _app_ctx_stack.top
>>> _request_ctx_stack.top
>>> _app_ctx_stack()
<LocalProxy unbound>
>>>
>>> from flask import current_app
>>> current_app
<LocalProxy unbound>
```

这也是 Flask 的一些使用者可能被坑的地方——比如编写一个离线脚本时, 如果直接在一个 Flask-SQLAlchemy 写成的 Model 上调用`User.query.get(user_id)`, 就会遇到`RuntimeError`。因为此时 App Context 还没被推入栈中, 而 Flask-SQLAlchemy 需要数据库连接信息时, 就会去`current_app.config`中获取, `current_app`指向的却是`_app_ctx_stack`为空的栈顶。

解决办法是运行脚本之前, 先将 App 的 App Context 推入栈中, 栈顶不为空后, `current_app`这个 LocalProxy 对象自然能将"取 config 属性"这个动作转发到当前的 App 上。

```python
>>> ctx = app.app_context()
>>> ctx.push()
>>> _app_ctx_stack.top
<flask.ctx.AppContext object at 0x102eac7d0>
>>> _app_ctx_stack.top is ctx
True
>>> current_app
<Flask '__main__'>
>>>
>>> ctx.pop()
>>> _app_ctx_stack.top
>>> current_app
<LocalProxy unbound>
```

那么为什么在应用运行时不需要手动调用`app_context().push()`呢? 因为 Flask App 作为 WSGI Application 运行时, 会在每个请求进入的时候将请求上下文推入`_app_ctx_stack`中, 而请求上下文一定是在应用上下文中的, 所以推入部分的逻辑有这样一条: 如果发现`_app_ctx_stack`为空, 则隐式的推入一个 App 上下文。

所以, 请求中是不需要手动推上下文入栈的, 但是离线脚本需要手动推入上下文。

## 两个疑问
这里还有两个疑问需要处理:
- 为什么 App Context 需要独立出来: 既然在 Web 应用运行时, App Context 和 Request Context 都是 Thread Local 对象, 那么为什么还要独立区分二者呢?
- 为什么要放在"栈"里: 在 Web 应用运行时, 一个线程同时只能处理一个请求, 那么`_app_ctx_stack`和`_request_ctx_stack`栈顶肯定只有一个元素, 为什么还要用栈这种结构呢?

这两个做法给予我们**多个 Flask App 共存**和**非 Web 应用运行时灵活控制 Context** 的可能性。

在对一个 Flask App 调用`app.run()`之后, 进程就进入阻塞模式并开始监听请求。此时是不可能让另一个 Flask App 在主线程运行起来的, 那么还有哪些场景需要多个 Flask App 共存呢?

之前说过 Flask App 实例就是一个 WSGI Application, 而 WSGI Middleware 是允许使用组合模式的, 比如:
```python
from werkzeug.wsgi import DispatcherMiddleware
from your_application import app
from your_application.admin import app as admin_app

application = DispatcherMiddleware(app, {'/admin': admin_app})
```
这个例子就是使用 Werkzeug 内置的 Middleware 将两个 Flask App 组合成一个 WSGI Application。这种情况下两个 App 都同时在运行, 只是根据请求 URL 的不同将请求分发到不同的 App 上进行处理。

> 需要注意的是, 这种用法和 Flask 的 Blueprint 是有区别的。Blueprint 虽然和这种用法很类似, 但 Blueprint 自己是没有 App Context 的, 只是同一个 Flask App 内部资源共享的一种方式, 所以多个 Blueprint 共享了同一个 Flask App; 而使用 Middleware 面向的是所有的 WSGI Application, 不仅仅是 Flask App, 即使是把 Django App 和 Flask App 用这种方式组合起来也是可行的。

## 在非 Web 环境运行 Flask 关联的代码
设想, 一个离线脚本需要操作两个甚至多个 Flask App 关联的上下文, 应该怎么办呢? 这时候栈结构的 App Context 的优势就展现出来了。
```python
from your_application import create_app
from your_application.admin import create_app as create_admin_app

app = create_app()
admin_app = create_admin_app()

def copy_data():
    with app.app_context():
        data = read_data()
        with admin_app.app_context():
            write_data(data)
        mark_data_copied()
```
无论有多少个 App, 只要主动去 push 它的 App Context, Context Stack 中就会累计起来。这样, 栈顶永远是当前操作的 App Context, 当一个 App Context 结束的时候, 相应的栈顶元素也随之出栈。如果在执行过程中抛出了异常, 对应的 App Context 中注册的`teardown`函数被传入带有异常信息的参数。

这么一来就解释了之前的两个疑问——在这种单线程运行环境中, 只有栈结构才能保存多个 Context 并在其中定位出哪个才是"当前"。而离线脚本只需要 App 关联的上下文, 不需要构造出请求, 所以 App Context 也应该和 Request Context 分离。