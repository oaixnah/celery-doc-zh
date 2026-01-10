---
subtitle: Canvas
description: Celery画布：设计工作流指南 - 详细介绍Celery任务签名、原语和工作流设计，包括group并行任务、chain任务链、chord带回调的组等高级功能，帮助开发者构建复杂分布式任务系统。
---

# 画布：设计工作流

## 签名

您刚刚在 [调用指南](calling.md){target="_blank"} 指南中学习了如何使用任务的 `delay()` 方法来调用任务，这通常就是您所需要的全部，但有时您可能希望将任务调用的签名传递给另一个进程或作为另一个函数的参数。

`signature()` 包装了单个任务调用的参数、关键字参数和执行选项，使其可以传递给函数，甚至可以序列化并通过网络发送。

- 您可以使用任务名称创建 `add()` 任务的签名，如下所示：

    ```pycon
    >>> from celery import signature
    >>> signature('tasks.add', args=(2, 2), countdown=10)
    tasks.add(2, 2)
    ```

    此任务具有 2 个参数（两个参数）的签名：`(2, 2)`，并将倒计时执行选项设置为 10。

- 或者您可以使用任务的 `signature()` 方法创建一个：

    ```pycon
    >>> add.signature((2, 2), countdown=10)
    tasks.add(2, 2)
    ```

- 还有一个使用星号参数的快捷方式：

    ```pycon
    >>> add.s(2, 2)
    tasks.add(2, 2)
    ```

- 也支持关键字参数：

    ```pycon
    >>> add.s(2, 2, debug=True)
    tasks.add(2, 2, debug=True)
    ```

- 从任何签名实例中，您可以检查不同的字段：

    ```pycon
    >>> s = add.signature((2, 2), {'debug': True}, countdown=10)
    >>> s.args
    (2, 2)
    >>> s.kwargs
    {'debug': True}
    >>> s.options
    {'countdown': 10}
    ```

- 它支持 `delay()`、`apply_async()` 等的"调用 API"，包括直接调用（`__call__`）。

    调用签名将在当前进程中内联执行任务：

    ```pycon
    >>> add(2, 2)
    4
    >>> add.s(2, 2)()
    4
    ```

    `delay()` 是我们喜爱的 `apply_async()` 快捷方式，接受星号参数：

    ```pycon
    >>> result = add.delay(2, 2)
    >>> result.get()
    4
    ```

    `apply_async()` 接受与 `app.Task.apply_async()` 方法相同的参数：

    ```pycon
    >>> add.apply_async(args, kwargs, **options)
    >>> add.signature(args, kwargs, **options).apply_async()

    >>> add.apply_async((2, 2), countdown=1)
    >>> add.signature((2, 2), countdown=1).apply_async()
    ```

- 您不能使用 `s()` 定义选项，但链式 `set()` 调用可以解决这个问题：

    ```pycon
    >>> add.s(2, 2).set(countdown=1)
    proj.tasks.add(2, 2)
    ```

### 部分签名

使用签名，您可以在工作进程中执行任务：

```pycon
>>> add.s(2, 2).delay()
>>> add.s(2, 2).apply_async(countdown=1)
```

或者您可以在当前进程中直接调用它：

```pycon
>>> add.s(2, 2)()
4
```

向 `apply_async`/`delay` 指定额外的参数、关键字参数或选项会创建部分签名：

- 添加的任何参数都将前置到签名中的参数之前：

    ```pycon
    >>> partial = add.s(2)          # 不完整的签名
    >>> partial.delay(4)            # 4 + 2
    >>> partial.apply_async((4,))   # 相同
    ```

- 添加的任何关键字参数将与签名中的关键字参数合并，新的关键字参数优先：

    ```pycon
    >>> s = add.s(2, 2)
    >>> s.delay(debug=True)                    # -> add(2, 2, debug=True)
    >>> s.apply_async(kwargs={'debug': True})  # 相同
    ```

- 添加的任何选项将与签名中的选项合并，新的选项优先：

    ```pycon
    >>> s = add.signature((2, 2), countdown=10)
    >>> s.apply_async(countdown=1)  # 倒计时现在是 1
    ```

您还可以克隆签名以创建衍生签名：

```pycon
>>> s = add.s(2)
proj.tasks.add(2)

>>> s.clone(args=(4,), kwargs={'debug': True})
proj.tasks.add(4, 2, debug=True)
```

### 不可变性

