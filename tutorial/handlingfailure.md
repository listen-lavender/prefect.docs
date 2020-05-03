> 
> 跟随终端演示：
> 
> ````bash
> cd examples/tutorial
> python 04_handle_failures.py
> ````
> 

## 如果失败了...

现在有一个有效的ETL工作流，让我们采取进一步的措施以确保其健壮性。**extract_** task正在用外部API发出Web请求，以获取数据。如果API短期不可用怎么办？或者单个请求由于未知原因超时？Prefect task可以在失败时重试；让我们将其添加到我们的**extract_** task中：

````Python
from datetime import timedelta
import aircraftlib as aclib
from prefect import task, Flow, Parameter


@task(max_retries=3, retry_delay=timedelta(seconds=10))
def extract_reference_data():
    # same as before ...
    ...


@task(max_retries=3, retry_delay=timedelta(seconds=10))
def extract_live_data(airport, radius, ref_data):
    # same as before ...
    ...

````

这是一种简单能帮助flow仅在指定的task中妥善处理瞬态错误的措施。现在，如果有任何失败的Web请求，将最多进行3次尝试，每次尝试之间等待10秒。

> 
> 更多处理失败的方法。
> 
> Prefect提供了其他机制来启用围绕故障的专门行为：
> 
>  - task触发器：根据上游task运行的状态有选择地执行task。
>  - state处理器: 当flow或者task改变状态时候，提供一个Python函数做响应处理，控制一切过程变化!
>  - 通知: 如对状态变化感兴趣要获取Slack通知，或组合task触发器和邮件task​​。
> 

接下来，自定义计划来周期性的调度执行flow。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/tutorial/04-handling-failure.html)
- [联系译者](https://github.com/listen-lavender)
