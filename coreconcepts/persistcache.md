开箱即用，Prefect Core不会永久保存数据。所有数据，结果和缓存状态都存储在运行该flow的Python进程的内存中。但是，Prefect Core提供了所有必要的钩子，用于在外部位置持久/查询数据。如果你需要开箱即用的持久层，则可以考虑使用Prefect Cloud。

Prefect提供了几种处理缓存数据的方法。在任何可能的情况下，缓存都将自动进行，或以最少的用户输入触发。

 - 缓存输入（Input Caching）
 - 缓存输出（Output Caching）
 - 检查点（Checkpointing）

## 缓存输入

在运行Prefect flow时，通常有一些需要在将来重新运行的task。例如，当task失败并需要重试时，或者task具有**manual_only**触发器时，可能会发生这种情况。

每当Prefect检测到将来需要运行某个task时，它将自动缓存该task需要运行的所有信息，并将其存储在结果状态中。下次Prefect遇到该task时，关键信息将反序列化并用于运行task。

> 
> **自动缓存**
> 
> 缓存输入是自动缓存。Prefect会在必要时自动应用它。
> 

## 缓存输出

有时，最好缓存task的输出，以避免将来重新计算它。这种模式的常见示例包括不太可能更改的昂贵或费时的计算。在这种情况下，用户可以指定应该将task缓存一定的持续时间，或者只要满足某些条件即可。

这种机制有时称为“时间旅行”，因为它使一个flow运行中计算出的结果可用于其他运行。

缓存输出由三个task参数控制：**cache_for**，**cache_validator**和**cache_key**。

 - cache_for：一个时间长度，表示应将输出缓存多长时间
 - cache_validator：一个可调用对象，表示缓存应如何过期。默认值为**duration_only**，这意味着缓存将在**cache_for**的持续时间内处于有效状态。其他验证器可以在**prefect.engine.cache_validators**中找到，并且包括用于在task接收到不同的输入或flow以不同的参数运行时失效缓存的机制
 - cache_key：用于存储输出缓存的可选参数；指定此参数将允许不同的task以及不同的flow共享一个公共缓存

````Python
# this task will be cached for 1 hour
task_1 = prefect.Task(
    cache_for=datetime.timedelta(hours=1))

# this task will be cached for 1 hour, but only if the flow is run with the same parameters
task_2 = prefect.Task(
    cache_for=datetime.timedelta(hours=1),
    cache_validator=prefect.engine.cache_validators.all_parameters)
````

> 
> **缓存存储在上下文中**
> 
> 注意，在本地运行Prefect Core时，task的缓存状态将存储在内存的prefect.context的对象中。

## 检查点

通常，将task的数据保存在外部存储很有用。你总是可以将此逻辑直接直接写入task本身，但这有时会使测试变得困难。Prefect提供了task“检查点”的概念，以确保每次成功运行task时都会调用其结果处理器。要配置用于检查点的task，就得提供结果处理器，并在task初始化时设置**checkpoint=True**：

````Python
from prefect.engine.result_handlers import LocalResultHandler
from prefect import task, Task


class MyTask(Task):
    def run(self):
        return 42


# create a task via initializing our custom Task class
class_task = MyTask(
    checkpoint=True, result_handler=LocalResultHandler(dir="~/.prefect")
)


# create a task via the task decorator
@task(checkpoint=True, result_handler=LocalResultHandler(dir="~/.prefect"))
def func_task():
    return 99
````

Prefect Core中的默认设置是关闭检查点，Prefect Cloud 0.9.1+中的默认设置是打开检查点。有关更多信息，阅读有关[结果和结果处理器](https://docs.prefect.io/core/concepts/results.html)的概念文档以及有关[使用结果处理器](https://docs.prefect.io/core/advanced_tutorials/using-result-handlers.html)的设置教程。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/persistence.html)
- [联系译者](https://github.com/listen-lavender)

