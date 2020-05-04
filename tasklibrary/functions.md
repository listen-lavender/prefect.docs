FunctionTask能将任何普通函数转换为任务，带有输入和输出。用户通常会通过Prefect的@task装饰器定义函数为task：

````Python
@task
def add_one(x):
    return x + 1

# roughly equivalent to:
FunctionTask(fn=lambda x: x + 1)
````

### FunctionTask

一个方便的task生成方式是函数式的在任意可调用的运行方法创建Task实例。这也是Prefect的@task装饰器的底层实现。

[API参考文档](https://docs.prefect.io/api/latest/tasks/function.html#prefect-tasks-core-function-functiontask)

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/api/latest/tasks/function.html)
- [联系译者](https://github.com/listen-lavender)
