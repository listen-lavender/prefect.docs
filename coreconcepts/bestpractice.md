## 编写Prefect代码

为了最大程度地提高Prefect代码的清晰度，建议设计以下几部分组织脚本：

 1. 导入需要的任何task类
 2. 定义任何自定义task函数
 3. 实例化任何task类
 4. 尽可能使用函数式API和适当的命令性API来构建flow


### 例子

````Python
#--------------------------------------------------------------
# Imports
#--------------------------------------------------------------

# basic imports
from prefect import Flow, Parameter, task

# specific task class imports
from prefect.tasks.shell import ShellTask


#--------------------------------------------------------------
# Define custom task functions
#--------------------------------------------------------------

@task
def plus_one(x):
    """A task that adds 1 to a number"""
    return x + 1

@task
def build_command(name):
    return 'echo "HELLO, {}!"'.format(name)

#--------------------------------------------------------------
# Instantiate task classes
#--------------------------------------------------------------

run_in_bash = ShellTask(name='run a command in bash')

#--------------------------------------------------------------
# Open a Flow context and use the functional API (if possible)
#--------------------------------------------------------------

with Flow('Best Practices') as flow:
    # store the result of each task call, even if you don't use the result again
    two = plus_one(1)

    # for clarity, call each task on its own line
    name = Parameter('name')
    cmd = build_command(name=name)
    shell_result = run_in_bash(command=cmd)

    # use the imperative API where appropriate
    shell_result.set_upstream(two)
````

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/best-practices.html)
- [联系译者](https://github.com/listen-lavender)

