---
subtitle: Calling
description: Celery任务调用API完整指南：详细讲解apply_async、delay和直接调用三种任务执行方式，包含ETA倒计时、过期时间、任务链接、序列化器选择、消息压缩等高级功能，帮助开发者掌握Celery分布式任务调用的最佳实践。
---

# 调用

## 基础

本文档描述了 Celery 的统一"调用 API"，该 API 由任务实例和 [画布指南](canvas.md){target="_blank"} 使用。

该 API 定义了一组标准的执行选项，以及三种方法：

| 方法                                 | 描述                                                               |
|------------------------------------|------------------------------------------------------------------|
| `apply_async(args[, kwargs[, …]])` | 发送任务消息。                                                          |
| `delay(*args, **kwargs)`           | 发送任务消息的快捷方式，但不支持执行选项。                                            |
| `__call__(*args, **kwargs)`        | 应用支持调用 API 的对象（例如，`add(2, 2)`）意味着任务不会由工作进程执行，而是在当前进程中执行（不会发送消息）。 |

??? example "快速参考表"

    | 函数                                             | 描述                                                                                      |
    |------------------------------------------------|-----------------------------------------------------------------------------------------|
    | `delay(arg, kwarg=value)`                      | 星号参数到 `.apply_async` 的快捷方式。（`.delay(*args, **kwargs)` 调用 `.apply_async(args, kwargs)`）。 |
    | `apply_async((arg,), {'kwarg': value})`        | -                                                                                       |
    | `apply_async(countdown=10)`                    | 从现在起 10 秒后执行。                                                                           |
    | `apply_async(eta=now + timedelta(seconds=10))` | 从现在起 10 秒后执行，使用 `eta` 指定。                                                               |
    | `apply_async(countdown=60, expires=120)`       | 从现在起一分钟后执行，但在 2 分钟后过期。                                                                  |
    | `apply_async(expires=now + timedelta(days=2))` | 在 2 天后过期，使用 :class:`~datetime.datetime` 设置。                                             |
    | `apply_async(task_id=f'my_own_task_id')`       | 将任务的 ID 设置为 my_own_task_id，而不是通常生成的 uuid。                                               |

### 示例

`delay()` 方法很方便，因为它看起来像调用常规函数：

```python
task.delay(arg1, arg2, kwarg1='x', kwarg2='y')
```

使用 `apply_async()` 则需要编写：

```python
task.apply_async(args=[arg1, arg2], kwargs={'kwarg1': 'x', 'kwarg2': 'y'})
```

!!! tip

    如果任务未在当前进程中注册，您可以使用 `send_task()` 通过名称调用任务。


所以 `delay` 显然很方便，但如果您想设置额外的执行选项，则必须使用 `apply_async()`。

本文档的其余部分将详细讨论任务执行选项。所有示例都使用一个名为 `add` 的任务，返回两个参数的和：

```python
@app.task
def add(x, y):
    return x + y
```

!!! example "还有另一种方式…"

    您将在阅读 [画布指南](canvas.md){target="_blank"} 时了解更多信息，但 `Signature` 是用于传递任务调用签名的对象（例如通过网络发送），它们也支持调用 API：

    ```python
    task.s(arg1, arg2, kwarg1='x', kwargs2='y').apply_async()
    ```

## 链接（回调/错误回调）

Celery 支持将任务链接在一起，以便一个任务跟随另一个任务。回调任务将使用父任务的结果作为部分参数来应用：

```python
add.apply_async((2, 2), link=add.s(16))
```

!!! question inline end "什么是 `s`？"

    这里使用的 `add.s` 调用被称为签名。如果您不知道它们是什么，您应该在[画布指南](canvas.md){target="_blank"} 中阅读相关内容。在那里您还可以了解 `chain`：一种更简单的链接任务的方式。

    实际上，`link` 执行选项被认为是一个内部原语，您可能不会直接使用它，而是使用链式调用。

这里第一个任务的结果（4）将被发送到一个新的任务，该任务将 16 加到前一个结果上，形成表达式 `(2 + 2) + 16 = 20`

您还可以在任务引发异常时应用回调（*错误回调*）。工作进程实际上不会将错误回调作为任务调用，而是直接调用错误回调函数，以便可以将原始请求、异常和回溯对象传递给它。

这是一个错误回调的示例：

```python
@app.task
def error_handler(request, exc, traceback):
    print('Task {0} raised exception: {1!r}\n{2!r}'.format(
          request.id, exc, traceback))
```

可以使用 `link_error` 执行选项将其添加到任务中：

```python
add.apply_async((2, 2), link_error=error_handler.s())
```

