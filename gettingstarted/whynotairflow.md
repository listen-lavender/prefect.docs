在数据工程生态系统，Airflow一直是一个重要的工具，技术工作者也利用它开发了很多业务。它开创性地兼备标准DAG模型和Python的灵活性，从而广泛的适用现实生活中的workflow业务。然而，Airflow被当做一个臃肿的批处理调度器来设计，目标使用者是团队打算招来协调组织其他系统的数据工程师，因此其可用性受到了很大限制。

今天，许多数据工程师正在与分析人员更直接地合作，计算存储都是便宜的，使得试错成本低，实验得到注重。加上业务workflow快速动态变化，尽管Airflow在许多事情上做对了，但是其核心目标从未预料到数据应用会变得如此多样化。很显然，它没有亟需的数据抽象和接口来实现那些日异月新的业务活动。

时间追溯到2016年如何改变Airflow来支撑新的多样化数据实践的讨论中，Prefect产品的第一次全面计划就此萌芽。令人失望的是，Aiflow在讨论后并未改进。我们才能开源了现代化的数据平台Prefect，早期的commit和评论反馈令我们很鼓舞。

我们知道用户关心关于Prefect和Airflow的对比，尤其是考虑到Prefect的血统与Airflow相关。本文主要对比阐述Prefect引擎是如何用特性解决一些Airflow无法解决的问题，并不是Prefect详尽的特性说明书，而是给熟悉Airflow的人提供用户指南，指明Prefect解决问题的方法。尽可能地讨论已解决的问题和方法，对于开源仓库中当前不可用内容的介绍试图保持平衡和克制，希望这对社区有益。

内容提要

- 概览
- API
- 调度和时间的关系
- 调度服务
- 数据流
- 参数化workflow
- 动态workflow
- 多版本workflow
- 本地测试
- UI操作界面
- 总结


## 概览

Airflow是用来为按照固定的时间调度运行偏静态的、task流转缓慢的workflow而设计，在这个目标上算是优秀的框架。Airflow也是第一个用代码成功实现强大灵活的workflow的有用榜样。事实证明，无需借助配置文件的定义，也可实现workflow。

然而，从能处理多样化的workflow业务的角度，尤其以现在的标准来看，Airflow为workflow设计提供的数据抽象和接口很有限。开发者为了实现业务，把业务勉强套进Airflow模型会遇到麻烦。Airflow不能很好地满足以下场景：

- 无调度或者脱离调度的DAG
- 同一时间多次并发启动的DAG
- 复杂分支逻辑的DAG
- 实时task执行的DAG
- 需要交换数据的DAG
- 参数化的DAG
- 动态的DAG

如果你的使用场景与上述任何一种相似，将无法直接使用Airflow，除非你在Airflow的抽象设计层面花费大量的工夫。因此，几乎每个中大型规模的公司最终都会编写自定义的DSL，或者维护大量专有插件来满足内部复杂多变的业务需求，又造成升级和故障维护的困难。

Prefect汲取了多年从事Airflow相关项目的经验。产品的研究覆盖了数百用户和公司，发现了Airflow难以解决的隐患。然后通过支撑适合大多数业务场景的架构抽象设计，它最终实现了难以置信的轻量好用的接口。**注意，本文Airflow DAG和Prefect Flow都是workflow工作流的意思**


## API

>   当workflow能够定义成代码时，他们变得更加
>   易于维护，方便多版本追踪，好测试，可协作。
> 
> 
> 
>   -- Airflow 官方文档

生产中应用workflow对复杂业务是非常有利的，并且牵扯到技术分工范围的多个利益协作者。因此考虑到workflow在数据处理技术栈处处可用，让workflow足够清晰简单是很重要的。Python是开发workflow的语言的最优选择。Airflow是第一个认识到这一点的框架，并且用Python实现API。

但是，Airflow的API是命令式（imperative）和面向对象（基于类）的。此外，由于Airflow对DAG的代码开发的诸多限制（后面展开讲），编写Airflow DAG代码就像是在开发Airflow特定风格的代码。

