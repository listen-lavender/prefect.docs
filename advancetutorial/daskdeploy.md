> 
> 如何在分布式Dask群集中运行Prefect flow？
> 

## Dask执行器

Prefect暴露一组“执行器”概念抽象，它们代表task应如何运行以及在何处运行的逻辑（例如，它应在子进程中运行还是在另一台计算机上运行？）。 在我们的案例中，想使用Prefect的DaskExecutor将task运行提交到已知的Dask集群。这提供了一些开箱即用的关键优势：

 - Dask可以为每一次运行管理所有“flow内调度”，例如在尝试运行下游task之前确定上游task何时完成。这使用户能够以不会使任何中枢调度器超载的方式部署具有许多微型task的flow
 - Dask处理许多资源决策，例如向哪个worker提交作业
 - Dask处理worker/调度器的通信，例如在worker之间序列化数据

## flow例子

如果想在Dask上实验，可以通过以下CLI命令安装Dask Distribution并启用带有两个Dask worker的本地集群：

````bash
> dask-scheduler
# Scheduler at: tcp://10.0.0.41:8786

# in new terminal windows
> dask-worker tcp://10.0.0.41:8786
> dask-worker tcp://10.0.0.41:8786
````
> 
> **工作窃取**
> 
> 强烈建议在执行Prefect流时关闭Dask集群中的Dask工作窃取。 这可以通过Dask集群中的环境变量来完成：
> 
> ````bash
> DASK_DISTRIBUTED__SCHEDULER__WORK_STEALING="False" # case sensitive
> ````
> 
> 在极少数情况下，工作窃取机制可能导致task尝试运行两次。
> 

集群启动并运行后，让我们在该集群上部署运行非常基本的流程。从分布式文档中重新调整了此示例的用途：

````Python
from prefect import task, Flow
import datetime
import random
from time import sleep


@task
def inc(x):
    sleep(random.random() / 10)
    return x + 1


@task
def dec(x):
    sleep(random.random() / 10)
    return x - 1


@task
def add(x, y):
    sleep(random.random() / 10)
    return x + y


@task(name="sum")
def list_sum(arr):
    return sum(arr)


with Flow("dask-example") as flow:
    incs = inc.map(x=range(100))
    decs = dec.map(x=range(100))
    adds = add.map(x=incs, y=decs)
    total = list_sum(adds)
````

到目前为止，我们所要做的就是定义一个flow，其中包含有关如何运行这些task的所有必要信息，尚未执行任何自定义task代码。为了使此flow在Dask集群上运行，我们要做的就是为**flow.run()**方法提供一个经过适当配置的DaskExecutor：

````Python
from prefect.engine.executors import DaskExecutor

executor = DaskExecutor(address="tcp://10.0.0.41:8786")
flow.run(executor=executor)
````

如果恰好安装了bokeh，则可以访问Dask Web UI并在flow开始运行后在Dashboard查看正在处理的task！

> 
> **高级Dask配置**
> 
> 要通过Dask Gateway与安全的、经过生产加固的Dask集群进行接口连接，可能需要向DaskExecutor提供TLS详细信息。这些细节可以在创建时在GatewayCluster对象上找到：
> 
> ````Python
> from dask_gateway import Gateway
> from prefect.engine.executors import DaskExecutor
> 
> # ...flow definition...
> 
> gateway = Gateway()
> cluster = gateway.new_cluster()
> executor = DaskExecutor(address=cluster.scheduler_address, security=cluster.security)
> flow.run(executor=executor)
> ````
>   
> 另外，可以手动提供TLS详细信息：
> 
> ````Python
> from dask_gateway.client import GatewaySecurity
> from prefect.engine.executors import DaskExecutor
> 
> # ...flow definition...
> 
> security = GatewaySecurity(tls_cert="path-to-cert", tls_key="path-to-key")
> executor = DaskExecutor(address="a-scheduler-address", security=security)
> flow.run(executor=executor)
> ````
> 

## 下一步

让我们再进一步：将调度计划附上该flow，并将其打包，以便可以将其指向我们选择的任何Dask集群，而无需编辑定义该flow的代码。为此，我们将首先在上面的脚本中添加一个main方法，以便可以通过CLI执行该方法：

````Python
def main():
    from prefect.schedules import IntervalSchedule

    every_minute = IntervalSchedule(start_date=datetime.datetime.utcnow(),
                                    interval=datetime.timedelta(minutes=1))
    flow.schedule = every_minute
    flow.run() # runs this flow on its schedule


if __name__ == "__main__":
    main()
````

注意对**flow.run()**的调用中未指定执行器。这是因为可以通过环境变量设置默认执行器（有关其工作方式的更多信息，请参见Prefect的文档）。 假设我们将其保存在名为dask_flow.py的文件中，我们现在可以如下指定执行器和Dask调度器地址：

````bash
> export PREFECT__ENGINE__EXECUTOR__DEFAULT_CLASS="prefect.engine.executors.DaskExecutor"
> export PREFECT__ENGINE__EXECUTOR__DASK__ADDRESS="tcp://10.0.0.41:8786"

> python dask_flow.py
````

现在，此flow将在本地Dask集群上每分钟运行一次，直到终止进程。

## 进一步

通过将flow存储在Docker容器中，并使用优秀的dask-kubernetes项目将它与Dask一起部署在Kubernetes上，将本示例提升到一个新的层次！

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/advanced_tutorials/dask-cluster.html)
- [联系译者](https://github.com/listen-lavender)
