# 使用 Redis

## 安装

要使用 Redis 支持，您需要安装额外的依赖项。
您可以使用 `celery[redis]` 一次性安装 Celery 和这些依赖项：

=== "uv"
    
    ```console
    uv add "celery[redis]"
    ```

=== "pip"
    
    ```console
    pip install -U "celery[redis]"
    ```

## 配置

配置很简单，只需配置 Redis 数据库的位置：

```python
app.conf.broker_url = 'redis://localhost:6379/0'
```

??? info "URL 格式说明"

    1. 默认格式
        
        ```text
        redis://:password@hostname:port/db_number
        ```
        
        scheme 后的所有字段都是可选的，将默认为端口 6379 上的 `localhost`，使用数据库 `0`。

    2. 使用 Redis 凭据提供程序
    
        ```text
        redis://@hostname:port/db_number?credential_provider=mymodule.myfile.myclass
        ```

    3. 使用 Unix 套接字：

        ```text
        redis+socket:///path/to/redis.sock
        ```

        使用 Unix 套接字时，可以通过向 URL 添加 `virtual_host` 参数来指定不同的数据库编号：
        
        ```text
        redis+socket:///path/to/redis.sock?virtual_host=db_number
        ```

    4. 连接 Redis Sentinel 列表：

        ```python
        app.conf.broker_url = 'sentinel://localhost:26379;sentinel://localhost:26380;sentinel://localhost:26381'
        app.conf.broker_transport_options = {'master_name': "cluster1"}
        ```
        
        可以使用 `sentinel_kwargs` 向 Sentinel 客户端传递其他选项：
        
        ```python
        app.conf.broker_transport_options = {'sentinel_kwargs': {'password': "password"}}
        ```

### 可见性超时（Visibility Timeout） {#redis-visibility_timeout}