Prefect的基本理念是，如果开发者能够保证编写的业务函数按预期正常运行，对workflow框架是无感知的。只有当bug出现，程序抛异常，workflow框架的管理作用才显现出来。这么看，workflow框架可算是流程风险管理工具，并且设计良好的情况下，应该降低workflow框架对业务代码的显式侵入，除非开发者明确需要，否则workflow框架不要让用户在开发路上觉得碍眼。

所以Prefect的设计目标是在业务代码正确的情况下是几乎零侵入透明的，同时在出错时，workflow框架能尽量协助业务重回正轨。无论哪种方式，Prefect都可以为业务提供相应级别的透明度和细节。

实现上述目标的基本方法是使用Prefect的功能API。在此模式下，Prefect的task行为类似于Python函数，可以直接定义输入输出和调用它。简单到只用一行Prefect装饰器代码，就将Python函数转换为task。像是Python函数一样自然，task之间的互相调用就地构建workflow，非常具有Python风格。这使得通常将已有业务代码和脚本转换成全栈Prefect的workflow是轻车熟路的。

不用担心，Prefect同时还为Airflow用户提供Airflow风格的API，这种过程式风格对于构建复杂task依赖的workflow很有用，可以实现底层抽象控制。用户可以根据需求和倾向，来切换这两种风格使用。


## 调度和时间的关系

>   时间是一种幻想，就像午餐时间的加倍。
>   
>
>
>
>   -- 银河系漫游指南

对于Airflow的新手来说，可能最常见的困惑是不同的时间概念的用途。如果你运行Airflow教程例子，你会发现要一直纠结这些不同的时间定义是什么意思呢？

````bash
airflow test tutorial print_date 2015–06–01

## Output
AIRFLOW_CTX_EXECUTION_DATE=2015–06–01T00:00:00+00:00
[2019–04–17 15:54:45,679] {bash_operator.py:110} INFO - Running command: date
[2019–04–17 15:54:45,685] {bash_operator.py:119} INFO - Output:
[2019–04–17 15:54:45,695] {bash_operator.py:123} INFO - Wed Apr 17 15:54:45 PDT 2019
````

Airflow严格依赖execution_date的时间定义。DAG要设置execution_date才可以运行，同一个DAG不可能生成2个运行execution_date相同的实例。当一个DAG需要同时开始执行两次的呢？Airflow就无法支持。这需要创建2个DAG ID不同业务逻辑完全相同的DAG来生成各自的可执行实例，或者同一个DAG生成execution_date相隔1ms的2个可执行实例，或者使用Hack方式达到目的。

更令人困惑的是，execution_date并不是Airflow DAG可执行实例的start_time，start_time=execution_date+一个周期间隔。这最初是由于ELT编排业务要求所致，execution_date为5月2日时间点的数据作业将在5月3日的同一时间点真正开始运行。时至今日，这都是用户尤其是新用户的最大阻碍。

下一个时间点才开始运行可执行实例产生的间隔设计来源于Airflow严格要求DAG有良好的执行计划。直到现在，它都不可能运行脱离计划调度的DAG，因为调度器无法正确地给这类DAG可执行实例安排start_time。其实临时可执行实例也是可以产生的，只要该临时的可执行实例不与同一个DAG下面其他的可执行实例有同一个start_time。

这意味对于以下需求，Airflow会是一个错误的选择：

- 不规则调度或者无调度的DAG
- 同一个DAG生成多份同时开始的可执行实例
- 维护经常手工触发产生可执行实例的DAG

> 作为对比，Prefect将workflow的可执行实例当作可以任何时候执行、任意并发的对象。
> 调度计划是一组预定义好start_time的可执行实例的编排，可以根据需求很灵活的定义简单或者复杂的调度计划。
> 如果workflow的可执行实例确实要严格遵守时间计划，只需要简单地将时间添加为workflow可执行实例的参数。


## 调度服务

> R2-D2（星球大战的宇航技工机器人），你比起你作为一台奇怪的计算机器了解的更好。
> 
> --C-3PO（星球大战的礼仪机器人）

Airflow的调度器是Airflow的核心部分，对Airflow使用有性能影响。主要负责如下：

