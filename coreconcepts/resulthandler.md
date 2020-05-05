Prefect允许数据作为一等公民操作在task之间传递。此外，Prefect在设计时考虑了安全性和隐私性。因此，Prefect的概念抽象使用户可以利用其所有特性，而无需放弃对其私有数据的控制、访问甚至可见性。

实现此目标的主要方法之一是使用**Results**和**Result Handlers**。在高层次上看，所有状态对象都有一个与之关联的结果对象。可以为task的结果对象配置提供结果处理器，该结果处理器指定如果出于任何原因需要持久化保存task的输出，应如何处理该task的输出。

## 结果对象（Result）

结果对象表示Prefect task输出。特别是，每当task运行时，其输出就会封装在结果对象中。如果需要稍后保存/检索该数据，则该对象保留有关数据是什么以及如何处理该数据的信息（例如，如果此task请求将其输出缓存或设置检查点，在持久化和缓存中了解有关差异的更多信息）。

实例化的结果对象具有以下属性：

 - **value**：Result对象的值是单个数据
 - **safe_value**：此属性维护对SafeResult对象的引用，该对象包含该值的“安全”表示形式，例如，**SafeResult**的值可能是指向原始数据所在位置的URI或文件名。在这种情况下，“安全”指该值可以安全地发送到Cloud
 - **result_handler**：持有用于将值读取为其处理的表示形式或者从其处理的表示形式写入值的ResultHandler对象

在未配置的情况下，flow产生的所有状态对象都有一个基本的结果对象，该结果对象仅设置了**Result.value**，没有附加结果处理器。

````bash
>>> type(state._result)
prefect.engine.result.Result
>>> type(state._result.value)  # this is the type of your Task's return value
dict
>>> type(state._result.safe_value)
prefect.engine.result.NoResultType
>>> type(state._result.result_handler)
NoneType
````

> 
> **NoResult和None Result**区别
> 
> 为了区分正在运行尚未返回结果和尚未运行的两种task，Prefect还提供了一个NoResult对象，该对象表示缺少计算/数据。这与值为“无”的正常结果对象意义不同。**to_result**/**store_safe_value**方法以及NoResult对象的**value**和**safe_value**属性都返回相同的NoResult对象。
> 

所有结果对象还配备了**to_result**/**store_safe_value**方法。这些方法对于附加了结果处理器的Result对象，分别从其处理器读取结果和使用其处理器写入结果。

### 和结果对象交互

用户可能需要直接与内存中的Result对象进行交互的最常见情况是在本地运行flow时。所有Prefect 状态对象都内置有结果对象，可以通过私有**_result**属性对其进行访问。为方便起见，使用公有**.result**属性检索基础Result对象的值。

````bash
>>> task_ref = flow.get_tasks[0]
>>> state = flow.run()
>>> state._result.value  # a Flow State's Result value is its Tasks' States
{<Task: add>: <Success: "Task run succeeded.">,
 <Task: add>: <Success: "Task run succeeded.">}
>>> state.result  # the public property aliases the same API as above
{<Task: add>: <Success: "Task run succeeded.">,
 <Task: add>: <Success: "Task run succeeded.">}
>>> state.result[task_ref]._result  # a Task State's Result contains the Task's return value
<Result: 1>
````

## 结果处理器（Result Handler）

结果处理器比Result基类更具通用型，因为它们提供持久化存储内存中Result对象的数据的方法。结果处理器是用于处理数据的读/写接口的特定实现。实现结果处理器的唯一约束是**.write**方法返回一个JSON格式兼容对象。

但是，总的来说，结果处理器的**.write**方法被实现为可以做两件事：

 - 在task状态的Result.value属性中序列化task内存的返回值，并将其持久化到结果处理器的存储后端的磁盘中
 - 从状态的**Result.safe_value**属性中分离出有关该存储的返回值的“安全值”（通常为元数据），对于Cloud用户，将该安全值保留在调度程序数据库中

在另一方面，结果处理器的读取方法通常访问上游**Result.safe_value**中的元数据，以查找并反序列化存储中上游Task的返回值。

例如，我们可以轻松想象出不同类型的结果处理器：

 - Google Cloud Storage处理器，用于将给定的数据写入Google Cloud Storage存储桶，并从该存储桶中读取数据，此实例中的**write**方法返回一个URI
 - 一个**LocalResultHandler**，它从本地文件存储中读取/写入数据。在这种情况下，**write**方法返回一个绝对文件路径

