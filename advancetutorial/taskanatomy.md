> 
> 深入了解的有关Prefect task的所有信息以及更多信息。
> 

## 基础

简单来说，Prefect task代表一个单独的工作单元。例如，以下所有条件都可以作为Prefect task：

 - 查询数据库
 - 格式化字符串
 - 玩转一个Spark集群
 - 在Slack上发送消息
 - 执行Kubernetes作业
 - 创建一个简单的字典

除了执行某些操作外，task还可以选择性接收输入和返回输出。所有这些运行时逻辑都存在Task类的**.run()**方法内。

> 
> **如果可以用Python编写，则可以转换成Prefect task**
> 
> 调用task的**.run()**方法时，它将作为Python代码执行。因此，如果可以在Python中编写此操作代码，则可以将其放入Prefect task的**.run()**方法中。
> 
> 但是，仍然有一些注意事项：
> 
>  - 如果使用超时，则有时会依赖于多进程/多线程，这可能会干扰判断你的task会需要多大资源
>  - 确保了解[task输入和输出](https://docs.prefect.io/core/advanced_tutorials/task-guide.html#task-inputs-and-outputs)的可能限制
> 

一般而言，有两种首选的方法来创建自己的Prefect task：在函数上使用@task装饰器，或对Prefect Task类进行子类化。通过编写一个将两个数字加在一起的自定义task来分别查看每种task示例。

### 子类化Task基类

当需要可重用的、可参数化的task模型（例如，将自定义task添加到[Prefect Task库](https://docs.prefect.io/core/task_library/)中）时，子类化Task基类是创建task的首选方法。所有运行时逻辑都在task的**.run()**方法内部实现：

````Python
from prefect import Task


class AddTask(Task):
    def run(self, x, y):
        return x + y

add_task = AddTask(name="Add")
add_task # <Task: Add>
````

在这里，我们创建了一个名为“Add”的Prefect task，它接收两个输入（称为x和y），并返回一个输出。注意必须实例化自定义Task类，以创建正确的Prefect task对象。

### @task装饰器

通过编写自定义Python函数并使用@task装饰器，我们可以完全避免面向对象的风格：

````Python
from prefect import task

@task(name="Add")
def add_task(x, y):
    return x + y

add_task # <Task: Add>
````

在这里，我们创建与上述task等效的结果，@task装饰器本质上为我们装配此函数作为基础Task类的**.run()**方法的执行逻辑。以下是一些观察结果：

 - @task装饰器会自动实例化一个Prefect task
 - 所有[task初始化关键字参数](https://docs.prefect.io/api/latest/core/task.html#task-2)都可以在装饰器设置
 - 如果选择不提供名称，则函数名称将用作task名称

> 
> **调用签名不能是任意的**
> 
> 在实现Prefect task时，无论是通过子类化还是装饰器，对task的调用签名都有一些较小的限制：
> 
> 首先，Prefect task的所有参数最终都被视为关键字参数。这意味着不允许使用顺序参数（通常写为*args）。
> 
> 除了禁止（*args）外，以下关键字是保留关键字，并且不能由自定义Prefect task使用：
> 
>  - **upstream_tasks**：用于指定不交换数据的上游依赖关系
>  - **mapped**：映射的内置实现
>  - **task_args**：在函数式API中创建实例时覆盖task属性的一种方式
>  - **flow**：工作流的内置实现
> 
> 最后，Prefect必须能够检查函数签名才能应用这些规则。某些函数的签名无法检查，包括内置函数和许多numpy函数。要使用它们，将它们包装在普通的Python task中：
> 
> ````Python
> @task
> def prefect_bool(x):
>     """
>     `prefect.task(bool)` doesn't work because `bool` is
>     a `builtin`, but this wrapper function gets around the restriction.
>     """
>     return bool(x)
> ````
> 
> 如果违反任何这些限制，则在创建task时会立即引发错误，并产生通知。
> 

### 有用的Prefect上下文

有时，在task的运行时逻辑中包含额外自省信息会很有用。这些信息是通过Prefect的**Context**对象提供的。**prefect.context**是一个类似于字典的对象（对键具有属性访问权限），其中包含有用的运行时信息。例如，tasklogger存储在Prefect Context中，用于向task添加自定义日志。

````Python
import prefect

@prefect.task
def log_hello():
    logger = prefect.context["logger"]
    logger.info("Hello!")
````

请注意，上下文仅在flow运行时填充。在flow运行之外测试task时要意识到这一点很重要。有关Prefect Context中可用数据的完整列表，请参阅[API文档](https://docs.prefect.io/api/latest/utilities/context.html)。有关上下文如何工作的更多信息，请参阅相关的[概念文档](https://docs.prefect.io/core/concepts/execution.html#context)。注意上下文有一个优雅的**.get**方法来访问不保证存在的键。

## 执行task

大多数用户希望在flow之外运行其task以测试其逻辑是否合理。有几种不同的本地调试task的方式，其复杂性各不相同：

### 调用task的**.run()**方法

这是测试task的运行时逻辑的最直接方法。使用上面的**add_task**（在此阶段初始化创建方法无关紧要），我们可以：

````Python
add_task.run(1, 2) # returns 3
add_task.run(x=5, y=-10) # returns -5
add_task.run(x=[1, 2], y=[3, 4]) # returns [1, 2, 3, 4]
````

这样，测试和运行task就像测试任何类方法一样简单，并且可以以最少的Prefect开销达到目标。

### 使用TaskRunner

如果有兴趣遵从Prefect的可见性测试task，那么使用TaskRunner可能会有所帮助。Prefect TaskRunner设计用于处理运行时逻辑周围的所有表面区域，例如：

 - 上游task完成了吗？
 - 上游task是否处于由task触发器指定的适当状态？
 - 是task映射，并且应该产生自己的多个实例吗？
 - Task是否在Cloud上运行，是否应在Cloud数据库中设置其状态？
 - task是否有需要调用的结果处理器或状态处理器？
 - task运行结束后应处于什么状态？

为了了解这一点，让我们运行一个无需输入的task。

````Python
from prefect import task
from prefect.engine import TaskRunner

@task
def number_task():
    return 42

runner = TaskRunner(task=number_task)
state = runner.run()

assert state.is_successful()
assert state.result == 42
````

在这种情况下，TaskRunner实际上不返回task的**.run()**方法的值，而是返回Prefect Success状态，该状态的返回值存储在**result**属性中。

现在，我们可以开始做更多有趣的事情，例如提供上游状态来测试触发逻辑：

````Python
from prefect import Task
from prefect.core import Edge
from prefect.engine.state import Failed

# the base Task class provides a useful stand-in for a no-op Task
upstream_edge = Edge(Task(), number_task)
upstream_state = Failed(message="Failed for some reason")

state = runner.run(upstream_states={upstream_edge: upstream_state})

# our Task run is in a TriggerFailed state
# and its corresponding result is the exception that was caught
assert state.is_failed()
assert isinstance(state.result, Exception)
````

这可以使你快速了解Prefect的实施细节，但是这里重要的一点是，使用TaskRunner让我们可以单独测试task上的所有Prefect设置（没有flow的上下文支持）。

如果有兴趣进一步进行此操作，则以下API参考文档可能会有用：
 - [Edge](https://docs.prefect.io/api/latest/core/edge.html)
 - [TaskRunner](https://docs.prefect.io/api/latest/engine/task_runner.html#taskrunner-2)
 - [State](https://docs.prefect.io/api/latest/engine/state.html#state-2)
 - [Trigger](https://docs.prefect.io/core/concepts/execution.html#triggers)

> 
> **flow也有执行者
> 
> 除了TaskRunner之外，Prefect还具有FlowRunner的概念，FlowRunner是响应flow及其task状态进行单次OK的对象。Task和Flow执行者的**.run()**方法上的关键字参数对于测试flow很有用。
> 

## task输入和输出

在许多工作流系统中，仅允许单个task报告有关其状态的最少信息集（例如，“完成”、“失败”或“成功”）。 在Prefect中，我们鼓励Task实际交换更丰富的信息，包括“任意”数据对象。

但是，可以限制哪些task可以作为输入接收和作为输出返回。特别是，Prefect flow的执行器是DaskExecutor，它允许用户在集群上执行task。由于每台机器运行的都是不同的Python进程，因此Dask必须将Python对象转换为可以在不同进程之间共享的字节码。因此，在Prefect平台周围传递的数据必须与此序列化协议兼容。具体来说，task必须对可通过[cloudpickle库](https://github.com/cloudpipe/cloudpickle)进行对象的序列化操作。

### cloudpickle可以序列化什么？

 - 大多数数据结构，包括列表，集合和词典
 - 函数（包括lambda函数）
 - 大多数类，只要它们的属性是可序列化的

### cloudpickle不能序列化什么？

 - 迭代器
 - 文件对象
 - 线程锁
 - 深度递归类

### 如何确定某些东西是否可序列化？

导入cloudpickle，并尝试序列化/反序列化：

````Python
import cloudpickle

cloudpickle.dumps(None)
# b'\x80\x04\x95\x02\x00\x00\x00\x00\x00\x00\x00N.'

obj = cloudpickle.loads(cloudpickle.dumps(None))
assert obj is None

custom_generator = (x for x in range(5))
cloudpickle.dumps(custom_generator)
# TypeError: can't pickle generator objects
````

实际中，这些细微和技术上的限制在task库中为很多设计决策提供信息（我们将很快看到）。此外，毫无疑问，用户有时会遇到这种情况，重要的是要认识到违反此约束时出现的错误类型。另外，在编写自己的流程时，在设计task以及它们如何传达信息时，请记住这一点。

### 添加task到flow

现在已经编写了task，是时候将它们添加到flow中。坚持使用上面创建的**number_task**，让我们使用Prefect的命令式API将此task添加到flow中：

````Python
from prefect import Flow

f = Flow("example")
f.add_task(number_task)

print(f.tasks) # {<Task: number_task>}
````

到目前为止，一切都很好，我们的flow现在仅包含一个task。我们如何使用函数式API将单个task添加到flow？在这种情况下，我们必须对task执行一些操作以将其注册到flow。通常，如果满足以下条件之一，则task将通过函数式API自动添加到flow：

 - 在flow上下文中调用task
 - 该task作为另一个task的依赖被调用

在这种情况下，因为只有一个task而没有依赖关系，所以我们必须借用实例化Task：

````Python
with Flow("example-v2") as f:
    result = number_task()

print(f.tasks) # {<Task: number_task>}
````

> 
> **调用task创建实例**
> 
> 使用上面示例，你可能会惊讶地发现：
> 
> ````Python
> number_task in f.tasks # False
> result in f.tasks # True
> ````
> 
> 这是因为调用Prefect task实际上会创建该task的实例并返回该实例。这允许使用优雅的Python函数，例如使用不同的输入多次调用task以产生不同的输出。
> 
> 每当创建实例时，都可以选择通过特殊的**task_args**关键字覆盖任何/所有task属性：
> 
> ````Python
> with Flow("example-v3") as f:
>     result = number_task(task_args={"name": "new-name"})
> 
> print(f.tasks) # {<Task: new-name>}
> ````
> 
> 这对于覆盖task触发器，标签，名称等很有用。
> 

为了了解其中的一些细微差别，让我们使用上面创建的**add_task** task来制作一个更复杂的示例。首先，使用命令式API的**set_dependencies**方法：

````Python
f = Flow("add-example")

f.set_dependencies(add_task, keyword_tasks={"x": 1, "y": 2})
print(f.tasks) # {<Task: add_task>}
````

现在，让我们将注意力转移到函数式API上，并完全重现上述示例：

````Python
with Flow("add-example-v2") as f:
    result = add_task(x=1, y=2)

print(f.tasks) # {<Task: add_task>}

add_task in f.tasks # False
result in f.tasks # True
````

我们在这里看到**add_task**的实例已创建并添加到flow中。

> 
> **Auto-generation of Tasks**
> 
> 注意Prefect将自动生成tasks来表示Python集合；因此，例如，将字典添加到flow实际上会为字典的键及其值创建task。
> 
> ````Python
> from prefect import task, Flow
> 
> @task
> def do_nothing(arg):
>     pass
> 
> with Flow("constants") as flow:
>     do_nothing({"x": 1, "y": [9, 10]})
> 
> flow.tasks
> 
> #  <Task: Dict>,
> #  <Task: List>, # corresponding to [9, 10]
> #  <Task: List>, # corresponding to the dictionary keys
> #  <Task: List>, # corresponding to the dictionary values
> #  <Task: do_nothing>}
> 
> 对于深度嵌套的Python集合而言，这可能会很麻烦。为了防止发生这种细粒度的自动生成，您始终可以将Python对象包装在常量task中：
> 
> ````Python
> from prefect.tasks.core.constants import Constant
> 
> with Flow("constants") as flow:
>     do_nothing(Constant({"x": 1, "y": [9, 10]}))
> 
> flow.tasks
> 
> # {<Task: Constant[dict]>, <Task: do_nothing>}
> ````
> 
> 常量task告诉Prefect将其输入视为原始常量，而无需进一步的自省。
> 

最后说明如何/何时在函数式API中将task添加到flow中，让我们将这些值提升为具有默认值的Prefect参数：

````Python
from prefect import Parameter

with Flow("add-example-v3") as f:
    # we will instantiate these Parameters here
    # but note that does NOT add them to the flow yet
    x = Parameter("x", default=1)
    y = Parameter("y", default=2)

    result = add_task(x=x, y=y) # <-- the moment at which x, y are added to the Flow

print(f.tasks) # {<Task: add_task>, <Parameter: y>, <Parameter: x>}

add_task in f.tasks # False
result in f.tasks # True

x in f.tasks # True
y in f.tasks # True
````

我们的最终观察结果是将参数按原样添加到flow中（未制作实例）。仅当调用task时才创建实例。

> 
> **可以使用函数式API指定非数据依赖task。**
> 
> 一个常见的误解是函数式API不允许用户指定无数据依赖的task（taskB应该在taskA之后运行，但不交换任何数据）。实际上，使用task的调用方法的特殊**upstream_tasks**关键字参数可以实现此目的。这是一个例子：
> 
> ````Python
> from prefect import task, Flow
> 
> @task(name="A")
> def task_A():
>     # does something interesting and stateful
>     return None
> 
> 
> @task(name="B")
> def task_B():
>     # also does something interesting and stateful
>     return None
> 
> with Flow("functional-example") as flow:
>     result = task_B(upstream_tasks=[task_A])
> ````
> 

## 映射

映射是Prefect的特性，它允许用户动态生成给定task的多个实例，以响应上游task的输出。映射task后，Prefect会自动为其输入数据的每个元素创建task的实例。一个task实例应用于一个元素。这意味着映射的task实际上代表许多单个下游task的计算。

如果正常（未映射）task依赖于映射task，则Prefect会自动应用规约合并操作以收集映射结果并将其传递给下游task实例。

但是，如果一个映射task依赖于另一个映射task，则Prefect不会规约合并上游结果。而是将第n个上游task实例节点连接到第n个下游task实例节点，从而创建独立的并行管道。

看一个简单的例子：

````Python
import prefect
from prefect import task, Flow

@task
def add_one(x):
    """Operates one number at a time"""
    return x + 1

@task(name="sum")
def sum_numbers(y):
    """Operates on an iterable of numbers"""
    result = sum(y)
    logger = prefect.context["logger"]
    logger.info("The sum is {}".format(result))
    return result


## Imperative API
flow = Flow("Map Reduce")
flow.set_dependencies(add_one, keyword_tasks={"x": [1, 2, 3, 4]}, mapped=True)
flow.set_dependencies(sum_numbers, keyword_tasks={"y": add_one})

## Functional API
with Flow("Map Reduce") as flow:
    mapped_result = add_one.map(x=[1, 2, 3, 4])
    summed_result = sum_numbers(mapped_result)
````

每当运行此flow时，都会看到以下日志：

````bash
[2019-07-20 21:35:00,968] - INFO - prefect.Task | The sum is 14
````

注意每个task实例都是一等公民的Prefect task。这意味着它们能执行正常task可以执行的任何操作，包括成功、失败、重试、暂停或跳过。

### 并非所有内容都必须映射

假设我们有一个task，接受许多关键字参数，并且我们只想映射这些参数的子集（部分参数）。在这种情况下，可以使用Prefect的未映射容器来指定不应映射到的那些输入：

````Python
from prefect import task, Flow
from prefect.utilities.tasks import unmapped

@task
def add_one(x, y):
    """Operates one number at a time"""
    return x + y

with Flow("unmapped example") as flow:
    result = add_one.map(x=[1, 2, 3], y=unmapped(5))
````

运行此flow时，仅x关键字将被映射；y将保持固定不变，值为5。

### 映射task不交换数据

实际上，Prefect的API为了提供最大的灵活性，可以响应上游输出而生成task的多个映射实例，而无需数据依赖。例如：

````Python
from prefect import task, Flow

@task
def return_list():
    return [1, 2, 3, 4]

@task
def print_hello():
    print("=" * 30)
    print("HELLO")
    print("=" * 30)

with Flow("print-example") as flow:
    result = print_hello.map(upstream_tasks=[return_list])
````

当此flow运行时，我们将看到四个print语句，每一个print语句对应**return_list**输出的每个值。当你具有多个映射层和复杂的依赖关系结构时，此模式（与未映射的容器结合使用）有时很有用。另外，值得注意的是，如果**return_list**返回一个空列表，则不会创建任何task实例，并且**print_hello** task会正常运行。

> 
> **顺序影响**
> 
> 注意顺序在Prefect映射中很重要。在内部，Prefect通过映射索引追踪映射的上下游task，映射索引描述了其在映射中的位置。 因此，我们不建议在没有顺序的情况下映射字典、集合或其他任何东西。
> 

## 编写task最佳实践

我们可以将上述大多数规则提炼成一些设计Prefecttask的建议。

### 注意task属性

与[task输入和输出](https://docs.prefect.io/core/advanced_tutorials/task-guide.html#task-inputs-and-outputs)最终需要序列化的方式完全相同，task属性也需要序列化。实际上，如果将flow部署到Prefect Cloud，则无论是否将flow提交到Dask集群，task属性都将需要通过cloudpickle传递。

例如，以下task设计可能很糟糕，无法部署到Cloud，也不能使用基于Dask的执行器运行：

````Python
import google.bigquery

class BigQuery(Task):
    def __init__(self, *args, **kwargs):
        self.client = bigquery.Client() # <-- not serializable
        super().__init__(*args, **kwargs)

    def run(self, query: str):
        self.client.query(query)
````

在这种情况下，在**.run()**方法之外实例化Client将阻止该task被cloudpickle序列化（相反，应在**.run()**方法中实例化Client）。

### 无状态

除了避免无法序列化的属性外，还应该避免依赖有状态的task。例如，在设计Task类时，依赖于存储状态的属性将不会表现出预期的行为：

````Python
class Bad(Task):
    def __init__(self, *args, **kwargs):
        self.run_count = 0
        super().__init__(*args, **kwargs)

    def run(self):
        self.run_count += 1
        return self.run_count
````

同样，依赖某些全局状态最终会导致麻烦：

````Python
global_state = {"info": []}

@task
def task_one():
    global_state["info"].append(1)

@task
def task_two():
    global_state["info"].append(2)
````
    
### 通过Task类设计业务task的模板

每当要提供可配置的task模板时，使用[子类化方式](https://docs.prefect.io/core/advanced_tutorials/subclassing-the-task-class)来设计task都是有益的，该task的默认值既可以在初始化时设置，也可以在运行时覆盖。 例如，让我们更改上面创建的add_task，为y提供默认值：

````Python
class AddTask(Task):
    def __init__(self, *args, y=None, **kwargs):
        self.y = y
        super().__init__(*args, **kwargs)

    def run(self, x, y=None):
        if y is None:
            y = self.y

        return x + y


add_task = AddTask(y=0)

with Flow("add-with-default") as f:
    result_one = add_task(x=1)
    result_two = add_task(x=0, y=10)
````

我们发现这种设置默认值的模式非常普遍，可以选择在运行时将其覆盖，因此我们创建一个[实用程序函数](https://docs.prefect.io/api/latest/utilities/tasks.html#prefect-utilities-tasks-defaults-from-attrs)以最小化样板。此外，子类化允许在一个地方组织自定义类方法。

> 
> **一定调用父task的初始化方法**
> 
> 任何时候将Task子类化，请确保调用父task初始化方法！这样可以确保自定义task能被Prefect识别。此外，我们强烈建议始终允许将字典关键字参数（即**kwargs）传递给task的**.__init__()**方法。这样确保仍然可以设置诸如task标签，自定义名称，结果处理器等内容。
> 

### 状态处理器

状态处理器是一种有用的方式，可以为task创建自定义hook，并响应task所经历的每个状态。状态处理器的常见用例包括：

 - 发送有关失败/成功的通知
 - 拦截结果并对其进行操作
 - 调出外部系统以确保task准备就绪，可以运行（如果具有隐式的非Prefect依赖项）

状态处理器是具有附加到task的特定调用签名的函数。例如：

````Python
from prefect import task
import requests


def send_notification(obj, old_state, new_state):
    """Sends a POST request with the error message if task fails"""
    if new_state.is_failed():
        requests.post("http://example.com", json={"error_msg": str(new_state.result)})
    return new_state


@task(state_handlers=[send_notification])
def do_nothing():
    pass
````

每当此task运行时，它将经历许多状态更改。例如，从**Pending**到**Running**。如果在管道中的任何步骤转换为**Failed**状态，则将发送POST请求，其中包含来自失败状态的错误信息。（注意task的**on_failure**关键字参数是状态处理器的便捷接口，该状态处理器仅在失败的状态上被调用）。此外，可以根据需要将任意数量的状态处理器附加到task，并且将按照提供的顺序来调用它们。

> 
> **状态处理器失败导致task表现为失败**
> 
> 由于状态处理器被视为Prefect中自定义task逻辑的一部分，因此，如果状态处理器由于任何原因引发错误，则task将会处于**Failed**状态。因此，我们强烈建议仔细考虑如何实现状态处理器/回调。
> 

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/advanced_tutorials/task-guide.html)
- [联系译者](https://github.com/listen-lavender)