- 每隔几秒钟重新分析一次DAG文件夹，生成DAG计划
- 检查DAG计划以确定是否生成DAG可执行实例来准备就绪
- 检查所有task依赖项，以确定是否有后续的task可以运行
- 在数据库设置最终的DAG运行实例和task运行实例的状态

相对的，Prefect将这么多架构设计解耦到不同的独立模块中。

### Prefect Flow Scheduling

Prefect Flow Scheduling（workflow可执行实例调度）是很轻量的，只需要简单创建一个新的可执行实例，并且设置成Scheduled状态。这就是Prefect Flow Scheduling实际的唯一责任，调度器绝对不会干涉任何workflow的依赖逻辑。

### Prefect Flow Logic

Prefect Flow Logic（workflow的依赖设置）是独立的编码部分，依赖逻辑绝不侵入业务，也不会把workflow相关的状态管理掺杂到业务里面。作为证据，你可以在本地运行空的workflow的情况，发现workflow进程几乎没有资源开销。

````bash
# run your first Prefect flow from the command line

python -c "from prefect import Flow; f = Flow('empty'); f.run()"
````

### Prefect Task Scheduling

当Prefect Flow的可执行实例在运行期间，只管调度属于自己的task，这非常重要：

- 从workflow的架构抽象看，workflow的可执行实例对象是唯一适合接管这个责任的
- 减轻中枢调度器（Prefect Flow Scheduling）的负担
- 针对各种独特场景做决策，例如动态生成task
- 将执行细节交给Dask等外部系统负责

最后一点很重要，尽管Airflow已经丰富地支持了多种执行器，包括本地进程执行器、Celery执行器、Dask执行器、Kubernetes执行器，调度器仍然是Airflow使用的瓶颈，调度器执行任何task都需要至少10s的额外时间，5s用来标记已经就绪，5s用来提交执行。不管外部执行系统例如Dask集群有多大，Airflow会每隔10s生成准备和执行task。

相比而言，Prefect采用了更现代化的模式。例如当Prefect在Dask上面执行的时候，可以利用Dask的毫秒级延迟task调度器来尽快运行所有task，并且能充分利用Dask集群的并行度。实际上Prefect Cloud的默认部署模式是在Kubernetes中使用可定制化的Dask集群。

除性能外，如何设计workflow影响很大。Airflow鼓励设计庞大的task，Prefect倾向设计微型的模块化task，也保留了实现大型task的能力。

另外，当在Prefect Clound运行task且数据库是自定义的，task和workflow的可执行实例负责维护更新数据库的数据状态，而不是由调度器维护。

> 作为对比，强中心化（管理所有设计抽象的调度业务）的Airflow调度循环使得当task启动遇到task依赖处理时，会产生不小的延迟。如果使用场景只涉及到少数几个长时间运行的task，这完全没问题，但是如果你想执行一个包含有大量的时间要求的task的DAG，这可能成为瓶颈。
> Airflow将时间计划和workflow紧密耦合一起，这意味着你即使在本地运行DAG，也需要初始化数据库和调度服务。对于生产环境无可厚非，但是对于业务代码的快速迭代和测试是很麻烦的。
> Airflow调度器的强中心化特性形成了系统的单点故障。
> 循环重新解析DAG会造成代码逻辑的严重不一致，很有可能调度器将要执行的task，发现不存在对应的DAG。
> 强中心化调度意味着多个task之间无法通信（没有通信依赖关系解决方案）。


## 数据流

>   它是一个陷阱。
>   
>
>
>
>   -- 《星球大战》贾尔·阿克巴上将

Airflow最常见的用途之一建立某种数据管道，但讽刺的是，Airflow并没有最好的方式实现它。

Airflow提供XCom，一个被设计用来在task之间交换少量数据的工具。如果你希望task A告诉task B大量的数据写到云端的某个可见的位置，这是鼓励的的。但当用户试图使用它建立数据管道，就会产生很多Airflow的使用bug。

XCom使用管理员权限将可执行的序列化数据写到元信息的数据库，有安全隐患。即使采用json格式，也还是有问题。这种数据没有过期时间或者有效期限制，会造成性能和存储成本问题。更重要的是，使用XComs会在Airflow调度器不知情的情况，在task之间创建严格的上下游依赖关系。如果用户不增加额外的编码定义，Airflow实际可能会用错误的顺序执行这些task。考虑以下情况：

