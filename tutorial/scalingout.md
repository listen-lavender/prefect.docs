> 
> 跟随终端演示：
> 
> ````bash
> cd examples/tutorial
> python 06_parallel_execution.py
> ````
> 

让我们调整flow以将其task分配到[Dask](https://dask.org/)集群上并行执行。听起来可能要做很多，但实际上这是迄今为止最简单的调整：

````Python
from prefect.engine.executors import DaskExecutor

# ...task definitions...

# ...flow definition...

flow.run(executor=DaskExecutor())
````

这将启动系统上的[Local Dask Cluster](https://distributed.dask.org/en/latest/local-cluster.html)以并行化task。如果你已经在其他地方部署Dask集群，则可以通过在**DaskExecutor**构造函数中指定地址参数来使用该集群：

````Python
flow.run(
    executor=DaskExecutor(
        address='some-ip:port/to-your-dask-scheduler'
        )
    )
````

此外，只要提供的对象实现**Executor**接口（即Submit，Map和Wait函数，等同于Python的**current.futures.Executor**接口），可以定制化**Executor**以用于任何Prefect flow。这样可算是没有使用的天花板！

接下来，Prefect还能做什么呢？

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/tutorial/06-parallel-execution.html)
- [联系译者](https://github.com/listen-lavender)
