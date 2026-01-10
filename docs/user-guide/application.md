---
subtitle: Application
description: Celery应用程序指南：详细讲解Celery应用程序的创建、配置管理、任务注册、主模块命名、配置加载方法（config_from_object、config_from_envvar）、惰性实例化机制、应用程序链最佳实践以及抽象任务和自定义任务类的实现。
---

# 应用程序

Celery 在使用前必须被实例化，这个实例被称为应用程序（或简称为 *app*）。

应用程序是线程安全的，因此具有不同配置、组件和任务的多个 Celery 应用程序可以在同一个进程空间中共存。

现在让我们创建一个：

```pycon
>>> from celery import Celery
>>> app = Celery()
>>> app
<Celery __main__:0x100469fd0>
```

最后一行显示了应用程序的文本表示：包括应用程序类的名称（`Celery`）、当前主模块的名称（`__main__`）和对象的内存地址（`0x100469fd0`）。

## 主名称（Main Name）

这些中只有一个是重要的，那就是主模块名称。让我们看看为什么是这样。

当你在 Celery 中发送任务消息时，该消息不会包含任何源代码，只包含你想要执行的任务的名称。这类似于互联网上主机名的工作方式：每个工作器维护一个任务名称到其实际函数的映射，称为 *任务注册表*。

每当你定义一个任务时，该任务也会被添加到本地注册表中：

```pycon
>>> @app.task
... def add(x, y):
...     return x + y

>>> add
<@task: __main__.add>

>>> add.name
__main__.add

>>> app.tasks['__main__.add']
<@task: __main__.add>
```

在那里你又看到了 `__main__`；每当 Celery 无法检测到函数属于哪个模块时，它会使用主模块名称来生成任务名称的开头。

这仅在有限的用例中是个问题：

1. 如果定义任务的模块作为程序运行。
2. 如果应用程序是在 Python shell（REPL）中创建的。

例如这里，任务模块也用于使用 `app.worker_main()` 启动工作器：

```python title="tasks.py"
from celery import Celery
app = Celery()

@app.task
def add(x, y): return x + y

if __name__ == '__main__':
    args = ['worker', '--loglevel=INFO']
    app.worker_main(argv=args)
```

当这个模块被执行时，任务将以 `__main__` 开头命名，但当模块被另一个进程导入时，比如调用任务，任务将以 `tasks`（模块的真实名称）开头命名：

```pycon
>>> from tasks import add
>>> add.name
tasks.add
```

你可以为 main 模块指定另一个名称：

```pycon
>>> app = Celery('tasks')
>>> app.main
'tasks'

>>> @app.task
... def add(x, y):
...     return x + y

>>> add.name
tasks.add
```

!!! quote "[任务指南 - 名称](tasks.md#task-names){target="_blank"}"

## 配置（Configuration）

你可以设置几个选项来改变 Celery 的工作方式。这些选项可以直接在应用程序实例上设置，或者你可以使用专用的配置模块。

配置可作为 `app.conf` 使用：

```pycon
>>> app.conf.timezone
'Europe/London'
```

你也可以直接设置配置值：

```pycon
>>> app.conf.enable_utc = True
```

或者使用 `update()` 方法一次更新多个键：

```pycon
>>> app.conf.update(
...     enable_utc=True,
...     timezone='Europe/London',
... )
```

配置对象由多个按顺序查询的字典组成：

1. 运行时进行的更改。
2. 配置模块（如果有）
3. 默认配置（`celery.app.defaults`）。

你甚至可以使用 `app.add_defaults()` 方法添加新的默认源。

!!! quote "[完整配置参考：查看所有可用设置及其默认值的完整列表。](configuration.md){target="_blank"}"

### `config_from_object`

`app.config_from_object()` 方法从配置对象加载配置。

这可以是一个配置模块，或者任何具有配置属性的对象。

请注意，当调用 `app.config_from_object()` 时，任何先前设置的配置都将被重置。如果你想设置额外的配置，应该在此之后进行。

#### 示例 1：使用模块名称

`app.config_from_object()` 方法可以接受 Python 模块的完全限定名称，甚至是 Python 属性的名称，例如：`celeryconfig`、`myproj.config.celery` 或 `myproj.config:CeleryConfig`：

```python
from celery import Celery

app = Celery()
app.config_from_object('celeryconfig')
```

