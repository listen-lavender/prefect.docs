## 概览

Prefect的执行模型围绕**FlowRunner**和**TaskRunner**这两个类构建，它们产生并在State对象上操作。实际执行由**Executor**类处理，该类可以与外部环境交互。

## Flow执行器

flow执行器接收flow并尝试运行其所有task。它收集结果状态，并在可能的情况下返回flow的最终状态。

flow执行器一次遍历所有task。如果task在那一遍之后仍未完成，例如，如果其中一个task需要重试，则将需要第二个循环来尝试完成它们。将所有task（以及flow本身）移至完成状态所需的尝试次数没有限制。

### 参数

具有参数的flow可能需要参数值（如果这些参数没有默认值）。运行时，必须将参数值传递到flow实例。

## Task执行器

task执行器负责执行单个task。它接收task的初始状态以及任何上游状态，并使用这些状态来表示执行管道的结果。例如：

 - task必须处于**Pending**状态
 - 上游task必须完成
 - task的触发函数必须执行

如果满足这些条件（以及其他一些条件），则task可以进入**Running**状态。

然后，根据task，它可以是**.run()**或可以被映射，这涉及创建动态的子task执行器。

最后，task在后期处理管道中移动，该管道检查是否应重试或缓存该task。

## 执行器

执行器类负责实际执行的task。例如，flow执行器将把每个task执行器提交给它的执行器，并等待结果。我们建议将Dask分布式执行器作为首选的执行引擎。

执行器具有相对简单的API：用户可以提交函数并等待其结果。

对于测试和开发，首选LocalExecutor。除非明确声明，否则它将在本地进程中同步运行每个函数，并且是flow的默认执行器。

SynchronousExecutor稍微复杂一些。它仍然在单个线程中运行函数，但是使用Dask的调度逻辑。

DaskExecutor是一个完全异步的引擎，可以在分布式Dask集群中运行函数。这是推荐用于生产的引擎。

## 使用Dask执行器

执行器可以在flow的运行时提供：

````Python
from prefect import task, Flow

@task
def say_hello():
    print("Hello, world!")

with Flow("Run Me") as flow:
    h = say_hello()

from prefect.engine.executors import DaskExecutor

executor = DaskExecutor(address="tcp://localhost:8786")
flow.run(executor=executor)
````

该DaskExecutor将通过地址tcp://localhost:8786连接到Dask调度器，并开始提交要在Dask工作者上执行的工作。

> 
> **动态调度器**
> 
> 如果没有为DaskExecutor指定调度器地址，则将创建一个进程内调度器，并在完成时将其清除掉。更多信息参见[DaskExecutor API文档](https://docs.prefect.io/api/latest/engine/executors.html#daskexecutor)。
> 

不同的调度器使用时是有些差别的。

> 
> **LocalDaskExecutor vs DaskExecutor**
> 
> LocalDaskExecutor和DaskExecutor之间的主要区别在于调度器的选择。可以将LocalDaskExecutor配置为使用任意数量的调度器，而DaskExecutor使用分布式调度器。这意味着LocalDaskExecutor可以帮助实现一些多线程/多进程，但是它没有提供与DaskExecutor一样多的分布式功能。
> 

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/engine.html)
- [联系译者](https://github.com/listen-lavender)
