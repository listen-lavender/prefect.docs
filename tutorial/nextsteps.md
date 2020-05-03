本节简单介绍Prefect能如何增强工作流行为。但是还有很多的Prefect特性不能覆盖到。

## Task日志

教程里面的task通过Python内置**print()**函数展示信息：

````Python
@task
def load_reference_data(ref_data):
    print("saving reference data...")

    db = aclib.Database()
    db.update_reference_data(ref_data)
````

相应的，task中已经有一个**Logger**对象可供使用：

````Python
@task
def load_reference_data(ref_data):
    logger = prefect.context.get("logger")
    logger.info("saving reference data...")

    db = aclib.Database()
    db.update_reference_data(ref_data)
````

> 
> 阅读有关日志的更多信息：
> 
>  - [使用Prefect内置日志API](https://docs.prefect.io/core/concepts/logging.html#logging)
>  - [定制化日志](https://docs.prefect.io/core/advanced_tutorials/custom-logs.html)
> 

## 丰富的存储handler

在这个特定的航班数据ETL示例中，我们没有明确的需要在过程的每步中保持中间结果。但是，Prefect提供**ResultHandler**抽象使用户能够将每个task返回的结果持久化存储到所选的存储中：

````Python
from prefect.engine.result_handlers import LocalResultHandler

handler = LocalResultHandler(dir="./my-results")

with Flow("Aircraft-ETL", result_handler=handler) as flow:
    ...

flow.run()
````

现在，flow执行后，各个task生成的所有结果都将写到./my-results目录中。Prefect提供支持许多的结果处理handler，以将结果持久化存储到常见的存储选项中，例如S3，Google Cloud Storage，Azure Storage等。

> 
> 阅读有关结果处理handler的更多信息：
> 
>  - [结果处理handler概念](https://docs.prefect.io/core/concepts/results.html#results-and-result-handlers)
>  - [task检查](https://docs.prefect.io/core/concepts/persistence.html#checkpointing)
>  - [Prefect内置提供的结果处理handler](https://docs.prefect.io/api/latest/engine/result_handlers.html)
> 

## 先进的控制结构

在本教程中，我们可以更进一步，要求获取多个Y机场的X半径内的所有航班数据。在这种情况下，mapping是解决问题的最简单方法，为每个所需的机场调用**extract_live_data** task。

> 
> 阅读有关控制结构的更多信息：
> 
>  - [Mapping](https://docs.prefect.io/core/concepts/mapping.html#mapping)
>  - [Task Looping](https://docs.prefect.io/core/examples/task_looping.html#task-looping)
>  - [Signaling](https://docs.prefect.io/core/getting_started/next-steps.html#signals)
> 

## 现成的task库

在我们的教程中，task始终是用户定义的函数，但是Prefect提供开箱即用task的库。这是flow使用**ShellTask​​s**运行任意命令的示例：

````Python
from prefect import Flow
from prefect.tasks.shell import ShellTask
 
shell = ShellTask()
with Flow("Simple Pipeline") as flow:
   flow.chain(
       shell(command='pip install -r requirements.txt'),
       shell(command='black --check .'),
       shell(command='pytest .'),
   )
 
flow.run()
````

task库包含与Kubernetes，GitHub，Slack，Docker，AWS，GCP等的集成！

接下来，Prefect还能做什么呢？

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/tutorial/07-next-steps.html#ready-made-task-library)
- [联系译者](https://github.com/listen-lavender)
