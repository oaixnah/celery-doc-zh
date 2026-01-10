---
description: Celery与Amazon SQS集成指南：安装配置、区域设置、可见性超时、轮询间隔、长轮询、队列前缀、预定义队列、退避策略、STS令牌认证及注意事项，帮助您高效使用SQS作为Celery消息代理。
---

# 使用 Amazon SQS

## 安装 {#broker-sqs-installation}

要支持 Amazon SQS，您需要安装额外的依赖项。
您可以使用 `celery[sqs]` 一次性安装 Celery 和这些依赖项：

=== "uv"

    ```console
    uv add "celery[sqs]"
    ```

=== "pip"

    ```console
    pip install "celery[sqs]"
    ```

## 配置 {#broker-sqs-configuration}

您必须在代理 URL 中指定 SQS:

```python
broker_url = 'sqs://ABCDEFGHIJKLMNOPQRST:ZYXK7NiynGlTogH8Nj+P9nlE73sq3@'
```

其中 URL 格式为：

```text
sqs://aws_access_key_id:aws_secret_access_key@
```

请注意，您必须记住在末尾包含 `@` 符号，并对密码进行编码，以便始终能够正确解析。例如：

```python
from kombu.utils.url import safequote

aws_access_key = safequote("ABCDEFGHIJKLMNOPQRST")
aws_secret_key = safequote("ZYXK7NiynG/TogH8Nj+P9nlE73sq3")

broker_url = "sqs://{aws_access_key}:{aws_secret_key}@".format(
    aws_access_key=aws_access_key, aws_secret_key=aws_secret_key,
)
```

!!! warning

    不要将此设置选项与 django 的 `debug=True` 一起使用。这可能导致已部署的 django 应用程序出现安全问题。
    
    在调试模式下，django 会显示环境变量，SQS URL 可能会暴露给互联网，包括您的 AWS 访问密钥和密钥。请在已部署的 django 应用程序上关闭调试模式，或考虑使用下面描述的设置选项。


登录凭据也可以使用环境变量 `AWS_ACCESS_KEY_ID` 和 `AWS_SECRET_ACCESS_KEY` 设置，在这种情况下，代理 URL 可能仅为 `sqs://`。

如果您在实例上使用 IAM 角色，可以将 BROKER_URL 设置为：`sqs://`，kombu 将尝试从实例元数据中检索访问令牌。

## 选项

### 区域

