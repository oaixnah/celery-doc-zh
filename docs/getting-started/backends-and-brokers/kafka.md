---
description: Celery Kafka配置指南：详细说明如何在Celery中使用Kafka作为消息代理，包括SASL认证配置、任务队列路由机制、canvas操作支持以及单工作进程限制的完整配置教程。
---

# 使用 Kafka

## 配置

```python title="celeryconfig.py"
import os

task_serializer = 'json'
broker_transport_options = {
    # "allow_create_topics": True,
}
broker_connection_retry_on_startup = True

# 对于使用 SQLAlchemy 作为后端
# result_backend = 'db+postgresql://postgres:example@localhost/postgres'

broker_transport_options.update({
    "security_protocol": "SASL_SSL",
    "sasl_mechanism": "SCRAM-SHA-512",
})
sasl_username = os.environ["SASL_USERNAME"]
sasl_password = os.environ["SASL_PASSWORD"]
broker_url = f"confluentkafka://{sasl_username}:{sasl_password}@broker:9094"
broker_transport_options.update({
    "kafka_admin_config": {
        "sasl.username": sasl_username,
        "sasl.password": sasl_password,
    },
    "kafka_common_config": {
        "sasl.username": sasl_username,
        "sasl.password": sasl_password,
        "security.protocol": "SASL_SSL",
        "sasl.mechanism": "SCRAM-SHA-512",
        "bootstrap_servers": "broker:9094",
    }
})
```
    
请注意，如果主题尚不存在，则需要 "allow_create_topics"，否则不需要。

```python title="tasks.py"
from celery import Celery

app = Celery('tasks')
app.config_from_object('celeryconfig')


@app.task
def add(x, y):
    return x + y
```

## 认证

参见上文。SASL 用户名和密码通过环境变量传递。

## 更多信息

Celery 队列会被路由到 Kafka 主题。例如，如果一个队列名为 "add_queue"，那么将在 Kafka 中创建/使用一个名为 "add_queue" 的主题。

对于 canvas，当使用支持它的后端时，典型的机制如 chain、group 和 chord 似乎可以工作。

## 限制

目前，使用 Kafka 作为代理意味着只能使用一个工作进程。参见 [Multiple celery fork pool workers don't work #1785](https://github.com/celery/kombu/issues/1785){target="_blank"} 。
