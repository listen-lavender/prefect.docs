## 日志

日志记录是任何生产环境的关键方面。Prefect公开了一系列用于自定义日志行为的配置设置和实用函数。

### 添加日志

可能想添加其他日志的最常见位置是自定义task中。可以在两个位置访问task的logger，具体取决于创建task的方式：

 - 如果task是作为**Task**基类的子类实现的，则**self.logger**属性包含**Task**基类的logger
 - 如果task是通过@task装饰器实现的，则可以从上下文访问logger：logger=prefect.context.get("logger")

### 配置

Prefect用户配置文件提供了一种在本地运行时轻松更改显示日志的方式。在用户配置文件中，添加具有以下结构的日志记录部分：

````bash
[logging]
# The logging level: NOTSET, DEBUG, INFO, WARNING, ERROR, or CRITICAL
level = "INFO"

# The log format
format = "[%(asctime)s] %(levelname)s - %(name)s | %(message)s"
````

另外，可以设置以下环境变量：

````bash
export PREFECT__LOGGING__LEVEL="INFO"
export PREFECT__LOGGING__FORMAT="[%(asctime)s] %(levelname)s - %(name)s | %(message)s"
````

### 添加处理器

除了更改日志格式外，还可以通过在执行之前直接与logger对象进行交互来使日志满足更多需要。例如，可以将新的处理器添加到logger中（注意日志处理器允许将日志发送到多个可配置的目标位置）。

这可以通过位于**prefect.utilities.logging**中的**get_logger**实用程序轻松完成。可以通过不带任何参数的**get_logger()**来访问根logger。注意task logger和flow logger分别与名称为“task”和“flow”的logger相关联。

## 例子

让我们来看一个基本的例子。首先，我们将在本地计算机上创建一个虚拟Web服务器，并将日志发布到：

````bash
# spins up a local webserver running at http://0.0.0.0:8000/
python3 -m http.server
````

接下来，创建一个logger处理器，并将该处理器添加到task的日志对象中。

````Python
import logging
import requests

import prefect
from prefect import task, Flow
from prefect.utilities.logging import get_logger


class MyHandler(logging.StreamHandler):
    def emit(self, record):
        requests.post("http://0.0.0.0:8000/", params=dict(msg=record.msg))


@task(name="Task A")
def task_a():
    return 3


@task(name="Task B")
def task_b(x):
    logger = prefect.context.get("logger")
    logger.debug("Beginning to run Task B with input {}".format(x))
    y = 3 * x + 1
    logger.debug("Returning the value {}".format(y))
    return y


with Flow("logging-example") as flow:
    result = task_b(task_a)


# now attach our custom handler to Task B's logger
task_logger = get_logger("Task")
task_logger.addHandler(MyHandler())


if __name__ == "__main__":
    flow.run()
````

如果我们将此代码存储在名为**logging_example.py**的文件中，则可以使用环境变量更新日志格式和级别，并开始以下流程运行：

````bash
export PREFECT__LOGGING__LEVEL="DEBUG"
export PREFECT__LOGGING__FORMAT="%(levelname)s - %(name)s | %(message)s"

python logging_example.py
````

像往常一样，我们应该看到日志发布到**stdout**。但是，如果导航到运行Web服务器的窗口，则还会看到类似以下内容：

````bash
127.0.0.1 - "POST /?msg=Beginning+to+run+Task+B+with+input+3 HTTP/1.1" 501
127.0.0.1 - "POST /?msg=Returning+the+value+10 HTTP/1.1" 501
````

我们看到501状态代码，因为我们的网络服务器未实现任何路由。

## 进一步

可以通过此示例进一步为日志创建真正的存储解决方案来。此外如果在Prefect Cloud上运行，则可以选择Prefect存储日志，并通过方便的GraphQL接口访问日志！

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/advanced_tutorials/custom-logs.html)
- [联系译者](https://github.com/listen-lavender)