此外，`link` 和 `link_error` 选项都可以表示为列表：

```python
add.apply_async((2, 2), link=[add.s(16), other_task.s()])
```

然后回调/错误回调将按顺序调用，并且所有回调都将使用父任务的返回值作为部分参数调用。

在 chord 的情况下，我们可以使用多种处理策略来处理错误。

## 关于消息处理

Celery 支持通过设置 `on_message` 回调来捕获所有状态变化。

例如，对于长时间运行的任务发送任务进度，您可以这样做：

```python
@app.task(bind=True)
def hello(self, a, b):
    time.sleep(1)
    self.update_state(state="PROGRESS", meta={'progress': 50})
    time.sleep(1)
    self.update_state(state="PROGRESS", meta={'progress': 90})
    time.sleep(1)
    return 'hello world: %i' % (a+b)
```

```python
def on_raw_message(body):
    print(body)

a, b = 1, 1
r = hello.apply_async(args=(a, b))
print(r.get(on_message=on_raw_message, propagate=False))
```

将生成类似这样的输出：

```text
{'task_id': '5660d3a3-92b8-40df-8ccc-33a5d1d680d7',
 'result': {'progress': 50},
 'children': [],
 'status': 'PROGRESS',
 'traceback': None}
{'task_id': '5660d3a3-92b8-40df-8ccc-33a5d1d680d7',
 'result': {'progress': 90},
 'children': [],
 'status': 'PROGRESS',
 'traceback': None}
{'task_id': '5660d3a3-92b8-40df-8ccc-33a5d1d680d7',
 'result': 'hello world: 10',
 'children': [],
 'status': 'SUCCESS',
 'traceback': None}
hello world: 10
```

## ETA 和倒计时 {#calling-eta}

ETA（预计到达时间）允许您设置一个特定的日期和时间，这是任务最早执行的时间。`countdown` 是通过秒数设置未来 ETA 的快捷方式。

```pycon
>>> result = add.apply_async((2, 2), countdown=3)
>>> result.get()    # this takes at least 3 seconds to return
4
```

任务保证在指定日期和时间*之后*的某个时间执行，但不一定在确切的时间执行。可能破坏截止时间的原因可能包括队列中有许多项目等待，或者网络延迟严重。为了确保您的任务及时执行，您应该监控队列的拥塞情况。使用 Munin 或类似工具接收警报，以便可以采取适当的措施来减轻工作负载。参见 :ref:`monitoring-munin`。

虽然 `countdown` 是一个整数，但 `eta` 必须是一个 `datetime` 对象，指定确切的日期和时间（包括毫秒精度和时区信息）：

```pycon
>>> from datetime import datetime, timedelta, timezone

>>> tomorrow = datetime.now(timezone.utc) + timedelta(days=1)
>>> add.apply_async((2, 2), eta=tomorrow)
```

!!! warning

    带有 `eta` 或 `countdown` 的任务会立即被工作进程获取，并且在预定时间过去之前，它们会驻留在工作进程的内存中。当使用这些选项来调度大量远期任务时，这些任务可能会在工作进程中累积并对 RAM 使用产生重大影响。

    此外，任务只有在工作进程开始执行它们时才会被确认。如果使用 Redis 作为代理，当 `countdown` 超过 `visibility_timeout` 时，任务将被重新传递（参见 :ref:`redis-caveats`）。

    因此，**不建议**使用 `eta` 和 `countdown` 来调度远期任务。理想情况下，使用不超过几分钟的值。对于更长的持续时间，请考虑使用基于数据库的周期性任务，例如在使用 Django 时使用 `django-celery-beat` 。

