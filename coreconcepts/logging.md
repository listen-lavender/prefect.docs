## 日志配置

Prefect具有多种从task生成日志的方式。

## Prefect内置Logger

Prefect的日志级别由**prefect.config.logging.level**控制，默认为**INFO**。但是设置只会影响Prefect内置logger对象，而不是Python全局的logger对象。

要更改默认日志级别，请设置环境变量**PREFECT__LOGGING__LEVEL=DEBUG**。

## task记录日志

要访问Prefect配置的logger对象，请使用**prefect.utilities.logging.get_logger**（<名称可选>）。如果你不提供名称，将收到Prefect的根logger对象。

### task类

在task类里面使用**self.logger**：

````Python
class MyTask(prefect.Task):
    def run(self):
        self.logger.info("An info message.")
        self.logger.warning("A warning message.")
````

### task装饰器

在**@task**装饰器生成的task的实例运行里，从上下文获取logger：

````Python
@task
def my_task():
    logger = prefect.context.get("logger")

    logger.info("An info message.")
    logger.warning("A warning message.")
````

> 
> **确保仅在task运行时访问上下文**
> 
> 运行task时，将填充Prefect上下文信息。因此，仅应在task运行时访问上下文的logger对象。例如，将无法工作的一个实例：
> 
> ````Python
> logger = prefect.context.get("logger")
> 
> @task
> def my_task():
> 
>     logger.info("An info message.")
>     logger.warning("A warning message.")
> ````
> 

### 日志输出

Prefect的task本身就支持将标准输出的输出转发到logger对象。可以通过在task上设置**log_stdout=True**来启用此功能。

````Python
@task(log_stdout=True)
def log_my_stdout():
    print("I will be logged!")
````

### 其他Logger库

当使用日志配置时，可以设置许多其他库（例如**boto3**和**snowflake.connector**）发出自己的内部日志。你甚至可以为团队使用具有相同日志功能提供内部共享库。

如果通过标准日志库执行此操作，则可以执行以下操作：

````Python
import logging
import sys
for l in ['snowflake.connector', 'boto3', 'custom_lib']:
    logger = logging.getLogger(l)
    logger.setLevel('INFO')
    log_stream = logging.StreamHandler(sys.stdout)
    log_stream.setFormatter(LOG_FORMAT)
    logger.addHandler(log_stream)
````

鉴于Prefect已经提供了一种配置本地日志和云日志记录的方法，你可以使用这些第三方的日志库，以使它们继承Prefect日志记录配置以在本地流传输并显示在云端。

为了使其正常工作，你需要提供一个列表至**prefect.config.logging.extra_loggers**。

这是TOML配置的样子：

````bash
[logging]
# Extra loggers for Prefect log configuration
extra_loggers = "['snowflake.connector', 'boto3', 'custom_lib']"
````

当做环境变量配置:

````bash
export PREFECT__LOGGING__EXTRA_LOGGERS="['snowflake.connector', 'boto3', 'custom_lib']"
````

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/logging.html)
- [联系译者](https://github.com/listen-lavender)
