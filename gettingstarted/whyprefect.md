## 理念细节

Prefect是一个工作流引擎，要用户相信：1）它能工作；2）它能工作好。因此，Perfict的设计以强大的数据工程哲学为基石，我们的代码都是以高标准来开发的。

Prefect已经比任何其他工作流引擎（包括整个Airflow平台）拥有更多的单元测试和更高的测试覆盖率。文档是最重要的：每个模块、类和函数不仅有docstring，docstring本身也经过一致性测试。类型注解（Type annotations）用于捕获错误。我们甚至对变量和函数的命名进行了用户测试，以确保它们是清晰的。它是值得用户相信的，因为这种谨慎的标准甚至覆盖了代码库的用户自己可能永远不会接触到某些部分。

## Task即函数

从一个简单的意义上讲，Prefect task是一些附着关于何时运行等特殊规则的函数：它们接受输入（可选）、处理某些业务和返回输出（可选）。task可以直接处理数据，或者编排外部系统，或者调用其他语言环境：task可以做什么几乎没有限制。此外，每个task在运行之前都会接收到关于其上游task的元数据，即使上下游之间没有显式的业务数据依赖，这使它有机会根据flow的状态更改其行为。

因为Prefect是一个容错（假设开发者的代码有bug）的工程框架，所以Prefect服务的稳定性不会受每个task运行的影响。进而对于flow能输入和输出什么没有限制。

## Flow即容器（上下文）

Prefect的每个组件都采用模块化设计，从执行引擎可以轻松自定义或替换，日志记录，数据序列化和存储，以及本身的状态处理。作为一种容错良好的工程工具，Prefect为了支持业务实践的正向工程导向而设计，而不会决定开发者的实践就是正向工程。

## 基于State通信

Prefect使用State抽象概念来反映工作流的运行时行为。task和flow都产生State。

## 大规模并发

Prefect flow的设计特点是任何时候、任何原因、任意量的并发都可以。按照调度计划运行flow只是一种特殊场景。

## 支持幂等（非必需的）

幂等性一般被视为工作流管理系统的“救星”。当task保证业务幂等时，工程的挑战就变得非常容易。但是，保证幂等的task实现起来更困难。因此我们在没有任何幂等假设要求的理念下设计了Prefect。用户应该喜欢幂等task，因为它们天生健壮，但是Prefect不强制要求task幂等。

## 自动化框架

确切的自动化框架包含三个关键组件：

 - Workflow Definition(构建Workflow的定义)
 - Workflow Engine(执行Workflow的引擎)
 - Workflow State(设置Workflow的状态)

Prefect Core满足上述三条，而且每个组件的设计都来源于用户实践研究和应用经验。

### Workflow Definition

从多方面看，定义工作流程是最容易的部分。这是偏向业务负向工程实践的一种显式描述依赖的机会：task如何相互依赖，应在何种情况下运行，对任何基础库有何依赖。

许多工作流框架都设计成在配置文件或详细数据结构中定义工作流。在设计Prefect时，我们进行的关于用户实践的所有研究中，没有一个人说他想要更明确的显式工作流定义。也没有一次听到要求更多的YAML配置。从未有人自愿重写其代码来适配工作流框架的API。

因此，Prefect将这种用于工作流定义的方法视为失败的设计。虽然Prefect能为每个工作流都构建一个完全自省和可定制化的DAG定义，但如果用户不希望与之打交道，就永远不需要它。对应的，Python就是API。用户定义Python普通函数并像在任何脚本中一样调用它们，Prefect看框架的工作就是收集task依赖，生成工作流结构。

### Workflow Engine

一旦定义了工作流，我们需要执行它。这是工程核心。

Prefect引擎是一个强壮的管道，它嵌入了用于调度执行工作流的逻辑。它的本质是一个规则系统，用于判断task是否要运行，运行时干什么，以及停止时干什么。

仅仅能按顺序启动每个task是不够的，task可能会因为成功，失败，跳过，暂停甚至崩溃而停止运行！任何一种结果都需要引擎或者下游task做出不同的响应。引擎的作用是检查工作流定义的依赖，并保证task遵从与之相关的依赖规则

### Workflow State

可以说，Prefect的主要创新还不在于简化工作流的定义或者强大的工作流引擎，而是工作流状态的丰富抽象。

大多数工作流框架的标准是默认工作流会成功。这意味着如果task停止而没有崩溃，则将其视为成功，否则才是失败。这是令人难以置信的世界观。如果你想跳过task呢？如果你想中止task呢？如果整个系统出现故障，是否可以恢复线程呢？如果要在不相交的环境的多个系统运行该task，要保证运行一次呢？

通过为每一个task和flow专门设计state的抽象概念，使得Prefect能在工作流执行前、执行中、执行后的任何时刻描述它。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/getting_started/why-prefect.html)
- [联系译者](https://github.com/listen-lavender)