部分签名旨在与回调一起使用，任何链接的任务或和弦回调都将使用父任务的结果来应用。有时您希望指定一个不接受额外参数的回调，在这种情况下，您可以将签名设置为不可变：

```pycon
>>> add.apply_async((2, 2), link=reset_buffers.signature(immutable=True))
```

`.si()` 快捷方式也可以用于创建不可变签名：

```pycon
>>> add.apply_async((2, 2), link=reset_buffers.si())
```

当签名不可变时，只能设置执行选项，因此无法使用部分参数/关键字参数调用签名。

!!! note

    在本教程中，我有时使用前缀运算符 `~` 来操作签名。您可能不应该在生产代码中使用它，但在 Python shell 中进行实验时，它是一个方便的快捷方式：

    ```pycon
    >>> ~sig

    >>> # 等同于
    >>> sig.delay().get()
    ```

### 回调

可以使用 `apply_async()` 的 `link` 参数将回调添加到任何任务：

```pycon
add.apply_async((2, 2), link=other_task.s())
```

回调仅在任务成功退出时应用，并且将使用父任务的返回值作为参数来应用。

正如我之前提到的，您添加到签名的任何参数都将前置到签名本身指定的参数之前！

如果您有签名：

```pycon
>>> sig = add.s(10)
```

那么 `sig.delay(result)` 变为：

```pycon
>>> add.apply_async(args=(result, 10))
```

...

现在让我们使用部分参数调用我们的 `add()` 任务并添加回调：

```pycon
>>> add.apply_async((2, 2), link=add.s(8))
```

正如预期的那样，这将首先启动一个计算 `2 + 2` 的任务，然后启动另一个计算 `4 + 8` 的任务。

## 原语

<table>
<thead>
<tr>
    <th>原语</th>
    <th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>group</code></td>
<td>group原语是一个签名，它接受一个任务列表，这些任务应该并行应用。</td>
</tr>
<tr>
<td><code>chain</code></td>
<td>chain原语允许我们将签名链接在一起，以便一个接一个地调用，本质上形成一个回调*链*。</td>
</tr>
<tr>
<td><code>chord</code></td>
<td>chord就像一个带有回调的group。一个chord由一个头部组和一个主体组成，其中主体是在头部所有任务完成后应该执行的任务。</td>
</tr>
<tr>
<td><code>map</code></td>
<td>
map原语的工作方式类似于内置的 <code>map</code> 函数，但会创建一个临时任务，其中参数列表被应用到任务上。例如，<code>task.map([1, 2])</code> -- 导致调用单个任务，按顺序将参数应用到任务函数，结果是：

```python
res = [task(1), task(2)]
```
</td>
</tr>
<tr>
<td><code>starmap</code></td>
<td>
工作方式与map完全相同，只是参数以 <code>*args</code> 的形式应用。
例如 <code>add.starmap([(2, 2), (4, 4)])</code> 导致单个任务调用：

```python
res = [add(2, 2), add(4, 4)]
```
</td>
</tr>
<tr>
<td><code>chunks</code></td>
<td>
分块将长参数列表分成多个部分，例如操作：

```pycon
>>> items = zip(range(1000), range(1000))  # 1000个项目
>>> add.chunks(items, 10)
```

将项目列表分成10个一组，产生100个任务（每个任务按顺序处理10个项目）。
</td>
</tr>
</tbody>
</table>

这些原语本身也是签名对象，因此它们可以以多种方式组合来构成复杂的工作流。

以下是一些示例：

- 简单链

    这是一个简单的链，第一个任务执行并将其返回值传递给链中的下一个任务，依此类推。

    ```pycon
    >>> from celery import chain

    >>> # 2 + 2 + 4 + 8
    >>> res = chain(add.s(2, 2), add.s(4), add.s(8))()
    >>> res.get()
    16
    ```

    也可以使用管道符号编写：

    ```pycon
    >>> (add.s(2, 2) | add.s(4) | add.s(8))().get()
    16
    ```

- 不可变签名

    签名可以是部分的，因此可以向现有参数添加参数，但您可能并不总是需要这样，例如，如果您不想要链中前一个任务的结果。

    在这种情况下，您可以将签名标记为不可变，这样参数就不能被更改：

    ```pycon
    >>> add.signature((2, 2), immutable=True)
    ```

    还有一个`.si()`快捷方式，这是创建签名的首选方式：

    ```pycon
    >>> add.si(2, 2)
    ```

    现在您可以创建一个独立任务的链：
    
    ```pycon
    >>> res = (add.si(2, 2) | add.si(4, 4) | add.si(8, 8))()
    >>> res.get()
    16

    >>> res.parent.get()
    8

    >>> res.parent.parent.get()
    4
    ```

