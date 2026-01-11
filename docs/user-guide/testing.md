---
subtitle: Testing
description: Celery测试指南：单元测试与集成测试的完整解决方案，涵盖pytest插件、任务模拟、实时工作器配置和最佳实践。
---

# 使用 Celery 进行测试

使用 Celery 进行测试分为两个部分：

- 单元测试和集成测试：使用 `celery.contrib.pytest`
- 冒烟测试/生产测试：使用 `pytest-celery` >= 1.0.0

安装 pytest-celery 插件也会安装 `celery.contrib.pytest` 基础设施，以及 pytest 插件基础设施。区别在于如何使用它们。

!!! warning

     这两个 API 彼此不兼容。pytest-celery 插件基于 Docker，而 `celery.contrib.pytest` 基于模拟（mock）。

要使用 `celery.contrib.pytest` 基础设施，请遵循以下说明。

pytest-celery 插件有[自己的文档](https://pytest-celery.readthedocs.io/)。

## 任务和单元测试

在单元测试中测试任务行为的首选方法是模拟（mocking）。

!!! tip "Eager 模式"

    由 `task_always_eager` 设置启用的 eager 模式根据定义不适合用于单元测试。

    当使用 eager 模式进行测试时，您只是在测试一个模拟的工作进程行为，而模拟和实际情况之间存在很多差异。

    请注意，默认情况下，急切执行的任务不会将结果写入后端。如果您想启用此功能，请查看 `task_store_eager_result`。

Celery 任务很像 Web 视图，因为它应该只定义在被作为任务调用时如何执行操作。

这意味着理想情况下，任务只处理序列化、消息头、重试等事情，而实际逻辑在其他地方实现。

假设我们有这样一个任务：

```python
from .models import Product


@app.task(bind=True)
def send_order(self, product_pk, quantity, price):
    price = Decimal(price)  # json 将其序列化为字符串。

    # 模型通过 id 传递，而不是序列化。
    product = Product.objects.get(product_pk)

    try:
        product.order(quantity, price)
    except OperationalError as exc:
        raise self.retry(exc=exc)
```

`注意`：一个任务被[绑定](https://docs.celeryq.dev/en/latest/userguide/tasks.html#bound-tasks)意味着任务的第一个参数将始终是任务实例（self）。这意味着您确实会得到一个 self 参数作为第一个参数，并且可以使用 Task 类的方法和属性。

您可以使用模拟为此任务编写单元测试，
如以下示例所示：

```python
from pytest import raises

from celery.exceptions import Retry

# 对于 python 2：使用来自 `pip install mock` 的 mock.patch。
from unittest.mock import patch

from proj.models import Product
from proj.tasks import send_order

class test_send_order:

    @patch('proj.tasks.Product.order')  # < 修补上面模块中的 Product
    def test_success(self, product_order):
        product = Product.objects.create(
            name='Foo',
        )
        send_order(product.pk, 3, Decimal(30.3))
        product_order.assert_called_with(3, Decimal(30.3))

    @patch('proj.tasks.Product.order')
    @patch('proj.tasks.send_order.retry')
    def test_failure(self, send_order_retry, product_order):
        product = Product.objects.create(
            name='Foo',
        )

        # 在修补的方法上设置副作用
        # 以便它们引发我们想要的错误。
        send_order_retry.side_effect = Retry()
        product_order.side_effect = OperationalError()

        with raises(Retry):
            send_order(product.pk, 3, Decimal(30.6))
```

## pytest

Celery 还提供了一个 `pytest` 插件，该插件添加了可以在集成（或单元）测试套件中使用的夹具。

### 启用

Celery 最初以禁用状态提供该插件。要启用它，您可以：

- `pip install celery[pytest]`
- 或添加环境变量 `PYTEST_PLUGINS=celery.contrib.pytest`
- 或在根目录的 conftest.py 中添加 `pytest_plugins = ("celery.contrib.pytest", )`

### 标记

`celery` - 设置测试应用配置。

`celery` 标记使您能够覆盖用于单个测试用例的配置：

```python
@pytest.mark.celery(result_backend='redis://')
def test_something():
    ...
```

或用于类中的所有测试用例：

```python
@pytest.mark.celery(result_backend='redis://')
class test_something:

    def test_one(self):
        ...

    def test_two(self):
        ...
```

### 夹具

#### 函数作用域

##### `celery_app`

用于测试的Celery应用。

此夹具返回一个可用于测试的Celery应用。

示例：

```python
def test_create_task(celery_app, celery_worker):
    @celery_app.task
    def mul(x, y):
        return x * y
    
    celery_worker.reload()
    assert mul.delay(4, 4).get(timeout=10) == 16
```

##### `celery_worker`

嵌入实时工作器。

此夹具启动一个Celery工作器实例，可用于集成测试。工作器将在*单独的线程*中启动，并在测试返回时立即关闭。

默认情况下，夹具将等待工作器完成未完成任务最多10秒，如果超过时间限制将引发异常。可以通过设置 `celery_worker_parameters` 夹具返回的字典中的 `shutdown_timeout` 键来自定义超时时间。

示例：

```python
# 将此放入conftest.py中
@pytest.fixture(scope='session')
def celery_config():
    return {
        'broker_url': 'amqp://',
        'result_backend': 'redis://'
    }

def test_add(celery_worker):
    mytask.delay()


# 如果希望仅在一个测试用例中覆盖某些设置
# - 可以使用`celery`标记：
@pytest.mark.celery(result_backend='rpc')
def test_other(celery_worker):
    ...
```

默认情况下禁用心跳，这意味着测试工作器不会发送 `worker-online`、`worker-offline` 和 `worker-heartbeat` 事件。要启用心跳，请修改 `celery_worker_parameters` 夹具：

```python
# 将此放入conftest.py中
@pytest.fixture(scope="session")
def celery_worker_parameters():
    return {"without_heartbeat": False}
    ...
```

#### 会话作用域

##### `celery_config`

覆盖以设置Celery测试应用配置。

您可以重新定义此夹具来配置测试Celery应用。

您的夹具返回的配置将用于配置 `celery_app` 和 `celery_session_app` 夹具。

示例：

```python
@pytest.fixture(scope='session')
def celery_config():
    return {
        'broker_url': 'amqp://',
        'result_backend': 'rpc',
    }
```

##### `celery_parameters`

覆盖以设置Celery测试应用参数。

您可以重新定义此夹具来更改测试 Celery 应用的 `__init__` 参数。与 `celery_config` 不同，这些参数在实例化 `Celery` 时直接传递。

您的夹具返回的配置将用于配置 `celery_app` 和 `celery_session_app` 夹具。

示例：

```python
@pytest.fixture(scope='session')
def celery_parameters():
    return {
        'task_cls':  my.package.MyCustomTaskClass,
        'strict_typing': False,
    }
```

##### `celery_worker_parameters`

覆盖以设置Celery工作器参数。

您可以重新定义此夹具来更改测试 Celery 工作器的 `__init__` 参数。这些参数在实例化 `WorkController` 时直接传递。

您的夹具返回的配置将用于配置 `celery_worker` 和 `celery_session_worker` 夹具。

示例：

```python
@pytest.fixture(scope='session')
def celery_worker_parameters():
    return {
        'queues':  ('high-prio', 'low-prio'),
        'exclude_queues': ('celery'),
    }
```

##### `celery_enable_logging`

覆盖以在嵌入工作器中启用日志记录。

这是一个您可以覆盖的夹具，用于在嵌入工作器中启用日志记录。

示例：

```python
@pytest.fixture(scope='session')
def celery_enable_logging():
    return True
```

##### `celery_includes`

为嵌入工作器添加额外的导入。

您可以覆盖此夹具以在嵌入工作器启动时包含模块。

您可以使其返回要导入的模块名称列表，这些模块可以是任务模块、注册信号的模块等。

示例：

```python
@pytest.fixture(scope='session')
def celery_includes():
    return [
        'proj.tests.tasks',
        'proj.tests.celery_signal_handlers',
    ]
```

##### `celery_worker_pool`

覆盖嵌入工作器使用的池。

您可以覆盖此夹具以配置嵌入工作器使用的执行池。

示例：

```python
@pytest.fixture(scope='session')
def celery_worker_pool():
    return 'prefork'
```

!!! warning

    您不能使用gevent/eventlet池，除非您的整个测试套件都启用了monkeypatch。

##### `celery_session_worker`

在整个会话期间存在的嵌入工作器。

此夹具启动一个在整个测试会话期间存在的工作器（不会为每个测试启动/停止）。

示例：

```python
# 将此添加到conftest.py中
@pytest.fixture(scope='session')
def celery_config():
    return {
        'broker_url': 'amqp://',
        'result_backend': 'rpc',
    }

# 在测试中执行此操作。
def test_add_task(celery_session_worker):
    assert add.delay(2, 2).get() == 4
```

!!! warning

    混合使用会话和临时工作器可能不是一个好主意...

##### `celery_session_app`

用于测试的Celery应用（会话作用域）。

当其他会话作用域的夹具需要引用Celery应用实例时，可以使用此夹具。

##### `use_celery_app_trap`

在回退到默认应用时引发异常。

这是一个您可以在 `conftest.py` 中覆盖的夹具，用于启用"应用陷阱"：如果某些东西尝试访问默认应用或 current_app，将引发异常。

示例：

```python
@pytest.fixture(scope='session')
def use_celery_app_trap():
    return True
```

如果测试想要访问默认应用，您必须使用 `depends_on_current_app` 夹具标记它：

```python
@pytest.mark.usefixtures('depends_on_current_app')
def test_something():
    something()
```
