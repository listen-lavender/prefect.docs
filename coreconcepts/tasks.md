## 概览

一个task表示Prefect工作流的离散行为。

task类似函数：接收可选输入，执行操作并产生可选输出结果。实际中，创建task的最简单方法是装饰Python函数：

````Python
from prefect import task

@task
def plus_one(x):
    return x + 1
````

对于可能需要自定义更复杂的task，可以直接对**Task**类进行子类化扩展：

````Python
from prefect import Task

class HTTPGetTask(Task):

    def __init__(self, username, password, **kwargs):
        self.username = username
        self.password = password
        super().__init__(**kwargs)

    def run(self, url):
        return requests.get(url, auth=(self.username, self.password))
````

所有Task子类都必须具有run()方法。

> 
> **task可以独立运行**
> 
> 任何时候都可以调用task的**.run()**进行测试
> 
> ````Python
> plus_one.run(2)  # 3
> ````
> 

task支持各种功能参数，这些参数可以提供给**Task**类构造函数或**@task**装饰器。有关完整说明，请参阅[Task API文档](https://docs.prefect.io/api/latest/core/task.html)。

> 
> **task该设计成多大？**
> 
> 人们经常想知道每个task要编写多少代码。
> 
> Prefect鼓励**small task**：每个task都应代表工作流中的一个逻辑步骤。这样，Prefect可以更好地包含task失败。
> 
> 需要明确的是，没有什么可以阻止你将所有代码放在单个task中：Prefect照样能运行它！但是，如果任何代码出bug，则整个task将失败，需要从头开始重试。通过将代码分成多个相互依赖的task，可以轻松避免这种情况。
> 

## 重试

将代码放入Prefect task的最常见原因之一是在失败时自动重试。要启用重试，请将适当的**max_retries**和**retry_delay**参数传递给task：

````Python
# this task will retry up to 3 times, waiting 10 minutes between each retry
Task(max_retries=3, retry_delay=datetime.timedelta(minutes=10))
````

## 触发器

在执行Prefect task之前，它会通过**触发器函数**以决定是否应该开始运行它。触发器是接收上游task状态并在下游task应运行时返回True，否则返回False（或引发错误）的函数。如果task的触发器失败并且未引发更具体的错误，则该task将进入**TriggerFailed**状态，这是**Failed**状态的一种更具体的类型，表明task无法运行，但是由于触发器问题而不是task自身的原因。

> 
> **跳过被视为成功**
> 
> 在Prefect中，将跳过的task视为成功。这是因为仅在用户要求时才进行跳过，因此它们表示用户设计意图的“成功”执行。但是，默认情况下，跳过的task的下游task也会被跳过：除非跳过的task接收**skip_on_upstream_skip=False**，否则task的跳过状态会沿着链路传播。
> 

内置的触发器包括以下功能：

 - **all_successful**：这是默认触发器，并且仅在所有上游task成功后才允许task运行。
 - **all_failed**：当所有上游task失败后才允许task运行。
 - **any_successful**：至少有一个上游task成功才允许task运行。
 - **any_failed**：至少有一个上游task失败才允许task运行。
 - **all_finished**：只要所有的上游task运行结束，就允许task运行。这意味着下游task一定会运行，只是Prefect task会在上游task结束时才能执行。
 - **manual_only**：此触发器的独特之处在于不会让task运行。当manual_only触发器运行时，task将始终进入暂停状态。用户可以通过将其显式置为“恢复”状态来使这些task运行。因此，此触发器是将强制性中断引入工作流的有用方法。

尽管我们鼓励用户将自定义逻辑放在task的**.run()**方法中，但用户也可以提供具有以下签名的任何函数：

````Python
trigger_fn(upstream_states: Set[State]) -> bool
````

## 常数

如果为task提供非task输入，它将自动转换为**Constant**常量抽象概念。

````Python
from prefect import Flow, task

@task
def add(x, y):
    return x + y

with Flow('Flow With Constant') as flow:
    add(1, 2)

assert len(flow.tasks) == 1
assert len(flow.constants) == 2
````

Prefect将自动尝试将Python对象转换为常量，包括集合（如列表，元组，集合和字典）。如果生成的常量直接用作task的输入，则会在task图中进行优化独立出来，并将其存储在**flow.constants**字典。但是，如果常量已映射，则它将保留在依赖图中。

## 运算符

使用函数式API时，Prefect task支持基本的数学和逻辑运算符。例如：

````Python
import random
from prefect import Flow, task

@task
def a_number():
    return random.randint(0, 100)

with Flow('Using Operators') as flow:
    a = a_number()
    b = a_number()

    add = a + b
    sub = a - b
    lt = a < b
    # etc
````
These operators automatically add new tasks to the active flow context.

这些操作符自动将新task添加到flow上下文中。

> 
> **运算符验证**
> 
> 由于Prefect flow在创建时不会执行，因此Prefect无法验证是否将运算符应用于兼容类型。例如，您可以用产生整数的task表达式减去产生列表的task。这将在运行时发生错误，而不是在task定义期间。
> 

## 集合

使用功能性API时，Prefect task可以自动在集合中使用。例如：

````Python
import random
from prefect import Flow, task

@task
def a_number():
    return random.randint(0, 100)

@task
def get_sum(x):
    return sum(x)

with Flow('Using Collections') as flow:
    a = a_number()
    b = a_number()
    s = get_sum([a, b])
````

在这个场景，将自动创建一个列表task，以获取a和b的结果并将它们放入列表中。自动创建的task成为s的唯一上游依赖。

Prefect将对Python的列表，元组，集合和字典执行自动集合提取。

## 可索引

使用函数式API时，可以为Prefect task建立索引以检索特定结果。

````Python
from prefect import Flow, task

@task
def fn():
    return {'a': 1, 'b': 2}


with Flow('Indexing Flow') as flow:
    x = fn()
    y = x['a']
````

这会自动将接收x作为输入并尝试执行x['a']的**GetItem** task添加到flow，该task的结果存储为y。

> 
> **key验证**
> 
> 因为Prefect flow是在运行时执行，Prefect无法提前验证索引键是否可用。所以，Prefect将允许你按任意值索引任何task。如果flow实际运行时该键不存在，则会引发运行时错误。
> 

## 映射

有关更多详细信息，请参见[映射概念文档](https://docs.prefect.io/core/concepts/mapping.html)。

一般来说，Prefect的函数式API允许像函数一样调用task。

另外，可以调用**Task.map()**在其输入上自动映射task。Prefect将为输入的每个元素生成task的动态副本。如果你不希望输入被视为可迭代的（例如，你希望将其提供给每个动态副本），则只需使用Prefect的无映射装函数将其装饰即可。

````Python
from prefect import task, unmapped

@task
def add(x, y):
    return x + y

add.map(x=[1, 2, 3], y=unmapped(1))
````

映射可组合，从而能创建强大的动态管道：

````Python
z1 = add.map(x=[1, 2, 3], y=unmapped(1))
z2 = add.map(x=z1, y=unmapped(100))
````

另外，如果将映射task的结果传递给无映射task（或用作映射task的无映射输入），则其结果将收集在列表中。这允许透明但十分灵活的map/reduce功能。

## 标识符

### 名称

可以给task指定一个可选名称；如果未提供，它将从task的类名（或@task装饰器的函数名）中组合获取。名称纯粹是为了用户方便在各种可视化中表示task，也可以在查询有关特定task信息时使用。task名称没有限制，两个task可以具有相同的名称。

````Python
# assigning a name to a Task instance
t = Task(name="My Task")

# assigning a name to a decorated task function
@task(name="My Other Task")
def my_task():
    pass
````

### slug

与ID相似，因为同一flow中Prefect不允许两个task具有相同的slug。因此，slug可以用作可选的阅读直观的唯一标识符。如果未提供，它是自动生成的UUID。

````Python
Task(slug="my-task")
````

### 标签

有时，以多种方式组合各种task很有用。为此，Prefect提供了标签。标签既可以指定为关键字参数，也可以使用便捷的上下文管理器指定。上下文管理器可以嵌套。

````Python
from prefect import tags

with tags('red', 'blue'):
    t = Task()

assert t.tags == {'red', 'blue'}
````

### 检索task

在Prefect Cloud中查询task并应用一些基于标记的高级特性时，各种标识符属性最有用，可以将它们与flow的**.get_tasks()**函数一起在本地使用。这将返回与所有提供的参数匹配的所有task。例如：

````Python
flow.get_tasks(name='my-task')
flow.get_tasks(tags=['red'])
````

## 状态处理器

状态处理器允许用户提供在task变更状态时触发的自定义逻辑。例如，如果task失败，可以发送Slack通知，这个功能放这儿恰到好处！

状态处理器必须具有以下签名：

````Python
state_handler(task: Task, old_state: State, new_state: State) -> State
````

每当task状态改变时，都会触发处理器，并将该task本身，旧状态和新状态作为输入参数。处理器返回的状态用作task的新状态。

If multiple handlers are provided, they are called in sequence. Each one will receive the "true" old_state and the new_state generated by the previous handler.

如果提供了多个状态处理器，则会依次调用它们。每个处理器都会收到由上一个状态处理器传递的**true**、原始**old_state**和生成的**new_state**。

处理器也可以与**Flow**，**TaskRunner**和**FlowRunner**类关联。task级处理器首先被调用。

## 缓存

在输出结果将复用于以后的运行的情况下，可以缓存task。例如，可能要确保在生成报告之前数据库已加载，但是你可能不想每次运行flow时都运行**load task**。没问题，仅将**load task**缓存24小时，以后的运行将重用缓存的以前成功的输出结果。

更多详情，请参见执行的[相关文档](https://docs.prefect.io/core/concepts/execution.html#caching)。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/tasks.html)
- [联系译者](https://github.com/listen-lavender)
