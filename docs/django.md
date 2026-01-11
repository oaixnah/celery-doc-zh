---
hide:
  - navigation
description: Celery与Django集成指南：从基础配置到高级功能，包括@shared_task装饰器使用、数据库事务处理、连接池管理以及django-celery-results和django-celery-beat扩展的详细配置说明。
---

# 与 Django 一起使用

## 在 Django 中使用 Celery

!!! note

    Celery 的早期版本需要一个单独的库来与 Django 一起工作，但从 3.1 版本开始就不再是这样了。Django 现在已得到开箱即用的支持，因此本文档仅包含集成 Celery 和 Django 的基本方法。您将使用与非 Django 用户相同的 API，因此建议您先阅读 :ref:`first-steps` 教程，然后再回到本教程。

!!! note

    Celery 5.5.x 支持 Django 2.2 LTS 或更新版本。如果您的 Django 版本低于 2.2，请使用 Celery 5.2.x；如果您的 Django 版本低于 1.11，请使用 Celery 4.4.x。

要在 Django 项目中使用 Celery，您必须首先定义 Celery 库的一个实例（称为 "app"）

如果您有一个现代的 Django 项目布局，例如：

```console
proj
├── manage.py
└── proj
    ├── __init__.py
    ├── settings.py
    └── urls.py
```

那么推荐的方式是创建一个新的 `proj/proj/celery.py` 模块来定义 Celery 实例：

```python title="proj/proj/celery.py"
import os

from celery import Celery

# 为 'celery' 程序设置默认的 Django 设置模块。
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'proj.settings')

app = Celery('proj')

# 在这里使用字符串意味着工作进程不需要将配置对象序列化到子进程。
# - namespace='CELERY' 表示所有与 celery 相关的配置键
#   都应该有一个 `CELERY_` 前缀。
app.config_from_object('django.conf:settings', namespace='CELERY')

# 从所有已注册的 Django 应用中加载任务模块。
app.autodiscover_tasks()


@app.task(bind=True, ignore_result=True)
def debug_task(self):
    print(f'Request: {self.request!r}')
```

然后您需要在 `proj/proj/__init__.py` 模块中导入此应用。这确保了当 Django 启动时应用会被加载，以便稍后提到的 `shared_task` 装饰器会使用它：

```python title="proj/proj/__init__.py"
# 这将确保应用在 Django 启动时总是被导入，
# 以便 shared_task 会使用此应用。
from .celery import app as celery_app

__all__ = ('celery_app',)
```

请注意，此示例项目布局适用于较大的项目，对于简单的项目，您可以使用一个包含应用和任务定义的单一模块。

让我们分解第一个模块中发生的事情，首先，我们为 `celery` 命令行程序设置默认的 `DJANGO_SETTINGS_MODULE` 环境变量：

```python
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'proj.settings')
```

您不需要这一行，但它可以避免您总是向 `celery` 程序传递设置模块。它必须始终在创建应用实例之前出现，就像我们接下来做的那样：

```python
app = Celery('proj')
```

这是我们的库实例，您可以拥有多个实例，但在使用 Django 时可能没有理由这样做。

我们还将 Django 设置模块添加为 Celery 的配置源。这意味着您不必使用多个配置文件，而是可以直接从 Django 设置中配置 Celery；但如果您愿意，也可以将它们分开。

```python
app.config_from_object('django.conf:settings', namespace='CELERY')
```

大写命名空间意味着所有 [配置选项](configuration.md) 必须用大写而不是小写指定，并以 `CELERY_` 开头，例如 `task_always_eager` 设置变为 `CELERY_TASK_ALWAYS_EAGER`，而 `broker_url` 设置变为 `CELERY_BROKER_URL`。这也适用于工作进程设置，例如 `worker_concurrency` 设置变为 `CELERY_WORKER_CONCURRENCY`。

例如，Django 项目的配置文件可能包含：

```python title="settings.py"
...

# Celery 配置选项
CELERY_TIMEZONE = "Australia/Tasmania"
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 30 * 60
```

您可以直接传递设置对象，但使用字符串更好，因为这样工作进程就不需要序列化对象。`CELERY_` 命名空间也是可选的，但推荐使用（以防止与其他 Django 设置重叠）。

接下来，对于可重用应用的一个常见做法是在单独的 `tasks.py` 模块中定义所有任务，而 Celery 确实有一种自动发现这些模块的方法：

```python
app.autodiscover_tasks()
```

使用上面的代码行，Celery 将自动从所有已安装的应用中发现任务，遵循 `tasks.py` 约定：

```console
app1/
├── tasks.py
└── models.py
app2/
├── tasks.py
└── models.py
```

这样您就不必手动将各个模块添加到 `CELERY_IMPORTS <imports>` 设置中。

最后，`debug_task` 示例是一个转储其自身请求信息的任务。这使用了 Celery 3.1 中引入的新 `bind=True` 任务选项，以便轻松引用当前任务实例。

### 使用 `@shared_task` 装饰器

您编写的任务可能存在于可重用应用中，而可重用应用不能依赖于项目本身，因此您也不能直接导入您的应用实例。

