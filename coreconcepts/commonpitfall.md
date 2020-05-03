## 有状态的task

### 在Task类的.run()方法持久化数据

task的**.run()**方法中存储的数据对以后的运行不可用。

尽管这在本地测试期间可能会起作用，但你应该假设每次Prefect task运行时，它都是在全新的环境中进行的。即使task运行两次，它也将无法访问在上一次运行中设置的本地状态。

所以，你不应该这么做：

````Python
class BadCounterTask(Task):

    def __init__(self, **kwargs):
        self.counter = 0
        super().__init__(**kwargs)

    def run(self):
        self.counter += 1 # this won't have the intended effect
        return self.counter
````

## 序列化

### task结果和输入

Prefect中的大多数基础通信模式都要求通信对象可以序列化为某种格式（通常是JSON或二进制）。例如，当task返回有效数据时，结果会使用**cloudpickle**序列化以与其他Dask工作者进行通信。因此，你应尝试确保创建并提交给Prefect的所有结果和对象都可以适当地序列化。 通常，（反）序列化错误可能是非常难以理解的。

### 完整的flow

此外，在创建和提交新的flow以进行部署时，应检查以确保可以正确序列化和反序列化你的flow。为了帮助解决此问题，Prefect具有内置的**is_serializable**实用程序函数，使你可以放心地处理flow：

````Python
from prefect.utilities.debug import is_serializable

is_serializable(my_flow) # returns True / False
````

注意这并不是保证，只是帮助尽早发现问题。

> 
> 提示
> 
> 一个好的经验法则是，你依赖的每个对象/函数都应显式附加到flow的某个属性（例如task），或者可以导入Docker打包的某些库。
> 

更健壮的开发流程是检查：flow可以序列化，部署能本地创建Docker存储，flow可以在容器中反序列化。有关如何执行此操作的详细信息，参见[本地调试](https://docs.prefect.io/core/advanced_tutorials/local-debugging.html)教程中的相应部分。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/common-pitfalls.html)
- [联系译者](https://github.com/listen-lavender)