Airflow框架是无法明确确定某个task依赖于另一个推类型task的行为。如果开发者没有通过代码明确向Airflow说明，调度器还是会无序编排运行这些task实例。即使开发者代码指定输入输出数据关系，Airflow也无法理解基于通信数据的依赖关系，而且当XCom推送数据失败，Airflow又不知道怎么响应。这是最常见的玄奥难解的Airflow bug之一。

不幸的是，常常怕什么来什么，Airflow新手过度使用XCom堵死了元信息的数据库，我们看到过有人创建了10GB规模的数据，并用XCom传递给各种task。一个DAG实例要是有10个task实例，一共会写100GB的数据到Airflow的元信息的数据库。

> 作为对比，Prefect将数据流抽象当做一等公民。task可以输入返回，Prefect用透明方式管理这些输入输出依赖。另外，Prefect永远不会将这些数据写入元信息的数据库，相应的，task实例的阶段结果在需要存储的时候，可以轻松配置安全的结果处理器。这样有许多好处：
> 
> - 开发者可以使用熟悉的Python函数风格编码
> - 引擎知道依赖关系，能提供更加透明流畅的调试体验
> - 尽管Prefect允许使用数据流，并不意味着只能这么用，所以仍然支持无依赖的Airflow风格，有时候鼓励使用无数据依赖风格
> - 因为task可以交换数据，Prefect能支持更复杂的毛细分支逻辑和更丰富的task状态，并且在workflow实例中的task实例之间有更严格的执行约束（例如，一个tas不能在数据库更新下游的task的状态）


## 参数化workflow

>   很抱歉Dave，恐怕我做不到。
>   
>
>
>
>   -- 《2001太空漫游》超级电脑HAL9000

有能力处理响应不同的输入是很方便的。例如，一个workflow可能就是一系列的操作步骤，可以重复处理不同外部数据（API、数据库或者ID）的信息，这些不同的数据需要复用相同的处理逻辑。甚至，你可能想用输入参数来控制影响workflow本身。

因为Airflow DAG要按固定的调度计划来执行，并且不接收调用的输入参数，Airflow不能很好的支持更多。当然也可以突破这个限制，但是解决方案涉及到围绕着Airflow调度器持续重复解析DAG文件和动态响应Airflow Variable对象开展工作。如果业务需要利用调度程序的内部实现细节，可能是落地方案错了。

Prefect为参数化workflow提供了方便的抽象。Prefect workflow的Parameter对象是一种支持可选参数的特殊task，它有配置的默认值，在运行期调用是可覆盖的。例如，我们在Prefect Cloud部署中运行时，Parameter对象的值可以通过简单的GraphQL调用或者Prefect的Python客户端来设置。

````Python
from prefect import task, Parameter, Flow


@task
def return_param(p):
    return p


with Flow("parameter-example") as flow:
    p = Parameter("p", default=42)
    result = return_param(p)


flow.run() # uses the value 42
flow.run(p=99) # uses the value 99
````

这提供了许多好处：

- 出问题时，数据的来龙去脉是透明的
- 不需要为不同的起始参数输入创建新的workflow，只需要根据参数生成新的workflow可执行实例
- 允许设计响应事件的workflow，workflow可执行实例是根据不同的类型内容的事件来选择执行某个分支

早些时候，我们就注意到Airflow没有同时开始运行一个DAG多个可执行实例的抽象。这有部分原因是Airflow DAG没有参数概念。当DAG无法响应输入，同时运行多个实例就没有意义。

但是，有了一等公民的参数抽象，就容易理解为什么我可能希望同时运行workflow的多个可执行实例，例如发送多封电子邮件，或者更新多个数据源，或者任何其他系列活动等workflow业务逻辑相同和起始参数不同的场景。


## 动态化workflow

>   你将需要更大的船。
>   
>
>
>
>   -- 电影《大白鲨》

