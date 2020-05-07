用于实现控制流结构（如分支和重新加入flow）的task和实用工具。

### If/Else

在工作流中建立条件分支。

如果条件评估为True（ish），则将运行**true_task**。如果它评估为False（ish），则将运行**false_task**。**Skipped**状态的task被跳过不执行，以及未设置**skip_on_upstream_skip=False**的所有下游task都会跳过。

[API参考文档](https://docs.prefect.io/api/latest/tasks/control_flow.html#prefect-tasks-control-flow-conditional-ifelse)

### Switch

SWITCH是一个条件task，会被添加到工作流。

条件task会计算结果，并将结果与case的键进行比较。匹配case键对应的task运行，其他所有task都将被跳过。除非将跳过的task下游的任何task设置为**skip_on_upstream_skip=False**，否则它们也会被跳过。

[API参考文档](https://docs.prefect.io/api/latest/tasks/control_flow.html#prefect-tasks-control-flow-conditional-switch)

### Merge

Merge是一个合并task，将条件分支合并在一起。

flow中的条件分支会导致一个或多个task继续进行，而一个或多个task会被跳过。将那些分支合并回单个结果通常很方便。此语义是实现该目标的简单方法。

合并task将返回遇到的第一个实际结果，或者**None**。如果多个task可能返回结果，则将它们与列表分组。

[API参考文档](https://docs.prefect.io/api/latest/tasks/control_flow.html#prefect-tasks-control-flow-conditional-merge)

### FilterTask

筛选结果列表的task。

默认的过滤器是过滤**NoResult**和**Exception**以过滤出映射结果。注意此task的默认触发器为**all_finished**，并且**skip_on_upstream_skip=False**。

[API参考文档](https://docs.prefect.io/api/latest/tasks/control_flow.html#prefect-tasks-control-flow-filter-filtertask)

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/task_library/control_flow.html)
- [联系译者](https://github.com/listen-lavender)
