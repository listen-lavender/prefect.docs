Prefect的丰富状态系统允许在运行时动态更改基础DAG结构进行工作流变形，同时仍提供基础工作流所有的原本保证，单个task可以具有自定义重试设置，交换数据，激活通知等。

以前，task映射允许用户在运行时将可并行化的for循环提升为一等公民的并行管道。task循环提供了几乎相同的好处，除了对于需要使用while循环模式的情况。

## 斐波那契数列的例子

作为一个鼓励例子，让我们实现一个Prefect flow，该流程计算小于给定数M的最大斐波那契数。对于此示例，我们将考虑斐波那契数的计算是一个只能通过API访问的完全黑盒。我们的实现将通过迭代查询下一个数字来进行，直到达到一个大于M的数字为止。将这种模式表达为工作流引出一个独特的问题，因为我们在运行时之前不知道需要多少个task。为了说明，请考虑斐波那契task的以下实现：

````Python
import requests
from prefect import task


@task
def compute_large_fibonacci(M):
    n = 1
    fib = 1

    while True:
        next_fib = requests.post(
            "https://nemo.api.stdlib.com/[email protected]/", data={"nth": n}
        ).json()
        if next_fib > M:
            break
        else:
            fib = next_fib
            n += 1

    return fib
````

假设我们的互联网连接短暂中断并且此task失败。哪个n值引起了故障，如何从n重新启动？为了捕获此类间歇性问题，我们可能考虑为该task提供重试设置，但这将导致整个循环重新运行。如果将每个n值单独都视为具有其自身设置（重试，通知等）的Prefect Task，这是否理想？

### task循环介绍

目标是将上述while循环的每个单独迭代提升到其自己的Prefect task，该Prefect task可以自己重试和处理。我们无法提前知道需要多少次迭代。幸运的是，Prefect以一等公民的方式支持这种动态模式。

从一个迭代到下一个迭代需要传递两段信息，循环计数以及循环有效负载（可以在各个迭代之间累积或更改）。为了传达此信息，我们将使用Prefect LOOP信号，如下所示：

````Python
import requests
from datetime import timedelta

import prefect
from prefect import task
from prefect.engine.signals import LOOP


@task(max_retries=5, retry_delay=timedelta(seconds=2))
def compute_large_fibonacci(M):
    # we extract the accumulated task loop result from context
    loop_payload = prefect.context.get("task_loop_result", {})

    n = loop_payload.get("n", 1)
    fib = loop_payload.get("fib", 1)

    next_fib = requests.post(
        "https://nemo.api.stdlib.com/[email protected]/", data={"nth": n}
    ).json()

    if next_fib > M:
        return fib  # return statements end the loop

    raise LOOP(message=f"Fib {n}={next_fib}", result=dict(n=n + 1, fib=next_fib))
````

像所有Prefect信号一样，LOOP信号接受message和result关键字。但是，在这种情况下，结果将包含在**task_loop_result**键的上下文中，并且可在下一次循环迭代时可用（task_loop_count也可用，但此处不需要该信息）。

### 拼凑完整

让我们以刚刚学到的内容为基础，将各部分放到一个实际的Prefect flow中。

````Python
from prefect import Flow, Parameter

with Flow("fibonacci") as flow:
    M = Parameter("M")
    fib_num = compute_large_fibonacci(M)
````

作为最佳实践，我们选择将M提升为Prefect参数，而不是对其值进行硬编码。这样，我们可以尝试使用较小的值，并最终在不重新编译Flow的情况下动态设置该值。

借助我们构建的flow，让我们计算小于100的最大斐波纳契数，然后计算1000！

````Python
flow_state = flow.run(M=100)
print(flow_state.result[fib_num].result) # 89

flow_state = flow.run(M=1000)
print(flow_state.result[fib_num].result) # 987
````

如果在本地运行此flow，将看到大量的日志，它们对应于**compute_large_fibonacci** task的每个迭代，如果这些单个迭代中的任何一个失败，则延迟2秒后task将重试，而成功的迭代无需再次运行！


## 进一步

Prefect中的循环是一等公民的操作。因此，它可以与映射结合使用，以实现真正的动态工作流！

````Python
with Flow("mapped-fibonacci") as mapped_flow:
    ms = Parameter("ms")
    fib_nums = compute_large_fibonacci.map(ms)
````

需要明确的是，我们编写的flow似乎只有两个task，不知道我们可能提供多少个M值，也不知道每个M都需要多少次迭代！

````Python
flow_state = mapped_flow.run(ms=[10, 100, 1000, 1500])
print(flow_state.result[fib_nums].result) # [8, 89, 987, 987]
````

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/advanced_tutorials/task-looping.html)
- [联系译者](https://github.com/listen-lavender)