- 简单组

    您可以轻松创建一组并行执行的任务：

    ```pycon
    >>> from celery import group
    >>> res = group(add.s(i, i) for i in range(10))()
    >>> res.get(timeout=1)
    [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
    ```

- 简单和弦

    chord原语使我们能够添加一个回调，当组中的所有任务执行完成时调用。这对于那些不是*令人尴尬的并行*的算法通常是必需的：

    ```pycon
    >>> from celery import chord
    >>> res = chord((add.s(i, i) for i in range(10)), tsum.s())()
    >>> res.get()
    90
    ```

    上面的示例创建了10个并行启动的任务，当所有任务完成后，返回值被组合成一个列表并发送到`tsum`任务。

    chord的主体也可以是不可变的，这样组的返回值就不会传递给回调：

    ```pycon
    >>> chord((import_contact.s(c) for c in contacts), notify_complete.si(import_id)).apply_async()
    ```

    注意上面使用了`.si`；这会创建一个不可变签名，意味着传递的任何新参数（包括前一个任务的返回值）都将被忽略。

- 通过组合让您大开眼界

    链也可以是部分的：

    ```pycon
    >>> c1 = (add.s(4) | mul.s(8))

    # (16 + 4) * 8
    >>> res = c1(16)
    >>> res.get()
    160
    ```

    这意味着您可以组合链：

    ```pycon
    # ((4 + 16) * 2 + 4) * 8
    >>> c2 = (add.s(4, 16) | mul.s(2) | (add.s(4) | mul.s(8)))

    >>> res = c2()
    >>> res.get()
    352
    ```

    将组与另一个任务链接在一起会自动将其升级为chord：

    ```pycon
    >>> c3 = (group(add.s(i, i) for i in range(10)) | tsum.s())
    >>> res = c3()
    >>> res.get()
    90
    ```

    组和chord也接受部分参数，因此在链中，前一个任务的返回值会转发给组中的所有任务：

    ```pycon
    >>> new_user_workflow = (create_user.s() | group(import_contacts.s(), send_welcome_email.s()))
    ... new_user_workflow.delay(username='artv', first='Art', last='Vandelay', email='art@vandelay.com')
    ```

    如果您不想将参数转发给组，那么您可以使组中的签名不可变：

    ```pycon
    >>> res = (add.s(4, 4) | group(add.si(i, i) for i in range(10)))()
    >>> res.get()
    [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

    >>> res.parent.get()
    8
    ```

!!! warning

    对于更复杂的工作流，已观察到默认的JSON序列化器由于递归引用而显著增加消息大小，导致资源问题。*pickle*序列化器不容易受到此问题的影响，因此在这样的情况下可能更可取。

### Chains

任务可以链接在一起：当任务成功返回时调用链接的任务：

```pycon
>>> res = add.apply_async((2, 2), link=mul.s(16))
>>> res.get()
4
```

链接的任务将以其父任务的结果作为第一个参数应用。在上面的例子中，结果是4，这将导致 `mul(4, 16)`。

结果将跟踪原始任务调用的任何子任务，这可以从结果实例中访问：

```pycon
>>> res.children
[<AsyncResult: 8c350acf-519d-4553-8a53-4ad3a5c5aeb4>]

>>> res.children[0].get()
64
```

结果实例还有一个 `collect()` 方法，它将结果视为图，使您能够迭代结果：

```pycon
>>> list(res.collect())
[(<AsyncResult: 7b720856-dc5f-4415-9134-5c89def5664e>, 4), (<AsyncResult: 8c350acf-519d-4553-8a53-4ad3a5c5aeb4>, 64)]
```

默认情况下，如果图未完全形成（其中一个任务尚未完成），`collect()` 方法将引发 `IncompleteStream` 异常，但您也可以获取图的中间表示：

```pycon
>>> for result, value in res.collect(intermediate=True):
...
```

您可以根据需要链接任意多个任务，签名也可以链接：

```pycon
>>> s = add.s(2, 2)
>>> s.link(mul.s(4))
>>> s.link(log_result.s())
```

您还可以使用 `on_error` 方法添加*错误回调*：

```pycon
>>> add.s(2, 2).on_error(log_error.s()).delay()
```

当签名被应用时，这将导致以下 `.apply_async` 调用：

```pycon
>>> add.apply_async((2, 2), link_error=log_error.s())
```