`celeryconfig` 模块可能看起来像这样：

```python title="celeryconfig.py"
enable_utc = True
timezone = 'Europe/London'
```

只要 `import celeryconfig` 是可能的，应用程序就能够使用它。

#### 示例 2：传递实际的模块对象

你也可以传递一个已经导入的模块对象，但这并不总是推荐的。

!!! tip

    推荐使用模块的名称，因为这意味着在使用 prefork 池时不需要序列化模块。如果你遇到配置问题或 pickle 错误，请尝试使用模块名称代替。

```python
import celeryconfig

from celery import Celery

app = Celery()
app.config_from_object(celeryconfig)
```

#### 示例 3：使用配置类/对象

```python
from celery import Celery

app = Celery()

class Config:
    enable_utc = True
    timezone = 'Europe/London'

app.config_from_object(Config)
# 或者使用对象的完全限定名称：
#   app.config_from_object('module:Config')
```

### `config_from_envvar`

`app.config_from_envvar()` 方法从环境变量中获取配置模块名称

例如——要从名为 `CELERY_CONFIG_MODULE` 的环境变量指定的模块加载配置：

```python
import os
from celery import Celery

#: 设置默认配置模块名称
os.environ.setdefault('CELERY_CONFIG_MODULE', 'celeryconfig')

app = Celery()
app.config_from_envvar('CELERY_CONFIG_MODULE')
```

然后你可以通过环境指定要使用的配置模块：

```console
CELERY_CONFIG_MODULE="celeryconfig.prod" celery worker -l INFO
```

### 审查配置

如果你想要打印出配置作为调试信息或类似用途，你可能还想过滤掉敏感信息，如密码和 API 密钥。

Celery 附带了一些用于呈现配置的有用工具，其中之一是 `humanize()`：

```pycon
>>> app.conf.humanize(with_defaults=False, censored=True)
```

此方法将配置作为表格字符串返回。默认情况下，这仅包含对配置的更改，但你可以通过启用 `with_defaults` 参数来包含内置的默认键和值。

如果你希望将配置作为字典处理，可以使用 `table()` 方法：

```pycon
>>> app.conf.table(with_defaults=False, censored=True)
```

请注意，Celery 无法删除所有敏感信息，因为它仅使用正则表达式搜索常用命名的键。如果你添加包含敏感信息的自定义设置，应该使用 Celery 识别为机密的名称来命名键。

如果配置设置的名称包含以下任何子字符串，将被审查：

`API`, `TOKEN`, `KEY`, `SECRET`, `PASS`, `SIGNATURE`, `DATABASE`

## 惰性（Laziness）

应用程序实例是惰性的，意味着它只有在实际需要时才会被求值。

创建 `Celery` 实例只会执行以下操作：

1. 创建一个逻辑时钟实例，用于事件。
2. 创建任务注册表。
3. 将自己设置为当前应用程序（但如果 `set_as_current` 参数被禁用则不会）
4. 调用 `app.on_init()` 回调（默认情况下不执行任何操作）。

`app.task()` 装饰器不会在定义任务时创建任务，而是将任务的创建推迟到任务被使用时，或者在应用程序被*最终化*之后。

这个例子展示了任务直到你使用任务或访问属性（在这种情况下是 `__repr__`）时才会被创建：

```pycon
>>> @app.task
>>> def add(x, y):
...    return x + y

>>> type(add)
<class 'celery.local.PromiseProxy'>

>>> add.__evaluated__()
False

>>> add        # <-- 导致 repr(add) 发生
<@task: __main__.add>

>>> add.__evaluated__()
True
```

应用程序的 *最终化* 可以通过显式调用 `app.finalize()` 发生，或者通过隐式访问 `app.tasks` 属性发生。

最终化对象将：

1. 复制必须在应用程序之间共享的任务默认情况下任务是被共享的，但如果任务装饰器的 `shared` 参数被禁用，那么任务将对其绑定的应用程序私有。
2. 评估所有挂起的任务装饰器。
3. 确保所有任务都绑定到当前应用程序。

    任务绑定到应用程序，以便它们可以从配置中读取默认值。

