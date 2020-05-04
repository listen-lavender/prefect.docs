该模块中的task可用于表示内置操作，包括算术，索引和逻辑比较。

通常，用户不会手动实例化这些task。当用户将内联Python运算符应用于task和另一个值时，它们将自动应用。

例如：

````bash
Task() or Task() # applies Or
Task() + Task() # applies Add
Task() * 3 # applies Mul
Task()['x'] # applies GetItem
````
    
### GetItem（获取项）

task助手，用于检索上游task结果的特定索引。

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-getitem)

### Add（加）

计算 **x + y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-add)

### Sub（减）

计算 **x - y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-sub)

### Mul（乘）

计算 **x * y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-mul)

### Div（除）

计算 **x / y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-div)

### FloorDiv（向下整除）

计算 **x // y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-floordiv)

### Pow（乘方）

计算 **x ** y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-pow)

### Mod（求余）

计算 **x % y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-mod)

### And（与）

计算 **x and y.**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-and)

### Or（或）

计算 **x or y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-or)

### Not（否）

计算 **not x**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-not)

### Equal（相等）

计算 **x == y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-equal)

### NotEqual（不相等）

计算 **x != y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-notequal)

### GreaterThanOrEqual（大于或等于）

计算 **x ≥ y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-greaterthanorequal)

### GreaterThan（大于）

计算 **x > y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-greaterthan)

### LessThanOrEqual（小于或等于）

计算 **x ≤ y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-lessthanorequal)

### LessThan（小于）

计算 **x < y**

[API参考文档](https://docs.prefect.io/api/latest/tasks/operators.html#prefect-tasks-core-operators-lessthan)

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/task_library/operators.html)
- [联系译者](https://github.com/listen-lavender)