工作进程实际上不会将错误回调作为任务调用，而是直接调用错误回调函数，以便可以将原始请求、异常和回溯对象传递给它。

以下是一个错误回调的示例：

```python
import os

from proj.celery import app

@app.task
def log_error(request, exc, traceback):
    with open(os.path.join('/var/errors', request.id), 'a') as fh:
        print('--\n\n{0} {1} {2}'.format(
            request.id, exc, traceback), file=fh)
```

为了使链接任务更容易，有一个特殊的签名 `chain()` 可以让您将任务链接在一起：

```pycon
>>> from celery import chain
>>> from proj.tasks import add, mul

>>> # (4 + 4) * 8 * 10
>>> res = chain(add.s(4, 4), mul.s(8), mul.s(10))
proj.tasks.add(4, 4) | proj.tasks.mul(8) | proj.tasks.mul(10)
```

调用链将在当前进程中调用任务，并返回链中最后一个任务的结果：

```pycon
>>> res = chain(add.s(4, 4), mul.s(8), mul.s(10))()
>>> res.get()
640
```

它还设置 `parent` 属性，以便您可以沿着链向上工作以获取中间结果：

```pycon
>>> res.parent.get()
64

>>> res.parent.parent.get()
8

>>> res.parent.parent
<AsyncResult: eeaad925-6778-4ad1-88c8-b2a63d017933>
```

也可以使用 `|`（管道）运算符创建链：

```pycon
>>> (add.s(2, 2) | mul.s(8) | mul.s(10)).apply_async()
```

#### 任务 ID（Task ID）

链将继承链中最后一个任务的任务ID。

#### 图（Graphs）

此外，您可以将结果图作为 `DependencyGraph` 来处理：

```pycon
>>> res = chain(add.s(4, 4), mul.s(8), mul.s(10))()

>>> res.parent.parent.graph
285fa253-fcf8-42ef-8b95-0078897e83e6(1)
    463afec2-5ed4-4036-b22d-ba067ec64f52(0)
872c3995-6fa0-46ca-98c2-5a19155afcf0(2)
    285fa253-fcf8-42ef-8b95-0078897e83e6(1)
        463afec2-5ed4-4036-b22d-ba067ec64f52(0)
```

您甚至可以将这些图转换为*dot*格式：

```pycon
>>> with open('graph.dot', 'w') as fh:
...     res.parent.parent.graph.to_dot(fh)
```

并创建图像：

```console
dot -Tpng graph.dot -o graph.png
```

![graph.png](../assets/result_graph.png){loading=lazy}

### Groups {#canvas-group}

!!! note

    与和弦类似，在组中使用的任务*不能*忽略其结果。。

组可以用于并行执行多个任务。

`group` 函数接受一个签名列表：

```pycon
>>> from celery import group
>>> from proj.tasks import add

>>> group(add.s(2, 2), add.s(4, 4))
(proj.tasks.add(2, 2), proj.tasks.add(4, 4))
```

如果您**调用**组，任务将在当前进程中依次应用，并返回一个 `GroupResult` 实例，可用于跟踪结果或了解有多少任务已准备就绪等：

```pycon
>>> g = group(add.s(2, 2), add.s(4, 4))
>>> res = g()
>>> res.get()
[4, 8]
```

组也支持迭代器：

```pycon
>>> group(add.s(i, i) for i in range(100))()
```

组是一个签名对象，因此可以与其他签名结合使用。

#### 回调和错误处理

组也可以链接回调和错误回调签名，但由于组不是真正的任务，只是将链接的任务传递给其封装的签名，因此行为可能有些令人惊讶。这意味着组的返回值不会被收集并传递给链接的回调签名。此外，链接任务*不*保证它仅在所有组任务完成后才激活。例如，以下使用简单 `add(a, b)` 任务的代码片段是有问题的，因为链接的 `add.s()` 签名不会像预期的那样接收最终的组结果。

```pycon
>>> g = group(add.s(2, 2), add.s(4, 4))
>>> g.link(add.s())
>>> res = g()
[4, 8]
```

请注意，前两个任务的最终结果已返回，但回调签名将在后台运行并引发异常，因为它没有收到期望的两个参数。

组错误回调也会传递给封装的签名，这可能导致仅链接一次的错误回调在组中多个任务失败时被多次调用。
例如，以下使用引发异常的 `fail()` 任务的代码片段可以预期为组中运行的每个失败任务调用一次 `log_error()` 签名。

```pycon
>>> g = group(fail.s(), fail.s())
>>> g.link_error(log_error.s())
>>> res = g()
```

