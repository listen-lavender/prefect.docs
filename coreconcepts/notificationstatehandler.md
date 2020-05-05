告警，通知以及对task状态的动态响应是任何工作流工具的重要特性。使用Prefect原语，用户可以添加Prefect的触发器逻辑使得创建的指定task运行或失败后发送通知。这将起作用，但不能涵盖通知逻辑的更多细微用法（例如，如果task重试，则接收通知）。因此，Prefect引入了一种灵活的概念抽象，称为“状态处理器”，可以附加到各个task或flow。从高层次上讲，状态处理器是在附着对象的每次状态更改时调用的函数；它们可以用于在发生故障时发送警报，成功时发送电子邮件，或者根据新旧状态中包含的信息进行更细微的处理。

除了直接使用**state_handler**API外，Prefect还提供了更高级的装饰器来实现常见的用例方案，例如故障回调。

> 
> **Prefect状态**
> 
> 状态处理器与Prefect的“状态”概念抽象密切相关。我们建议先阅读有关的[状态概念文档](https://docs.prefect.io/core/concepts/states.html)，然后再继续深入下去。
> 

## 状态处理器

让我们从状态处理器的定义开始：

````Python
def state_handler(obj: Union[Task, Flow], old_state: State, new_state: State) -> Optional[State]:
    """
    Any function with this signature can serve as a state handler.

    Args:
        - obj (Union[Task, Flow]): the underlying object to which this state handler
            is attached
        - old_state (State): the previous state of this object
        - new_state (State): the proposed new state of this object

    Returns:
        - Optional[State]: the new state of this object (typically this is just `new_state`)
    """
    pass
````

如您所见，状态处理器API是一种在每次状态更改时执行任意Python代码的简单方法。

一个简单的例子将阐明其用法：

````Python
from prefect import Task, Flow

def my_state_handler(obj, old_state, new_state):
    msg = "\nCalling my custom state handler on {0}:\n{1} to {2}\n"
    print(msg.format(obj, old_state, new_state))
    return new_state

my_flow = Flow(name="state-handler-demo",
               tasks=[Task()],
               state_handlers=[my_state_handler])
my_flow.run()
````

忽略日志，应该输出：

````bash
Calling my custom state handler on <Flow: name=state-handler-demo>:
Scheduled() to Running("Running flow.")

Calling my custom state handler on <Flow: name=state-handler-demo>:
Running("Running flow.") to Success("All reference tasks succeeded.")
````

以完全相同的方式，我们可以将此状态处理器附加到单个task而不是flow：

````Python
t = Task(state_handlers=[my_state_handler])
flow = Flow(name="state-handler-demo", tasks=[t])
flow.run()
````

再次忽略日志，这应该输出：

````bash
Calling my custom state handler on <Task: Task>:
Pending() to Running("Starting task run.")

Calling my custom state handler on <Task: Task>:
Running("Starting task run.") to Success("Task run succeeded.")
````

归根结底，仅此而已！但是，API的简单性掩盖了此特性的许多可能的使用模式，这就是我们接下来要介绍的内容。

> 
> **注意**
> 
> 为了简单起见，在本文档的其余部分中，我们将重点介绍task状态处理器，但是我们讨论的所有内容均适用于flow。
> 

## 发送一个简单的通知

如我们上面的例子所示，拦截task状态并对其进行响应非常容易。实际上，我们前面的示例可以认为是一个简单的通知系统，在该系统中，我们将通知打印到stdout。让我们更进一步，编写一个通知程序，每当我们的task完成运行时，该通知程序就会向Slack发布更新。为了使此示例正常工作，需要传入Webhook进行Slack应用设置。如果您不使用Slack或没有应用，则无需担心，只需将Slack URL与你可能会访问的任何其他web服务交换即可。

````Python
import requests
from prefect import Task, Flow

def post_to_slack(task, old_state, new_state):
    if new_state.is_finished():
        msg = "Task {0} finished in state {1}".format(task, new_state)

        # replace URL with your Slack webhook URL
        requests.post("https://XXXXX", json={"text": msg})

    return new_state


t = Task(state_handlers=[post_to_slack])
flow = Flow(name="state-handler-demo", tasks=[t])
flow.run()
````

在这里，我们通过仅在task状态被视为完成时发送通知来响应状态。这包括**Success**状态和**Failed**状态，但不包括诸如**Retrying**或**Scheduled**状态。

> 
> **通知失败会使task变成失败**
> 
> 如果POST请求返回非200状态码，则可能引发错误。很好，但是要警告，Prefect将状态处理器视为task执行的组成部分，因此，如果在调用task的状态处理器时引发错误，则task实例运行将被中止，并且该task将被标记为**Failed**状态。
> 

关于第三方通知API的认证。

> 
> **状态处理器可以使用Prefect Secrets**
> 
> 大多数通知系统将需要某种形式的身份认证。不要失望，状态处理器可以像task一样查询使用Prefect Secrets。
> 

## **响应状态变化**

到目前为止，我们的大多数示例都会告知用户task何时会不会进入特定状态。Prefect的state对象包含丰富的数据，使我们可以发挥更大的创造力！让我们重新访问**post_to_slack**通知程序，让它在task进入**Retrying**状态时提醒我们，以及等待多久重试发生：

````Python
import requests
from prefect import Task, Flow

def post_to_slack(task, old_state, new_state):
    if new_state.is_retrying():
        msg = "Task {0} failed and is retrying at {1}".format(task, new_state.start_time)

        # replace URL with your Slack webhook URL
        requests.post("https://XXXXX", json={"text": msg})

    return new_state


t = Task(state_handlers=[post_to_slack])
flow = Flow(name="state-handler-demo", tasks=[t])
flow.run() # the notifier is never run
````

这例子使用**Retrying**状态的start_time属性向用户发出更多有用的信息的警告。

> 
> **创造性思考**
> 
> 因为Prefect允许task返回数据，所以实际上我们可以使状态处理器根据task的输出进行响应。更加有趣的是，任何Prefect状态都可以携带数据，其中包括**Failed**状态。
> 

让我们实现一个具有特殊失败模式的task。如果发生此故障模式，我们希望立即得到警告。

````Python
from prefect import task, Flow
from prefect.engine import signals

def alert_on_special_failure(task, old_state, new_state):
    if new_state.is_failed():
        if getattr(new_state.result, "flag", False) is True:
            print("Special failure mode!  Send all the alerts!")
            print("a == b == {}".format(new_state.result.value))

    return new_state


@task(state_handlers=[alert_on_special_failure])
def mission_critical_task(a, b):
    if a == b:
        fail_signal = signals.FAIL("a equaled b!")
        fail_signal.flag = True
        fail_signal.value = a
        raise fail_signal
    else:
        return 1 / (b - a)


with Flow(name="state-inspection-handler") as flow:
    result = mission_critical_task(1, 1)

flow.run()
# Special failure mode!  Send all the alerts!
# a == b == 1
````

注意我们能复用这种将数据附加到跨越多个task的**FAIL**信号，因此，在其他情况也可以复用此状态处理器。

## 使用多个状态处理器

你可能已经注意到**state_handlers**参数为复数形式，并可以接受列表。这是因为Prefect允许你根据需要将尽可能多的状态处理器附加到task上！此模式对于组合具有不同用例场景的状态处理器很有用（例如，用于失败的特殊处理器和用于重试的特殊处理器）。如果要在某些紧急情况下向你发出警报，它也很有用，你可以实现许多不同的状态处理器，以确保尽快收到警报。

> 
> **依次调用状态处理器**
> 
> 如果选择为一个task提供多个状态处理器，请注意将按照提供它们的顺序依次调用它们。
> 

## 高级状态处理器API

Prefect提供了用于从更小、更模块化的部分创建状态处理器的工具。特别是，位于**prefect.utilities.notifications**中的**callback_factory**helper的工具函数使你可以从两个简单的函数创建状态处理器，一个实现动作，另一个执行检查（或过滤）以确定何时应执行该动作。让我们使用此工具重新实现**post_to_slack**重试处理功能：

````Python
from prefect.utilities.notifications import callback_factory

def send_post(task, state):
    msg = "Task {0} failed and is retrying at {1}".format(task, state.start_time)
    requests.post("https://XXXXX", json={"text": msg})

post_to_slack = callback_factory(send_post, lambda s: s.is_retrying())
````

你可以检查该状态处理器的行为是否与我们实现的第一个相同。这种工厂方法允许用户使用简单的API来混合和匹配逻辑部分。

常见检查是状态是否为失败。因此，Prefect提供了高级的API来构造失败回调。特别是给定带签名的函数。

````Python
def f(obj: Union[Task, Flow], state: State) -> None
````

可以在task和flow中使用**on_failure**关键字参数来自动创建适当的状态处理器。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/notifications.html)
- [联系译者](https://github.com/listen-lavender)