在以下示例中，假设配置了一个flow，以使用**LocalResultHandler**将每个task的输出指向**~/Desktop/HelloWorld/results**目录。**Result.value**是3，即task的实际返回值。**Result.safe_value**是本地文件路径的字符串。

````bash
>>> state.result[first_result]._result.value
3
>>> state.result[first_result]._result.safe_value           
<SafeResult: '/Users/prefect/Desktop/HelloWorld/results/prefect-result-2020-02-23t18-38-40-381223-00-00'>
````

该文件内部是一个值为3的cloudpickle序列化数据，因为**LocalResultHandler**的实现将**Result.value**中的数据作为序列化保留在**Result.safe_value**的本地文件系统位置。

````bash
>>> f = open(state.result[first_result]._result.safe_value.value, 'rb')
>>> content = f.read()     
>>> content                
b'\x80\x05K\x03.'
>>> import cloudpickle     
>>> cloudpickle.loads(content)                              
3
````

但是，根据你的情况和数据的大小，你可以配置结果处理器以序列化task的整个结果，而不仅仅是序列化其元数据。例如，Prefect Core的基础**JSONResultHandler**将整个Result对象序列化为其返回值。如果你有一个task返回一小段数据（例如字符串或一小段数字），那么直接在返回的JSON字符串中处理该数据就足够了！

为了说明这一点，下面是相同要求的示例，其flow配置为使用**JSONResultHandler**检查每个task的输出。注意，现在**Result.value**和**Result.safe_value**都包含对其task的实际返回值（整数3）的引用。

````bash
>>> state.result[first_result]._result.value                 
3
>>> state.result[first_result]._result.safe_value            
<SafeResult: '3'>
````

在Prefect Core中，返回值在内存中与flow状态附带的Result对象关联。对于Cloud用户，结果处理器的**write**方法的返回值也将保留在Prefect Cloud的数据库中。

> 
> **谨慎处理数据**
> 
> 使用结果处理器意味着数据将被保存到Prefect的Python进程之外的存​​储位置，因此值得花一些时间来考虑哪些数据可以安全地保存在何处。
> 
> 特别的是，在Prefect Cloud上运行时，结果处理器的**write**方法的返回值存储在Cloud数据库中，并在Cloud UI中可见。仔细考虑您配置的结果处理器的**write**方法。
> 

### 如何指定结果处理器（Result Handler）

仅当启用检查点时，Prefect才会调用**ResultHandler**的读写方法。设置好此配置后，将有一个层次结构来确定对给定的数据使用什么**ResultHandler**：

 - 可以使用**result_handler**关键字参数在flow初始化时指定flow级别结果处理器。如果你从不指定其他结果处理器，则此处理器将用于该特定flow中的所有task
 - 最后，你可以设置task级别的结果处理器。这是通过在task初始化时（或在@task装饰器中）使用**result_handler**关键字参数来实现的。如果在此处提供结果处理器，则无论出于任何原因都需要缓存此task的输出，将始终使用该结果处理器。

> 
> **层级结构**
> 
> task级别结果处理器将始终在flow级处理器上使用。如果task的检查点kwarg设置为**False**，或者全局**prefect.config.tasks.checkpointing**值设置为False，则不会使用任何处理器。
> 

结果处理器的位置。

> 
> **结果处理器始终附加到task输出**
> 
> 例如，假设task A具有结果处理器Ah，task B具有结果处理器Bh，并且A将数据向下游传递给B。如果B失败并请求重试，则它需要缓存其输入，其中之一来自A。如果使用Cloud，则Cloud将使用结果处理器来保留输入缓存，并且由于数据来自task A，因此它将使用结果处理器Ah。
> 

参数抽象是例外。

> 
> **参数是不同的**
> 
> 结果处理器的规则有一个例外：Prefect参数。 Prefect参数始终使用**JSONResultHandler**，以便可以在UI中检查其缓存的值。
> 

要了解有关使用结果处理器的更多信息，请查看有关[使用结果处理器的教程](https://docs.prefect.io/core/advanced_tutorials/using-result-handlers.html)。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/results.html)
- [联系译者](https://github.com/listen-lavender)