考虑到这一点，通常建议创建幂等或计数任务，这些任务能够容忍被重复调用以用作错误回调。

这些用例更适合由 `chord` 类处理，该类在某些后端实现中受支持。

#### 结果

组任务也返回一个特殊的结果，这个结果的工作方式类似于普通任务结果，
只是它作用于整个组：

```pycon
>>> from celery import group
>>> from tasks import add

>>> job = group([add.s(2, 2), add.s(4, 4), add.s(8, 8), add.s(16, 16), add.s(32, 32)])

>>> result = job.apply_async()

>>> result.ready()  # 所有子任务都完成了吗？
True
>>> result.successful() # 所有子任务都成功了吗？
True
>>> result.get()
[4, 8, 16, 32, 64]
```

`GroupResult` 接受一个 `AsyncResult` 实例列表，并像操作单个任务一样操作它们。

它支持以下操作：

| 方法                  | 描述                                                                                    |
|---------------------|---------------------------------------------------------------------------------------|
| `successful()`      | 如果所有子任务都成功完成（例如，没有引发异常），则返回 `True`。                                                   |
| `failed()`          | 如果有任何子任务失败，则返回 `True`。                                                                |
| `waiting()`         | 如果有任何子任务尚未就绪，则返回 `True`。                                                              |
| `ready()`           | 如果所有子任务都已就绪，则返回 `True`。                                                               |
| `completed_count()` | 返回已完成的子任务数量。请注意，在此上下文中 `complete` 意味着 `successful`。换句话说，此方法的返回值是 `successful` 任务的数量。  |
| `revoke()`          | 撤销所有子任务。                                                                              |
| `join()`            | 收集所有子任务的结果，并按它们被调用的顺序返回（作为列表）。                                                        |

#### 展开

当链接时，包含单个签名的组将展开为单个签名。这意味着以下组可能会根据组中项目的数量向链传递结果列表或单个结果。

```pycon
>>> from celery import chain, group
>>> from tasks import add
>>> chain(add.s(2, 2), group(add.s(1)), add.s(1))
add(2, 2) | add(1) | add(1)
>>> chain(add.s(2, 2), group(add.s(1), add.s(2)), add.s(1))
add(2, 2) | %add((add(1), add(2)), 1)
```

这意味着您应该小心，并确保 `add()` 任务可以接受列表或单个项目作为输入，如果您计划将其用作更大画布的一部分。

!!! warning

    在 Celery 4.x 中，由于一个错误，下面的组不会展开为链，而是画布将升级为和弦。

    ```pycon
    >>> from celery import chain, group
    >>> from tasks import add
    >>> chain(group(add.s(1, 1)), add.s(2))
    %add([add(1, 1)], 2)
    ```

    在 Celery 5.x 中，此错误已修复，组正确展开为单个签名。

    ```pycon
    >>> from celery import chain, group
    >>> from tasks import add
    >>> chain(group(add.s(1, 1)), add.s(2))
    add(1, 1) | add(2)
    ```

### Chords

!!! note

    在 chord 中使用的任务*不能*忽略它们的结果。如果在你的 chord 中*任何*任务（header 或 body）的结果后端被禁用。Chords 目前不支持 RPC 结果后端。

Chord 是一个只有在组中所有任务都执行完成后才会执行的任务。

让我们计算表达式 `1 + 1 + 2 + 2 + 3 + 3 ... n + n` 到一百位数字的和。

首先你需要两个任务，`add()` 和 `tsum()`（`sum()` 已经是一个标准函数）：

```python
@app.task
def add(x, y):
    return x + y

@app.task
def tsum(numbers):
    return sum(numbers)
```

现在你可以使用 chord 来并行计算每个加法步骤，然后得到结果数字的总和：

```pycon
>>> from celery import chord
>>> from tasks import add, tsum

>>> chord(add.s(i, i) for i in range(100))(tsum.s()).get()
9900
```

这显然是一个非常人为的例子，消息传递和同步的开销使得这比其 Python 对应版本慢很多：

```pycon
>>> sum(i + i for i in range(100))
```

同步步骤成本很高，所以你应该尽可能避免使用 chords。尽管如此，chord 仍然是工具箱中一个强大的原语，因为同步是许多并行算法所需的步骤。

让我们分解 chord 表达式：

```pycon
>>> callback = tsum.s()
>>> header = [add.s(i, i) for i in range(100)]
>>> result = chord(header)(callback)
>>> result.get()
9900
```

