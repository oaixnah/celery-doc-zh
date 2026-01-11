---
subtitle: Configuration
description: Celery配置和默认值完整指南 - 包含所有可用配置选项的详细说明，涵盖任务执行、结果后端、时间设置、序列化器、速率限制等关键配置参数，帮助您优化Celery分布式任务队列的性能和可靠性。
---

# 配置和默认值

本文档描述了可用的配置选项。

如果您使用默认加载器，必须创建 `celeryconfig.py` 模块，并确保它在 Python 路径中可用。

## 示例配置文件

这是一个示例配置文件，可以帮助您入门。它应该包含运行基本 Celery 设置所需的所有内容。

```python
## Broker 设置。
broker_url = 'amqp://guest:guest@localhost:5672//'

# Celery worker 启动时要导入的模块列表。
imports = ('myapp.tasks',)

## 使用数据库存储任务状态和结果。
result_backend = 'db+sqlite:///results.db'

task_annotations = {'tasks.add': {'rate_limit': '10/s'}}
```

## 新的小写设置

版本 4.0 引入了新的小写设置和设置组织方式。

与先前版本的主要区别，除了小写名称外，还包括一些前缀的重命名，例如 `celery_beat` 改为 `beat`，`celeryd` 改为 `worker`，并且大多数顶级 `celery` 设置已移至新的 `task` 前缀中。

!!! warning

    Celery 在 Celery 6.0 之前仍能读取旧的配置文件。之后，对旧配置文件的支持将被移除。我们提供了 `celery upgrade` 命令，应该可以处理大多数情况。

    请尽快迁移到新的配置方案。

## 配置选项

### 通用设置

#### `accept_content`

默认值：`{'json'}`（Set、List 或 Tuple）。

允许的内容类型/序列化器的白名单。

如果接收到不在该列表中的消息，则该消息将被丢弃并报错。

默认情况下仅启用 json，但可以添加任何内容类型，包括 pickle 和 yaml；在这种情况下，请确保不受信任的方无法访问您的代理。

示例：

```python
# 使用序列化器名称
accept_content = ['json']

# 或实际的内容类型（MIME）
accept_content = ['application/json']
```

#### `result_accept_content`

默认值：`None`（可以设置为 Set、List 或 Tuple）。

允许用于结果后端的内容类型/序列化器的白名单。

如果接收到不在该列表中的消息，则该消息将被丢弃并报错。

默认情况下，它与 `accept_content` 使用相同的序列化器。但是，可以为结果后端的接受内容指定不同的序列化器。通常在使用了签名消息传递且结果以未签名方式存储在结果后端时需要这样做。

示例：

```python
# 使用序列化器名称
result_accept_content = ['json']

# 或实际的内容类型（MIME）
result_accept_content = ['application/json']
```

### 时间和日期设置

#### `enable_utc`

默认值：自版本 3.0 起默认启用。

如果启用，消息中的日期和时间将被转换为使用 UTC 时区。

注意：运行 Celery 版本低于 2.5 的工作进程将假定所有消息使用本地时区，因此只有在所有工作进程都已升级后才启用此设置。

#### `timezone`

默认值：`"UTC"`。