`shared_task` 装饰器让您可以在没有任何具体应用实例的情况下创建任务：

```python title="demoapp/tasks.py"
# 在这里创建您的任务

from demoapp.models import Widget

from celery import shared_task


@shared_task
def add(x, y):
    return x + y


@shared_task
def mul(x, y):
    return x * y


@shared_task
def xsum(numbers):
    return sum(numbers)


@shared_task
def count_widgets():
    return Widget.objects.count()


@shared_task
def rename_widget(widget_id, name):
    w = Widget.objects.get(id=widget_id)
    w.name = name
    w.save()
```

### 在数据库事务结束时触发任务

Django 的一个常见陷阱是立即触发任务而不等待数据库事务结束，这意味着 Celery 任务可能在所有更改持久化到数据库之前运行。例如：

```python
# views.py
def create_user(request):
    # 注意：简化示例，请使用表单验证输入
    user = User.objects.create(username=request.POST['username'])
    send_email.delay(user.pk)
    return HttpResponse('User created')

# task.py
@shared_task
def send_email(user_pk):
    user = User.objects.get(pk=user_pk)
    # 发送邮件 ...
```

在这种情况下，`send_email` 任务可能在视图提交事务到数据库之前启动，因此任务可能无法找到用户。

一个常见的解决方案是使用 Django 的 [on_commit](https://docs.djangoproject.com/en/stable/topics/db/transactions/#django.db.transaction.on_commit) 钩子在事务提交后触发任务：

```diff
- send_email.delay(user.pk)
+ transaction.on_commit(lambda: send_email.delay(user.pk))
```

由于这是一个非常常见的模式，Celery 5.4 为此引入了一个方便的快捷方式，使用 `celery.contrib.django.task.DjangoTask`。您应该调用 `celery.contrib.django.task.DjangoTask.delay_on_commit` 而不是调用 `celery.app.task.Task.delay`：

```diff
- send_email.delay(user.pk)
+ send_email.delay_on_commit(user.pk)
```

此 API 负责为您将调用包装到 `on_commit`_ 钩子中。在极少数情况下，如果您想不等待就触发任务，现有的 `celery.app.task.Task.delay` API 仍然可用。

与 `delay` 方法相比，一个关键区别是 `delay_on_commit` 不会将任务 ID 返回给调用者。当您调用该方法时，任务不会发送到代理，只有在 Django 事务完成时才会发送。如果您需要任务 ID，最好坚持使用 `celery.app.task.Task.delay`。

如果您按照上面的设置步骤操作，此任务类应该会自动使用。您需要从 `celery.contrib.django.task.DjangoTask` 继承而不是 `celery.app.task.Task` 以获得此行为。

### Django 连接池

从 Django 5.1+ 开始，内置支持数据库连接池。如果您在 Django `DATABASES` 设置中启用它，Celery 将自动通过 `close_pool` 数据库后端方法处理工作进程中的连接池关闭，因为[跨进程共享连接是不可能的](https://github.com/psycopg/psycopg/issues/544#issuecomment-1500886864)

您可以在 [Django 文档](https://docs.djangoproject.com/en/dev/ref/databases/#connection-pool) 中找到有关连接池的更多信息。

## 扩展

### `django-celery-results`

使用 Django ORM/缓存作为结果后端

`django-celery-results` 扩展提供了使用 Django ORM 或 Django 缓存框架的结果后端。

要在您的项目中使用此扩展，您需要遵循以下步骤：

1. 安装 `django-celery-results` 库：

    ```console
    pip install django-celery-results
    ```

2. 在您的 Django 项目的 `settings.py` 中将 `django_celery_results` 添加到 `INSTALLED_APPS`：

    ```python
    INSTALLED_APPS = (
        ...,
        'django_celery_results',
    )
    ```

    请注意模块名称中没有破折号，只有下划线。

3. 通过执行数据库迁移来创建 Celery 数据库表：

    ```console
    python manage.py migrate django_celery_results
    ```

4. 配置 Celery 使用 `django-celery-results` 后端。

    假设您使用 Django 的 `settings.py` 来配置 Celery，添加以下设置：

    ```python
    CELERY_RESULT_BACKEND = 'django-db'
    ```

    当使用缓存后端时，您可以指定在 Django 的 CACHES 设置中定义的缓存。

    ```python
    CELERY_RESULT_BACKEND = 'django-cache'

    # 从 CACHES 设置中选择哪个缓存。
    CELERY_CACHE_BACKEND = 'default'

    # django 设置。
    CACHES = {
        'default': {
            'BACKEND': 'django.core.cache.backends.db.DatabaseCache',
            'LOCATION': 'my_cache_table',
        }
    }
    ```

### `django-celery-beat`

基于数据库的周期性任务，带有管理界面。

## 启动工作进程

对于测试和开发，能够使用 `celery worker` 管理命令启动工作进程实例非常有用，就像您使用 Django 的 `manage.py runserver` 一样：

```console
celery -A proj worker -l INFO
```

要获取可用命令行选项的完整列表，请使用帮助命令：

```console
celery --help
```