!!! warning

    当使用 RabbitMQ 作为消息代理时，如果指定超过 15 分钟的 `countdown`，您可能会遇到工作进程终止并引发 :exc:`~amqp.exceptions.PreconditionFailed` 错误的问题：

    ```pycon
    amqp.exceptions.PreconditionFailed: (0, 0): (406) PRECONDITION_FAILED - consumer ack timed out on channel
    ```

    在 RabbitMQ 自版本 3.8.15 起，`consumer_timeout` 的默认值为 15 分钟。自版本 3.8.17 起，它增加到 30 分钟。如果消费者在超过超时值的时间内没有确认其交付，其通道将以 `PRECONDITION_FAILED` 通道异常关闭。有关更多信息，请参阅 [交付确认超时](https://www.rabbitmq.com/docs/consumers#acknowledgement-timeout)。

    要解决此问题，在 RabbitMQ 配置文件 `rabbitmq.conf` 中，您应该指定 `consumer_timeout` 参数大于或等于您的倒计时值。例如，您可以指定一个非常大的值`consumer_timeout = 31622400000`，这相当于 1 年的毫秒数，以避免将来出现问题。

## 过期时间

`expires` 参数定义了一个可选的过期时间，可以是任务发布后的秒数，也可以是使用 `datetime` 的特定日期和时间：

```pycon
>>> # 任务从现在起一分钟后过期
>>> add.apply_async((10, 10), expires=60)

>>> # 也支持 datetime
>>> from datetime import datetime, timedelta, timezone
>>> add.apply_async((10, 10), kwargs, expires=datetime.now(timezone.utc) + timedelta(days=1))
```

当工作进程接收到过期的任务时，它会将任务标记为 `REVOKED` (`TaskRevokedError`)。

## 消息发送重试

Celery 会在连接失败时自动重试发送消息，并且重试行为可以配置——比如重试频率、最大重试次数——或者完全禁用。

要禁用重试，您可以将 `retry` 执行选项设置为 `False`：

```python
add.apply_async((2, 2), retry=False)
```

## 相关设置

### 重试策略

重试策略是一个映射，用于控制重试的行为，
可以包含以下键：

<table>
<thead>
<tr>
<th>Key</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>max_retries</code></td>
<td>
放弃前的最大重试次数，在这种情况下，
导致重试失败的异常将被抛出。<br/><br/>
值为 <code>None</code> 表示将无限重试。<br/><br/>
默认重试 3 次。
</td>
</tr>
<tr>
<td><code>interval_start</code></td>
<td>
定义重试之间等待的秒数（浮点数或整数）。默认值为 0（第一次重试将是即时的）。
</td>
</tr>
<tr>
<td><code>interval_step</code></td>
<td>
在每次连续重试时，此数字将被添加到重试延迟中（浮点数或整数）。默认值为 0.2。
</td>
</tr>
<tr>
<td><code>interval_max</code></td>
<td>
重试之间等待的最大秒数（浮点数或整数）。默认值为 0.2。
</td>
</tr>
<tr>
<td><code>retry_errors</code></td>
<td>
`retry_errors` 是一个异常类的元组，这些异常应该被重试。如果未指定，将被忽略。默认值为 None（忽略）。<br/><br/>
例如，如果您只想重试超时的任务，可以使用 <code>TimeoutError</code>：

```python
from kombu.exceptions import TimeoutError

add.apply_async((2, 2), retry=True, retry_policy={
    'max_retries': 3,
    'retry_errors': (TimeoutError, ),
})
```
</td>
</tr>
</tbody>
</table>

例如，默认策略对应：

```python
add.apply_async((2, 2), retry=True, retry_policy={
    'max_retries': 3,
    'interval_start': 0,
    'interval_step': 0.2,
    'interval_max': 0.2,
    'retry_errors': None,
})
```

重试花费的最大时间将是 0.4 秒。默认设置相对较短，因为如果代理连接断开，连接失败可能导致重试堆积效应——例如，许多 Web 服务器进程等待重试，阻塞其他传入请求。

## 连接错误处理

当您发送任务且消息传输连接丢失，或无法建立连接时，将引发 `OperationalError` 错误：

```pycon
>>> from proj.tasks import add
>>> add.delay(2, 2)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "celery/app/task.py", line 388, in delay
        return self.apply_async(args, kwargs)
  File "celery/app/task.py", line 503, in apply_async
    **options
  File "celery/app/base.py", line 662, in send_task
    amqp.send_task_message(P, name, message, **options)
  File "celery/backends/rpc.py", line 275, in on_task_call
    maybe_declare(self.binding(producer.channel), retry=True)
  File "/opt/celery/kombu/kombu/messaging.py", line 204, in _get_channel
    channel = self._channel = channel()
  File "/opt/celery/py-amqp/amqp/connection.py", line 272, in connect
    self.transport.connect()
  File "/opt/celery/py-amqp/amqp/transport.py", line 100, in connect
    self._connect(self.host, self.port, self.connect_timeout)
  File "/opt/celery/py-amqp/amqp/transport.py", line 141, in _connect
    self.sock.connect(sa)
  kombu.exceptions.OperationalError: [Errno 61] Connection refused
```

您也可以处理此错误：

```pycon
>>> from celery.utils.log import get_logger
>>> logger = get_logger(__name__)

>>> try:
...     add.delay(2, 2)
... except add.OperationalError as exc:
...     logger.exception('Sending task raised: %r', exc)
```

!!! note

    对于 RabbitMQ，这些错误仅表示代理无法访问。当代理达到资源限制时，消息仍可能被静默丢弃。在 `broker_transport_options` 中启用 `confirm_publish` 来检测这种情况。

## 序列化器 {#calling-serializers}

!!! tip inline end "安全"

    pickle 模块允许执行任意函数，请参阅 [安全指南](security.md){target="_blank"}。

    Celery 还附带了一个特殊的序列化器，它使用密码学来签名您的消息。

客户端和工作器之间传输的数据需要被序列化，因此 Celery 中的每条消息都有一个 `content_type` 头，用于描述用于编码的序列化方法。

默认的序列化器是 `JSON`，但您可以使用 `task_serializer` 设置来更改它，或者为每个单独的任务，甚至每条消息更改。

内置支持 `JSON`、:mod:`pickle`、`YAML` 和 `msgpack`，您还可以通过将自定义序列化器注册到 Kombu 序列化器注册表中来添加自己的序列化器。

每种选项都有其优缺点。

### json

JSON 在许多编程语言中都得到支持，现在是 Python 的标准部分（自 2.6 版起），并且解码速度相当快。

JSON 的主要缺点是它限制您使用以下数据类型：字符串、Unicode、浮点数、布尔值、字典和列表。十进制数和日期明显缺失。

二进制数据将使用 Base64 编码传输，与支持原生二进制类型的编码格式相比，传输数据的大小增加了 34%。

但是，如果您的数据符合上述约束条件并且您需要跨语言支持，JSON 的默认设置可能是您的最佳选择。

有关更多信息，请参阅 http://json.org。

!!! note

    （来自 Python 官方文档 https://docs.python.org/3.6/library/json.html）JSON 键/值对中的键始终是 :class:`str` 类型。当字典转换为 JSON 时，字典的所有键都被强制转换为字符串。因此，如果字典被转换为 JSON 然后再转换回字典，该字典可能不等于原始字典。也就是说，如果 x 有非字符串键，则 `loads(dumps(x)) != x`。

!!! warning

    在使用 :ref:`guide-canvas` 创建的更复杂的工作流中，观察到 JSON 序列化器由于递归引用而大幅增加消息大小，导致资源问题。*pickle* 序列化器不易受此影响，因此在某些情况下可能更可取。

### pickle

如果您不希望支持除 Python 之外的任何语言，那么使用 pickle 编码将为您提供所有内置 Python 数据类型（类实例除外）的支持，发送二进制文件时消息更小，并且比 JSON 处理速度略有提升。

### yaml

YAML 具有与 json 相同的许多特性，但它原生支持更多数据类型（包括日期、递归引用等）。

但是，Python 的 YAML 库比 JSON 库要慢得多。

如果您需要更具表现力的数据类型集并且需要保持跨语言兼容性，那么 YAML 可能比上述选项更适合。

要使用它，请安装 Celery：

```console
pip install celery[yaml]
```

有关更多信息，请参阅 http://yaml.org/。

### msgpack

msgpack 是一种二进制序列化格式，其特性更接近 JSON。该格式压缩效果更好，因此解析和编码速度比 JSON 更快。

要使用它，请安装 Celery：

```console
pip install celery[msgpack]
```

有关更多信息，请参阅 http://msgpack.org/。

要使用自定义序列化器，您需要将内容类型添加到 `accept_content`。默认情况下，只接受 JSON，包含其他内容头的任务将被拒绝。

以下顺序用于确定发送任务时使用的序列化器：

1. `serializer` 执行选项。
2. `Task.serializer` 属性
3. `task_serializer` 设置。

为单个任务调用设置自定义序列化器的示例：

```pycon
>>> add.apply_async((10, 10), serializer='json')
```

## 压缩 {#calling-compression}

Celery 可以使用以下内置方案压缩消息：

### `brotli`

brotli 针对 Web 进行了优化，特别适用于小型文本文档。它在提供静态内容（如字体和 HTML 页面）时最为有效。

要使用它，请安装带有 brotli 支持的 Celery：

```console
pip install celery[brotli]
```

### `bzip2`

bzip2 创建的文件比 gzip 更小，但压缩和解压缩速度明显比 gzip 慢。

要使用它，请确保您的 Python 可执行文件已编译支持 bzip2。

如果您遇到以下 `ImportError`：

```pycon
>>> import bz2
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
ImportError: No module named 'bz2'
```

这意味着您应该重新编译支持 bzip2 的 Python 版本。

### `gzip`

gzip 适用于需要小内存占用的系统，使其成为内存受限系统的理想选择。它通常用于生成带有 ".tar.gz" 扩展名的文件。

要使用它，请确保您的 Python 可执行文件已编译支持 gzip。

如果您遇到以下 `ImportError`：

```pycon
>>> import gzip
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
ImportError: No module named 'gzip'
```

这意味着您应该重新编译支持 gzip 的 Python 版本。

### `lzma`

lzma 提供良好的压缩比，并以快速的压缩和解压缩速度执行，但代价是更高的内存使用量。

要使用它，请确保您的 Python 可执行文件已编译支持 lzma，并且您的 Python 版本为 3.3 及以上。

如果您遇到以下 `ImportError`：

```pycon
>>> import lzma
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
ImportError: No module named 'lzma'
```

这意味着您应该重新编译支持 lzma 的 Python 版本。

或者，您也可以使用以下命令安装向后移植版本：

```console
pip install celery[lzma]
```

### `zlib`

zlib 是 Deflate 算法的库形式抽象，在其 API 中同时支持 gzip 文件格式和轻量级流格式。它是许多软件系统的关键组件 - 仅举几例，如 Linux 内核和 Git 版本控制系统。

要使用它，请确保您的 Python 可执行文件已编译支持 zlib。

如果您遇到以下 `ImportError`：

```pycon
>>> import zlib
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
ImportError: No module named 'zlib'
```

这意味着您应该重新编译支持 zlib 的 Python 版本。

### `zstd`

zstd 针对 zlib 级别的实时压缩场景和更好的压缩比。它由非常快速的熵阶段支持，由 Huff0 和 FSE 库提供。

要使用它，请安装带有 zstd 支持的 Celery：

```console
pip install celery[zstd]
```

以下顺序用于决定发送任务时使用的压缩方案：

1. `compression` 执行选项
2. `Task.compression` 属性
3. `task_compression` 设置

示例指定调用任务时使用的压缩方案：

```pycon
>>> add.apply_async((2, 2), compression='zlib')
```

## 连接

!!! tip inline end "自动连接池支持"

    自版本 2.3 起支持自动连接池，因此您无需手动处理连接和发布者来重用连接。

    自版本 2.5 起默认启用连接池。

    有关更多信息，请参阅 `broker_pool_limit` 设置。

您可以通过创建发布者来手动处理连接：

```python
numbers = [(2, 2), (4, 4), (8, 8), (16, 16)]
results = []
with add.app.pool.acquire(block=True) as connection:
    with add.get_publisher(connection) as publisher:
        try:
            for i, j in numbers:
                res = add.apply_async((i, j), publisher=publisher)
                results.append(res)
print([res.get() for res in results])
```

不过这个特定示例更适合用组来表达：

```pycon
>>> from celery import group

>>> numbers = [(2, 2), (4, 4), (8, 8), (16, 16)]
>>> res = group(add.s(i, j) for i, j in numbers).apply_async()

>>> res.get()
[4, 8, 16, 32]
```

## 路由选项

Celery 可以将任务路由到不同的队列。

简单的路由（名称 <-> 名称）可以通过使用 `queue` 选项来实现：：

```python
add.apply_async(queue='priority.high')
```

然后您可以通过使用工作者的 `celery worker -Q` 参数将工作者分配给 `priority.high` 队列：

```console
celery -A proj worker -l INFO -Q celery,priority.high
```

!!! quote

    在代码中硬编码队列名称不推荐，最佳实践是使用配置路由器（`task_routes`）。

    要了解更多关于路由的信息，请参阅 [路由指南](routing.md){target="_blank"}。

## 结果选项

您可以使用 `task_ignore_result` 设置或 `ignore_result` 选项来启用或禁用结果存储：

```pycon
>>> result = add.apply_async((1, 2), ignore_result=True)
>>> result.get()
None

>>> # 不忽略结果（默认）
...
>>> result = add.apply_async((1, 2), ignore_result=False)
>>> result.get()
3
```

如果您想在结果后端存储有关任务的额外元数据，请将 `result_extended` 设置设为 `True`。

!!! quote "有关任务的更多信息，请参阅 [任务指南](tasks.md){target="_blank"}。"

### 高级选项

这些选项适用于希望利用 AMQP 完整路由功能的高级用户。感兴趣的读者可以阅读 [路由指南](routing.md){target="_blank"}。

| 选项            | 描述                                                                   |
|---------------|----------------------------------------------------------------------|
| `exchange`    | 要发送消息到的交换器名称（或 `Exchange`）。                                          |
| `routing_key` | 用于确定的路由键。                                                            |
| `priority`    | 介于 `0` 和 `255` 之间的数字，其中 `255` 是最高优先级。支持：RabbitMQ、Redis（优先级反转，0 为最高）。 |
