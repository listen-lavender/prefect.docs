![prefect](prefect-core.svg)

Prefect Core是一种新型的工作流管理工具，使得构建数据pipeline非常容易，并且能轻松添加重试、日志、动态映射、缓存、失败告警以及更多的附加功能。

我们从一个基本的前提开始：

> 
> 你的代码可能正常运行，但是有时候可能不行。
> 

当你的代码按照预期运行，你可能甚至不需要工作流框架。我们将为了实现业务逻辑开发代码视为正向工程实践。只有当出现问题时候，一个类似Prefect的系统的价值才会凸显。代码掺杂业务目标和成功失败稳定性保证的是负向工程实践。从这个角度看，工作流框架实际上是风险管理工具，像保险，需要的时候就在那里，不需要的时候看不到。

我们还没有一个工作流工具是按照风险管理理念设计的。那些工作流框架系统自认为是正向工程的工具，却以某种方式使用户做了风险管理之外的工作。结果造成，它们问心无愧地让开发者多写一个配置文件，或者将业务代码强行扭曲成复杂的DAG结构。Prefect已经知道你会写出意外的代码，它只是想保证代码的运行。

Prefect将代码转化成一个健壮的，分布式的pipeline。开发者能继续使用已有工具、语言、基础结构和脚本。Prefect按照支持正向工程实践的原则，支持丰富的DAG结构，并且不会阻碍业务。开发者可以通过少量的函数式钩子和功能API就能转化脚本，或者你可以直接访问延迟的计算图，或者任何组合。这由你决定。

> 
> 我们对于负向工程实践最常听闻的就是：“这很容易！”
> 
> 现在Prefect做到了。
> 
> 开心编码构建工程！
> 
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Prefect团队
> 