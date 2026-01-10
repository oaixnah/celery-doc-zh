---
description: Celery与Google Pub/Sub集成指南：详细说明如何配置和使用Google Cloud Pub/Sub作为Celery的消息代理，包括安装依赖、URL配置、资源过期设置、确认截止时间、轮询间隔调整以及使用注意事项。
---

# 使用 Google Pub/Sub

## 安装

要支持 Google Pub/Sub，您需要安装额外的依赖项。您可以使用 `celery[gcpubsub]` 一次性安装 Celery 和这些依赖项：

=== "uv"

    ```console
    uv add "celery[gcpubsub]"
    ```

=== "pip"

    ```console
    pip install "celery[gcpubsub]"
    ```

## 配置

您必须在代理 URL 中指定 gcpubsub 和 Google 项目：

```python
broker_url = 'gcpubsub://projects/project-id'
```

URL 格式为：

```text
gcpubsub://projects/project-id
```

请注意，您必须在 URL 中为 project-id 添加 `projects/` 前缀。

登录凭据将是您在环境中设置的常规 GCP 凭据。

## 选项

### 资源过期

默认设置旨在尽可能简单、经济高效且直观，以便"开箱即用"。Pub/Sub 消息和订阅设置为 24 小时后过期，可以通过配置 `expiration_seconds` 设置来调整：

```python
expiration_seconds = 86400
```

!!! note "[Google Cloud Pub/Sub 设置的概述](https://cloud.google.com/pubsub/docs){target=_blank}"

### 确认截止时间

`ack_deadline_seconds` 定义了 Pub/Sub 基础设施在将消息重新传递给另一个工作器之前，应等待工作器确认任务的秒数。

此选项通过 [`broker_transport_options`](../../user-guide/configuration.md#broker_transport_options) 设置进行配置：

```python
broker_transport_options = {'ack_deadline_seconds': 60}  # 1 分钟。
```

默认的可见性超时为 240 秒，工作器会自动延长其所有待处理消息的确认时间。

!!! note "[Pub/Sub 截止时间的概述](https://cloud.google.com/pubsub/docs/lease-management){target=_blank}"

### 轮询间隔

轮询间隔决定了在轮询失败之间休眠的秒数。此值可以是整数或浮点数。默认值为 *0.1 秒*。但这并不意味着当没有更多消息可读时，工作器会每 0.1 秒轰炸 Pub/Sub API，因为它会被对 Pub/Sub API 的阻塞调用所阻塞，该调用只会在有新消息可读或 10 秒后返回。

轮询间隔可以通过 [`broker_transport_options`](../../user-guide/configuration.md#broker_transport_options) 设置进行配置：

```python
broker_transport_options = {'polling_interval': 0.3}
```

非常频繁的轮询间隔可能导致*忙循环*，导致工作器使用大量 CPU 时间。如果您需要亚毫秒级的精度，应考虑使用其他传输方式，如 [RabbitMQ](rabbitmq.md){target=_blank} 或 [Redis](redis.md){target=_blank}。

### 队列前缀

默认情况下，Celery 会为队列名称分配 `kombu-` 前缀。如果您有其他使用 Pub/Sub 的服务，可以通过 [`broker_transport_options`](../../user-guide/configuration.md#broker_transport_options) 设置进行配置：

```python
broker_transport_options = {'queue_name_prefix': 'kombu-'}
```

### 结果

Google Cloud Storage (GCS) 可能是存储结果的一个不错的选择。。

## 注意事项

- 使用 celery flower 时，需要 --inspect-timeout=10 选项才能正确检测工作器状态。

- GCP 订阅的空闲订阅（没有排队的消息）配置为 24 小时后删除。
  这旨在降低成本。

- 排队和未确认的消息设置为 24 小时后自动清理。
  原因同上。

- 通道队列大小是近似值，可能不准确。
  原因是 Pub/Sub API 不提供获取订阅中确切消息数量的方法。

- 孤儿（无订阅）Pub/Sub 主题不会被自动删除！！
  由于 GCP 对每个项目引入了 10k 个主题的硬限制，
  建议定期手动删除孤儿主题。

- 最大消息大小限制为 10MB，作为解决方法，您可以使用 GCS 后端
  将消息存储在 GCS 中，并将 GCS URL 传递给任务。