!!! note ""默认应用程序""

    Celery 并不总是有应用程序，过去只有基于模块的 API。在 Celery 5.0 发布之前，旧位置提供了兼容性 API，但已被移除。

    Celery 总是创建一个特殊的应用程序 - "默认应用程序"，如果没有实例化自定义应用程序，就会使用这个应用程序。

    `celery.task` 模块不再可用。使用应用程序实例上的方法，而不是基于模块的 API：

    ```python
    from celery.task import Task   # << 旧的 Task 基类

    from celery import Task        # << 新的基类
    ```

## 打破链（Breaking the chain）

虽然可以依赖当前设置的应用程序，但最佳实践是始终将应用程序实例传递给任何需要它的地方。

我称之为"应用程序链（app chain）"，因为它创建了一个依赖于传递的应用程序的实例链。

以下示例被认为是糟糕的做法：

```python
from celery import current_app

class Scheduler:

    def run(self):
        app = current_app
```

相反，它应该将 `app` 作为参数：

```python
class Scheduler:

    def __init__(self, app):
        self.app = app
```

在内部，Celery 使用 `celery.app.app_or_default()` 函数，以便所有内容也能在基于模块的兼容性 API 中工作

```python
from celery.app import app_or_default

class Scheduler:
    def __init__(self, app=None):
        self.app = app_or_default(app)
```

在开发中，你可以设置 `CELERY_TRACE_APP` 环境变量，以便在应用程序链断开时引发异常：

```console
CELERY_TRACE_APP=1 celery worker -l INFO
```

!!! note "API 的演进"

    Celery 自 2009 年最初创建以来已经发生了很大变化。

    例如，在开始时，可以使用任何可调用对象作为任务：

    ```pycon
    def hello(to):
        return 'hello {0}'.format(to)

    >>> from celery.execute import apply_async

    >>> apply_async(hello, ('world!',))
    ```

    或者你也可以创建一个 `Task` 类来设置某些选项，或覆盖其他行为

    ```python
    from celery import Task
    from celery.registry import tasks

    class Hello(Task):
        queue = 'hipri'

        def run(self, to):
            return 'hello {0}'.format(to)
    tasks.register(Hello)

    >>> Hello.delay('world!')
    ```

    后来，人们认为传递任意可调用对象是一种反模式，因为它使得使用除 pickle 之外的其他序列化器变得非常困难，该功能在 2.0 版本中被移除，被任务装饰器取代：

    ```python
    from celery import app

    @app.task(queue='hipri')
    def hello(to):
        return 'hello {0}'.format(to)
    ```

## 抽象任务（Abstract Tasks）

所有使用 `app.task()` 装饰器创建的任务都将继承应用程序的基础 `Task` 类。

你可以使用 `base` 参数指定不同的基类：

```python
@app.task(base=OtherTask)
def add(x, y):
    return x + y
```

要创建自定义任务类，你应该继承中性基类：`celery.Task`。

```python
from celery import Task

class DebugTask(Task):

    def __call__(self, *args, **kwargs):
        print('TASK STARTING: {0.name}[{0.request.id}]'.format(self))
        return self.run(*args, **kwargs)
```

!!! tip

    如果你重写了任务的 `__call__` 方法，那么非常重要的一点是你也要调用 `self.run` 来执行任务的主体。不要调用 `super().__call__`。中性基类 `celery.Task` 的 `__call__` 方法仅用于参考。为了优化，这已经被展开到 `celery.app.trace.build_tracer.trace_task` 中，如果没有定义 `__call__` 方法，它会直接在自定义任务类上调用 `run`。

中性基类是特殊的，因为它还没有绑定到任何特定的应用程序。一旦任务绑定到应用程序，它将读取配置来设置默认值，等等。

要实现一个基类，你需要使用 `app.task()` 装饰器创建一个任务：

```python
@app.task(base=DebugTask)
def add(x, y):
    return x + y
```

甚至可以通过更改应用程序的 `app.Task()` 属性来更改应用程序的默认基类：

```pycon
>>> from celery import Celery, Task

>>> app = Celery()

>>> class MyBaseTask(Task):
...    queue = 'hipri'

>>> app.Task = MyBaseTask
>>> app.Task
<unbound MyBaseTask>

>>> @app.task
... def add(x, y):
...     return x + y

>>> add
<@task: __main__.add>

>>> add.__class__.mro()
[<class add of <Celery __main__:0x1012b4410>>,
 <unbound MyBaseTask>,
 <unbound Task>,
 <type 'object'>]
```
