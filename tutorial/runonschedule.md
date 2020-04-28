> 
> 跟随终端演示：
> 
> ````bash
> cd examples/tutorial
> python 05_schedules.py
> ````
> 

现在，航班数据ETL工作流流已经足够值得信赖，我们希望能够按计划持续运行它。Prefect提供了可以附加到flow的**Schedule**对象，来设置调度计划：

````Python
from datetime import timedelta, datetime
from prefect.schedules import IntervalSchedule

# ... task definitions ...

schedule = IntervalSchedule(
    start_date=datetime.utcnow() + timedelta(seconds=1),
    interval=timedelta(minutes=1),
)

with Flow("Aircraft-ETL", schedule=schedule) as flow:
    airport = Parameter("airport", default="IAD")
    radius = Parameter("radius", default=200)

    reference_data = extract_reference_data()
    live_data = extract_live_data(airport, radius, reference_data)

    transformed_live_data = transform(live_data, reference_data)

    load_reference_data(reference_data)
    load_live_data(transformed_live_data)
````

当调用**flow.run()**时，我们的工作流将永远不会停止，总是每分钟开始一次新的执行。

> 
> **Schedules附加信息。**
> 
> 有多种方法可以配置调度计划以满足各种需求。有关Schedules的更多信息，请参见[后续文档](https://docs.prefect.io/core/concepts/schedules.html#schedules)。
> 

接下来，轻松并发执行task。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/tutorial/05-running-on-a-schedule.html)
- [联系译者](https://github.com/listen-lavender)