除了参数化的workflow之外，经常遇到workflow内部有task的逻辑需要被重复运行几次的场景。假设task A从数据库查询所有的新客户列表，需要把里面的每一个客户ID交给另一个task做一些事务处理。在Airflow中，只有一个选择，实现一个下游task B，该task B使用客户ID列表，并循环一一处理。这种方式有以下缺点：

- UI操作界面无法可视化task内部循环造成的动态负载，执行过程不可见
- 如果任何一条数据记录执行失败，则整个任务失败或者被丢弃形成业务黑洞
- 重试的话，还要考虑规模和实现幂等逻辑，因为失败中途重试，系统很难理解哪些数据已经处理过，需要跳过不处理

由于这是一种常见场景，因此Prefect将其专门抽象成称之为task mapping的功能。task mapping是指能够运行时根据上游task的输出结果来动态生成不同数量的下游task的能力。映射强大到可以在映射过的任务继续映射，从而轻松创建并行管道。而归纳收集结果就是将mapped task作为Parameter输入到non-mapped task这么简单。考虑一个简单例子，生成一个4项item的list，遍历每一项item两次做+1处理并设置item，然后计算list的item之和。

````Python
from prefect import task, Flow


@task
def create_list():
    return [1, 1, 2, 3]

@task
def add_one(x):
    return x + 1

@task
def get_sum(x):
    return sum(x)

with Flow("simple-map") as f:
    plus_one = add_one.map(create_list)
    plus_two = add_one.map(plus_one)
    result = get_sum(plus_two)

f.run()

````

该workflow运行实例明确有10个Prefect task，1个是list创建，8个是4项item X 2次+1映射处理，还有一个求和规约。任务映射提供许多好处：

- mapping模式易于在workflow中表达
- 每一个动态产生的task都是单独实例，意味着可以独立于其他task单独重试告警
- workflow的每一个task实例都能触发动态生成不同数量的下游task实例
- 作为一等公民的task mapping抽象，UI操作界面也能准确可视化映射关系


## 多版本workflow

>   为了通过编码解决工作问题，这还不够。
>   
>
>
>
>   -- 《代码整洁之道》的作者[美]Robert C. Martin

任何涉及代码的系统的都需要一个重要能力，是能够对代码进行版本控制。

回想一下在Airflow里面，DAG是中枢调度器在检测指定的DAG目录包含的Python文件，并执行代码文件来发现DAG定义。这意味着，如果你更新某个DAG，Airflow将会重新加载盲目执行，而并不会感知代码的迭代改变。如果你的DAG定义代码定期更新，这会引起一些麻烦。

- 能够重新访问甚至运行旧的DAG的能力要求你存储旧版本代码，并且需要在Airflow生态系统中抽象出单独的实体
- UI操作界面对版本系统一无所知，就无法可视化DAG有效的多版本信息

在实践中，团队只能借助第三方版本工具GitHub控制版本，并将版本信息附加到文件名中实现部分业务目标。这种设计实现，如果你的workflow代码变化缓慢，也不会有负担。但是，由于数据工程日益成为快节奏的科学，包括实验和频繁更新，仅仅支持部署新的代码模型参数，这种方式很快会失败。

在Prefect Cloud中，我们已将版本化数据提升为一等公民。任何workflow都是版本化数据的一部分，以轻松追踪维护历史记录。我们照常有合理的默认值。

- 一个项目下部署一个同名的workflow，版本系统会自动追踪记录为版本数据
- 当一个workflow被版本化，会获得递增的版本号，并且对之前的老版本workflow关闭自动调度和进行存档

如果你有更复杂的版本需求，这些都是有可配置自定义的。例如，绕开项目下workflow同名追踪规则，你可以直接指定某一个workflow是另外一个workflow的不同版本。你可以覆盖自动版本升级，来取消存档并留用旧版本（例如，用于A/B测试）。或者，你只是简单使用版本系统来维护workflow历史，而不会复杂化UI操作界面。


## 本地测试

>   可能出错的事物和不可能出错的事物之间最大的区别是，
>   当不可能出错的事物出错时，
>   通常发现根本无法调试。
>
>
>   -- 工具书《Mostly Harmless》