可见性超时定义了在消息重新传递给另一个 worker 之前等待 worker 确认任务的秒数。请务必查看下面的 [注意事项](#redis-caveats)。

此选项通过 [`broker_transport_options`](../../user-guide/configuration.md#broker_transport_options){target="_blank"} 设置进行配置：

```python
app.conf.broker_transport_options = {'visibility_timeout': 3600}    # 默认 1 小时。
```

### 结果存储（Result Backend） {#redis-results-configuration}

如果您还想在 Redis 中存储任务的状态和返回值，您应该配置这些设置：

```python
app.conf.result_backend = 'redis://localhost:6379/0'
```

有关 Redis 结果后端支持的选项的完整列表，请参阅 [Redis 后端设置](../../user-guide/configuration.md#conf-redis-result-backend){target="_blank"}。

如果您使用 Sentinel，应使用 [`result_backend_transport_options`](../../user-guide/configuration.md#result_backend_transport_options){target="_blank"} 设置指定 master_name：

```python
app.conf.result_backend_transport_options = {'master_name': "my_master"}
```

#### 全局键前缀 {#redis-result-backend-global-keyprefix}

全局键前缀将添加到结果后端使用的所有键之前，这在 Redis 数据库由不同用户共享时非常有用。
默认情况下，不添加前缀。

要配置 Redis 结果后端的全局键前缀，请在 [`result_backend_transport_options`](../../user-guide/configuration.md#result_backend_transport_options){target="_blank"} 下使用 `global_keyprefix` 键：

```python
app.conf.result_backend_transport_options = {
    'global_keyprefix': 'my_prefix_'
}
```

#### 连接超时 {#redis-result-backend-timeout}

要配置 Redis 结果后端的连接超时，请在 [`result_backend_transport_options`](../../user-guide/configuration.md#result_backend_transport_options){target="_blank"} 下使用 `retry_policy` 键：

```python
app.conf.result_backend_transport_options = {
    'retry_policy': {
        'timeout': 5.0
    }
}
```

有关可能的重试策略选项，请参阅 `retry_over_time()`。

## Serverless {#redis-serverless}

Celery 支持使用远程无服务器 Redis，这可以显著降低运营开销和成本，使其成为微服务架构或最小化运营费用至关重要的环境中的有利选择。无服务器 Redis 提供了必要的功能，无需手动设置、配置和管理，因此与 Celery 提倡的自动化和可扩展性原则非常契合。

### Upstash

[Upstash](http://upstash.com/?code=celery.oaix.tech){target="_blank"} 提供无服务器 Redis 数据库服务，为希望利用无服务器架构的 Celery 用户提供无缝解决方案。Upstash 的无服务器 Redis 服务采用最终一致性模型和持久存储，通过多层存储架构实现。

与 Celery 的集成很简单，如 [Upstash 提供的示例](https://github.com/upstash/examples/tree/main/examples/using-celery){target="_blank"} 所示。

### Dragonfly

[Dragonfly](https://www.dragonflydb.io/){target="_blank"} 是一个即插即用的 Redis 替代品，可降低成本并提升性能。Dragonfly 旨在充分利用现代云硬件的功能，满足现代应用程序的数据需求，使开发人员摆脱传统内存数据存储的限制。

## 注意事项 {#redis-caveats}

### 可见性超时（Visibility Timeout）{#redis-visibility_timeout}

如果任务被确认，任务将被重新传递给另一个 worker 并执行。

这会导致 ETA/倒计时/重试 任务出现问题，其中执行时间超过可见性超时；实际上，如果发生这种情况，任务将再次执行，并循环执行。

为了解决这个问题，您可以增加可见性超时以匹配您计划使用的最长 ETA 时间。但是，这不推荐，因为它可能对可靠性产生负面影响。Celery 会在 worker 关闭时重新传递消息，因此较长的可见性超时只会延迟在电源故障或强制终止 worker 时"丢失"任务的重新传递。

Broker 不是数据库，因此如果您需要为更远的将来调度任务，基于数据库的周期性任务可能是更好的选择。周期性任务不会受到可见性超时的影响，因为这是一个与 ETA/倒计时 分开的概念。

您可以通过配置以下所有同名选项来增加此超时（需要设置所有选项）：

```python
app.conf.broker_transport_options = {'visibility_timeout': 43200}
app.conf.result_backend_transport_options = {'visibility_timeout': 43200}
app.conf.visibility_timeout = 43200
```

该值必须是一个描述秒数的整数。

注意：如果多个应用程序共享同一个 Broker，但设置不同，将使用 _最短_ 的值。这包括如果未设置该值，并且发送了默认值。

### 软关闭（Soft Shutdown）

在 [停止 worker 进程](../../user-guide/workers.md#worker-stopping) 期间，worker 将尝试重新排队任何未确认的消息（启用 [`task_acks_late`](../../user-guide/configuration.md#task_acks_late) 时）。但是，如果 worker 被强制终止 [`冷关闭`](../../user-guide/workers.md#worker-cold-shutdown)，
worker 可能无法按时重新排队任务，并且它们将不会再次被消费，直到 [可见性超时](#redis-visibility_timeout) 过去。当 [可见性超时](#redis-visibility_timeout) 非常高且 worker 在刚接收到任务后需要关闭时，这会产生问题。如果在这种情况下任务没有被重新排队，它将需要等待较长的可见性超时过去才能再次被消费，导致任务执行可能出现非常长的延迟。

[`软关闭`](../../user-guide/workers.md#worker-soft-shutdown) 在 [`冷关闭`](../../user-guide/workers.md#worker-cold-shutdown) 之前引入了一个有时间限制的暖关闭阶段。这个时间窗口显著增加了在关闭期间重新排队任务的机会，从而缓解了长可见性超时的问题。

要启用 [`软关闭`](../../user-guide/workers.md#worker-soft-shutdown)，请将 [`worker_soft_shutdown_timeout`](../../user-guide/configuration.md#worker_soft_shutdown_timeout) 设置为大于 0 的值。该值必须是一个描述秒数的浮点数。在此期间，worker 将继续处理正在运行的任务，直到超时到期，之后将自动启动 [`冷关闭`](../../user-guide/workers.md#worker-cold-shutdown) 以优雅地终止 worker。

如果 **TERM** 在环境变量中配置为 **SIGQUIT**，并且设置了 [`worker_soft_shutdown_timeout`](../../user-guide/configuration.md#worker_soft_shutdown_timeout)，则 worker 在接收到 `TERM` 信号（和 `QUIT` 信号）时将启动 [`软关闭`](../../user-guide/workers.md#worker-soft-shutdown)。

### 键驱逐（Key Eviction）

在某些情况下，Redis 可能会从数据库中驱逐键。

如果您遇到类似以下错误：

```pycon
InconsistencyError: Probably the key ('_kombu.binding.celery') has been
removed from the Redis database.
```

那么您可能希望配置 **redis-server** 不驱逐键，通过在 Redis 配置文件中设置：

- `maxmemory` 选项
- `maxmemory-policy` 选项设置为 `noeviction` 或 `allkeys-lru`

有关驱逐策略的详细信息，请参阅 [Key eviction | Docs](https://redis.io/docs/latest/develop/reference/eviction/){target="_blank"}

### 组结果排序（Group Result Ordering） {#redis-group-result-ordering}

Celery 4.4.6 及更早版本使用未排序的列表在 Redis 后端中存储组的 result 对象。这可能导致这些结果以与原始组实例化中关联任务不同的顺序返回。Celery 4.4.7 引入了一个可选行为，修复了此问题，并确保组结果按照任务定义的相同顺序返回，与其他后端的行为匹配。在 Celery 5.0 中，此行为更改为选择退出。该行为由 `result_chord_ordered` 配置选项控制，可以像这样设置：

```python
# 为运行 Celery 4.4.6 或更早版本的 worker 指定此选项无效
app.conf.result_backend_transport_options = {
    'result_chord_ordered': True    # 或 False
}
```

这是共享相同 Redis 后端进行结果存储的 worker 运行时行为的不兼容更改，因此所有 worker 必须遵循新行为或旧行为以避免中断。对于运行 Celery 4.4.6 或更早版本的一些 worker 的集群，这意味着运行 4.4.7 的 worker 不需要特殊配置，而运行 5.0 或更高版本的 worker 必须将 `result_chord_ordered` 设置为 `False`。对于没有运行 4.4.6 或更早版本但有一些运行 4.4.7 的 worker 的集群，建议将所有 worker 的 `result_chord_ordered` 设置为 `True`，以便于将来的迁移。行为之间的迁移将破坏当前保存在 Redis 后端中的结果，如果迁移的 worker 运行下游任务，将导致中断 - 请相应地进行规划。
