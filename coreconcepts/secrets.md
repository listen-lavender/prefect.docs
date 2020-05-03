## 概览

通常，工作流需要使用敏感数据：API密钥，密码，令牌，凭据等。作为最佳实践，绝不应将此类信息硬编码到工作流的源代码中，因为代码本身的保护级别不同。 此外，不应通过Prefect参数提供敏感数据，因为与许多task一样，**Parameter**概念抽象会存储其结果。

Prefect提供一种称为“Secret”的概念抽象来维护敏感数据。

 - **Secret**，和Prefect参数一样也是一种特殊的task，在flow中维护敏感数据。与常规task不同，**Secret** task被设计用来在运行时访问敏感数据，并使用特殊的**ResultHandler**来确保不存储结果
 - **prefect.client.secrets** API，提供用于维护敏感数据的接口。该API可以在task不可用的地方使用，包括通知，状态处理器和结果处理器

> 
> **Secret保密**
> 
> 尽管Prefect采取措施确保**Secret**对象不会泄露敏感数据，但其他task可能并不那么小心。一旦将敏感数据加载到flow中，就可以将其用于任何目的。在处理敏感数据时，请务必谨慎。
> 

## 机制

### 本地上下文

Secret基类首先通过**prefect.context.secrets**检查本地上下文中的敏感数据。这对于本地测试很有用，因为可以通过设置环境变量PREFECT__CONTEXT__SECRETS__FOO来将敏感数据添加到上下文中，该值绑定对应到上下文中的**secrets.foo**（如果操作系统区分大小写，则为secrets.FOO）。

### Prefect云环境

如果在本地上下文中未找到敏感数据并且**config.cloud.use_local_secrets=False**，则Secret基类会向Prefect Cloud API查询存储的敏感数据。只有经过身份验证的Prefect Cloud Agent才能成功进行此调用。

### 环境变量

EnvVarSecret类从环境变量读取敏感数据。

````Python
from prefect import task, Flow
from prefect.tasks.secrets import EnvVarSecret

@task
def print_value(x):
    print(x)

with Flow("Example") as flow:
    secret = EnvVarSecret("PATH")
    print_value(secret)

flow.run() # prints the value of the "PATH" environment variable
````

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/secrets.html)
- [联系译者](https://github.com/listen-lavender)