因为Airflow和Prefect都是用Python编写的，因此可以按照标准的Python模式来对独立的task/operator逻辑进行单元测试。例如，在Airflow中，您可以导入DagBag提取单个DAG，并对其结构或包含的task进行各种断言测试。同样，在Prefect中，你可以轻松导入和测试workflow。此外，在Airflow和Prefect中，对于独立的task逻辑，都可以进行Python风格的单元测试。

然而，在Airflow中测试DAG要比Prefect的workflow复杂的多。这是由于以下多种原因：

- Airflow中的DAG级别执行是由中枢调度器来控制协调的，意味着模拟数据来执行DAG也需要初始化准备数据和调度服务。将其纳入CI持续集成流程中测试对很多人来说都是个大麻烦
- Airflow的状态类型是字符串，这增加了测试数据传递的复杂性，或者引发什么类型异常，这需要数据库查询来确定

另外，在Prefect中，workflow作为参数简单传递给FlowRunner可以直接本地执行。此外，这类接口都提供了丰富的额外参数，专门用于帮助测试流程，还贴心的包括一种手动指定上游task任意状态的方法。

例如，要确保测试逻辑适用于单个任务。因为Prefect返回上游task的结果是组合完整的状态对象（包括数据、异常状态和重试次数等信息），这样你能通过task_states关键字参数模拟传递所有的上游任务状态，进而你可以只对关心的task返回状态做断言。


## UI操作界面

>   我选择相信。
>   
>
>
>
>   -- 电影《X档案》Fox Mulder

Airflow最受欢迎的是web界面。在UI操作界面上，你可以轻松打开关闭DAG，可视化DAG运行实例的进度，甚至可以对Airflow数据库进行查询。这是一种访问Airflow元信息的非常有用的方法。

因此从设计Prefect的第一天开始，就注重支持漂亮的实时的UI操作界面的理念，但Prefect并没有Airflow那样直接裸露数据库数据模型的操作，而是将最佳实践都制作成可视化界面，来应对用户使用过程最关心的问题：系统运行状况如何？并且如果出现问题，我如何快速定位？

![prefect](prefect-ui.gif)

Prefect UI操作界面支持如下：

- 系统概况的仪表版
- 调度新的输入参数执行
- 实时更新task和对应的运行状态
- 人工干预更新状态
- 流式日志，包括立即跳转最新错误日志的能力
- 完整的交互式的GraphQL API接口
- 全局搜索
- 代理控制
- 项目的workflow组成
- 团队管理和权限
- API访问Token生成
- 密码管理
- 全局并发限制
- 时区（适合Airflow用户）
- ......还有很多

UI操作界面是Prefect Cloud的一部分，它以能提供生产级workflow管理的底层结构作为支持。但是我们致力于从开源产品的云端免费套餐开始，逐渐将其完整地提供给用户。我们正在研究将元素抽象通过UI操作界面交付给开源用户的方式。


## Prefect的简单安装使用

安装
````bash
pip install prefect
````

task编写
````Python
from prefect import task, Flow, Parameter


@task(log_stdout=True)
def say_hello(name):
    print("Hello, {}!".format(name))


with Flow("My First Flow") as flow:
    name = Parameter('name')
    say_hello(name)


flow.run(name='world') # "Hello, world!"
flow.run(name='Marvin') # "Hello, Marvin!"

````

管理server启动
````bash
prefect server start
````


## 总结

>   如果我比其他人看的更远，是因为我站在巨人的肩上。
>   
>
>
>
>   -- 艾萨克·牛顿

Airflow普及了很多今天工程师认为理所当然的框架抽象和语义。不幸的是，随着数据科学工程的发展，它无法满足公司更多的动态业务需求。

Prefect是一种新的引擎工具，反映了从数以百计的工业用户那里收集的很多变化的需求，可用于启用定制化workflow，为各种数据应用程序提供丰富的数据抽象和接口。

如果你想了解更多信息，请查看我们的文档或访问我们的[源码仓库](https://github.com/PrefectHQ/prefect.git)。如果你想试用Prefect Cloud，可以注册一个免费账号。

***

- [英版原文](https://docs.prefect.io/core/getting_started/why-not-airflow.html)
- [开源库](https://github.com/PrefectHQ/prefect.git)
