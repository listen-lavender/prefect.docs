前文介绍了创建task和组合task成flow。Prefect的函数式和声明式API轻松达到上述目的，但生成结果仍是相当原始的数据管道。本文将探讨Prefect如何启用高级机制协助工作流实现复杂业务逻辑。

可以查看有关构建实际数据应用的[完整教程](https://docs.prefect.io/core/tutorial/01-etl-before-prefect.html)。

## 触发器（Trigger）

到目前为止，我们已经能处理遵循“正常”数据管道规则的task：只要task成功，它的下游task就会开始运行。

经常需要建立task互相响应复杂的flow，一个简单的例子是清理职责的task是必须执行的，即使前面的task失败了。假设第一个task是创建Spark集群，第二个task是提交一个job到集群，第三个task解散清理集群。在正常业务管道世界中，如果第二个task失败了，则第三个task不会执行。这就导致Spark集群永不停止，一直占用物理资源。

Prefect引入称之为触发器的抽象概念来应对这种场景，会根据一组上游task状态调用设置的下游task触发器，触发task实例执行。如果触发函数未生效，则该对应的task不会触发执行。Prefect具有多种内置触发器，包括**all_successful**（默认所有task的触发器）、**all_failed**、**any_successful**、**any_failed**，甚至是一个有趣特别的触发器，叫作**manual_only**。

让我们用适当的用显而易见的伪代码来设置Spark集群流程：

````Python
from prefect import task, Flow

@task
def create_cluster():
    cluster = create_spark_cluster()
    return cluster

@task
def run_spark_job(cluster):
    submit_job(cluster)

@task
def tear_down_cluster(cluster):
    tear_down(cluster)


with Flow("Spark") as flow:
    # define data dependencies
    cluster = create_cluster()
    submitted = run_spark_job(cluster)
    tear_down_cluster(cluster)

    # wait for the job to finish before tearing down the cluster
    tear_down_cluster.set_upstream(submitted)
````

当我们运行此flow时，只要Spark作业成功运行，一切业务正常。但是如果作业失败，**run_spark_job**task实例将设置为**Failed**状态。因为**tear_down_cluster**task上的默认触发器是**all_successful**，所以该task被触发失败（相当于当前task Failed的特殊情况，引起下游task通知失败的不是运行时错误，是触发器条件）。

为了解决该类问题，只需在定义task时调整触发器即可。当前情况，使用**always_run**触发器，它是**all_finished**的别名：

````Python
@task(trigger=prefect.triggers.always_run)  # one new line
def tear_down_cluster(cluster):
    tear_down(cluster)
````

所有其他代码保持不变。现在**tear_down_cluster**task实例一定会产生并执行，即使是上游task失败了。

> 
> **task永远不会在上游task完成之前执行**
> 
> Prefect永远不会在上游task完成之前运行task。这就是为什么**all_finished**和**always_run**触发器是同义词。
> 

## flow状态表征的关联task（Reference Task）

前面，我们使用触发器来确保**tear_down_cluster**task始终运行。最后一个task实例一定会成功，让我们选择始终得到成功运行的结果。实际上这是一个工程实践问题，因为工作流管理系统根据最后一个task的状态来设置整个flow的状态。于是，我们要运行的作业失败了，最后一个task**tear_down_cluster**也将运行并成功，或者相反。

对业务来说，这是一个严重问题：工作流管理系统和一般的工作流具有不一定与业务逻辑一致的语义

这说明了一个重要的问题：工作流管理系统和flow的语义不一定和业务逻辑一致。其实，用户是希望如果作业没有运行，业务流程表征应该是失败了，同时工作流在系统层面（包括清理task）成功结束！

Prefect引入了一种称之为**状态关联task**的机制来解决问题。在所有task都进入**Finished**状态之前，flow将保持**Running**状态。那时flow实际通过将**all_successful**触发器应用于其状态关联task来决定自己的最终状态。默认情况下，flow状态表征的关联task是最后一个task，但开发者可以将其指定为flow的任意一个或多个task，例如如下代码。


````Python
flow.set_reference_tasks([submitted])
````

现在flow的状态取决于业务task实例会发生什么。如果业务失败，flow状态会表征失败，即使最后一个清理职责的task实例成功了。

## 信号（Signal）

Prefect状态系统允许用户通过触发器和关联task来设置高级控制。每一个task都会根据**.run()**函数的执行进入最终状态：如果该函数正常完成，task进入**Success**状态；如果函数遇到错误，task进入**Failed**状态。贴心的是，Prefect通过**signal**机制给用户提供了操作task状态的细粒度控制。

信号（signal）是告诉Prefect引擎应将task立即移至特定状态的方式。

例如，任何时候，task可以在其**.run**方法正常完成之前发出SUCCESS信号表示该task成功执行。相对的，如果用户希望对task生成的**Failed**状态进行更谨慎的验证，就可以发出FAILED信号。


还有其他更有趣的signal可用。例如，发出RETRY信号将使task立即自行重试，发出PAUSE信号将使task进入暂停状态，直到task被明确唤醒为止。

另一个有用的signal是SKIP信号。一个task被跳过，意味着它的下游task也会被跳过，除非专门设置了skip_on_upstream_skip = False。这意味着用户在条件不满足时，可以设置整个工作流分支都能被跳过。

> 
> **SKIP被当做SUCCESS对待。**
> 
> 当一个task被跳过时，通常将其视为执行成功。这是因为仅当用户特意引入跳过处理逻辑时，task才会跳过。因此结果符合用户的设计意图。
> 

在task的**.run()**方法中，可以编码发出一个信号：

````Python
from prefect.engine import signals

@task
def signal_task(message):
    if message == 'go!':
        raise signals.SUCCESS(message='going!')
    elif message == 'stop!':
        raise signals.FAIL(message='stopping!')
    elif message == 'skip!':
        raise signals.SKIP(message='skipping!')
````

如果传递给该task的message是"stop！"，则该task将失败。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/getting_started/next-steps.html)
- [联系译者](https://github.com/listen-lavender)
