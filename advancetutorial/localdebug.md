无论您是使用**flow.run()**在本地运行Prefect，还是在使用Prefect Cloud将其投入生产之前对想法方案进行实验，Prefect都可以提供大量用于调试flow和诊断问题的工具。

## 使用FlowRunner进行无状态执行

如果问题与重试有关，或者想按计划运行flow，则可以首先考虑使用**run_on_schedule=False**重新运行flow。这可以通过环境变量**PREFECT__FLOWS__RUN_ON_SCHEDULE=false**或关键字**flow.run(run_on_schedule=False)**来完成。相反，如果想实现单个flow运行，请考虑直接使用**FlowRunner**：

````Python
from prefect.engine.flow_runner import FlowRunner

# ... your flow construction

runner = FlowRunner(flow=my_flow)
flow_state = runner.run(return_tasks=my_flow.tasks)
````

无论其调度计划如何，将立即执行flow。注意默认情况下，FlowRunner不提供任何单独的task状态，因此，如果想要该信息，则必须使用**return_tasks**关键字参数来请求它。

## 执行器选择

### LocalExecutor

选择执行器对于调试flow的难易程度是非常重要的。DaskExecutor依赖于多进程和多线程，这可能很难定位问题。当试图找出问题所在时，诸如共享状态和打印语句之类的东西会消失在虚空中，可能是一件真正的麻烦事。幸运的是，Prefect提供LocalExecutor，它是最简化的执行器-所有task都将立即执行，并且是在主进程中按照适当的顺序执行。

要换本地执行器，请按照示例进行操作：

````Python
from prefect.engine.executors import LocalExecutor

# ... your flow construction

state = flow.run(executor=LocalExecutor()) # <-- the executor needs to be initialized
````

这样就设定了！executor关键字也可以提供给**FlowRunner.run**方法。

### LocalDaskExecutor

如果想升级为功能更强大的执行器，但仍保持易于调试的环境，则建议使用LocalDaskExecutor。该执行器确实使用dask推迟了计算，但是避免了任何并行性，从而使执行流水线更容易推理追踪。初始化此执行器时，可以通过设置**scheduler=threads**或**scheduler=processes**来打开并行性。

> 
> **Prefect设置**
> 
> 可以将LocalDaskExecutor设置为本地计算机上的默认执行器。要更改Prefect设置（包括默认执行器），可以：
>  - 修改**~/.prefect/config.toml**配置文件
>  - 更新OS环境变量；可以通过设置PREFECT__SECTION__SUBSECTION__KEY覆盖配置文件中的每个值。例如，要更改默认执行器，可以设置**PREFECT__ENGINE__EXECUTOR__DEFAULT_CLASS=prefect.engine.executors.LocalExecutor**
> 

### DaskExecutor

最后，如果问题实际上与并行性有关，则需要使用DaskExecutor。有两个初始化关键字参数，在调试时非常有用：

 - local_processes，它是一个布尔值，指定是否要使用多进程。此标志的默认值为False。尝试切换它以查看问题是否与多进程或多线程有关
 - debug，这是另一个用于设置**dask.distributed**日志记录级别的布尔值； 默认值是从Prefect配置文件中设置的，并且应该为True，以从dask的日志中获取最详细的输出

## 运行时抛异常

有时不希望Prefect强大的错误处理机制捕获异常，你希望将它们引发，以便可以立即自己调试它们！发生错误时，使用**raise_on_exception**上下文管理器抛出异常：

````Python
from prefect import Flow, task
from prefect.utilities.debug import raise_on_exception


@task
def div(x):
    return 1 / x

with Flow("My Flow") as f:
    val = div(0)

with raise_on_exception():
    f.run()

---------------------------------------------------------------------------

ZeroDivisionError                         Traceback (most recent call last)

<ipython-input-1-82c40dd24406> in <module>()
     11
     12 with raise_on_exception():
---> 13     f.run()


... # the full traceback is long

<ipython-input-1-82c40dd24406> in div(x)
      5 @task
      6 def div(x):
----> 7     return 1 / x
      8
      9 with Flow("My Flow") as f:


ZeroDivisionError: division by zero
````

现在可以使用自己喜欢的调试器进行跟踪并像往常一样继续进行。

> 
> 提示
>
> 注意该实用工具不需要知道错误发生的位置。
> 

## 事后异常重放

假设想让整个管道运行，并且不想在运行时引发陷阱错误。假设错误被捕获并置task于**Failed**状态，则完整的异常存储在task状态的**result**属性中。知道这一点，就可以在本地重新引发它并从那里进行调试！

展示：

````Python
from prefect import Flow, task


@task
def gotcha():
    tup = ('a', ['b'])
    try:
        tup[1] += ['c']
    except TypeError:
        assert len(tup[1]) == 1


flow = Flow(name="tuples", tasks=[gotcha])

state = flow.run()
state.result # {<Task: gotcha>: Failed("Unexpected error: AssertionError()")}

failed_state = state.result[gotcha]
raise failed_state.result

---------------------------------------------------------------------------

TypeError                                 Traceback (most recent call last)

<ipython-input-50-8efcdf8dacda> in gotcha()
      7     try:
----> 8         tup[1] += ['c']
      9     except TypeError:

TypeError: 'tuple' object does not support item assignment

During handling of the above exception, another exception occurred:

AssertionError                            Traceback (most recent call last)
<ipython-input-1-f0f986d2f159> in <module>
     22
     23 failed_state = state.result[gotcha]