默认区域是 `us-east-1`，但您可以通过配置 [`broker_transport_options`](../../user-guide/configuration.md#broker_transport_options) 设置来选择其他区域:

```python
broker_transport_options = {'region': 'eu-west-1'}
```

!!! note "[AWS 全球基础设施](https://aws.amazon.com/cn/about-aws/global-infrastructure/){target=_blank}"

### 可见性超时 {#sqs-visibility-timeout}

可见性超时定义了在消息重新传递给另一个工作器之前，等待工作器确认任务的秒数。另请参阅下面的注意事项。

此选项通过 [`broker_transport_options`](../../user-guide/configuration.md#broker_transport_options) 设置:

```python
broker_transport_options = {'visibility_timeout': 1800}  # 默认 30 分钟。
```

### 轮询间隔

轮询间隔决定了在轮询失败之间休眠的秒数。此值可以是整数或浮点数。默认值为 *一秒*：这意味着当没有更多消息可读取时，工作器将休眠一秒。

您必须注意 **更频繁的轮询也更昂贵，因此增加轮询间隔可以节省您的资金**。

轮询间隔可以通过 [`broker_transport_options`](../../user-guide/configuration.md#broker_transport_options) 设置:

```python
broker_transport_options = {'polling_interval': 0.3}
```

非常频繁的轮询间隔可能导致 *忙循环*，导致工作器使用大量 CPU 时间。如果您需要亚毫秒精度，应考虑使用其他传输方式，如 [Redis](redis.md){target=_blank} 或 [RabbitMQ](rabbitmq.md){target=_blank}。

### 长轮询

默认启用 [SQS 长轮询](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html#sqs-long-polling){target=_blank}，并且 [ReceiveMessage](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ReceiveMessage.html){target=_blank} 操作的 `WaitTimeSeconds` 参数设置为 10 秒。

`WaitTimeSeconds` 参数的值可以通过 [`broker_transport_options`](../../user-guide/configuration.md#broker_transport_options) 设置:

```python
broker_transport_options = {'wait_time_seconds': 15}
```

有效值为 0 到 20。请注意，新创建的队列本身（即使由 Celery 创建）也将为 "接收消息等待时间" 队列属性设置默认值 0。

### 队列前缀

默认情况下，Celery 不会为队列名称分配任何前缀，如果您有其他使用 SQS 的服务，可以通过 [`broker_transport_options`](../../user-guide/configuration.md#broker_transport_options) 设置进行配置:

```python
broker_transport_options = {'queue_name_prefix': 'celery-'}
```

### 预定义队列 {#predefined-queues}

如果您希望 Celery 使用 AWS 中的一组预定义队列，并且从不尝试列出 SQS 队列，也不尝试创建或删除它们，请使用 [预定义队列](#predefined_queues) 设置传递队列名称到 URL 的映射:

```python
broker_transport_options = {
    'predefined_queues': {
        'my-q': {
            'url': 'https://ap-southeast-2.queue.amazonaws.com/123456/my-q',
            'access_key_id': 'xxx',
            'secret_access_key': 'xxx',
        }
    }
}
```

!!! warning

    **重要：** 使用 预定义队列 时，请勿对 `access_key_id` 和 `secret_access_key` 值使用 URL 编码的凭据（`safequote`）。
    URL 编码应仅应用于代理 URL 中的凭据。
    
    在 预定义队列 中使用 URL 编码的凭据将导致签名不匹配错误，例如："我们计算的请求签名与您提供的签名不匹配。"

**结合代理 URL 和预定义队列的正确示例：**

```python
import os
from kombu.utils.url import safequote
from celery import Celery

# 来自环境的原始凭据
AWS_ACCESS_KEY_ID = os.getenv("AWS_ACCESS_KEY_ID")
AWS_SECRET_ACCESS_KEY = os.getenv("AWS_SECRET_ACCESS_KEY")

# 仅对代理 URL 进行 URL 编码
aws_access_key_encoded = safequote(AWS_ACCESS_KEY_ID)
aws_secret_key_encoded = safequote(AWS_SECRET_ACCESS_KEY)

# 在代理 URL 中使用编码的凭据
broker_url = f"sqs://{aws_access_key_encoded}:{aws_secret_key_encoded}@"

celery_app = Celery("tasks", broker=broker_url)
celery_app.conf.broker_transport_options = {
    "region": "us-east-1",
    "predefined_queues": {
        "my-queue": {
            "url": "https://sqs.us-east-1.amazonaws.com/123456/my-queue",
            # 在此处使用原始凭据（非编码）
            "access_key_id": AWS_ACCESS_KEY_ID,
            "secret_access_key": AWS_SECRET_ACCESS_KEY,
        },
    },
}
```

使用此选项时，可见性超时应在 SQS 队列中（在 AWS 中）设置，而不是通过 [可见性超时](#sqs-visibility-timeout) 选项设置。

### 退避策略

退避策略使用 SQS 可见性超时机制来改变任务重试之间的时间差。该机制将消息特定的 可见性超时 从队列的 `Default visibility timeout` 更改为策略配置的超时时间。重试次数由 SQS 管理（具体通过 `ApproximateReceiveCount` 消息属性），用户无需进一步操作。

配置队列和退避策略:

```python
broker_transport_options = {
    'predefined_queues': {
        'my-q': {
            'url': 'https://ap-southeast-2.queue.amazonaws.com/123456/my-q',
            'access_key_id': 'xxx',
            'secret_access_key': 'xxx',
            'backoff_policy': {1: 10, 2: 20, 3: 40, 4: 80, 5: 320, 6: 640},
            'backoff_tasks': ['svc.tasks.tasks.task1']
        }
    }
}
```

`backoff_policy` 字典，其中键是重试次数，值是重试之间的延迟秒数（即 SQS 可见性超时）
`backoff_tasks` 应用上述策略的任务名称列表

上述策略：

| **尝试次数**  | **延迟** |
|-----------|--------|
| `第 2 次尝试` | 20 秒   |
| `第 3 次尝试` | 40 秒   |
| `第 4 次尝试` | 80 秒   |
| `第 5 次尝试` | 320 秒  |
| `第 6 次尝试` | 640 秒  |

### STS 令牌认证

https://docs.aws.amazon.com/cli/latest/reference/sts/assume-role.html

通过使用 `sts_role_arn` 和 `sts_token_timeout` 代理传输选项支持 AWS STS 认证。`sts_role_arn` 是我们用于授权访问 SQS 的假定 IAM 角色 ARN。`sts_token_timeout` 是令牌超时时间，默认为 900 秒（最小值）。在指定时间段后，将创建新令牌:

```python
broker_transport_options = {
    'predefined_queues': {
        'my-q': {
            'url': 'https://ap-southeast-2.queue.amazonaws.com/123456/my-q',
            'access_key_id': 'xxx',
            'secret_access_key': 'xxx',
            'backoff_policy': {1: 10, 2: 20, 3: 40, 4: 80, 5: 320, 6: 640},
            'backoff_tasks': ['svc.tasks.tasks.task1']
        }
    },
'sts_role_arn': 'arn:aws:iam::<xxx>:role/STSTest', # 可选
'sts_token_timeout': 900 # 可选
}
```

## 注意事项 {#sqs-caveats}

- 如果任务在 可见性超时 内未被确认，任务将被重新传递给另一个工作器并执行。

    这会导致 ETA/倒计时/重试任务出现问题，其中执行时间超过可见性超时；实际上，如果发生这种情况，
    任务将再次执行，并循环执行。
    
    因此，您必须增加可见性超时以匹配您计划使用的最长 ETA 时间。
    
    请注意，Celery 将在工作器关闭时重新传递消息，因此较长的可见性超时只会延迟在电源故障或强制终止工作器时
    "丢失"任务的重新传递。
    
    周期性任务不会受到可见性超时的影响，因为这是一个与 ETA/倒计时分离的概念。
    
    截至本文撰写时，AWS 支持的最大可见性超时为 12 小时（43200 秒）:
    
    ```python
    broker_transport_options = {'visibility_timeout': 43200}
    ```

- SQS 尚不支持工作器远程控制命令。

- SQS 尚不支持事件，因此不能与 `celery events`、`celerymon` 或 Django Admin 监视器一起使用。

- 对于 FIFO 队列，在发布消息时可能需要设置额外的消息属性，如 `MessageGroupId` 和 `MessageDeduplicationId`。

    消息属性可以作为关键字参数传递给 `apply_async()`：

    ```python
    message_properties = {
        'MessageGroupId': '<YourMessageGroupId>',
        'MessageDeduplicationId': '<YourMessageDeduplicationId>'
    }
    task.apply_async(**message_properties)
    ```

- 在 [停止 worker 进程](../../user-guide/workers.md#worker-stopping) 期间，工作器将尝试重新排队任何未确认的消息（启用 `task_acks_late`）。
    但是，如果工作器被强制终止（[`冷关闭`](../../user-guide/workers.md#worker-cold-shutdown)），工作器可能无法及时重新排队任务，
    并且它们将不会被再次消费，直到 :ref:`sqs-visibility-timeout` 过去。当 :ref:`sqs-visibility-timeout` 非常高且
    工作器在接收到任务后需要关闭时，这会产生问题。如果在这种情况下任务没有被重新排队，它将需要等待长的可见性超时
    过去才能再次被消费，导致任务执行可能非常长的延迟。

    [`软关闭`](../../user-guide/workers.md#worker-soft-shutdown) 在 [`冷关闭`](../../user-guide/workers.md#worker-cold-shutdown) 之前引入了一个有时间限制的温关闭阶段。
    这个时间窗口显著增加了在关闭期间重新排队任务的机会，从而缓解了长可见性超时的问题。

    要启用 [`软关闭`](../../user-guide/workers.md#worker-soft-shutdown)，请将 [`worker_soft_shutdown_timeout`](../../user-guide/configuration.md#worker_soft_shutdown_timeout) 设置为大于 0 的值。
    该值必须是一个描述秒数的浮点数。在此期间，工作器将继续处理正在运行的任务，直到超时到期，
    之后将自动启动 [`冷关闭`](../../user-guide/workers.md#worker-cold-shutdown) 以优雅地终止工作器。

    如果 **SIGTERM** 在环境变量中配置为 **SIGQUIT**，并且设置了 [`worker_soft_shutdown_timeout`](../../user-guide/configuration.md#worker_soft_shutdown_timeout)，
    工作器将在接收到 `TERM` 信号（和 `QUIT` 信号）时启动 [`软关闭`](../../user-guide/workers.md#worker-soft-shutdown)。

### 结果 {#sqs-results-configuration}

Amazon Web Services 系列中的多个产品可能是存储或发布结果的良好候选者，但目前没有包含这样的结果后端。

!!! warning "注意"

    不要将 ``amqp`` 结果后端与 SQS 一起使用。

    它将为每个任务创建一个队列，并且队列不会被收集。这可能会花费您的资金，而这些资金最好用于为 Celery 贡献一个 AWS 结果存储后端。