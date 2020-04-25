在本教程中，我们将使用Prefect来改进前面章节中ETL工作流的整体结构。

> 
> 跟随终端演示：
> 
> 教学代码一览
> ````bash
> cd examples/tutorial
> python 02_etl_flow.py
> ````
> 

## Extract,Transform,Load

Prefect的最小工作单元是Python函数。因此，我们的首要任务是将示例分解成函数。

可能想到的第一个问题是“函数应该有多大/小？”。一种简单的方法是显式地将工作分解为“提取”，“转换”和“加载”函数，如下所示：

````Python
def extract(...):
    # fetch all aircraft and reference data
    ...
    return all_data

def transform(all_data):
    # clean the live data
    ...

def load(all_data):
    # save all transformed data and reference data to the database
    ...
````

这比较函数式，但它仍然不能解决原来的代码中的一些问题：

 - 如果提取实时航班数据失败，已经获取的关联数据怎么办？
 - 如果存储数据库不可用，已经转换的数据怎么办？

这些要点突出了一个事实，即**.extract()**和**.load()**仍是任意范围的。在决定每个功能的大小时，这使我们有了一个经验法则：查看工作流在每个步骤所需的输入和输出数据。在我们的场景，关联数据和实时航班数据来自不同的来源，并分开存储。考虑到这一新见解，让我们进行更多重构：

````Python
def extract_reference_data(...):
    # fetch reference data
    ...
    return reference_data

def extract_live_data(...):
    # fetch live data
    ...
    return live_data

def transform(live_data, reference_data):
    # clean the live data
    ...
    return transformed_data

def load_reference_data(reference_data):
    # save reference data to the database
    ...

def load_live_data(transformed_data):
    # save transformed live data to the database
    ...
````

## 套用Prefect

现在我们有适当大小的函数，并且知道这些函数之间的关系，可以用Prefect封装我们的工作流。

### 第1步

使用**prefect.task**装饰需要Prefect调度执行的任何函数：

````Python
from prefect import task, Flow

@task
def extract_reference_data(...):
    # fetch reference data
    ...
    return reference_data

@task
def extract_live_data(...):
    # fetch live data
    ...
    return live_data

@task
def transform(live_data, reference_data):
    # clean the live data
    ...
    return transformed_data

@task
def load_reference_data(reference_data):
    # save reference data to the database
    ...

@task
def load_live_data(transformed_data):
    # save transformed live data to the database
    ...
````

### 第2步

在prefect.Flow上下文中指定业务数据和task依赖关系：

````Python
# ...task definitions above

with Flow("Aircraft-ETL") as flow:
    reference_data = extract_reference_data()
    live_data = extract_live_data()

    transformed_live_data = transform(live_data, reference_data)

    load_reference_data(reference_data)
    load_live_data(transformed_live_data)
````
注意：此时没有实际执行任何task，因为使用**Flow(...):**上下文管理器允许Prefect推理出task之间的依赖关系，并构建稍后将执行的执行图。这个业务场景的执行图如下所示：

![Prefect Aircraft ETL](prefectetl.png)

与最初的实现相比有了很大的改进！

### 第3步

执行Flow

````Python
# ...flow definition above

flow.run()
````

此时，以适当的顺序执行Task（我们的Python函数），并按照执行图中的指定从task到task传递业务数据。

> 
> Prefect Task库
> 
> Prefect提供一个task库，其中包含常见的task实现以及与Kubernetes，GitHub，Slack，Docker，AWS，GCP等的集成！
> 

接下来，让我们对flow进行参数化以使其复用性更好。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/tutorial/02-etl-flow.html)
- [联系译者](https://github.com/listen-lavender)