---> 24 raise failed_state.result

~/Developer/prefect/src/prefect/engine/runner.py in inner(self, state, *args, **kwargs)
     58
     59         try:
---> 60             new_state = method(self, state, *args, **kwargs)
     61         except ENDRUN as exc:
     62             raise_end_run = True

~/Developer/prefect/src/prefect/engine/task_runner.py in get_task_run_state(self, state, inputs, timeout_handler)
    697             self.logger.info("Running task...")
    698             timeout_handler = timeout_handler or main_thread_timeout
--> 699             result = timeout_handler(self.task.run, timeout=self.task.timeout, **inputs)
    700
    701         # inform user of timeout

~/Developer/prefect/src/prefect/utilities/executors.py in multiprocessing_timeout(fn, timeout, *args, **kwargs)
     68
     69     if timeout is None:
---> 70         return fn(*args, **kwargs)
     71     else:
     72         timeout_length = timeout.total_seconds()

<ipython-input-1-f0f986d2f159> in gotcha()
      9         tup[1] += ['c']
     10     except TypeError:
---> 11         assert len(tup[1]) == 1
     12
     13

AssertionError:

%debug # using the IPython magic method to start a pdb session
```` 

````bash
ipdb> tup
('a', ['b', 'c'])
ipdb> exit
````

啊哈！这就是在元组中放置可变对象所得到的！

## 调修损坏的flow

到目前为止，已经谈到了如何识别错误，但是如何解决损坏的flow呢？一种方法是从头开始创建flow实例重新运行逻辑。在生产中，对于如何创建flow有一个单一的绝对来源可能是个好主意。但是，当快速迭代或尝试建立概念验证时，这可能会令人沮丧。

尝试更新flow时，有两种方便的flow方法：

 - **flow.get_tasks**用于检索满足某些条件的task
 - **flow.replace**准备好更新/修复的版本替换task

使用上面的错漏flow，让我们定义一个新task，并使用以下方法将其替换掉：

````Python
@task
def fixed():
    tup = ('a', ('b',))
    try:
        tup[1] += ('c',)
    except TypeError:
        assert len(tup[1]) == 1


broken = flow.get_tasks(name="gotcha")[0]
flow.replace(broken, fixed)

flow.run() # Success("All reference tasks succeeded.")
````

如果有一个复杂的依赖关系图，**flow.replace**可以成为真正的节省时间的工具，用于快速替换task，同时保留所有依赖关系！

> 
> **调用签名**
> 
> 注意**flow.replace**保留原型，这意味着新旧task需要具有完全相同的调用签名。
> 

## 本地检查flow的Docker存储

flow在生产中可能意外中断（或根本无法运行）的另一个原因是它的存储是否损坏（例如，如果忘记了为该flow定义Docker存储时的Python依赖关系）。幸运的是，在本地检查flow的存储很容易！让我们来看一个例子：

````Python
from prefect import task, Flow
from prefect.environments.storage import Docker


# import a non-prefect package used for scraping reddit
import praw


@task
def whoami():
    reddit = praw.Reddit(client_id='SI8pN3DSbt0zor',
                         client_secret='xaxkj7HNh8kwg8e5t4m6KvSrbTI',
                         password='1guiwevlfo00esyy',
                         user_agent='testscript by /u/fakebot3',
                         username='fakebot3')
    return reddit.user.me()


storage = Docker(base_image="python:3.6", registry_url="http://my.personal.registry")
flow = Flow("reddit-flow", storage=storage, tasks=[whoami])
````

如果打算将此flow部署到Cloud，将不会再的到它的反馈。为什么？我们来了解一下，首先，在本地构建Docker存储而无需推送注册：

````Python
# note that this will require either a valid registry_url, or no registry_url
# push=False is important here; otherwise your local image will be deleted
built_storage = flow.storage.build(push=False)
````

注意Docker输出的倒数第二行中包含的镜像ID：

````bash
Successfully built 0f3b0851148b # your ID will be different
````

现在，在本地计算机上有一个Docker容器，其中包含我们的flow。要访问它，我们首先确定flow的存储位置：

````bash
built_storage.flows
# {"reddit-flow": "/root/.prefect/reddit-flow.prefect"}
````

然后通过以下方式连接到容器：

````bash
# connect to an interactive python session running in the container
docker run -it 0f3b0851148b python
````

最后，我们可以使用cloudpickle将文件反序列化为flow对象：

````Python
import cloudpickle

with open("/root/.prefect/reddit-flow.prefect", "rb") as f:
    flow = cloudpickle.load(f)
````

这将导致非常明确的回溯！

````Python
Traceback (most recent call last):
    flow = cloudpickle.loads(decrypted_pickle)
  File "/usr/local/lib/python3.6/site-packages/cloudpickle/cloudpickle.py", line 944, in subimport
    __import__(name)
ModuleNotFoundError: No module named 'praw'
````

在这种特殊情况下，我们忘了为Docker存储的python_dependencies中包含praw；通常，这是确保flow顺利通过部署过程的一种方法。

> 
> **更多知识点**
> 
> 实际上，我们发现此过程非常有用，我们已为您自动化了！ Prefect现在在推送Docker镜像之前执行“健康状况检查”，该镜像实际上运行上述代码，并确保flow可在其容器中反序列化。但是，了解这种情况的机制仍然很有用。
> 

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/advanced_tutorials/local-debugging.html)
- [联系译者](https://github.com/listen-lavender)