记住，回调只能在 header 中的所有任务都返回后才能执行。header 中的每个步骤都作为一个任务执行，并行地，可能在不同的节点上。然后使用 header 中每个任务的返回值来应用回调。`chord()` 返回的任务 id 是回调的 id，所以你可以等待它完成并获取最终的返回值。

#### 错误处理

那么如果其中一个任务引发异常会发生什么？

Chord 回调结果将转换到失败状态，错误被设置为 `ChordError` 异常：

```pycon
>>> c = chord([add.s(4, 4), raising_task.s(), add.s(8, 8)])
>>> result = c()
>>> result.get()
```

```pycon
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "*/celery/result.py", line 120, in get
    interval=interval)
  File "*/celery/backends/amqp.py", line 150, in wait_for
    raise meta['result']
celery.exceptions.ChordError: Dependency 97de6f3f-ea67-4517-a21c-d867c61fcb47
    raised ValueError('something something',)
```

虽然跟踪回溯可能因使用的结果后端而异，但你可以看到错误描述包括失败任务的 id 和原始异常的字符串表示。你也可以在 `result.traceback` 中找到原始跟踪回溯。

请注意，其余任务仍将执行，所以第三个任务（`add.s(8, 8)`）即使中间任务失败也会执行。此外，`ChordError` 只显示首先（按时间）失败的任务：它不尊重 header 组的顺序。

因此，要在 chord 失败时执行操作，你可以将 errback 附加到 chord 回调：

```python
@app.task
def on_chord_error(request, exc, traceback):
    print('Task {0!r} raised error: {1!r}'.format(request.id, exc))
```

```pycon
>>> c = (group(add.s(i, i) for i in range(10)) | tsum.s().on_error(on_chord_error.s())).delay()
```

Chords 可以有回调（callback）和错误回调（errback）签名链接到它们，这解决了将签名链接到组的一些问题。这样做会将提供的签名链接到 chord 的 body，期望在 body 完成时优雅地调用回调一次，或者在 chord header 或 body 中的任何任务失败时调用错误回调一次。

