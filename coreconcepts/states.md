## 概览

状态（state）是Prefect的“货币”。有关task和flow的所有信息都是通过丰富的状态对象传输的。虽然不需要了解的状态系统设计的详细信息就能使用Prefect，但可以利用它来增强工作流。

在任何时候，都可以通过检查task或flow的当前状态或历史状态来了解任何需要了解的信息。例如，一个状态可以说明：

 - 该task在一小时内被调度进行第三次运行
 - task成功了以及产生了什么数据
 - task已暂停并等待用户恢复它
 - task的输出已缓存，并将在以后的运行中复用
 - task因超时而失败
 - 等等很多

通过处理相对少量的task状态，Prefect工作流可以驾驭这种复杂性。

> 
> **只有运行实例才有状态**
> 
> 我们经常提到flow或task的状态，其实是指flow运行或task运行的状态。flow和task是描述系统行为的模板。只有当我们运行系统时，它才会处于一种状态。因此，尽管我们可能将task称为**Running**或**Success**，但实际上是指某task的特定实例处于该状态。
> 

## 状态对象（State object）

所有State对象都具有三个重要特征：类型，消息和结果。一些状态对象还有有其他字段，例如，**Retrying**状态具有一个字段指示何时重试task。

### 消息（Message）

State messages usually explain why the state was entered. In the case of Failed states, they often contain the error message associated with the failure.

状态消息通常会解释为什么输入状态。在“失败”状态下，它们通常包含与失败相关的错误消息。

````Python
Success(message="The task succeeded!")
````

或者

````Python
Pending(message="This task is waiting to start")
````

### 结果（Result）

状态结果携带与状态关联的数据。对于task**Success**状态，这是task产生的数据。对于**Failed**状态，通常是Python Exception对象导致的失败。

> 
> **失败的结果**
> 
> 因为所有状态对象都有一个结果字段，所以这意味着task可以处理失败的上游task的结果。这很令人吃惊，但功能强大。例如，在失败的task之后运行的task可以查看上游task失败的结果，以了解失败的确切原因。跳过task之后的task可能会收到一条说明为什么要跳过该上游task的消息。
> 
> 需要明确的是：默认触发器将不会运行失败task之后的task，因此用户将必须选择手工设置要执行。
> 

## 状态类型（State types）

共有三种主要状态：**Pending**、**Running**和**Finished**。flow和task通常按照该状态顺序发生，可能不止一次。每个状态类型都有很多子类。例如，**Scheduled**和**Retrying**均为**Pending**的子状态，**Success**和**Failed**均为**Finished**的子状态。

在执行流水线的每个阶段，当前状态决定将要采取的行动。例如，如果尝试以**Success**状态运行task，它将退出流水线，因为**Finished**状态永远不会重新运行。如果尝试以**Retrying**状态运行task，则仅在该状态的重试计划时间已经到了，task才会再次进行。这样，状态将携带Prefect引擎用来做出有关工作流逻辑决策的所有关键信息。

> 
> **元状态**
> 
> 实际上，存在第四种状态，称为**MetaState**，但它不会影响执行流水线。取而代之的是，Prefect使用元状态来扩展现有状态的附加信息。例如，两个元状态为“Submitted”和“Queued​​”。这些用于以使原始状态可恢复的方式“包装”其他状态。例如，可以将**Scheduled**状态置于**Submitted**状态，用来说明执行已提交，但是引擎需要原始的**Scheduled**来执行运行时逻辑。通过将**Scheduled**状态与**Submitted**元状态包装在一起，而不是替换它，引擎能恢复所需的原始信息。
> 

## 状态处理器和回调

通常希望在某个事件发生时（例如，task失败）采取措施。为此，Prefect提供了**state_handlers**。flow和task可能具有一个或多个状态处理器函数，只要task的状态发生更改，就会调用这些函数。状态处理程序的签名为：

````Python
    def state_handler(obj: Union[Flow, Task], old_state: State, new_state: State) -> State:
        return new_state
````

每当task的状态更改时，处理器都将被调用，传入task本身，旧（上一个）状态和新（当前）状态参数。处理器必须返回一个状态对象，该对象用作task的新状态。这提供了对某些状态做出反应甚至修改它们的机会。如果提供多个处理器，则将依次调用它们，上一个返回的状态变为下一个的**new_state**值。

例如，当一个task重试时，发送通知：

````Python
def notify_on_retry(task, old_state, new_state):
    if isinstance(new_state, state.Retrying):
        send_notification() # function that sends a notification
    return new_state

task_that_notifies = Task(state_handlers=[notify_on_retry])
````

## flow状态变迁

flow在相对少一点的几个状态之间变迁。

> 
> Scheduled -> Running -> Success / Failed
> 

通常，flow实例运行以**Scheduled**状态开始，该状态指示实例应何时开始运行。**Scheduled**状态是**Pending**状态的子类。开始运行时，它将变迁为**Running**状态。最后，当所有终端task完成时，流程将移至**Finished**状态。根据状态关联task的状态，最终状态将为**Success**或**Failed**。

## task状态变迁

task状态变迁会有多种状态，因为它们的执行可以导致许多不同的结果。通常，它们将反复从**Pending**状态变迁为**Running**状态，直到最终进入**Finished**状态。

task的状态变迁有几种常见模式：

### Success

> 
> Pending -> Running -> Success
> 

最常见的task的状态变迁模式是在创建task实例后处于就绪状态，开始后是运行状态，结束是成功状态。

### Failure

> 
> Pending -> Running -> Failed
> 

第二种常见的task状态变迁模式是运行时遇到错误，结束是失败状态。

#Failure (before running)

> 
> Pending -> TriggerFailed
> 

如果task实例的触发器没有返回True，然后task实例在运行之前就会失败，结束是触发失败状态。

### Retry

> 
> Failed -> Retrying -> Running
> 

从失败状态开始，正确配置的task可以自动进入重试状态。经过指定的时间后，task将重新移回运行状态。

### Skip (while running)

> 
> Running -> Skipped
> 

用户可以通过发出**SKIP**信号来使task跳过自己。跳过的状态通常被视为成功状态，但有一些其他警告。例如，默认情况下，已跳过task下游task将自动跳过自身。

### Skip (before running)

> 
> Pending -> Skipped
> 

如果上游task被跳过，并且task具有**skip_on_upstream_skip = True**（默认设置），则该task在将在运行之前的就绪状态就会自动跳过自身。这使用户可以绕过整个task链，而无需配置每个task。

## 特殊task状态变迁

task还具有一些不常见但重要的状态变迁模式。


### Pause transition (while running)

> 
> Running -> Paused -> Resume -> Running
> 

用户可以通过发出**PAUSE**信号来暂停task。暂停后，必须将task置于恢复状态才能重回运行状态。暂停和恢复都是**Pending**的子类状态。

### Pause transition (before running)

> 
> Pending -> Paused
> 

如果task具有manual_only触发器，它将进入暂停状态。这将在运行之前发生，并且用户将必须手工启动这些task才能继续。

### Mapped transition

> 
> Running -> Mapped
> 

如果task映射到其输入，则它将在运行后进入**Mapped**状态。这表明它没有做任何工作，而只是动态生成task实例来执行映射的函数。子状态可以通过Mapped.map_states访问。**Mapped**是一种完成状态，它是**Success**的子类状态。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/states.html)
- [联系译者](https://github.com/listen-lavender)
