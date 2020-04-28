在Prefect引擎中，用户可以通过多种方式来影响执行。

 - [Triggers（触发器）](https://docs.prefect.io/core/concepts/execution.html#triggers)
 - [State signals（状态信号量）](https://docs.prefect.io/core/concepts/execution.html#state-signals)
 - [Context（上下文）](https://docs.prefect.io/core/concepts/execution.html#context)

     - [Adding context globally（添加全局上下文）](https://docs.prefect.io/core/concepts/execution.html#adding-context-globally)
     - [Modifying context at runtime（运行时修改上下文）](https://docs.prefect.io/core/concepts/execution.html#modifying-context-at-runtime)
     - [Prefect-supplied context（Prefect提供的上下文）](https://docs.prefect.io/core/concepts/execution.html#prefect-supplied-context)

## 触发器

触发函数根据上游task的状态决定task是否准备就绪。除非触发器通过，否则高级task将无法运行。

默认情况下，task具有**all_successful**触发器，这意味着除非所有上游task都成功，否则它们将不会运行。通过更改触发函数，您可以控制task相对于上游task的行为。其他触发器包括**all_failed**，**any_successful**，**any_failed**，**all_finished**和**manual_only**。这些可用于创建仅在前面的task失败时运行、或者一定运行、或者根本不会自动运行的task！

仅当所有上游task都处于**Finished**状态时，task才会执行其触发器。因此，**all_finished**触发器与**always_run**触发器相同。

> 
> 使用**manual_only**触发器暂停flow
> 
> **manual_only**始终将task插入**Paused**状态，因此该flow永远不会自动运行**manual_only**task。这允许用户在运行中暂停flow。要恢复，请将task插入**Resume**状态，这将被视为没有上游task的根task，并完全跳过触发器函数的执行检查。
> 

例如，假设我们要用一个根task构造一个flow；如果此task成功，则我们要运行task B。如果失败，我们将要运行task C。我们可以通过使用触发器来完成此模式：

````Python
import random

from prefect.triggers import all_successful, all_failed
from prefect import task, Flow


@task(name="Task A")
def task_a():
    if random.random() > 0.5:
        raise ValueError("Non-deterministic error has occured.")

@task(name="Task B", trigger=all_successful)
def task_b():
    # do something interesting
    pass

@task(name="Task C", trigger=all_failed)
def task_c():
    # do something interesting
    pass


with Flow("Trigger example") as flow:
    success = task_b(upstream_tasks=[task_a])
    fail = task_c(upstream_tasks=[task_a])

## note that as written, this flow will fail regardless of the path taken
## because *at least one* terminal task will fail;
## to fix this, we want to set Task B as the "reference task" for the Flow
## so that it's state uniquely determines the overall Flow state
flow.set_reference_tasks([success])

flow.run()
````

## 状态信号

Prefect会尽力推断正在运行的task的状态。如果**.run()**方法成功结束，则Prefect会将状态设置为**Success**，并记录所有返回的数据。如果**.run()**方法引发错误，则Prefect会通过相应的消息将状态设置为**Failed**。

有时，你可能希望对task状态进行更好的控制。例如，你可能希望跳过task或强制重试task。在那种场景下，请发出适当的Prefect信号。它将被拦截并转换为适当的状态。

Prefect为大多数状态提供信号，包括**RETRY**、**SKIP**、**FAIL**、**SUCCESS**和**PAUSE**。

````Python
from prefect import task
from prefect.engine import signals

def retry_if_negative(x):
    if x < 0:
        raise signals.RETRY()
    else:
        return x
````

Prefect信号的另一种常见用法是，将讨论的task嵌套在其他可能会捕获其正常结果或错误的函数里。在那种情况下，可以使用一个信号来“冒泡”所需的状态变化，并绕过正常的返回机制。

## 上下文

Prefect提供一个强大的Context对象，在task的**.run()**方法上不使用显式参数的情况下，可以共享信息。

Prefect会在flow和task执行期间向可以随时访问的上下文填充信息。上下文对象本身可以填充任意的用户定义的键值对，然后可以使用A.key对象方式或类似字典的A[key]索引方式对其进行访问。例如，以下代码假定用户提供上下文{a=1}，并且可以使用以下三种方式之一进行访问：

````bash
>>> import prefect
>>> prefect.context.a
1
>>> prefect.context['a']
1
>>> prefect.context.get('a')
1
````

### 添加全局上下文

可以通过config.toml的[context]部分在全局范围内添加上下文。对于此处指定的任何键，只要Prefect Core不会在内部覆盖此键，就可以从prefect.context全局访问它，甚至可以在flow运行实例之外访问。

````bash
# config.toml
[context]
a = 1
````

````bash    
>>> import prefect
>>> prefect.context.a
1
````

### 运行时修改上下文

修改上下文，甚至全局修改上下文键，在特殊时候能用
Modifying context, even globally set context keys, at specific times is possible using a provided context manager:

使用提供的上下文管理器可以在特定时间修改上下文，甚至可以全局设置上下文密钥：

````bash
>>> import prefect
>>> with prefect.context(a=2):
...    print(prefect.context.a)
...
2
````

这对于在不同情况下运行flow以进行快速迭代开发通常很有用：

````Python
@task
def try_unlock():
    if prefect.context.key == 'abc':
        return True
    else:
        raise signals.FAIL()

with Flow('Using Context') as flow:
    try_unlock()

flow.run() # this run fails

with prefect.context(key='abc'):
    flow.run() # this run is successful
````

### Prefect提供的上下文

除了您自己的上下文键之外，Prefect还可以在flow实例运行和task实例运行期间动态地将上下文提供给上下文对象。该上下文提供有关当前flow或task的一些标准信息。例如，正在运行的task已经从Prefect提供的上下文中知道了它们的运行日期：

````Python
@task
def report_start_day():
    logger = prefect.context.get("logger")
    logger.info(prefect.context.today)

with Flow('My flow') as flow:
    report_start_day()

flow.run()
````

````bash
[2020-03-02 22:15:58,779] INFO - prefect.FlowRunner | Beginning Flow run for 'My flow'
[2020-03-02 22:15:58,780] INFO - prefect.FlowRunner | Starting flow run.
[2020-03-02 22:15:58,786] INFO - prefect.TaskRunner | Task 'report_start_time': Starting task run...
[2020-03-02 22:15:58,786] INFO - prefect.Task: report_start_day | 2020-03-02
[2020-03-02 22:15:58,788] INFO - prefect.TaskRunner | Task 'report_start_time': finished task run for task with final state: 'Success'
[2020-03-02 22:15:58,789] INFO - prefect.FlowRunner | Flow run SUCCESS: all reference tasks succeeded
````

使用此上下文对于编写可感知时间的task有帮助，例如使用**prefect.context.tomorrow**触发与开始时间有关的未来作业的task，或者使用**prefect.context.yesterday**仅处理前一天的数据。

> 
> **上下文中还有什么？**
> 
> 有关可以在上下文中找到的值的详尽列表，请参见相应的[API文档](https://docs.prefect.io/api/latest/utilities/context.html)。
> 

还有些要注意的。

>
> **修改Prefect提供的上下文的提醒**
> 
> 由于Prefect内部使用某些上下文在flow和task运行逻辑期间跟踪元数据，因此修改Prefect提供的上下文键值可能会产生意想不到的后果。通常建议避免覆盖API文档中描述的保留键名。
>
> 与此相关的一个例外是与时间戳相关的键，例如**prefect.context.today**。用户可能希望在每个flow运行实例中修改此上下文，以实现信息回填，其中各个flow运行实例在时间序列数据的子集上执行。
> 

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/execution.html)
- [联系译者](https://github.com/listen-lavender)