这种行为可以通过使用 [task_allow_error_cb_on_chord_header](configuration.md#task_allow_error_cb_on_chord_header) 标志来操作，以允许对 chord header 进行错误处理。启用此标志将导致 chord header 调用 body 的错误回调（默认行为）*以及* chord header 中任何失败的任务。

#### 重要注意事项

在 chord 中使用的任务*不能*忽略它们的结果。实际上这意味着你必须启用 `result_backend` 才能使用 chords。此外，如果在你的配置中 `task_ignore_result` 设置为 `True`，请确保要在 chord 中使用的各个任务都定义为 `ignore_result=False`。这适用于 Task 子类和装饰的任务。

Task 子类示例：

```python
class MyTask(Task):
    ignore_result = False
```

装饰任务示例：

```python
@app.task(ignore_result=False)
def another_task(project):
    do_something()
```

默认情况下，同步步骤是通过让一个循环任务每秒轮询组的完成情况，在准备好时调用签名来实现的。

示例实现：

```python
from celery import maybe_signature

@app.task(bind=True)
def unlock_chord(self, group, callback, interval=1, max_retries=None):
    if group.ready():
        return maybe_signature(callback).delay(group.join())
    raise self.retry(countdown=interval, max_retries=max_retries)
```

这被除 Redis、Memcached 和 DynamoDB 之外的所有结果后端使用：它们在 header 中的每个任务之后递增一个计数器，然后在计数器超过集合中的任务数时应用回调。

Redis、Memcached 和 DynamoDB 的方法是一个更好的解决方案，但不容易在其他后端中实现（欢迎建议！）。

!!! note

    Chords 在 Redis 2.2 版本之前不能正常工作；你需要升级到至少 redis-server 2.2 才能使用它们。

!!! note

    如果你使用 Redis 结果后端和 chords，并且还重写了 `Task.after_return()` 方法，你需要确保调用 super 方法，否则 chord 回调将不会被应用。

    ```python
    def after_return(self, *args, **kwargs):
        do_something()
        super().after_return(*args, **kwargs)
    ```

### Map & Starmap

`map` 和 `starmap` 是内置任务，它们为序列中的每个元素调用提供的调用任务。

它们与 `group` 的不同之处在于：

- 只发送一个任务消息
- 操作是顺序执行的

例如使用 `map`：

```pycon
>>> from proj.tasks import add

>>> ~tsum.map([list(range(10)), list(range(100))])
[45, 4950]
```

等同于有一个任务执行：

```python
@app.task
def temp():
    return [tsum(range(10)), tsum(range(100))]
```

使用 `starmap`：

```pycon
>>> ~add.starmap(zip(range(10), range(10)))
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

等同于有一个任务执行：

```python
@app.task
def temp():
    return [add(i, i) for i in range(10)]
```

`map` 和 `starmap` 都是签名对象，因此可以像其他签名一样使用，并可以在组等中组合使用，例如在10秒后调用starmap：

```pycon
>>> add.starmap(zip(range(10), range(10))).apply_async(countdown=10)
```

### Chunks

分块允许您将可迭代的工作分成多个部分，这样如果您有一百万个对象，您可以创建10个任务，每个任务处理十万个对象。

有些人可能担心对任务进行分块会导致并行性降低，但对于繁忙的集群来说，这种情况很少发生，并且在实践中，由于您避免了消息传递的开销，它可能会显著提高性能。

要创建分块的签名，您可以使用 `chunks()`：

```pycon
>>> add.chunks(zip(range(100), range(100)), 10)
```

与 `group` 类似，当调用时，发送分块消息的操作将在当前进程中发生：

```pycon
>>> from proj.tasks import add

>>> res = add.chunks(zip(range(100), range(100)), 10)()
>>> res.get()
[[0, 2, 4, 6, 8, 10, 12, 14, 16, 18],
 [20, 22, 24, 26, 28, 30, 32, 34, 36, 38],
 [40, 42, 44, 46, 48, 50, 52, 54, 56, 58],
 [60, 62, 64, 66, 68, 70, 72, 74, 76, 78],
 [80, 82, 84, 86, 88, 90, 92, 94, 96, 98],
 [100, 102, 104, 106, 108, 110, 112, 114, 116, 118],
 [120, 122, 124, 126, 128, 130, 132, 134, 136, 138],
 [140, 142, 144, 146, 148, 150, 152, 154, 156, 158],
 [160, 162, 164, 166, 168, 170, 172, 174, 176, 178],
 [180, 182, 184, 186, 188, 190, 192, 194, 196, 198]]
```

而调用 `.apply_async` 将创建一个专用任务，以便各个任务在工作者中应用：

```pycon
>>> add.chunks(zip(range(100), range(100)), 10).apply_async()
```

您还可以将分块转换为组：

```pycon
>>> group = add.chunks(zip(range(100), range(100)), 10).group()
```

并使用组将每个任务的倒计时按增量为一进行偏移：

```pycon
>>> group.skew(start=1, stop=10)()
```

这意味着第一个任务的倒计时为一秒，第二个任务的倒计时为两秒，依此类推。

## 标记

Stamping API 的目标是为签名及其组件提供标记能力，以便于调试信息的目的。例如，当画布（canvas）是一个复杂结构时，可能需要标记已形成结构的部分或全部元素。当嵌套组展开或链元素被替换时，复杂性会进一步增加。在这种情况下，可能需要了解某个元素属于哪个组或处于什么嵌套级别。这需要一个遍历画布元素并用特定元数据标记它们的机制。标记 API 基于访问者模式（Visitor pattern）实现这一功能。

例如，

```pycon
>>> sig1 = add.si(2, 2)
>>> sig1_res = sig1.freeze()
>>> g = group(sig1, add.si(3, 3))
>>> g.stamp(stamp='your_custom_stamp')
>>> res = g.apply_async()
>>> res.get(timeout=TIMEOUT)
[4, 6]
>>> sig1_res._get_task_meta()['stamp']
['your_custom_stamp']
```

将初始化一个组 `g` 并用标记 `your_custom_stamp` 标记其组件。

为了使此功能有用，您需要将 `result_extended` 配置选项设置为 `True` 或指令 `result_extended = True`。

### 画布标记

我们还可以使用自定义标记逻辑来标记画布，使用访问者类 `StampingVisitor` 作为自定义标记访问者的基类。

### 自定义标记

如果需要更复杂的标记逻辑，可以基于访问者模式实现自定义标记行为。
实现此自定义逻辑的类必须继承 `StampingVisitor` 并实现适当的方法。

例如，以下示例 `InGroupVisitor` 将通过标签 `in_group` 标记位于某个组内的任务。

```python
class InGroupVisitor(StampingVisitor):
    def __init__(self):
        self.in_group = False

    def on_group_start(self, group, **headers) -> dict:
        self.in_group = True
        return {"in_group": [self.in_group], "stamped_headers": ["in_group"]}

    def on_group_end(self, group, **headers) -> None:
        self.in_group = False

    def on_chain_start(self, chain, **headers) -> dict:
        return {"in_group": [self.in_group], "stamped_headers": ["in_group"]}

    def on_signature(self, sig, **headers) -> dict:
        return {"in_group": [self.in_group], "stamped_headers": ["in_group"]}
```

以下示例展示了另一个自定义标记访问者，它为所有任务标记一个自定义的 `monitoring_id`，该 ID 可以代表外部监控系统的 UUID 值，通过包含此 ID 的访问者实现来跟踪任务执行。这个 `monitoring_id` 可以是一个随机生成的 UUID，或者是外部监控系统使用的 span id 的唯一标识符等。

```python
class MonitoringIdStampingVisitor(StampingVisitor):
    def on_signature(self, sig, **headers) -> dict:
        return {'monitoring_id': uuid4().hex}
```

!!! tip

    `on_signature()`（或任何其他访问者方法）返回的字典中的 `stamped_headers` 键是**可选的**：

    ```python
    # 方法 1：不使用 stamped_headers - 所有键都被视为标记
    def on_signature(self, sig, **headers) -> dict:
        return {'monitoring_id': uuid4().hex}  # monitoring_id 成为标记

    # 方法 2：使用 stamped_headers - 只有列出的键是标记
    def on_signature(self, sig, **headers) -> dict:
        return {
            'monitoring_id': uuid4().hex,      # 这将是一个标记
            'other_data': 'value',             # 这将不是标记
            'stamped_headers': ['monitoring_id']  # 只有 monitoring_id 被标记
        }
    ```

    如果未指定 `stamped_headers` 键，标记访问者将假定返回字典中的所有键都是标记头。

接下来，让我们看看如何使用 `MonitoringIdStampingVisitor` 示例标记访问者。

```python
sig_example = signature('t1')
sig_example.stamp(visitor=MonitoringIdStampingVisitor())

group_example = group([signature('t1'), signature('t2')])
group_example.stamp(visitor=MonitoringIdStampingVisitor())

chord_example = chord([signature('t1'), signature('t2')], signature('t3'))
chord_example.stamp(visitor=MonitoringIdStampingVisitor())

chain_example = chain(signature('t1'), group(signature('t2'), signature('t3')), signature('t4'))
chain_example.stamp(visitor=MonitoringIdStampingVisitor())
```

最后，需要说明的是，上述示例中的每个监控 ID 标记在任务之间都是不同的。

### 回调标记

标记 API 还支持隐式标记回调。这意味着当回调添加到任务时，标记访问者也会应用于回调。

!!! warning

    回调必须在标记之前链接到签名。

例如，让我们检查以下使用隐式方法的自定义标记访问者，其中所有返回的字典键都会自动被视为标记头，而无需显式指定 `stamped_headers`。

```python
class CustomStampingVisitor(StampingVisitor):
    def on_signature(self, sig, **headers) -> dict:
        # 'header' 将自动被视为标记头
        # 无需指定 'stamped_headers': ['header']
        return {'header': 'value'}

    def on_callback(self, callback, **header) -> dict:
        # 'on_callback' 将自动被视为标记头
        return {'on_callback': True}

    def on_errback(self, errback, **header) -> dict:
        # 'on_errback' 将自动被视为标记头
        return {'on_errback': True}
```

此自定义标记访问者将用 `{'header': 'value'}` 标记签名、回调和错误回调，并分别用 `{'on_callback': True}` 和 `{'on_errback': True}` 标记回调和错误回调，如下所示。

```python
c = chord([add.s(1, 1), add.s(2, 2)], xsum.s())
callback = signature('sig_link')
errback = signature('sig_link_error')
c.link(callback)
c.link_error(errback)
c.stamp(visitor=CustomStampingVisitor())
```

此示例将产生以下标记：

```pycon
>>> c.options
{'header': 'value', 'stamped_headers': ['header']}
>>> c.tasks.tasks[0].options
{'header': 'value', 'stamped_headers': ['header']}
>>> c.tasks.tasks[1].options
{'header': 'value', 'stamped_headers': ['header']}
>>> c.body.options
{'header': 'value', 'stamped_headers': ['header']}
>>> c.body.options['link'][0].options
{'header': 'value', 'on_callback': True, 'stamped_headers': ['header', 'on_callback']}
>>> c.body.options['link_error'][0].options
{'header': 'value', 'on_errback': True, 'stamped_headers': ['header', 'on_errback']}
```