配置 Celery 使用自定义时区。时区值可以是 [ZoneInfo](https://docs.python.org/3/library/zoneinfo.html) 库支持的任何时区。

如果未设置，则使用 UTC 时区。为了向后兼容，还有一个 `enable_utc` 设置，当此设置设为 false 时，将使用系统本地时区。

### 任务设置

#### `task_annotations`

默认值：`None`。

此设置可用于从配置中重写任何任务属性。该设置可以是一个字典，或者是一个过滤任务并返回要更改的属性映射的注释对象列表。

这将更改 `tasks.add` 任务的 `rate_limit` 属性：

```python
task_annotations = {'tasks.add': {'rate_limit': '10/s'}}
```

或者为所有任务更改相同的属性：

```python
task_annotations = {'*': {'rate_limit': '10/s'}}
```

您也可以更改方法，例如 `on_failure` 处理程序：

```python
def my_on_failure(self, exc, task_id, args, kwargs, einfo):
    print('Oh no! Task failed: {0!r}'.format(exc))

task_annotations = {'*': {'on_failure': my_on_failure}}
```

如果您需要更多灵活性，可以使用对象而不是字典来选择要注释的任务：

```python
class MyAnnotate:

    def annotate(self, task):
        if task.name.startswith('tasks.'):
            return {'rate_limit': '10/s'}

task_annotations = (MyAnnotate(), {other,})
```

#### `task_compression`

默认值：`None`

用于任务消息的默认压缩方式。可以是 `gzip`、`bzip2`（如果可用），或者在 Kombu 压缩注册表中注册的任何自定义压缩方案。

默认是发送未压缩的消息。

#### `task_protocol`

默认值：2（自 4.0 版本起）。

设置用于发送任务的默认任务消息协议版本。支持的协议：1 和 2。

协议 2 由 3.1.24 和 4.x+ 版本支持。

#### `task_serializer`

默认值：`"json"`（自 4.0 版本起，早期版本为：pickle）。

标识要使用的默认序列化方法的字符串。可以是 `json`（默认）、`pickle`、`yaml`、`msgpack`，或者在 `kombu.serialization.registry` 中注册的任何自定义序列化方法。

#### `task_publish_retry`

默认值：启用。

决定在连接丢失或其他连接错误的情况下是否重试发布任务消息。

#### `task_publish_retry_policy`

默认值：请参阅 [重试策略](calling.md#calling-retry){target="_blank"}。

定义在连接丢失或其他连接错误的情况下重试发布任务消息时的默认策略。

### 任务执行设置

#### `task_always_eager`

默认值：Disabled。

如果设置为 `True`，所有任务将在本地通过阻塞方式执行，直到任务返回。`apply_async()` 和 `Task.delay()` 将返回一个 `celery.result.EagerResult` 实例，该实例模拟了 `celery.result.AsyncResult` 的 API 和行为，只是结果已经被评估。

也就是说，任务将在本地执行，而不是被发送到队列中。

#### `task_eager_propagates`

默认值：Disabled。

如果设置为 `True`，急切执行的任务（通过 `task.apply()` 应用，或者当 `task_always_eager` 设置启用时）将传播异常。

这与始终使用 `throw=True` 运行 `apply()` 相同。

#### `task_store_eager_result`

默认值：Disabled。

如果设置为 `True`，并且 `task_always_eager` 为 `True` 且 `task_ignore_result` 为 `False`，急切执行的任务结果将被保存到后端。

默认情况下，即使 `task_always_eager` 设置为 `True` 且 `task_ignore_result` 设置为 `False`，结果也不会被保存。

#### `task_remote_tracebacks`

默认值：Disabled。

如果启用，任务结果将在重新引发任务错误时包含工作进程的堆栈信息。

这需要 `tblib` 库，可以使用 `pip` 安装：

```console
pip install celery[tblib]
```

#### `task_ignore_result`

默认值：Disabled。

是否存储任务返回值（墓碑）。如果您仍然希望存储错误，只是不存储成功的返回值，可以设置 `task_store_errors_even_if_ignored`。

#### `task_store_errors_even_if_ignored`

默认值：Disabled。

如果设置，工作进程将把所有任务错误存储在结果存储中，即使 `celery.app.task.Task.ignore_result` 已启用。

#### `task_track_started`

默认值：Disabled。

如果设置为 `True`，任务在工作进程执行时将报告其状态为 'started'。默认值为 `False`，因为正常行为是不报告这种粒度的状态。任务要么是待处理、已完成，要么是等待重试。拥有 'started' 状态对于长时间运行的任务以及需要报告当前正在运行什么任务的情况很有用。

#### `task_time_limit`

默认值：无时间限制。

任务硬时间限制（以秒为单位）。当超过此限制时，处理该任务的工作进程将被终止并替换为新进程。

#### `task_allow_error_cb_on_chord_header`

默认值：Disabled。

启用此标志将允许将错误回调链接到和弦头部，默认情况下在使用 `link_error()` 时不会链接，并且如果头部中的任何任务失败，将阻止和弦主体的执行。

考虑以下禁用标志的画布（默认行为）：

```python
header = group([t1, t2])
body = t3
c = chord(header, body)
c.link_error(error_callback_sig)
```

如果*任何*头部任务失败（`t1` 或 `t2`），默认情况下，和弦主体（`t3`）将**不会执行**，并且 `error_callback_sig` 将被调用**一次**（针对主体）。

启用此标志将通过以下方式改变上述行为：

1. `error_callback_sig` 将被链接到 `t1` 和 `t2`（以及 `t3`）。
2. 如果*任何*头部任务失败，`error_callback_sig` 将被调用**针对每个**失败的头部任务**和** `body`（即使主体没有运行）。

现在考虑启用标志后的以下画布：

```python
header = group([failingT1, failingT2])
body = t3
c = chord(header, body)
c.link_error(error_callback_sig)
```

如果*所有*头部任务都失败（`failingT1` 和 `failingT2`），那么和弦主体（`t3`）将**不会执行**，并且 `error_callback_sig` 将被调用**3次**（两次针对头部，一次针对主体）。

最后，考虑启用标志后的以下画布：

```python
header = group([failingT1, failingT2])
body = t3
upgraded_chord = chain(header, body)
upgraded_chord.link_error(error_callback_sig)
```

此画布的行为将与之前的完全相同，因为 `chain` 将在内部升级为 `chord`。

#### `task_soft_time_limit`

默认值：无软时间限制。

任务软时间限制（以秒为单位）。

当超过此限制时，将引发 `SoftTimeLimitExceeded` 异常。例如，任务可以捕获此异常以在硬时间限制到来之前进行清理：

```python
from celery.exceptions import SoftTimeLimitExceeded

@app.task
def mytask():
    try:
        return do_work()
    except SoftTimeLimitExceeded:
        cleanup_in_a_hurry()
```

#### `task_acks_late`

默认值：Disabled。

延迟确认意味着任务消息将在任务**执行后**被确认，而不是*正好在执行前*（默认行为）。

#### `task_acks_on_failure_or_timeout`

默认值：Enabled

启用后，所有任务的消息都将被确认，即使它们失败或超时。

配置此设置仅适用于在执行**后**被确认的任务，并且仅当 `task_acks_late` 已启用时。

#### `task_reject_on_worker_lost`

默认值：Disabled。

即使 `task_acks_late` 已启用，当执行任务的工作进程突然退出或被信号中断（例如，`KILL`/`INT` 等）时，工作进程仍将确认任务。

将此设置为 true 允许消息重新排队，以便任务可以由同一工作进程或其他工作进程再次执行。

!!! warning

    启用此功能可能导致消息循环；请确保您知道自己在做什么。

#### `task_default_rate_limit`

默认值：无速率限制。

任务的全局默认速率限制。

此值用于没有自定义速率限制的任务

.. seealso:

    `worker_disable_rate_limits` 设置可以禁用所有速率限制。

### 任务结果后端设置

#### `result_backend`

默认值：默认未启用任何结果后端。

用于存储任务结果（墓碑）的后端。可以是以下之一：

| 后端                      |
|---------------------------|
| rpc                       |
| database                  |
| redis                     |
| cache                     |
| mongodb                   |
| cassandra                 |
| elasticsearch             |
| ironcache                 |
| couchbase                 |
| arangodb                  |
| couchdb                   |
| cosmosdbsql(实验性)       |
| filesystem                |
| consul                    |
| azureblockblob            |
| s3                        |
| gcs                       |

!!! warning

    虽然AMQP结果后端非常高效，但您必须确保
    您只接收一次相同的结果。请参阅：:doc:`userguide/calling`)。

#### `result_backend_always_retry`

默认值：`False`

如果启用，后端将在发生可恢复异常时尝试重试，而不是传播异常。它将在两次重试之间使用指数退避睡眠时间。

#### `result_backend_max_sleep_between_retries_ms`

默认值：10000

这指定了两次后端操作重试之间的最大睡眠时间。

#### `result_backend_base_sleep_between_retries_ms`

默认值：10

这指定了两次后端操作重试之间的基本睡眠时间量。

#### `result_backend_max_retries`

默认值：Inf

这是在可恢复异常情况下的最大重试次数。

#### `result_backend_thread_safe`

默认值：False

如果为True，则后端对象在线程间共享。这对于使用共享连接池而不是为每个线程创建连接可能很有用。

#### `result_backend_transport_options`

默认值：`{}`（空映射）。

传递给底层传输的额外选项字典。

有关支持的选项（如果有），请参阅您的传输用户手册。

设置可见性超时的示例（Redis和SQS传输支持）：

```python
result_backend_transport_options = {'visibility_timeout': 18000}  # 5小时
```

#### `result_serializer`

默认值：自4.0起为 `json`（早期：pickle）。

结果序列化格式。

有关支持的序列化格式的信息，请参阅：`calling-serializers`。

#### `result_compression`

默认值：无压缩。

用于任务结果的可选压缩方法。支持与 `task_compression`设置相同的选项。

#### `result_extended`

默认值：`False`

启用将扩展的任务结果属性（名称、参数、关键字参数、工作进程、重试次数、队列、投递信息）写入后端。

#### `result_expires`

默认值：1天后过期。

存储的任务墓碑将被删除的时间（以秒为单位，或 `datetime.timedelta` 对象）。

一个内置的周期性任务将在此时间后删除结果（`celery.backend_cleanup`），假设`celery beat`已启用。该任务每天凌晨4点运行。

`None`或0的值意味着结果永不过期（取决于后端规范）。

!!! note

    目前这仅适用于AMQP、数据库、缓存、Couchbase、文件系统和Redis后端。

    当使用数据库或文件系统后端时，`celery beat`必须运行才能使结果过期。

#### `result_cache_max`

默认值：默认禁用。

启用客户端结果缓存。

这对于旧的已弃用的'amqp'后端可能很有用，其中结果在一个结果实例消费后立即不可用。

这是在淘汰较旧结果之前要缓存的结果总数。值为 `0` 或 `None` 表示无限制，值为 `-1` 将禁用缓存。

默认禁用。

#### `result_chord_join_timeout`

默认值：3.0。

在 chord 内连接组结果的超时时间（秒，int/float）。

#### `result_chord_retry_interval`

默认值：1.0。

重试 chord 任务的默认间隔。

#### `override_backends`

默认值：默认禁用。

实现后端的类的路径。

允许覆盖后端实现。如果您需要存储有关已执行任务的额外元数据，覆盖重试策略等，这可能很有用。

示例：

```python
override_backends = {"db": "custom_module.backend.class"}
```

### 数据库后端设置

#### 数据库 URL 示例

要使用数据库后端，您必须配置 `result_backend` 设置，使用连接 URL 和 `db+` 前缀：

```python
result_backend = 'db+scheme://user:password@host:port/dbname'
```

示例：

```python
# sqlite (文件名)
result_backend = 'db+sqlite:///results.sqlite'

# mysql
result_backend = 'db+mysql://scott:tiger@localhost/foo'

# postgresql
result_backend = 'db+postgresql://scott:tiger@localhost/mydatabase'

# oracle
result_backend = 'db+oracle://scott:tiger@127.0.0.1:1521/sidname'
```

请参阅 [支持的数据库](https://docs.sqlalchemy.org/en/20/core/engines.html#supported-databases) 获取支持的数据库列表，以及 [连接字符串](https://docs.sqlalchemy.org/en/20/core/engines.html#database-urls) 获取有关连接字符串的更多信息（这是 URI 中 `db+` 前缀之后的部分）。

#### `database_create_tables_at_setup`

默认值：默认为 True。

- 如果为 `True`，Celery 将在设置期间在数据库中创建表。
- 如果为 `False`，Celery 将延迟创建表，即等待第一个任务执行后再创建表。

!!! note

    在 celery 5.5 之前，表是延迟创建的，即相当于 `database_create_tables_at_setup` 设置为 False。

#### `database_engine_options`

默认值：`{}`（空映射）。

要指定额外的 SQLAlchemy 数据库引擎选项，您可以使用 `database_engine_options` 设置：

```python
# echo 启用 SQLAlchemy 的详细日志记录。
app.conf.database_engine_options = {'echo': True}
```

#### `database_short_lived_sessions`

默认值：默认禁用。

短生命周期会话默认禁用。如果启用，它们会显著降低性能，尤其是在处理大量任务的系统上。此选项对于低流量工作器很有用，这些工作器由于缓存的数据库连接因不活动而过时而遇到错误。例如，间歇性错误如 `(OperationalError) (2006, 'MySQL server has gone away')` 可以通过启用短生命周期会话来修复。此选项仅影响数据库后端。

#### `database_table_schemas`

默认值：`{}`（空映射）。

当 SQLAlchemy 配置为结果后端时，Celery 会自动创建两个表来存储任务的结果元数据。此设置允许您自定义表的模式：

```python
# 为数据库结果后端使用自定义模式。
database_table_schemas = {
    'task': 'celery',
    'group': 'celery',
}
```

#### `database_table_names`

默认值：`{}`（空映射）。

当 SQLAlchemy 配置为结果后端时，Celery 会自动创建两个表来存储任务的结果元数据。此设置允许您自定义表名：

```python
# 为数据库结果后端使用自定义表名。
database_table_names = {
    'task': 'myapp_taskmeta',
    'group': 'myapp_groupmeta',
}
```

### RPC 后端设置

#### `result_persistent`

默认值：默认禁用（瞬态消息）。

如果设置为 `True`，结果消息将是持久化的。这意味着在代理重启后消息不会丢失。

#### 配置示例

```python
result_backend = 'rpc://'
result_persistent = False
```

**请注意**：使用此后端可能会触发 `celery.backends.rpc.BacklogLimitExceeded` 异常，如果任务墓碑过于*陈旧*。

例如：

```python
for i in range(10000):
    r = debug_task.delay()

print(r.state)  # 这会引发 celery.backends.rpc.BacklogLimitExceeded
```

### 缓存后端设置

!!! note

    缓存后端支持 `pylibmc` 和 `python-memcached` 库。只有在 `pylibmc` 未安装时才会使用后者。

使用单个 Memcached 服务器：

```python
result_backend = 'cache+memcached://127.0.0.1:11211/'
```

使用多个 Memcached 服务器：

```python
result_backend = "cache+memcached://172.19.26.240:11211;172.19.26.242:11211/".strip()
```

"memory" 后端仅将缓存存储在内存中：

```python
result_backend = 'cache'
cache_backend = 'memory'
```

#### `cache_backend_options`

默认值：`{}`（空映射）。

您可以使用 `cache_backend_options` 设置来配置 `pylibmc` 选项：

```python
cache_backend_options = {
    'binary': True,
    'behaviors': {'tcp_nodelay': True},
}
```

#### `cache_backend`

此设置不再用于 Celery 的内置后端，因为现在可以直接在 `result_backend` 设置中指定缓存后端。

!!! note

    `django-celery-results` 库使用 `cache_backend` 来选择 Django 缓存。

### MongoDB 后端设置

!!! note

    MongoDB 后端需要 `pymongo` 库：http://github.com/mongodb/mongo-python-driver/tree/master

#### mongodb_backend_settings

这是一个支持以下键的字典：

| 键                   | 描述                                                                                                                                          |
|---------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| database            | 要连接的数据库名称。默认为 `celery`。                                                                                                                     |
| taskmeta_collection | 存储任务元数据的集合名称。默认为 `celery_taskmeta`。                                                                                                         |
| max_pool_size       | 作为 max_pool_size 传递给 PyMongo 的 Connection 或 MongoClient 构造函数。这是在给定时间保持与 MongoDB 的 TCP 连接的最大数量。如果打开的连接数超过 max_pool_size， 套 接字将在释放时关闭。默认为 10。 |
| options             | 传递给 mongodb 连接构造函数的附加关键字参数。请参阅 `pymongo` 文档以查看支持的参数列表。                                                                                      |

!!! note

    在 pymongo>=4.14 中，选项是区分大小写的，而之前是不区分大小写的。请参阅 `pymongo.mongo_client.MongoClient` 来确定正确的大小写。

#### 示例配置

```python
result_backend = 'mongodb://localhost:27017/'
mongodb_backend_settings = {
    'database': 'mydb',
    'taskmeta_collection': 'my_taskmeta_collection',
}
```

### Redis 后端设置 {#conf-redis-result-backend}

#### 配置后端 URL

!!! note

    Redis 后端需要 `redis` 库。
    
    要安装此包，请使用 `pip`:

    ```console
    pip install celery[redis]
    ```

此后端需要将 `result_backend` 设置设置为 Redis 或 [Redis over TLS](https://www.iana.org/assignments/uri-schemes/prov/rediss) URL:

```python
result_backend = 'redis://username:password@host:port/db'
```

例如:

```python
result_backend = 'redis://localhost/0'
```

等同于:

```python
result_backend = 'redis://'
```

使用 `rediss://` 协议通过 TLS 连接到 redis:

```python
result_backend = 'rediss://username:password@host:port/db?ssl_cert_reqs=required'
```

请注意，`ssl_cert_reqs` 字符串应为 `required`、`optional` 或 `none` 之一（尽管为了向后兼容旧版 Celery，该字符串也可能是 `CERT_REQUIRED`、`CERT_OPTIONAL`、`CERT_NONE` 之一，但这些值仅适用于 Celery，不直接适用于 Redis）。

如果应使用 Unix 套接字连接，URL 需要采用以下格式:

```python
result_backend = 'socket:///path/to/redis.sock'
```

URL 的字段定义如下：

<table>
<thead>
<tr>
<th>字段</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>username</code></td>
<td>
用于连接数据库的用户名。<br/><br/>
请注意，这仅在 Redis>=6.0 和安装了 py-redis>=3.4.0 时受支持。<br/><br/>
如果您使用较旧的数据库版本或较旧的客户端版本，
可以省略用户名：

```python
result_backend = 'redis://:password@host:port/db'
```
</td>
</tr>
<tr>
<td><code>password</code></td>
<td>用于连接数据库的密码。</td>
</tr>
<tr>
<td><code>host</code></td>
<td>Redis 服务器的主机名或 IP 地址（例如，<code>localhost</code>）。</td>
</tr>
<tr>
<td><code>port</code></td>
<td>Redis 服务器的端口。默认为 6379。</td>
</tr>
<tr>
<td><code>db</code></td>
<td>要使用的数据库编号。默认为 0。db 可以包含可选的前导斜杠。</td>
</tr>
</tbody>
</table>

使用 TLS 连接（协议为 `rediss://`）时，您可以将 `broker_use_ssl` 中的所有值作为查询参数传递。证书路径必须进行 URL 编码，并且 `ssl_cert_reqs` 是必需的。示例：

```python
result_backend = 'rediss://:password@host:port/db?\
ssl_cert_reqs=required\
&ssl_ca_certs=%2Fvar%2Fssl%2Fmyca.pem\                  # /var/ssl/myca.pem
&ssl_certfile=%2Fvar%2Fssl%2Fredis-server-cert.pem\     # /var/ssl/redis-server-cert.pem
&ssl_keyfile=%2Fvar%2Fssl%2Fprivate%2Fworker-key.pem'   # /var/ssl/private/worker-key.pem
```

请注意，`ssl_cert_reqs` 字符串应为 `required`、`optional` 或 `none` 之一（尽管为了向后兼容，该字符串也可能是 `CERT_REQUIRED`、`CERT_OPTIONAL`、`CERT_NONE`）。

#### `redis_backend_health_check_interval`

默认：未配置

Redis 后端支持健康检查。此值必须设置为一个整数，其值为健康检查之间的秒数。如果在健康检查期间遇到 ConnectionError 或 TimeoutError，连接将被重新建立，命令将精确重试一次。

#### `redis_backend_use_ssl`

默认：禁用。

Redis 后端支持 SSL。此值必须以字典形式设置。有效的键值对与 `broker_use_ssl` 下的 `redis` 子节中提到的相同。

#### `redis_backend_credential_provider`

默认：禁用。

Redis 后端支持凭据提供程序。此值必须设置为类路径字符串或类实例的形式。例如 `mymodule.myfile.myclass` 有关更多详细信息，请参阅 `RedisCredentialProvider` 文档。

#### `redis_max_connections`

默认：无限制。

用于发送和检索结果的 Redis 连接池中可用的最大连接数。

!!! warning
    
    如果并发连接数超过最大值，Redis 将引发 `ConnectionError`。

#### `redis_socket_connect_timeout`

默认：`None`

从结果后端连接到 Redis 的套接字超时时间，以秒为单位（整数/浮点数）

#### `redis_socket_timeout`

默认：120.0 秒。

与 Redis 服务器进行读/写操作的套接字超时时间，以秒为单位（整数/浮点数），由 redis 结果后端使用。

#### `redis_retry_on_timeout`

默认：`False`

用于在发生 TimeoutError 时重试与 Redis 服务器的读/写操作，由 redis 结果后端使用。如果使用 Unix 套接字连接 Redis，不应设置此变量。

#### `redis_socket_keepalive`

默认：`False`

套接字 TCP keepalive，用于保持与 Redis 服务器的连接健康，由 redis 结果后端使用。

#### `redis_client_name`

默认：`None`

设置结果后端使用的 Redis 连接的客户端名称。这有助于在 Redis 监控工具中识别连接。

### Cassandra/AstraDB 后端设置

!!! note

    此 Cassandra 后端驱动程序需要 `cassandra-driver`。

    此后端可以指常规的 Cassandra 安装或托管的 Astra DB 实例。根据具体情况，必须提供 `cassandra_servers` 和 `cassandra_secure_bundle_path` 设置中的一个（但不能同时提供两个）。

    要安装，请使用 `pip`：

    ```console
    pip install celery[cassandra]
    ```

此后端需要设置以下配置指令。

#### `cassandra_servers`

默认值：`[]`（空列表）。

`host` Cassandra 服务器列表。连接到 Cassandra 集群时必须提供此设置。传递此设置与 `cassandra_secure_bundle_path` 严格互斥。示例：：

```python
cassandra_servers = ['localhost']
```

#### `cassandra_secure_bundle_path`

默认值：None。

连接到 Astra DB 实例的 secure-connect-bundle zip 文件的绝对路径。传递此设置与 `cassandra_servers` 严格互斥。
示例：：

```python
cassandra_secure_bundle_path = '/home/user/bundles/secure-connect.zip'
```

连接到 Astra DB 时，必须指定纯文本认证提供程序以及相关的用户名和密码，这些分别取值为为 Astra DB 实例生成的有效令牌的客户端 ID 和客户端密钥。
请参阅下面的 Astra DB 配置示例。

#### `cassandra_port`

默认值：9042。

联系 Cassandra 服务器的端口。

#### `cassandra_keyspace`

默认值：None。

存储结果的键空间。

```python
cassandra_keyspace = 'tasks_keyspace'
```

#### `cassandra_table`

默认值：None。

存储结果的表（列族）。

```python
cassandra_table = 'tasks'
```

#### `cassandra_read_consistency`

默认值：None。

使用的读取一致性。值可以是 `ONE`、`TWO`、`THREE`、`QUORUM`、`ALL`、`LOCAL_QUORUM`、`EACH_QUORUM`、`LOCAL_ONE`。

#### `cassandra_write_consistency`

默认值：None。

使用的写入一致性。值可以是 `ONE`、`TWO`、`THREE`、`QUORUM`、`ALL`、`LOCAL_QUORUM`、`EACH_QUORUM`、`LOCAL_ONE`。

#### `cassandra_entry_ttl`

默认值：None。

状态条目的生存时间。它们将在添加后的指定秒数后过期并被删除。值为 `None`（默认）意味着它们永远不会过期。

#### `cassandra_auth_provider`

默认值：`None`。

要使用的 `cassandra.auth` 模块中的 AuthProvider 类。值可以是 `PlainTextAuthProvider` 或 `SaslAuthProvider`。

#### `cassandra_auth_kwargs`

默认值：`{}`（空映射）。

传递给认证提供程序的命名参数。例如：

```python
cassandra_auth_kwargs = {
    username: 'cassandra',
    password: 'cassandra'
}
```

#### `cassandra_options`

默认值：`{}`（空映射）。

传递给 `cassandra.cluster` 类的命名参数。

```python
cassandra_options = {
    'cql_version': '3.2.1'
    'protocol_version': 3
}
```

#### 示例配置（Cassandra）

```python
result_backend = 'cassandra://'
cassandra_servers = ['localhost']
cassandra_keyspace = 'celery'
cassandra_table = 'tasks'
cassandra_read_consistency = 'QUORUM'
cassandra_write_consistency = 'QUORUM'
cassandra_entry_ttl = 86400
```

#### 示例配置（Astra DB）

```python
result_backend = 'cassandra://'
cassandra_keyspace = 'celery'
cassandra_table = 'tasks'
cassandra_read_consistency = 'QUORUM'
cassandra_write_consistency = 'QUORUM'
cassandra_auth_provider = 'PlainTextAuthProvider'
cassandra_auth_kwargs = {
  'username': '<<CLIENT_ID_FROM_ASTRA_DB_TOKEN>>',
  'password': '<<CLIENT_SECRET_FROM_ASTRA_DB_TOKEN>>'
}
cassandra_secure_bundle_path = '/path/to/secure-connect-bundle.zip'
cassandra_entry_ttl = 86400
```

#### 附加配置

Cassandra 驱动程序在建立连接时，会经历与服务器协商协议版本的阶段。类似地，会自动提供负载均衡策略（默认为 `DCAwareRoundRobinPolicy`，它又有一个 `local_dc` 设置，也由驱动程序在连接时确定）。
在可能的情况下，应在配置中明确提供这些设置：此外，未来版本的 Cassandra 驱动程序将要求至少指定负载均衡策略（使用[执行配置文件](https://docs.datastax.com/en/developer/python-driver/3.25/execution_profiles)，如下所示）。

因此，Cassandra 后端的完整配置将包含以下附加行：

```python
from cassandra.policies import DCAwareRoundRobinPolicy
from cassandra.cluster import ExecutionProfile
from cassandra.cluster import EXEC_PROFILE_DEFAULT

myEProfile = ExecutionProfile(
  load_balancing_policy=DCAwareRoundRobinPolicy(
    local_dc='datacenter1', # 替换为您的 DC 名称
  )
)

cassandra_options = {
  'protocol_version': 5,    # 对于 Cassandra 4，根据需要更改
  'execution_profiles': {EXEC_PROFILE_DEFAULT: myEProfile},
}
```

类似地，对于 Astra DB：

```python
from cassandra.policies import DCAwareRoundRobinPolicy
from cassandra.cluster import ExecutionProfile
from cassandra.cluster import EXEC_PROFILE_DEFAULT

myEProfile = ExecutionProfile(
  load_balancing_policy=DCAwareRoundRobinPolicy(
    local_dc='europe-west1',  # 对于 Astra DB，区域名称 = dc 名称
  )
)

cassandra_options = {
  'protocol_version': 4,      # 对于 Astra DB
  'execution_profiles': {EXEC_PROFILE_DEFAULT: myEProfile},
}
```

### S3 后端设置

!!! note

    此 S3 后端驱动程序需要 `s3`。

    要安装，请使用 `s3`

    ```console
    pip install celery[s3]
    ```

此后端需要设置以下配置指令。

#### `s3_access_key_id`

默认值：None。

S3 访问密钥 ID。

```python
s3_access_key_id = 'access_key_id'
```

#### `s3_secret_access_key`

默认值：None。

S3 秘密访问密钥。

```python
s3_secret_access_key = 'access_secret_access_key'
```

#### `s3_bucket`

默认值：None。

S3 存储桶名称。

```python
s3_bucket = 'bucket_name'
```

#### `s3_base_path`

默认值：None。

S3 存储桶中用于存储结果键的基础路径。

```python
s3_base_path = '/prefix'
```

#### `s3_endpoint_url`

默认值：None。

自定义 S3 端点 URL。用于连接到自定义自托管的 S3 兼容后端（Ceph、Scality 等）。

```python
s3_endpoint_url = 'https://.s3.custom.url'
```

#### `s3_region`

默认值：None。

S3 AWS 区域。

```python
s3_region = 'us-east-1'
```

#### 示例配置

```python
s3_access_key_id = 's3-access-key-id'
s3_secret_access_key = 's3-secret-access-key'
s3_bucket = 'mybucket'
s3_base_path = '/celery_result_backend'
s3_endpoint_url = 'https://endpoint_url'
```

### Azure Block Blob 后端设置

要使用 `AzureBlockBlob`_ 作为结果后端，您只需使用正确的 URL 配置 `result_backend` 设置。

所需的 URL 格式为 `azureblockblob://` 后跟存储连接字符串。您可以在 Azure 门户中存储帐户资源的 `访问密钥` 窗格中找到存储连接字符串。

#### 示例配置

```python
result_backend = 'azureblockblob://DefaultEndpointsProtocol=https;AccountName=somename;AccountKey=Lou...bzg==;EndpointSuffix=core.windows.net'
```

#### `azureblockblob_container_name`

默认值：celery。

用于存储结果的存储容器的名称。

#### `azureblockblob_base_path`

默认值：None。

存储容器中用于存储结果键的基本路径。

```python
azureblockblob_base_path = 'prefix/'
```

#### `azureblockblob_retry_initial_backoff_sec`

默认值：2。

第一次重试的初始回退间隔（以秒为单位）。后续重试采用指数策略。

#### `azureblockblob_retry_increment_base`

默认值：2。

#### `azureblockblob_retry_max_attempts`

默认值：3。

最大重试尝试次数。

#### `azureblockblob_connection_timeout`

默认值：20。

建立 Azure Block Blob 连接的超时时间（以秒为单位）。

#### `azureblockblob_read_timeout`

默认值：120。

读取 Azure Block Blob 的超时时间（以秒为单位）。

### GCS 后端设置

!!! note

    此 GCS 后端驱动程序需要 `google-cloud-storage` 和 `google-cloud-firestore`。

    要安装，请使用 `gcs`：

    ```console
    pip install celery[gcs]
    ```

    有关组合多个扩展要求的信息，请参阅 :ref:`bundles`。

GCS 可以通过 `result_backend` 中提供的 URL 进行配置，例如：：

```python
result_backend = 'gs://mybucket/some-prefix?gcs_project=myproject&ttl=600'
result_backend = 'gs://mybucket/some-prefix?gcs_project=myproject?firestore_project=myproject2&ttl=600'
```
此后端需要设置以下配置指令：

#### `gcs_bucket`

默认值：None。

GCS 存储桶名称。

```python
gcs_bucket = 'bucket_name'
```

#### `gcs_project`

默认值：None。

GCS 项目名称。

```python
gcs_project = 'test-project'
```

#### `gcs_base_path`

默认值：None。

在 GCS 存储桶中用于存储所有结果键的基本路径。

```python
gcs_base_path = '/prefix'
```

#### `gcs_ttl`

默认值：0。

结果 blob 的生存时间（以秒为单位）。
需要启用"删除"对象生命周期管理操作的 GCS 存储桶。
用于从云存储存储桶中自动删除结果。

例如，要在 24 小时后自动删除结果：

```python
gcs_ttl = 86400
```

#### `gcs_threadpool_maxsize`

默认值：10。

GCS 操作的线程池大小。相同的值定义了连接池大小。允许控制并发操作的数量。

```python
gcs_threadpool_maxsize = 20
```

#### `firestore_project`

默认值：gcs_project。

用于 Chord 引用计数的 Firestore 项目。允许本机 chord 引用计数。
如果未指定，则默认为 `gcs_project`。
例如：：

```python
firestore_project = 'test-project2'
```

#### 示例配置

```python
gcs_bucket = 'mybucket'
gcs_project = 'myproject'
gcs_base_path = '/celery_result_backend'
gcs_ttl = 86400
```

### Elasticsearch 后端设置

要将 `Elasticsearch`_ 用作结果后端，您只需使用正确的 URL 配置 `result_backend` 设置。

#### 示例配置

```python
result_backend = 'elasticsearch://example.com:9200/index_name/doc_type'
```

#### `elasticsearch_retry_on_timeout`

默认值：`False`

超时是否应在不同节点上触发重试？

#### `elasticsearch_max_retries`

默认值：3。

在异常传播之前最大重试次数。

#### `elasticsearch_timeout`

默认值：10.0 秒。

全局超时时间，由 elasticsearch 结果后端使用。

#### `elasticsearch_save_meta_as_text`

默认值：`True`

元数据应保存为文本还是原生 json。结果始终序列化为文本。

### AWS DynamoDB 后端设置

!!! note

    Dynamodb 后端需要 `boto3` 库。

    要安装此包，请使用 `pip`：

    ```console
    pip install celery[dynamodb]
    ```

!!! warning

    Dynamodb 后端与定义了排序键的表不兼容。

    如果您想基于分区键以外的内容查询结果表，请定义全局二级索引 (GSI)。

此后端要求将 `result_backend` 设置设置为 DynamoDB URL:

```python
result_backend = 'dynamodb://aws_access_key_id:aws_secret_access_key@region:port/table?read=n&write=m'
```

例如，指定 AWS 区域和表名:

```python
result_backend = 'dynamodb://@us-east-1/celery_results'
```

或者从环境变量中检索 AWS 配置参数，使用默认表名 (`celery`)并指定读取和写入预置吞吐量:

```python
result_backend = 'dynamodb://@/?read=5&write=5'
```

或者使用 DynamoDB 的 [可下载版本](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html) [本地端点](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.Endpoint.html):

```python
result_backend = 'dynamodb://@localhost:8000'
```

或者使用可下载版本或部署在任何主机上符合 API 的其他服务:

```python
result_backend = 'dynamodb://@us-east-1'
dynamodb_endpoint_url = 'http://192.168.0.40:8000'
```

`result_backend` 中 DynamoDB URL 的字段定义如下：

<table>
<thead>
<tr>
<th>字段</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<code>aws_access_key_id</code><br/>
<code>aws_secret_access_key</code>
</td>
<td>
用于访问 AWS API 资源的凭据。这些也可以通过 `boto3` 库从各种来源解析，
如<a href="http://boto3.readthedocs.io/en/latest/guide/configuration.html#configuring-credentials">此处</a>所述。
</td>
</tr>
<td><code>region</code></td>
<td>
AWS 区域，例如 <code>us-east-1</code> 或用于 <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html">可下载版本</a> 的 <code>localhost</code>。
有关定义选项，请参阅 <code>boto3</code> 库 <a href="http://boto3.readthedocs.io/en/latest/guide/configuration.html#environment-variable-configuration">文档</a>。
</td>
</tr>
<tr>
<td><code>port</code></td>
<td>
本地 DynamoDB 实例的监听端口，如果您使用的是可下载版本。
如果您没有将 <code>region</code> 参数指定为 <code>localhost</code>，
设置此参数将<b>无效</b>。
</td>
</tr>
<tr>
<td><code>table</code></td>
<td>
要使用的表名。默认为 <code>celery</code>。
有关允许的字符和长度的信息，请参阅 <a href="http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Limits.html#limits-naming-rules">DynamoDB 命名规则</a>。
</td>
</tr>
<tr>
<td><code>read & write</code></td>
<td>
创建的 DynamoDB 表的读取和写入容量单位。默认情况下，读取和写入均为 <code>1</code>。
更多详细信息可以在 <a href="http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ProvisionedThroughput.html">预置吞吐量文档</a> 中找到。
</td>
</tr>
<tr>
<td><code>ttl_seconds</code></td>
<td>
结果在过期前的生存时间（以秒为单位）。默认情况下，结果不会过期，同时保持 DynamoDB 表的生存时间设置不变。如果 <code>ttl_seconds</code> 设置为正值，结果将在指定的秒数后过期。将 <code>ttl_seconds</code>
设置为负值意味着结果不会过期，并且主动禁用 DynamoDB 表的生存时间设置。请注意，尝试在已有项目的表上启用生存时间将导致错误。更多详细信息可以在<a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html">DynamoDB TTL 文档</a> 中找到。
</td>
</tr>
</tbody>
</table>

### IronCache 后端设置

!!! note

    IronCache 后端需要 `iron_celery` 库：

    要安装此包，请使用 `pip`：

    ```console
    pip install iron_celery
    ```

IronCache 通过 `result_backend` 中提供的 URL 进行配置，

```python
result_backend = 'ironcache://project_id:token@'
```

或者更改缓存名称：

```python
ironcache://project_id:token@/awesomecache
```

更多信息，请参阅：https://github.com/iron-io/iron_celery

### Couchbase 后端设置

!!! note

    Couchbase 后端需要 `couchbase` 库。

    要安装此包，请使用 `pip`：

    ```console
    pip install celery[couchbase]
    ```

可以通过将 `result_backend` 设置为 Couchbase URL 来配置此后端：

```python
result_backend = 'couchbase://username:password@host:port/bucket'
```

#### `couchbase_backend_settings`

默认值：`{}`（空映射）。

这是一个支持以下键的字典：

| 键          | 描述                                   |
|------------|--------------------------------------|
| `host`     | Couchbase 服务器的主机名。默认为 `localhost`。   |
| `port`     | Couchbase 服务器正在监听的端口。默认为 `8091`。     |
| `bucket`   | Couchbase 服务器写入的默认存储桶。默认为 `default`。 |
| `username` | 用于验证 Couchbase 服务器的用户名（可选）。          |
| `password` | 用于验证 Couchbase 服务器的密码（可选）。           |

### ArangoDB 后端设置

!!! note

    ArangoDB 后端需要 `pyArango` 库。

    要安装此包，请使用 `pip`：

    ```console
    pip install celery[arangodb]
    ```

此后端可以通过将 `result_backend` 设置为 ArangoDB URL 来配置：

```python
result_backend = 'arangodb://username:password@host:port/database/collection'
```

#### `arangodb_backend_settings`

默认值：`{}`（空映射）。

这是一个支持以下键的字典：

| 键               | 描述                                       |
|-----------------|------------------------------------------|
| `host`          | ArangoDB 服务器的主机名。默认为 `localhost`。        |
| `port`          | ArangoDB 服务器监听的端口。默认为 `8529`。            |
| `database`      | ArangoDB 服务器中写入的默认数据库。默认为 `celery`。      |
| `collection`    | ArangoDB 服务器数据库中写入的默认集合。默认为 `celery`。    |
| `username`      | 用于验证 ArangoDB 服务器的用户名（可选）。               |
| `password`      | 用于验证 ArangoDB 服务器的密码（可选）。                |
| `http_protocol` | ArangoDB 服务器连接的 HTTP 协议。默认为 `http`。      |
| `verify`        | 创建 ArangoDB 连接时的 HTTPS 验证检查。默认为 `False`。 |

### CosmosDB 后端设置（实验性）

要将 `CosmosDB` 用作结果后端，您只需使用正确的 URL 配置 `result_backend` 设置。

#### 示例配置

```python
result_backend = 'cosmosdbsql://:{InsertAccountPrimaryKeyHere}@{InsertAccountNameHere}.documents.azure.com'
```

#### `cosmosdbsql_database_name`

默认值：celerydb。

用于存储结果的数据库名称。

#### `cosmosdbsql_collection_name`

默认值：celerycol。

用于存储结果的集合名称。

#### `cosmosdbsql_consistency_level`

默认值：Session。

表示 Azure Cosmos DB 客户端操作支持的一致性级别。

一致性级别按强度顺序排列为：Strong、BoundedStaleness、Session、ConsistentPrefix 和 Eventual。

#### `cosmosdbsql_max_retry_attempts`

默认值：9。

请求的最大重试次数。

#### `cosmosdbsql_max_retry_wait_time`

默认值：30。

重试过程中等待请求的最大等待时间（秒）。

### CouchDB 后端设置

!!! note

    CouchDB 后端需要 `pycouchdb` 库：

    要安装此 Couchbase 包，请使用 `pip`：

    ```console
    pip install celery[couchdb]
    ```

此后端可以通过将 `result_backend` 设置为 CouchDB URL 来配置：

```python
result_backend = 'couchdb://username:password@host:port/container'
```

URL 由以下部分组成：

| 键           | 描述                                |
|-------------|-----------------------------------|
| `username`  | 用于验证 CouchDB 服务器的用户名（可选）。         |
| `password`  | 用于验证 CouchDB 服务器的密码（可选）。          |
| `host`      | CouchDB 服务器的主机名。默认为 `localhost`。  |
| `port`      | CouchDB 服务器监听的端口。默认为 `8091`。      |
| `container` | CouchDB 服务器写入的默认容器。默认为 `default`。 |

### 文件系统后端设置

此后端可以使用文件 URL 进行配置

```python
CELERY_RESULT_BACKEND = 'file:///var/celery/results'
```

配置的目录需要被所有使用该后端的服务器共享并可写入。

如果您在单个系统上尝试使用 Celery，可以简单地使用该后端而无需进一步配置。对于较大的集群，您可以使用 NFS、[GlusterFS](http://www.gluster.org/)、CIFS、[HDFS](http://hadoop.apache.org/)（使用 FUSE）或任何其他文件系统。

### Consul K/V 存储后端设置

!!! note

    Consul 后端需要 `python-consul2` 库：

    要安装此包，请使用 `pip`：

    ```console
    pip install python-consul2
    ```

Consul 后端可以使用 URL 进行配置，例如：：

```python
CELERY_RESULT_BACKEND = 'consul://localhost:8500/'
```

或者：：

```python
result_backend = 'consul://localhost:8500/'
```

该后端将结果存储在 Consul 的 K/V 存储中作为单独的键。后端支持使用 Consul 中的 TTL 自动过期结果。URL 的完整语法是：

```text
consul://host:port[?one_client=1]
```

URL 由以下部分组成：

| 键            | 描述                                                                                                                                                                                                                                                                |
|--------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `host`       | Consul 服务器的主机名。默认为 `localhost`。                                                                                                                                                                                                                                   |
| `port`       | Consul 服务器监听的端口。默认为 `8500`。                                                                                                                                                                                                                                       |
| `one_client` | 默认情况下，为了正确性，后端为每个操作使用单独的客户端连接。在极端负载情况下，新连接的创建速率可能导致 Consul 服务器在负载下返回 HTTP 429 "连接过多" 错误响应。推荐的解决方法是使用 https://github.com/poppyred/python-consul2/pull/31 处的补丁在 `python-consul2` 中启用重试。或者，如果设置了 `one_client`，将使用单个客户端连接来处理所有操作。这应该可以消除 HTTP 429 错误，但后端存储结果可能会变得不可靠。 |

### 消息路由

#### `task_queues`

默认值：`None`（从默认队列设置中获取队列）。

大多数用户不需要指定此设置，而应该使用 [自动路由功能](routing.md#routing-automatic)。

如果您确实想要配置高级路由，此设置应该是 worker 将从中消费的 `kombu.Queue` 对象列表。

注意，可以通过 `celery worker -Q` 选项覆盖此设置，或者可以使用 `celery worker -X` 选项从此列表中排除单个队列（按名称）。

另请参阅 [路由基础知识](routing.md#routing-basics) 获取更多信息。

默认是一个队列/交换/绑定键为 `celery`，交换类型为 `direct`。

#### `task_routes`

默认值：`None`。

用于将任务路由到队列的路由器列表或单个路由器。在决定任务的最终目的地时，会按顺序咨询路由器。

路由器可以指定为以下任一形式：

- 具有签名 `(name, args, kwargs, options, task=None, **kwargs)` 的函数
- 提供路由器函数路径的字符串
- 包含路由器规范的字典：将被转换为 `celery.routes.MapRoute` 实例
- `(pattern, route)` 元组列表：将被转换为 `celery.routes.Map oute` 实例

示例：

```python
task_routes = {
    'celery.ping': 'default',
    'mytasks.add': 'cpu-bound',
    'feed.tasks.*': 'feeds',                           # <-- 通配符模式
    re.compile(r'(image|video)\.tasks\..*'): 'media',  # <-- 正则表达式
    'video.encode': {
        'queue': 'video',
        'exchange': 'media',
        'routing_key': 'media.video.encode',
    },
}

task_routes = ('myapp.tasks.route_task', {'celery.ping': 'default'})
```

其中 `myapp.tasks.route_task` 可以是：

```python
def route_task(self, name, args, kwargs, options, task=None, **kw):
    if task == 'celery.ping':
        return {'queue': 'default'}
```

`route_task` 可以返回字符串或字典。字符串表示它是 `task_queues` 中的队列名称，字典表示它是自定义路由。

发送任务时，会按顺序咨询路由器。第一个不返回 `None` 的路由器就是要使用的路由。然后消息选项会与找到的路由设置合并，其中任务的设置具有优先级。

如果 `celery.execute.apply_async` 有以下参数：

```python
Task.apply_async(immediate=False, exchange='video', routing_key='video.compress')
```

而路由器返回：

```python
{'immediate': True, 'exchange': 'urgent'}
```

最终的消息选项将是：

```python
immediate=False, exchange='video', routing_key='video.compress'
```

（以及 `celery.app.task.Task` 类中定义的任何默认消息选项）

合并两者时，`task_routes` 中定义的值优先于 `task_queues` 中定义的值。

使用以下设置：

```python
task_queues = {
    'cpubound': {
        'exchange': 'cpubound',
        'routing_key': 'cpubound',
    },
}

task_routes = {
    'tasks.add': {
        'queue': 'cpubound',
        'routing_key': 'tasks.add',
        'serializer': 'json',
    },
}
```

`tasks.add` 的最终路由选项将变为：

```javascript
    {'exchange': 'cpubound',
     'routing_key': 'tasks.add',
     'serializer': 'json'}
```

请参阅 [路由指南](routing.md) 获取更多示例。

#### `task_queue_max_priority`

默认值：`None`。

#### `task_default_priority`

默认值：`None`。

#### `task_inherit_parent_priority`

默认值：`False`。

如果启用，子任务将继承父任务的优先级。

```python
# 链中的最后一个任务也将优先级设置为5。
chain = celery.chain(add.s(2) | add.s(2).set(priority=5) | add.s(3))
```

当使用 `delay` 或 `apply_async` 从父任务调用子任务时，优先级继承也有效。

#### `worker_direct`

默认值：禁用。

此选项启用后，每个 worker 都会有一个专用队列，以便可以将任务路由到特定的 worker。

每个 worker 的队列名称基于 worker 主机名和 `.dq` 后缀自动生成，使用 `C.dq2` 交换。

例如，节点名称为 `w1@example.com` 的 worker 的队列名称变为:

    w1@example.com.dq

然后您可以通过将主机名指定为路由键和 `C.dq2` 交换来将任务路由到该 worker:

```python
task_routes = {
    'tasks.add': {'exchange': 'C.dq2', 'routing_key': 'w1@example.com'}
}
```

#### `task_create_missing_queues`

默认值：启用。

如果启用（默认），任何在 `task_queues` 中未定义但指定的队列将自动创建。请参阅 :ref:`routing-automatic`。

#### `task_create_missing_queue_type`

默认值：`"classic"`

当 Celery 需要声明一个不存在的队列时（即当 `task_create_missing_queues` 启用时），此设置定义要创建的 RabbitMQ 队列类型。

- `"classic"`（默认）：声明标准经典队列。
- `"quorum"`：声明 RabbitMQ 仲裁队列（添加 `x-queue-type: quorum`）。

#### `task_create_missing_queue_exchange_type`

默认值：`None`

如果此选项为 None 或空字符串（默认），Celery 将保持交换与您的 `app.amqp.Queues.autoexchange` 钩子返回的完全相同。

您可以将其设置为特定的交换类型，例如 `"direct"`、`"topic"` 或 `"fanout"`，以使用该交换类型创建缺失的队列。

!!! tip

    将此设置与 task_create_missing_queue_type = "quorum" 结合使用以创建绑定到主题交换的仲裁队列，例如:
    
    ```python
    app.conf.task_create_missing_queues=True
    app.conf.task_create_missing_queue_type="quorum"
    app.conf.task_create_missing_queue_exchange_type="topic"
    ```

!!! note

    与上面的队列类型设置一样，此选项不会影响您在 `task_queues` 中明确定义的队列；它仅适用于在运行时隐式创建的队列。

#### `task_default_queue`

默认值：`"celery"`。

如果消息没有路由或未指定自定义队列，`.apply_async` 使用的默认队列名称。

此队列必须列在 `task_queues` 中。如果未指定 `task_queues`，则会自动创建包含一个队列条目，其中此名称用作该队列的名称。

#### `task_default_queue_type`

默认值：`"classic"`。

此设置用于允许更改 `task_default_queue` 队列的默认队列类型。另一个可行的选项是 `"quorum"`，它仅受 RabbitMQ 支持，并使用 `x-queue-type` 队列参数将队列类型设置为 `quorum`。

如果启用了 `worker_detect_quorum_queues` 设置，worker 将自动检测队列类型并相应地禁用全局 QoS。

!!! warning

    仲裁队列需要启用确认发布。
    使用 `broker_transport_options` 通过以下设置启用确认发布：

    ```python
    broker_transport_options = {"confirm_publish": True}
    ```

    有关更多信息，请参阅 [RabbitMQ 文档](https://www.rabbitmq.com/docs/quorum-queues#use-cases)。

#### `task_default_exchange`

默认值：使用为 `task_default_queue` 设置的值。

当在 `task_queues` 设置中未为键指定自定义交换时使用的默认交换名称。

#### `task_default_exchange_type`

默认值：`"direct"`。

当在 `task_queues` 设置中未为键指定自定义交换类型时使用的默认交换类型。

#### `task_default_routing_key`

默认值：使用为 `task_default_queue` 设置的值。

当在 `task_queues` 设置中未为键指定自定义路由键时使用的默认路由键。

#### `task_default_delivery_mode`

默认值：`"persistent"`。

可以是 `transient`（消息不写入磁盘）或 `persistent`（写入磁盘）。

### Broker 设置 {#conf-broker-settings}

#### `broker_url`

默认值：`"amqp://"`

默认的 broker URL。这必须是一个格式如下的 URL：：

    transport://userid:password@hostname:port/virtual_host

只有方案部分（`transport://`）是必需的，其余部分是可选的，并默认为特定传输的默认值。

传输部分是使用的 broker 实现，默认是 `amqp`，（如果安装了 `librabbitmq` 则使用它，否则回退到 `pyamqp`）。还有其他可用的选择，包括：`redis://`、`sqs://` 和 `qpid://`。

方案也可以是你自己传输实现的完全限定路径：

```python
broker_url = 'proj.transports.MyTransport://localhost'
```

也可以指定多个相同传输的 broker URL。broker URL 可以作为分号分隔的单个字符串传递：

```python
broker_url = 'transport://userid:password@hostname:port//;transport://userid:password@hostname:port//'
```

或作为列表：

```python
broker_url = [
    'transport://userid:password@localhost:port//',
    'transport://userid:password@hostname:port//'
]
```

然后这些 broker 将用于 `broker_failover_strategy`。

有关更多信息，请参阅 Kombu 文档中的 [连接 URL](https://kombu.readthedocs.io/en/latest/reference/kombu.connection.html#connection-urls)。

#### `broker_read_url` / `broker_write_url`

默认值：取自 `broker_url`。

这些设置可以配置，以替代 `broker_url`，为用于消费和生产的 broker 连接指定不同的连接参数。

示例：

```python
broker_read_url = 'amqp://user:pass@broker.example.com:56721'
broker_write_url = 'amqp://user:pass@broker.example.com:56722'
```

这两个选项也可以指定为故障转移备选的列表，有关更多信息，请参阅 `broker_url`。

#### `broker_failover_strategy`

默认值：`"round-robin"`。

broker Connection 对象的默认故障转移策略。如果提供，可以映射到 'kombu.connection.failover_strategies' 中的一个键，或者是对从提供的列表中产生单个项目的任何方法的引用。

示例：：

```python
# 随机故障转移策略
def random_failover_strategy(servers):
    it = list(servers)  # 不要修改调用者的列表
    shuffle = random.shuffle
    for _ in repeat(None):
        shuffle(it)
        yield it[0]

broker_failover_strategy = random_failover_strategy
```

#### `broker_heartbeat`

默认值：`120.0`（由服务器协商）。

注意：此值仅由 worker 使用，客户端目前不使用心跳。

仅使用 TCP/IP 并不总是能够及时检测到连接丢失，因此 AMQP 定义了一种称为心跳的机制，由客户端和 broker 共同使用来检测连接是否已关闭。

如果心跳值为 10 秒，则心跳将在 `broker_heartbeat_checkrate` 设置指定的间隔内进行监控（默认情况下，这设置为心跳值的双倍速率，因此对于 10 秒的心跳，每 5 秒检查一次心跳）。

#### `broker_heartbeat_checkrate`

默认值：2.0。

worker 会定期监控 broker 是否错过了太多心跳。检查的频率是通过将 `broker_heartbeat` 值除以该值来计算的，因此如果心跳为 10.0，速率为默认的 2.0，则每 5 秒执行一次检查（心跳发送速率的两倍）。

#### `broker_use_ssl`

默认值：禁用。

切换 broker 连接上的 SSL 使用和 SSL 设置。

此选项的有效值因传输而异。

##### `pyamqp`

如果为 `True`，连接将使用默认 SSL 设置。如果设置为字典，将根据指定的策略配置 SSL 连接。使用的格式是 Python 的 `ssl.wrap_socket` 选项。

请注意，SSL 套接字通常由 broker 在单独的端口上提供服务。

提供客户端证书并根据自定义证书颁发机构验证服务器证书的示例：

```python
import ssl

broker_use_ssl = {
  'keyfile': '/var/ssl/private/worker-key.pem',
  'certfile': '/var/ssl/amqp-server-cert.pem',
  'ca_certs': '/var/ssl/myca.pem',
  'cert_reqs': ssl.CERT_REQUIRED
}
```

!!! info "5.1"

    从 Celery 5.1 开始，py-amqp 将始终验证从服务器接收的证书，不再需要手动将 `cert_reqs` 设置为 `ssl.CERT_REQUIRED`。

    之前的默认值 `ssl.CERT_NONE` 不安全，我们不鼓励使用它。
    如果你想恢复到之前的不安全默认值，请将 `cert_reqs` 设置为 `ssl.CERT_NONE`。

##### `redis`

该设置必须是一个包含以下键的字典：

- `ssl_cert_reqs`（必需）：`SSLContext.verify_mode` 值之一：
    - `ssl.CERT_NONE`
    - `ssl.CERT_OPTIONAL`
    - `ssl.CERT_REQUIRED`
- `ssl_ca_certs`（可选）：CA 证书的路径
- `ssl_certfile`（可选）：客户端证书的路径
- `ssl_keyfile`（可选）：客户端密钥的路径

#### `broker_pool_limit`

默认值：10。

连接池中可以打开的最大连接数。

自版本 2.5 起，池默认启用，默认限制为十个连接。这个数字可以根据使用连接的线程/绿色线程（eventlet/gevent）的数量进行调整。例如，运行 eventlet 时，如果有 1000 个使用 broker 连接的 greenlet，可能会出现争用，你应该考虑增加限制。

如果设置为 `None` 或 0，连接池将被禁用，每次使用都会建立和关闭连接。

#### `broker_connection_timeout`

默认值：4.0。

在放弃建立与 AMQP 服务器的连接之前的默认超时时间（秒）。使用 gevent 时，此设置被禁用。

!!! note

    broker 连接超时仅适用于尝试连接到 broker 的 worker。它不适用于生产者发送任务的情况，有关如何为该情况提供超时，请参阅 `broker_transport_options`。

#### `broker_connection_retry`

默认值：启用。

如果初始连接建立后丢失，自动尝试重新建立与 AMQP broker 的连接。

每次重试之间的时间都会增加，并且在超过 `broker_connection_max_retries` 之前不会耗尽重试次数。

!!! warning

    broker_connection_retry 配置设置将不再决定在 Celery 6.0 及更高版本中是否在启动期间进行 broker 连接重试。
    如果你希望在启动时不重试连接，你应该将 `broker_connection_retry_on_startup` 设置为 False。

#### `broker_connection_retry_on_startup`

默认值：启用。

如果 AMQP broker 不可用，在 Celery 启动时自动尝试建立连接。

每次重试之间的时间都会增加，并且在超过 `broker_connection_max_retries` 之前不会耗尽重试次数。

#### `broker_connection_max_retries`

默认值：100。

在放弃重新建立与 AMQP broker 的连接之前的最大重试次数。

如果设置为 `None`，我们将永远重试。

#### `broker_channel_error_retry`

默认值：禁用。

如果返回了任何无效响应，自动尝试重新建立与 AMQP broker 的连接。

重试计数和间隔与 `broker_connection_retry` 相同。
此外，当 `broker_connection_retry` 为 `False` 时，此选项不起作用。

#### `broker_login_method`

默认值：`"AMQPLAIN"`。

设置自定义的 AMQP 登录方法。

#### `broker_native_delayed_delivery_queue_type`

:transports supported: `pyamqp`

默认值：`"quorum"`。

此设置用于允许更改原生延迟传递队列的默认队列类型。另一个可行的选项是 `"classic"`，它仅由 RabbitMQ 支持，并使用 `x-queue-type` 队列参数将队列类型设置为 `classic`。

#### `broker_transport_options`

默认值：`{}`（空映射）。

传递给底层传输的附加选项字典。

有关支持的选项（如果有），请参阅你的传输用户手册。

设置可见性超时的示例（由 Redis 和 SQS 传输支持）：

```python
broker_transport_options = {'visibility_timeout': 18000}  # 5 小时
```

设置生产者连接最大重试次数的示例（这样如果 broker 在第一次任务执行时不可用，生产者不会永远重试）：

```python
broker_transport_options = {'max_retries': 5}
```

启用发布者确认的示例（由 `pyamqp` 传输支持）。如果没有这个，当 broker 达到资源限制时，消息可能会被静默丢弃：

```python
broker_transport_options = {'confirm_publish': True}
```

### Worker

#### `imports`

默认值：`[]`（空列表）。

在 worker 启动时要导入的模块序列。

这用于指定要导入的任务模块，同时也用于导入信号处理程序和额外的远程控制命令等。

模块将按原始顺序导入。

#### `include`

默认值：`[]`（空列表）。

与 `imports` 具有完全相同的语义，但可以作为一种手段来拥有不同的导入类别。

此设置中的模块在 `imports` 中的模块之后导入。

#### `worker_deduplicate_successful_tasks`

默认值：False

在每次任务执行之前，指示 worker 检查此任务是否是重复消息。

去重仅发生在具有相同标识符、启用了延迟确认、由消息代理重新传递并且在结果后端中状态为 `SUCCESS` 的任务上。

为了避免用查询淹没结果后端，在查询结果后端之前会检查一个本地缓存的已成功执行的任务，以防该任务已经被接收该任务的同一 worker 成功执行。

通过设置 `worker_state_db` 设置可以使此缓存持久化。

如果结果后端不是[持久化的](https://github.com/celery/celery/blob/main/celery/backends/base.py#L102)（例如 RPC 后端），则忽略此设置。

#### `worker_concurrency`

默认值：CPU 核心数。

执行任务的并发 worker 进程/线程/绿色线程的数量。

如果您主要进行 I/O 操作，可以拥有更多进程，但如果主要是 CPU 密集型，请尝试保持接近机器上的 CPU 数量。如果未设置，将使用主机上的 CPU/核心数。

#### `worker_prefetch_multiplier`

默认值：4。

一次预取多少条消息乘以并发进程数。默认值为 4（每个进程四条消息）。默认设置通常是一个不错的选择，但是——如果您有非常长时间运行的任务在队列中等待，并且您必须启动 workers，请注意第一个启动的 worker 最初将收到四倍的消息数量。因此，任务可能不会公平地分配给 workers。

要将代理限制为每次只向每个进程传递一条消息，请将 `worker_prefetch_multiplier` 设置为 1。将该设置更改为 0 将允许 worker 继续消费任意数量的消息。

如果您需要在使用早期确认的同时完全禁用代理预取，请启用 `worker_disable_prefetch`。当启用此选项时，worker 仅在其某个进程可用时才从代理获取任务。

!!! note

    此功能目前仅在使用 Redis 作为代理时受支持。

您也可以通过 `celery worker --disable-prefetch` 命令行标志启用此功能。

#### `worker_eta_task_limit`

默认值：无限制（None）。

worker 可以一次在内存中保存的最大 ETA/倒计时任务数量。当达到此限制时，worker 将不会从代理接收新任务，直到一些现有的 ETA 任务被执行。

当队列包含大量具有 ETA/倒计时值的任务时，此设置有助于防止内存耗尽，因为这些任务在内存中保存直到它们的执行时间。如果没有此限制，workers 可能会将数千个 ETA 任务提取到内存中，可能导致内存不足问题。

!!! note

    具有 ETA/倒计时的任务不受预取限制的影响。

#### `worker_disable_prefetch`

默认值：`False`。

启用后，worker 仅在其有可用进程执行任务时才从代理消费消息。这在使用早期确认的同时禁用了预取，确保任务在 workers 之间公平分配。

!!! note

    此功能目前仅在使用 Redis 作为代理时受支持。在其他代理上使用此设置将导致警告并且该设置将被忽略。

#### `worker_enable_prefetch_count_reduction`

默认值：启用。

`worker_enable_prefetch_count_reduction` 设置控制在与消息代理的连接丢失后，预取计数恢复到其最大允许值的行为。默认情况下，此设置是启用的。

在连接丢失时，Celery 将尝试自动重新连接到代理，前提是 `broker_connection_retry_on_startup` 或 `broker_connection_retry` 未设置为 False。在连接丢失期间，消息代理不会跟踪已获取的任务数量。因此，为了有效管理任务负载并防止过载，Celery 会根据当前正在运行的任务数量减少预取计数。

预取计数是 worker 一次从代理获取的消息数量。减少的预取计数有助于确保在重新连接期间不会过度获取任务。

当 `worker_enable_prefetch_count_reduction` 设置为其默认值（启用）时，每次在连接丢失前正在运行的任务完成时，预取计数将逐渐恢复到其最大允许值。此行为有助于在有效管理负载的同时保持任务在 workers 之间的平衡分配。

要在重新连接时禁用预取计数的减少和恢复到其最大允许值，请将 `worker_enable_prefetch_count_reduction` 设置为 False。在需要固定预取计数来控制任务处理速率或管理 worker 负载的场景中，禁用此设置可能很有用，尤其是在连接不稳定的环境中。

`worker_enable_prefetch_count_reduction` 设置提供了一种方法来控制在连接丢失后预取计数的恢复行为，有助于在 workers 之间保持平衡的任务分配和有效的负载管理。

#### `worker_lost_wait`

默认值：10.0 秒。

在某些情况下，worker 可能会在没有适当清理的情况下被杀死，并且 worker 可能在终止之前发布了结果。此值指定在引发 `WorkerLostError` 异常之前我们等待任何缺失结果的时间。

#### `worker_max_tasks_per_child`

池工作进程在被新进程替换之前可以执行的最大任务数。默认无限制。

#### `worker_max_memory_per_child`

默认值：无限制。
类型：int（千字节）

在 worker 被新 worker 替换之前可以消耗的最大驻留内存量，以千字节（1024 字节）为单位。如果单个任务导致 worker 超过此限制，该任务将完成，然后 worker 将被替换。

示例：

```python
worker_max_memory_per_child = 12288  # 12 * 1024 = 12 MB
```

#### `worker_disable_rate_limits`

默认值：禁用（速率限制启用）。

禁用所有速率限制，即使任务设置了明确的速率限制。

#### `worker_state_db`

默认值：`None`。

用于存储持久化 worker 状态（如已撤销任务）的文件名。 可以是相对路径或绝对路径，但请注意后缀 `.db` 可能会附加到文件名（取决于 Python 版本）。

也可以通过 `celery worker --statedb` 参数设置。

#### `worker_timer_precision`

默认值：1.0 秒。

设置 ETA 调度器在重新检查调度之间可以休眠的最大时间（以秒为单位）。

将此值设置为 1 秒意味着调度器的精度将为 1 秒。如果您需要接近毫秒的精度，可以将其设置为 0.1。

#### `worker_enable_remote_control`

默认值：默认启用。

指定是否启用 workers 的远程控制。

#### `worker_proc_alive_timeout`

默认值：4.0。

等待新 worker 进程启动的超时时间（以秒为单位，int/float）。

#### `worker_cancel_long_running_tasks_on_connection_loss`

默认值：默认禁用。

在连接丢失时杀死所有启用了延迟确认的长时间运行任务。

在连接丢失之前未被确认的任务无法再被确认，因为它们的通道已消失，并且任务被重新传递回队列。这就是为什么启用了延迟确认的任务必须是幂等的，因为它们可能被执行多次。在这种情况下，每次连接丢失时任务都会被执行两次（有时在其他 workers 中并行执行）。

当启用此选项时，那些尚未完成的任务将被取消并终止其执行。在连接丢失之前以任何方式完成的任务都会在结果后端中记录，只要 `task_ignore_result` 未被启用。

!!! warning

    此功能是作为未来的破坏性变更引入的。如果关闭此功能，Celery 将发出警告消息。

    在 Celery 6.0 中，`worker_cancel_long_running_tasks_on_connection_loss` 将默认设置为 `True`，因为当前行为带来的问题比解决的问题更多。

#### `worker_detect_quorum_queues`

默认值：启用。

自动检测 `task_queues` 中的任何队列是否为仲裁队列（包括 `task_default_queue`），如果检测到任何仲裁队列，则禁用全局 QoS。

#### `worker_soft_shutdown_timeout`

默认值：0.0。

标准的 [软关闭](workers.md#worker-soft-shutdown) 将在关闭之前等待所有任务完成，除非触发了冷关闭。[软关闭](workers.md#worker-soft-shutdown) 将在启动冷关闭之前添加一个等待时间。此设置指定 worker 在启动冷关闭并终止之前将等待多长时间。

这也适用于 worker 在没有先进行热关闭的情况下启动 [冷关闭](workers.md#worker-cold-shutdown) 时。

如果该值设置为 0.0，软关闭实际上将被禁用。无论值如何，如果没有任务正在运行，软关闭将被禁用（除非启用了 `worker_enable_soft_shutdown_on_idle`）。

尝试使用此值来找到在 worker 终止之前任务优雅完成的最佳时间。推荐值可以是 10、30、60 秒。太高的值可能导致 worker 终止前等待时间过长，并触发 `KILL` 信号由主机系统强制终止 worker。

#### `worker_enable_soft_shutdown_on_idle`

默认值：False。

如果 `worker_soft_shutdown_timeout` 设置为大于 0.0 的值，如果没有任务正在运行，worker 将跳过 [软关闭](workers.md#worker-soft-shutdown)。此设置将启用软关闭，即使没有任务正在运行。

!!! tip

    当 worker 接收到 ETA 任务，但 ETA 尚未到达，并且启动了关闭时，如果没有任务正在运行，worker 将**跳过**软关闭并立即启动冷关闭。这可能导致在 worker 拆卸期间重新排队 ETA 任务失败。为了缓解这种情况，启用此配置以确保 worker 无论如何都会等待，这为优雅关闭和成功重新排队 ETA 任务提供了足够的时间。

### Events

#### `worker_send_task_events`

默认值：默认禁用。

发送任务相关事件，以便可以使用像 `flower` 这样的工具来监控任务。设置工作者的默认值 `celery worker -E` 参数。

#### `task_send_sent_event`

默认值：默认禁用。

如果启用，将为每个任务发送一个 `task-sent` 事件，以便在任务被工作者消费之前可以跟踪它们。

#### `event_queue_ttl`

默认值：5.0 秒。

消息过期时间（秒，整数/浮点数），用于当发送到监控客户端事件队列的消息被删除时（`x-message-ttl`）。

例如，如果此值设置为 10，则传递到此队列的消息将在 10 秒后被删除。

#### `event_queue_expires`

默认值：60.0 秒。

过期时间（秒，整数/浮点数），用于在监控客户端事件队列将被删除时（`x-expires`）。

#### `event_queue_durable`

默认值：`False`

如果启用，事件接收器的队列将被标记为 *持久化*，意味着它将在代理重启后继续存在。

#### `event_queue_exclusive`

默认值：`False`

如果启用，事件队列将 *独占* 于当前连接，并在连接关闭时自动删除。

!!! warning

    您 **不能** 同时将 `event_queue_durable` 和 `event_queue_exclusive` 设置为 `True`。
    如果两者都设置，Celery 将引发 :exc:`ImproperlyConfigured` 错误。

#### `event_queue_prefix`

默认值：`"celeryev"`。

用于事件接收器队列名称的前缀。

#### `event_exchange`

默认值：`"celeryev"`。

事件交换器的名称。

!!! warning

    此选项处于实验阶段，请谨慎使用。

#### `event_serializer`

默认值：`"json"`。

发送事件消息时使用的消息序列化格式。

#### `events_logfile`

默认值：`None`

`celery events` 用于记录日志的可选文件路径（默认为 `stdout`）。

#### `events_pidfile`

默认值：`None`

`celery events` 用于创建/存储其 PID 文件的可选文件路径（默认为不创建 PID 文件）。

#### `events_uid`

默认值：`None`

当事件 `celery events` 降低其权限时使用的可选用户 ID（默认为不更改 UID）。

#### `events_gid`

默认值：`None`

当 `celery events` 守护进程降低其权限时使用的可选组 ID（默认为不更改 GID）。

#### `events_umask`

默认值：`None`

当 `celery events` 在守护进程化时创建文件（日志、pid 等）时使用的可选 `umask`。

#### `events_executable`

默认值：`None`

`celery events` 在守护进程化时使用的可选 `python` 可执行文件路径（默认为 :data:`sys.executable`）。

### 远程控制命令

!!! note

    要禁用远程控制命令，请参阅 `worker_enable_remote_control` 设置。

#### `control_queue_ttl`

默认值：300.0

时间（以秒为单位），远程控制命令队列中的消息在过期之前的时间。

如果使用默认的300秒，这意味着如果发送了远程控制命令，并且在300秒内没有工作进程拾取它，该命令将被丢弃。

此设置也适用于远程控制回复队列。

#### `control_queue_expires`

默认值：10.0

时间（以秒为单位），未使用的远程控制命令队列从代理中删除之前的时间。

此设置也适用于远程控制回复队列。

#### `control_exchange`

默认值：`"celery"`。

控制命令交换的名称。

!!! warning

    此选项处于实验阶段，请谨慎使用。

### `control_queue_durable`

默认值：`False`

类型：`bool`

如果设置为 `True`，控制交换机和队列将是持久的——它们将在代理重启后继续存在。

### `control_queue_exclusive`

默认值：`False`

类型：`bool`

如果设置为 `True`，控制队列将专属于单个连接。在分布式环境中通常不推荐这样做。

!!! warning

    同时将 `control_queue_durable` 和 `control_queue_exclusive` 设置为 `True` 是不支持的，会引发错误。

### 日志记录

#### `worker_hijack_root_logger`

默认值：默认启用（劫持根日志记录器）。

默认情况下，根日志记录器上任何先前配置的处理程序都将被移除。如果您想要自定义自己的日志处理程序，可以通过设置 `worker_hijack_root_logger = False` 来禁用此行为。

!!! note

    日志记录也可以通过连接到 `celery.signals.setup_logging` 信号来自定义。

#### `worker_log_color`

默认值：如果应用程序正在记录到终端，则启用。

启用/禁用 Celery 应用程序日志输出中的颜色。

#### `worker_log_format`

默认值：

```text
"[%(asctime)s: %(levelname)s/%(processName)s] %(message)s"
```

用于日志消息的格式。

有关日志格式的更多信息，请参阅 Python `logging` 模块。

#### `worker_task_log_format`

默认值：

```text
"[%(asctime)s: %(levelname)s/%(processName)s]
        %(task_name)s[%(task_id)s]: %(message)s"
```

用于任务中记录的日志消息的格式。

有关日志格式的更多信息，请参阅 Python `logging` 模块。

#### `worker_redirect_stdouts`

默认值：默认启用。

如果启用，`stdout` 和 `stderr` 将被重定向到当前日志记录器。

由 `celery worker` 和 `celery beat` 使用。

#### `worker_redirect_stdouts_level`

默认值：`WARNING`。

输出到 `stdout` 和 `stderr` 的日志级别。可以是 `DEBUG`、`INFO`、`WARNING`、`ERROR` 或 `CRITICAL` 之一。

### 安全

#### `security_key`

默认值: `None`.

包含私钥的文件的相对或绝对路径，当使用 `message-signing` 时用于签名消息。

#### `security_key_password`

默认值: `None`.

当使用 `message-signing` 时用于解密私钥的密码。

#### `security_certificate`

默认值: `None`.

X.509 证书文件的相对或绝对路径，当使用 `message-signing` 时用于签名消息。

#### `security_cert_store`

默认值: `None`.

包含用于 `message-signing` 的 X.509 证书的目录。可以是带有通配符的 glob 模式，（例如 `/etc/certs/*.pem`）。

#### `security_digest`

默认值: `sha256`.

当使用 `message-signing` 时用于签名消息的加密摘要。
https://cryptography.io/en/latest/hazmat/primitives/cryptographic-hashes/#module-cryptography.hazmat.primitives.hashes

### 自定义组件类（高级）

#### `worker_pool`

Default: `"prefork"` (`celery.concurrency.prefork:TaskPool`).

工作进程使用的池类名称。

!!! tip "Eventlet/Gevent"

    切勿使用此选项来选择 eventlet 或 gevent 池。您必须使用 `celery worker -P` 选项来 `celery worker`，以确保猴子补丁不会应用得太晚，导致以奇怪的方式出现问题。

#### `worker_pool_restarts`

Default: 默认禁用。

如果启用，可以使用 `pool_restart` 远程控制命令重新启动工作进程池。

#### `worker_autoscaler`

Default: `"celery.worker.autoscale:Autoscaler"`.

要使用的自动缩放器类名称。

#### `worker_consumer`

Default: `"celery.worker.consumer:Consumer"`.

工作进程使用的消费者类名称。

#### `worker_timer`

Default: `"kombu.asynchronous.hub.timer:Timer"`.

工作进程使用的 ETA 调度器类名称。默认由池实现设置。

#### `worker_logfile`

Default: `None`

`celery worker` 的可选文件路径，用于记录日志（默认为 `stdout`）。

#### `worker_pidfile`

Default: `None`

:`celery worker` 的可选文件路径，用于创建/存储其 PID 文件（默认为不创建 PID 文件）。

#### `worker_uid`

Default: `None`

当 `celery worker` 守护进程降低其权限时使用的可选用户 ID（默认为不更改 UID）。

#### `worker_gid`

Default: `None`

当 `celery worker` 守护进程降低其权限时使用的可选组 ID（默认为不更改 GID）。

#### `worker_umask`

Default: `None`

当 `celery worker` 守护进程创建文件（日志、pid 等）时使用的可选 `umask`。

#### `worker_executable`

Default: `None`

:`celery worker` 在守护进程化时使用的可选 `python` 可执行文件路径（默认为 :data:`sys.executable`）。

### Beat Settings (`celery beat`)

#### `beat_schedule`

默认值：`{}`（空映射）。

由 `celery.bin.beat` 使用的周期性任务计划。

#### `beat_scheduler`

默认值：`"celery.beat:PersistentScheduler"`。

默认的调度器类。例如，如果与 `django-celery-beat` 扩展一起使用，可以设置为 `"django_celery_beat.schedulers:DatabaseScheduler"`。

也可以通过 `celery beat -S` 参数设置。

#### `beat_schedule_filename`

默认值：`"celerybeat-schedule"`。

`PersistentScheduler` 用于存储周期性任务最后运行时间的文件名。可以是相对路径或绝对路径，但要注意后缀 `.db` 可能会附加到文件名上（取决于 Python 版本）。

也可以通过 `celery beat --schedule` 参数设置。

#### `beat_sync_every`

默认值：0。

在发出另一个数据库同步之前可以调用的周期性任务数量。值为 0（默认）表示基于时间的同步 - 由 scheduler.sync_every 确定的默认值为 3 分钟。如果设置为 1，beat 将在每个任务消息发送后调用同步。

#### `beat_max_loop_interval`

默认值：0。

`celery.bin.beat` 在检查计划之间可以睡眠的最大秒数。

此值的默认值取决于调度器。对于默认的 Celery beat 调度器，该值为 300（5 分钟），但对于 `django-celery-beat` 数据库调度器，它是 5 秒，因为计划可能会在外部更改，因此必须考虑对计划的更改。

此外，当在 Jython 上作为线程运行嵌入式 Celery beat（`celery worker -B`）时，最大间隔会被覆盖并设置为 1，以便能够及时关闭。

#### `beat_cron_starting_deadline`

默认值：None。

使用 cron 时，`celery.bin.beat` 在决定 cron 计划是否到期时可以回溯的秒数。当设置为 `None` 时，过期的 cron 作业将始终立即运行。

!!! warning

    强烈不建议将此值设置为高于 3600（1 小时）。

#### `beat_logfile`

默认值：`None`

`celery beat` 用于记录日志的可选文件路径（默认为 `stdout`）。

#### `beat_pidfile`

默认值：`None`

`celery beat` 用于创建/存储其 PID 文件的可选文件路径（默认为不创建 PID 文件）。

#### `beat_uid`

默认值：`None`

当 `celery beat` 降低其权限时使用的可选用户 ID（默认为不更改 UID）。

#### `beat_gid`

默认值：`None`

当 `celery beat` 守护进程降低其权限时使用的可选组 ID（默认为不更改 GID）。

#### `beat_umask`

默认值：`None`

当 `celery beat` 在守护进程化时创建文件（日志、pid 等）时使用的可选 `umask`。

#### `beat_executable`

默认值：`None`

`celery beat` 在守护进程化时使用的可选 `python` 可执行文件路径（默认为 `sys.executable`）。
