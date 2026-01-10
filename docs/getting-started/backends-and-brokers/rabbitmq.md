---
description: Celery与RabbitMQ集成指南：详细讲解RabbitMQ作为默认代理的安装配置方法，包括macOS系统设置、Quorum队列配置、原生延迟交付功能实现，以及ETA和倒计时任务的正确调度方式。
---

# 使用 RabbitMQ

## 安装与配置

RabbitMQ 是默认的代理，因此除了您想要使用的代理实例的 URL 位置外，不需要任何额外的依赖项或初始配置：

```python
broker_url = 'amqp://myuser:mypassword@localhost:5672/myvhost'
```

有关代理 URL 的描述以及 Celery 可用的各种代理配置选项的完整列表，请参阅 [`conf-broker-settings`](../../user-guide/configuration.md#conf-broker-settings)，并参阅下文设置用户名、密码和虚拟主机。

## 安装 RabbitMQ Server

请参阅 RabbitMQ 网站上的 [Installing RabbitMQ](https://www.rabbitmq.com/docs/download){target="_blank"}。

### 设置 RabbitMQ

要使用 Celery，我们需要创建一个 RabbitMQ 用户、一个虚拟主机，并允许该用户访问该虚拟主机：

```console
sudo rabbitmqctl add_user myuser mypassword

sudo rabbitmqctl add_vhost myvhost

sudo rabbitmqctl set_user_tags myuser mytag

sudo rabbitmqctl set_permissions -p myvhost myuser ".*" ".*" ".*"
```

将上述 `myuser`、`mypassword` 和 `myvhost` 替换为适当的值。

有关 [access control](https://www.rabbitmq.com/docs/access-control){target="_blank"} 的更多信息，请参阅 RabbitMQ [Authentication, Authorisation, Access Control](https://www.rabbitmq.com/docs/access-control){target="_blank"}。

### 在 macOS 上安装 RabbitMQ

在 macOS 上安装 RabbitMQ 最简单的方法是使用 [Homebrew](https://brew.sh/zh-cn/){target="_blank"}。

```console
brew install rabbitmq
```

使用 `brew` 安装 RabbitMQ 后，您需要将以下内容添加到您的路径中，以便能够启动和停止代理：将其添加到您的 shell 启动文件中（例如 `.bash_profile` 或 `.profile`）。

```shell
PATH=$PATH:/usr/local/sbin
```

#### 配置系统主机名

如果您使用的 DHCP 服务器给您一个随机的主机名，您需要永久配置主机名。这是因为 RabbitMQ 使用主机名与节点通信。

使用 `scutil` 命令永久设置您的主机名：

```console
sudo scutil --set HostName myhost.local
```

然后将该主机名添加到 `/etc/hosts` 中，以便可以将其解析回 IP 地址：

```text
127.0.0.1       localhost myhost myhost.local
```

如果您启动 `rabbitmq-server`，您的 rabbit 节点现在应该是 `rabbit@myhost`，可以通过 `rabbitmqctl` 验证：

```console
sudo rabbitmqctl status
Status of node rabbit@myhost ...
[{running_applications,[{rabbit,"RabbitMQ","1.7.1"},
                    {mnesia,"MNESIA  CXC 138 12","4.4.12"},
                    {os_mon,"CPO  CXC 138 46","2.2.4"},
                    {sasl,"SASL  CXC 138 11","2.1.8"},
                    {stdlib,"ERTS  CXC 138 10","1.16.4"},
                    {kernel,"ERTS  CXC 138 10","2.13.4"}]},
{nodes,[rabbit@myhost]},
{running_nodes,[rabbit@myhost]}]
...done.
```

如果您的 DHCP 服务器给您一个以 IP 地址开头的主机名（例如 `23.10.112.31.comcast.net`），这一点尤其重要。在这种情况下，RabbitMQ 将尝试使用 `rabbit@23`：一个非法的主机名。

#### 启动/停止 RabbitMQ 服务器

要启动服务器：

```console
sudo rabbitmq-server
```

您也可以通过添加 `-detached` 选项在后台运行它（注意：只有一个破折号）：

```console
sudo rabbitmq-server -detached
```

切勿使用 `kill` (`kill(1)`) 停止 RabbitMQ 服务器，而是使用 `rabbitmqctl` 命令：

```console
sudo rabbitmqctl stop
```

## 使用 Quorum 队列

!!! warning

    Quorum 队列需要禁用全局 QoS，这意味着某些功能将无法按预期工作。
    有关详细信息，请参阅 `limitations`。

Celery 支持 `Quorum Queues`，通过将 `x-queue-type` 标头设置为 `quorum`，如下所示：

```python
from kombu import Queue

task_queues = [Queue('my-queue', queue_arguments={'x-queue-type': 'quorum'})]
broker_transport_options = {"confirm_publish": True}
```

如果您想更改默认队列的类型，请将 [`task_default_queue_type`](../../user-guide/configuration.md#task_default_queue_type) 设置为 `"quorum"`。

配置 [Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues){target="_blank"} 的另一种方法是依赖默认设置并使用 `task_routes`：

```python
task_default_queue_type = "quorum"
task_default_exchange_type = "topic"
task_default_queue = "my-queue"
broker_transport_options = {"confirm_publish": True}

task_routes = {
    "*": {
        "routing_key": "my-queue",
    },
}
```

Celery 使用 [`worker_detect_quorum_queues`](../../user-guide/configuration.md#worker_detect_quorum_queues) 设置自动检测是否使用了 quorum 队列。我们建议保持默认行为开启。

要从经典镜像队列迁移到 quorum 队列，请参阅 [Migrating from Mirrored Classic Queues to Quorum Queues](https://www.rabbitmq.com/blog/2023/03/02/quorum-queues-migration){target="_blank"}。

### 限制

禁用全局 QoS 意味着每通道 QoS 现在是静态的。这意味着在使用 Quorum 队列时，某些 Celery 功能将无法工作。

自动缩放依赖于在实例化或终止新进程时增加和减少预取计数，因此在检测到 Quorum 队列时它将无法工作。

类似地，[`worker_enable_prefetch_count_reduction`](../../user-guide/configuration.md#worker_enable_prefetch_count_reduction) 设置在检测到 Quorum 队列时即使设置为 `True` 也将无效。

此外，[`ETA 和倒计时`](../../user-guide/calling.md#calling-eta) 在接收到时会阻塞工作进程，直到 ETA 到达，因为我们无法再增加预取计数并从队列中获取另一个任务。

为了正确调度 ETA 和倒计时 任务，我们自动检测是否使用了 quorum 队列，如果使用了，Celery 会自动启用 [原生延迟交付](#native-delayed-delivery)。

### 原生延迟交付 {#native-delayed-delivery}

由于带有 ETA 和倒计时 的任务会阻塞工作进程，直到它们被调度执行，
我们需要使用 RabbitMQ 的原生功能来调度任务的执行。

该设计借鉴自 NServiceBus。如果您对实现细节感兴趣，请参阅 [RabbitMQ Delayed Delivery](https://docs.particular.net/transports/rabbitmq/delayed-delivery){target="_blank"}。

原生延迟交付在检测到 quorum 队列时自动启用。

默认情况下，原生延迟交付队列是 quorum 队列。如果您想将它们更改为经典队列，可以将 [`broker_native_delayed_delivery_queue_type`](../../user-guide/configuration.md#broker_native_delayed_delivery_queue_type) 设置为 classic。
