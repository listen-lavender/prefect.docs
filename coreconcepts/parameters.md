参数（Parameter）是一种特殊task，它们在运行flow时可以接收用户输入。默认值设置是可选的。

````Python
from prefect import task, Flow, Parameter

@task
def print_plus_one(x):
    print(x + 1)

with Flow('Parameterized Flow') as flow:
    x = Parameter('x', default = 2)
    print_plus_one(x=x)

flow.run(parameters=dict(x=1)) # prints 2
flow.run(parameters=dict(x=100)) # prints 101
flow.run() #prints 3
````

> 
> **参数（Parameter）名称**
> 
> 尽管参数（Parameter）可以具有任何名称，但是flow中只能有一个具有该名称的参数。这是通过要求Parameter的slug（唯一标识）与其名称相同来实现的。
> 

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/parameters.html)
- [联系译者](https://github.com/listen-lavender)
