Prefect引入了灵活的**map/reduce**模型，用于动态生成执行并行task。

经典的**map/reduce**是一个功能强大的两阶段编程模型，可用于在收集和处理所有结果（reduce阶段）之前分配和生成并行化作业（map阶段）。

典型的**map/reduce**设置需要三件事：

 - 可迭代的输入
 - 一次在单个项目上运行的**map**功能
 - 一次处理一组项目的**reduce**功能

例如，我们可以使用**map/reduce**来获取数字列表，将它们全部加一，然后求和：

````Python
numbers = [1, 2, 3]
map_fn = lambda x: x + 1
reduce_fn = lambda x: sum(x)

mapped_result = [map_fn(n) for n in numbers]
reduced_result = reduce_fn(mapped_result)
assert reduced_result == 9
````

## Prefect方法

Prefect的**map/reduce**版本比经典实现灵活得多。

映射task时，Prefect会为其输入数据的每个元素自动创建task的实例。task实例仅应用于该元素。这意味着映射的task实际上代表许多单个task实例的计算。

如果正常（非映射的）task依赖于映射task，则Prefect会自动应用规约操作以收集映射结果并将其传递给下游task。

但是，如果一个映射task依赖于另一个映射task，则Prefect不会规约上游结果。而是将第n个上游子实例连接到第n个下游子实例，从而创建独立的并行管道。

这是前面的示例改写成Perfect flow：

````Python
from prefect import Flow, task

numbers = [1, 2, 3]
map_fn = task(lambda x: x + 1)
reduce_fn = task(lambda x: sum(x))

with Flow('Map Reduce') as flow:
    mapped_result = map_fn.map(numbers)
    reduced_result = reduce_fn(mapped_result)

state = flow.run()
assert state.result[reduced_result].result == 9
````

> 
> **动态生成的task实例是一等公民的task**
>
> 即使用户没有明确创建它们，映射task的task实例也是一等公民的Prefect task。它们可以执行正常task可以执行的任何操作，包括成功，失败，重试，暂停或跳过。
> 

## 简单映射（map）

最简单的Prefect映射函数接收一个task，并将其应用于其输入的每个元素。

例如，如果我们定义一个将数字加10的task，则可以将该task简单地应用于列表的每个元素：

````Python
from prefect import Flow, task

@task
def add_ten(x):
    return x + 10

with Flow('simple map') as flow:
    mapped_result = add_ten.map([1, 2, 3])
````

运行flow时，**mapped_result** task的结果将为[11，12，13]。

## 迭代映射（map）

由于**mapped_result**只不过是具有可迭代结果的task，因此我们可以立即将其用作另一轮映射的输入：

````Python
from prefect import Flow, task

@task
def add_ten(x):
    return x + 10

with Flow('iterated map') as flow:
    mapped_result = add_ten.map([1, 2, 3])
    mapped_result_2 = add_ten.map(mapped_result)
````

运行此flow时，**mapped_result_2** task的结果将为[21、22、23]，这是两次应用映射函数的结果。

> 
> 无需规约（reduce）
> 
> 即使我们观察到**mapping_result**的结果是一个列表，除非用户需要，否则Prefect不会应用规约步骤来收集该列表。在此示例中，我们不需要整个列表（我们只需要它的每个元素），因此没有发生规约。这两个映射的task生成了三个完全独立的管道，每个管道包含两个task。
> 

## 规约（reduce）

如果非映射task需要映射结果，Prefect会自动将映射结果收集到列表中。因此，所有需要规约操作映射task结果的用户都将映射task的结果提供给task！

````Python
from prefect import Flow, task

@task
def add_ten(x):
    return x + 10

@task
def sum_numbers(y):
    return sum(y)

with Flow('reduce') as flow:
    mapped_result = add_ten.map([1, 2, 3])
    mapped_result_2 = add_ten.map(mapped_result)
    reduced_result = sum_numbers(mapped_result_2)
````

在此示例中，**sum_numbers**从**mapped_result**接收了自动规约的结果列表。它适时地计算总和：66。

## 未映射的输入

当task映射到其输入上时，它将保留相同的调用签名和参数，但会在输入上进行迭代以生成其task实例。有时，我们不想迭代其中一个输入，也许它是一个常量值，或者是需要的完整列表。为此，Prefect提供一个方便的**unmapped()**函数。

````Python
from prefect import Flow, task, unmapped

@task
def add(x, y):
    return x + y

with Flow('unmapped inputs') as flow:
    result = add.map(x=[1, 2, 3], y=unmapped(10))
````
This map will iterate over the x inputs but not over the y input. The result will be [11, 12, 13].

The unmapped function can be applied to any number of input arguments. This means that a mapped task can depend on both mapped and reduced upstream tasks seamlessly.

该映射将遍历输入参数x，但不会遍历输入参数y。结果将是[11，12，13]。

未映射的函数可以应用于任意数量的输入参数。这意味着映射的task可以无缝地依赖于映射的和规约的上游task。

## 映射task的状态行为

每当映射task被下游task规约时，Prefect都会将映射task子级视为该下游的task的输入。这意味着，除其他外，触发器函数将应用于所有映射的子级，而不是映射的父级。

如果规约task具有**all_successful** task，但是映射的子级之一失败了，则规约task的触发器将变成失败。这与已手动创建映射的子级代并将其传递给规约task的行为相同。跳过状态也会发生类似的行为。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/mapping.html)
- [联系译者](https://github.com/listen-lavender)

