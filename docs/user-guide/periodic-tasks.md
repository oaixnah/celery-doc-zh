---
subtitle: Periodic Tasks
description: Celery周期性任务使用指南 - 详细介绍Celery Beat调度器的配置、时区设置、任务调度类型（包括定时、crontab和太阳事件调度）以及如何启动和管理周期性任务。
---

# 周期性任务

## 介绍

`celery beat` 是一个调度器；它按固定的时间间隔启动任务，然后由集群中可用的工作节点执行这些任务。

默认情况下，条目取自 `beat_schedule` 设置，但也可以使用自定义存储，例如将条目存储在 SQL 数据库中。

您必须确保每个调度只有一个调度器在运行，否则会出现重复任务。使用集中式方法意味着调度不需要同步，服务可以在不使用锁的情况下运行。

## 时区

周期性任务调度默认使用 UTC 时区，但您可以使用 `timezone` 设置来更改使用的时区。

一个示例时区可以是 `Europe/London`：

```python
timezone = 'Europe/London'
```

此设置必须添加到您的应用程序中，可以通过直接配置使用 (`app.conf.timezone = 'Europe/London'`)，或者通过添加到您的配置模块（如果您已使用 `app.config_from_object` 设置了一个）。
有关配置选项的更多信息，请参阅 [配置](../getting-started/first-steps-with-celery.md#celerytut-configuration){target="_blank"}。

默认调度器（将计划存储在 `celerybeat-schedule` 文件中）将自动检测到时区已更改，因此将重置计划本身，但其他调度器可能不那么智能（例如，Django 数据库调度器，见下文），在这种情况下，您必须手动重置计划。

??? tip "Django 用户"

    Celery 推荐并与 Django 1.4 中引入的 `USE_TZ` 设置兼容。

    对于 Django 用户，将使用 `TIME_ZONE` 设置中指定的时区，
    或者您可以通过使用 :setting:`timezone` 设置单独为 Celery 指定自定义时区。

    数据库调度器在时区相关设置更改时不会重置，因此您必须手动执行此操作：

    ```console
    $ python manage.py shell
    >>> from djcelery.models import PeriodicTask
    >>> PeriodicTask.objects.update(last_run_at=None)
    ```

    Django-Celery 仅支持 Celery 4.0 及以下版本，对于 Celery 4.0 及以上版本，请按以下方式操作：

    ```console
    $ python manage.py shell
    >>> from django_celery_beat.models import PeriodicTask
    >>> PeriodicTask.objects.update(last_run_at=None)
    ```

## 条目

要定期调用任务，您必须向 beat 调度列表中添加一个条目。

```python
from celery import Celery
from celery.schedules import crontab

app = Celery()

@app.on_after_configure.connect
def setup_periodic_tasks(sender: Celery, **kwargs):
    # 每10秒调用 test('hello')
    sender.add_periodic_task(10.0, test.s('hello'), name='add every 10')

    # 每30秒调用 test('hello')
    # 它使用与前一个任务相同的签名，定义了一个显式名称
    # 以避免此任务替换先前定义的任务
    sender.add_periodic_task(30.0, test.s('hello'), name='add every 30')

    # 每30秒调用 test('world')
    sender.add_periodic_task(30.0, test.s('world'), expires=10)

    # 每周一早上7:30执行
    sender.add_periodic_task(
        crontab(hour=7, minute=30, day_of_week=1),
        test.s('Happy Mondays!'),
    )

@app.task
def test(arg):
    print(arg)

@app.task
def add(x, y):
    z = x + y
    print(z)
```

在 `on_after_configure` 处理程序内设置这些意味着在使用 `test.s()` 时我们不会在模块级别评估应用程序。请注意 `on_after_configure` 是在应用设置完成后发送的，因此位于应用声明模块之外的任务（例如，在由 `celery.Celery.autodiscover_tasks` 定位的 `tasks.py` 文件中）必须使用稍后的信号，例如 `on_after_finalize`。

`add_periodic_task` 函数将在后台将条目添加到 `beat_schedule` 设置中，相同的设置也可以用于手动设置周期性任务：

示例：每30秒运行 `tasks.add` 任务。

```python
app.conf.beat_schedule = {
    'add-every-30-seconds': {
        'task': 'tasks.add',
        'schedule': 30.0,
        'args': (16, 16)
    },
}
app.conf.timezone = 'UTC'
```

!!! note

    如果您想知道这些设置应该放在哪里，请参阅 [配置](../getting-started/first-steps-with-celery.md#celerytut-configuration){target="_blank"}。您可以直接在您的应用上设置这些选项，也可以保留一个单独的配置模块。

    如果您想为 `args` 使用单元素元组，请不要忘记构造函数是一个逗号，而不是一对括号。

使用 `timedelta` 作为调度意味着任务将以30秒的间隔发送（第一个任务将在 `celery beat` 启动后30秒发送，然后在每次运行后30秒发送）。

还存在类似 Crontab 的调度，请参阅 `Crontab schedules` 部分。

与 `cron` 类似，如果第一个任务在下一个任务开始前未完成，任务可能会重叠。如果这是一个问题，您应该使用锁定策略来确保一次只能运行一个实例。

### 可用字段

<table>
<thead>
<tr>
<th>字段</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>task</code></td>
<td>
要执行的任务名称。<br/><br/>
任务名称在用户指南的 [任务名称](../user-guide/tasks.md#task-names){target="_blank"} 部分中描述。
请注意，这不是任务的导入路径，即使默认
命名模式是这样构建的。
</td>
</tr>
<tr>
<td><code>schedule</code></td>
<td>
执行频率。<br/><br/>
这可以是整数秒数、<code>timedelta</code> 或 
<code>crontab</code>。您还可以通过扩展 <code>schedule</code> 的接口来定义自己的自定义调度类型。
</td>
</tr>
<tr>
<td><code>args</code></td>
<td>
位置参数（<code>list</code> 或 <code>tuple</code>）。
</td>
</tr>
<tr>
<td><code>kwargs</code></td>
<td>
关键字参数（<code>dict</code>）。
</td>
</tr>
<tr>
<td><code>options</code></td>
<td>
执行选项（<code>dict</code>）。<br/><br/>
这可以是 <code>apply_async</code> 支持的
任何参数——`exchange`、`routing_key`、`expires` 等。
</td>
</tr>
<tr>
<td><code>relative</code></td>
<td>
如果 `relative` 为 true，<code>timedelta</code> 调度将按
"时钟"调度。这意味着频率将根据 <code>timedelta</code> 的
周期四舍五入到最接近的秒、分钟、小时或天。<br/><br/>
默认情况下 <code>relative</code> 为 false，频率不会被四舍五入，并且将
相对于 <code>celery beat</code> 启动的时间。
</td>
</tr>
</tbody>
</table>

## Crontab 调度

如果您想对任务执行时间有更多控制，例如特定的时间点或星期几，您可以使用 `crontab` 调度类型：

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    # 每周一早上7:30执行
    'add-every-monday-morning': {
        'task': 'tasks.add',
        'schedule': crontab(hour=7, minute=30, day_of_week=1),
        'args': (16, 16),
    },
}
```

这些 Crontab 表达式的语法非常灵活。

一些示例：

| 示例                                                              | 含义                                                                                |
|-----------------------------------------------------------------|-----------------------------------------------------------------------------------|
| `crontab()`                                                     | 每分钟执行。                                                                            |
| `crontab(minute=0, hour=0)`                                     | 每天午夜执行。                                                                           |
| `crontab(minute=0, hour='*/3')`                                 | 每三小时执行：午夜、凌晨3点、早上6点、上午9点、中午12点、下午3点、下午6点、晚上9点。                                    |
| `crontab(minute=0, hour='0,3,6,9,12,15,18,21')`                 | 同上。                                                                               |
| `crontab(minute='*/15')`                                        | 每15分钟执行。                                                                          |
| `crontab(day_of_week='sunday')`                                 | 每周日每分钟执行（！）。                                                                      |
| `crontab(minute='*', hour='*', day_of_week='sun')`              | 同上。                                                                               |
| `crontab(minute='*/10', hour='3,17,22', day_of_week='thu,fri')` | 每十分钟执行，但仅在周四或周五的凌晨3-4点、下午5-6点、晚上10-11点之间执行。                                       |
| `crontab(minute=0, hour='*/2,*/3')`                             | 每偶数小时执行，以及每能被三整除的小时执行。这意味着：在每个小时执行，*除了*：凌晨1点、凌晨5点、早上7点、上午11点、下午1点、下午5点、晚上7点、晚上11点 |
| `crontab(minute=0, hour='*/5')`                                 | 每能被五整除的小时执行。这意味着它在下午3点触发，而不是下午5点（因为下午3点等于24小时制中的"15"，可以被5整除）。                     |
| `crontab(minute=0, hour='*/3,8-17')`                            | 每能被三整除的小时执行，以及在办公时间（上午8点-下午5点）的每个小时执行。                                            |
| `crontab(0, 0, day_of_month='2')`                               | 每月的第二天执行。                                                                         |
| `crontab(0, 0, day_of_month='2-30/2')`                          | 在每个偶数日执行。                                                                         |
| `crontab(0, 0, day_of_month='1-7,15-21')`                       | 在每月的第一周和第三周执行。                                                                    |
| `crontab(0, 0, day_of_month='11', month_of_year='5')`           | 每年5月11日执行。                                                                        |
| `crontab(0, 0, month_of_year='*/3')`                            | 在每个季度的第一个月的每一天执行。                                                                 |

有关更多文档，请参阅 `crontab`。

## Solar 调度

如果你有一个任务需要根据日出、日落、黎明或黄昏来执行，你可以使用 `solar` 调度类型：

```python
from celery.schedules import solar

app.conf.beat_schedule = {
    # 在墨尔本日落时执行
    'add-at-melbourne-sunset': {
        'task': 'tasks.add',
        'schedule': solar('sunset', -37.81753, 144.96715),
        'args': (16, 16),
    },
}
```

参数很简单：`solar(event, latitude, longitude)`

请确保使用正确的纬度和经度符号：

| 符号   | 参数          | 含义      |
|------|-------------|---------|
| `+`  | `latitude`  | 北纬      |
| `-`  | `latitude`  | 南纬      |
| `+`  | `longitude` | 东经      |
| `-`  | `longitude` | 西经      |

可能的事件类型有：

| 事件                   | 含义                                                 |
|----------------------|----------------------------------------------------|
| `dawn_astronomical`  | 在天色不再完全黑暗的时刻执行。此时太阳位于地平线以下18度。                     |
| `dawn_nautical`      | 在有足够阳光使地平线和某些物体可区分时执行；正式来说，当太阳位于地平线以下12度时。         |
| `dawn_civil`         | 在有足够光线使物体可区分，从而可以开始户外活动时执行；正式来说，当太阳位于地平线以下6度时。     |
| `sunrise`            | 在早晨太阳的上边缘出现在东方地平线上时执行。                             |
| `solar_noon`         | 在太阳当天达到地平线最高点时执行。                                  |
| `sunset`             | 在傍晚太阳的后边缘消失在西方地平线上时执行。                             |
| `dusk_civil`         | 在民用黄昏结束时执行，此时物体仍然可区分，一些星星和行星可见。正式来说，当太阳位于地平线以下6度时。 |
| `dusk_nautical`      | 在太阳位于地平线以下12度时执行。物体不再可区分，地平线也不再肉眼可见。               |
| `dusk_astronomical`  | 在天色完全变暗的时刻之后执行；正式来说，当太阳位于地平线以下18度时。                |

所有太阳事件都使用UTC时间计算，因此不受您时区设置的影响。

在极地地区，太阳可能不会每天升起或落下。调度器能够处理这些情况（例如，在太阳不升起的日子，`sunrise` 事件不会运行）。唯一的例外是 `solar_noon`，它被正式定义为太阳穿过天球子午线的时刻，即使太阳在地平线以下，它也会每天发生。

黄昏被定义为黎明和日出之间的时期；以及日落和黄昏之间的时期。您可以根据您对黄昏的定义（民用、航海或天文）以及您希望事件在黄昏开始还是结束时发生，使用上面列表中的适当事件来安排"黄昏"相关的事件。

有关更多文档，请参阅 `solar`。

## 启动调度器 {#starting-the-scheduler}

要启动 `celery beat` 服务：

```console
celery -A proj beat
```

你也可以通过启用工作器的 `celery worker -B` 选项将 `beat` 嵌入到工作器中，如果你永远不会运行超过一个工作节点，这很方便，但不常用，因此不推荐在生产环境中使用：

```console
celery -A proj worker -B
```

Beat 需要将任务的最后运行时间存储在本地数据库文件中（默认名为 `celerybeat-schedule`），因此它需要访问当前目录的写入权限，或者你可以为此文件指定自定义位置：

```console
celery -A proj beat -s /home/celery/var/run/celerybeat-schedule
```

!!! note

    要将 beat 守护进程化，请参阅 [守护进程指南](daemonizing.md)。

### 使用自定义调度器类

自定义调度器类可以在命令行上指定（使用 `celery beat --scheduler` 参数）。

默认调度器是 `celery.beat.PersistentScheduler`，它只是在本地 `shelve` 数据库文件中跟踪最后运行时间。

还有 `django-celery-beat` 扩展，它将计划存储在 Django 数据库中，并提供了一个方便的管理界面来在运行时管理周期性任务。

要安装和使用此扩展：

1. 使用 `pip` 安装包：

    ```console
    pip install django-celery-beat
    ```

2. 将 `django_celery_beat` 模块添加到 Django 项目的 `settings.py` 中的 `INSTALLED_APPS`：：

    ```python
    INSTALLED_APPS = (
        ...,
        'django_celery_beat',
    )
    ```   

    注意模块名称中没有破折号，只有下划线。

3. 应用 Django 数据库迁移以创建必要的表：

    ```console
    python manage.py migrate
    ```

4. 使用 `django_celery_beat.schedulers:DatabaseScheduler` 调度器启动 `celery beat` 服务：

    ```console
    celery -A proj beat -l INFO --scheduler django_celery_beat.schedulers:DatabaseScheduler
    ```

    注意：你也可以直接将其添加为 `beat_scheduler` 设置。

5. 访问 Django-Admin 界面设置一些周期性任务。
