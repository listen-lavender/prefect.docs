该模块中的task可用于表示task结果的集合，例如列表，元组，集合和字典。

通常，用户不会手动实例化这些task。当用户在task和其他对象的集合之间创建依赖关系时，它们将自动应用。

例如：

````Python
@task
def count_more_than_2(inputs: set) -> int:
    return len([s for s in inputs if s > 2])

# note these tasks don't actually return a useful value
set_of_tasks = {Task(), Task(), Task()}

# automatically applies Set
count_more_than_2(inputs=set_of_tasks)
````

## List

自动将其输入合并到Python列表中。

[API参考文档](https://docs.prefect.io/api/latest/tasks/collections.html#prefect-tasks-core-collections-tuple)

## Tuple

自动将其输入组合到Python元组中。

[API参考文档](https://docs.prefect.io/api/latest/tasks/collections.html#prefect-tasks-core-collections-tuple)

## Set

自动将其输入组合到Python集合中。

[API参考文档](https://docs.prefect.io/api/latest/tasks/collections.html#prefect-tasks-core-collections-set)

## Dict

自动将其输入组合到Python字典中。

[API参考文档](https://docs.prefect.io/api/latest/tasks/collections.html#prefect-tasks-core-collections-dict)

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/task_library/collections.html)
- [联系译者](https://github.com/listen-lavender)

