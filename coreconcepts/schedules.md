## 概览

Prefect假设可以出于任何原因随时运行flow实例。但是，在指定时间自动执行flow实例通常很有用。可以通过**schedule**关键字参数简单的将调度计划附加到flow。对于更详细或更复杂的调度计划，Prefect提供通用的**schedule**对象，该对象可进行细微的日期时间调整和过滤，以及根据计划的时间更新参数值。

## 简单调度计划

可以通过**schedule**关键字参数简单的将调度计划附加到flow：

````Python
from prefect import task, Flow
from datetime import timedelta
from prefect.schedules import IntervalSchedule

@task
def say_hello():
    print("Hello, world!")

schedule = IntervalSchedule(interval=timedelta(minutes=2))

with Flow("Hello", schedule) as flow:
    say_hello()

flow.run()
````

你可以在此[示例](https://docs.prefect.io/core/examples/daily_github_stats_to_airtable.html)中看到cron计划。

## 复杂调度计划

Prefect调度计划包含三个组件：

 - 发出事件的时钟。例如，一个**IntervalClock**可能每小时发出一个事件；一个**CronClock**可以根据cron定时类型字符串发出事件；单个调度计划可能包含多个时钟。时钟还可以用于为每个flow运行实例指定不同的参数值
 - 决定是否应包含事件的过滤器。例如，可能设置了仅允许在工作日或在工作时间内发送事件的过滤器
 - 可通过过滤器修改事件的调整。例如，一个调整可以将事件延迟到下一个工作日或该月的最后一个工作日。

这三个组件允许用户将简单函数组合为复杂行为。

### 时间间隔时钟

最基本的Prefect时钟是**IntervalClock**。它采用一个时间间隔参数，并定期发出事件。可以设置可选参数**start_date**，在这种情况下，时间间隔是相对于该日期的，还可以提供可选参数**end_date**。

Prefect不支持小于或等于分钟级别的计划。

````Python
from datetime import timedelta
from prefect.schedules import Schedule
from prefect.schedules.clocks import IntervalClock

schedule = Schedule(clocks=[IntervalClock(timedelta(hours=24))])

schedule.next(5)
````

> 
> **时区**
> 
> 想要将调度计划固定在某个时区吗？为你的时钟指定一个与该时区相对应的**start_date**，例如：
> 
> ````Python
> schedules.clocks.IntervalClock(
>     start_date=pendulum.datetime(2019, 1, 1, tz="America/New_York"),
>     interval=timedelta(days=1)
> )
> ````
> 

关于夏令时。

> 
> **夏令时**
> 
> 如果**IntervalClock**的开始时间带有DST观察时区，则调度计划将进行适当的调整。大于24小时的时间间隔将遵循DST标准，而小于24小时的时间间隔将遵循UTC标准。例如，一个小时的调度计划将在每个UTC小时触发一次，甚至跨越DST边界。当时钟回拨时，这将导致两个运行实例似乎都安排在本地时间凌晨1点进行，即使它们与UTC时间相隔一个小时。对于较长的时间间隔，例如每日调度计划，时间间隔调度计划将针对DST边界进行调整，以使时钟的小时保持不变。这意味着始终在上午9点触发的每日调度计划会遵守DST标准，并会继续在当地时区的上午9点触发。
> 
> 注意此行为与**CronClock**不同。
> 

### 时间点定时时钟

时钟也可以通过用cron定时类型字符串设置Prefect的**CronClock**来生成：

````Python
from datetime import timedelta
from prefect.schedules import Schedule
from prefect.schedules.clocks import CronClock

schedule = Schedule(clocks=[CronClock("0 0 * * *")])

schedule.next(5)
````

> 
> **夏令时**
> 
> 如果**CronClock**的开始时间带有DST观察时区，则调度计划将自行调整。Cron的DST规则基于时钟时间，而不是时间间隔。这意味着每小时cron计划将在每个新的小时的时间点而非每隔一个小时触发。例如，如果将时钟回拨，这将导致两个小时的时间点触发暂停，因为调度计划将在120分钟后的第一次凌晨1点触发和第一次凌晨2点触发。较长的调度计划（例如每天早上9点触发的调度计划）会自动调整DST。
> 
> 注意此行为与**IntervalClock**不同。
> 

### 日期时钟

对于更不同的场景的调度计划，Prefect提供一个**DatesClock**，仅在用户指定的特定日期触发。

````Python
from datetime import timedelta
import pendulum
from prefect.schedules import Schedule
from prefect.schedules.clocks import DatesClock

schedule = Schedule(
    clocks=[DatesClock([pendulum.now().add(days=1), pendulum.now().add(days=2)])])

schedule.next(2)
````

### 调度计划产生Prefect参数（0.9.2以上版本支持）

所有时钟都支持可选参数**parameter_defaults**，该参数允许用户为由此时钟生成的每个flow运行实例指定不同的Prefect参数。例如，假设我们有以下flow记录传递给它的Parameter的值：

````Python
import prefect
from prefect import task, Flow, Parameter

@task
def log_param(p):
    logger = prefect.context['logger']
    logger.info("Received parameter value {}".format(p))

p = Parameter("p", default=None, required=False)

with Flow("Varying Parameters") as flow:
    log_param(p)
````

每次运行此flow时，我们都可以选择为Prefect参数p传递一个新值，如果要按固定的调度计划运行flow，则可能要根据调用哪个调度计划为p设定不同的值，我们可以通过使用时钟来做到这一点：

````Python
import datetime
from prefect.schedules import clocks, Schedule

now = datetime.datetime.utcnow()

clock1   = clocks.IntervalClock(start_date=now, 
                                interval=datetime.timedelta(minutes=1), 
                                parameter_defaults={"p": "CLOCK 1"})
clock2   = clocks.IntervalClock(start_date=now + datetime.timedelta(seconds=30), 
                                interval=datetime.timedelta(minutes=1), 
                                parameter_defaults={"p": "CLOCK 2"})

# the full schedule
schedule = Schedule(clocks=[clock1, clock2])

flow.schedule = schedule # set the schedule on the Flow
flow.run()
````

当按上述调度计划运行此flow时，每次新的实例运行都会在日志中看到Prefect参数值变化：

````bash
...
INFO - prefect.Task: log_param | Received parameter value CLOCK 2
...
INFO - prefect.Task: log_param | Received parameter value CLOCK 1
...
````

### 过滤器

Prefect提供了丰富多样的事件过滤器，包括：

 - on_datetime（允许在特定日期时间发送事件）
 - on_date（允许在特定日期发送事件，例如3月15日）
 - at_time（允许在特定时间发送事件，例如下午3:30）
 - between_datetimes（允许两个特定日期时间之间发送事件）
 - between_times（允许两个事件之间发送事件，例如上午9点至下午5点）
 - between_dates（允许两个日历日期之间发送事件，例如1月1日至3月31日）
 - is_weekday（允许在工作日发送事件）
 - is_weekend（允许在周末发送事件）
 - is_month_end（允许在月末发送事件）

Filters can be provided to schedules in three different ways:
可以通过三种不同的方式向计划提供过滤器：

 - filters：所有过滤器必须返回True才能发送事件
 - or_filters：至少有一个filter返回True才能发送事件
 - not_filters：所有过滤器必须返回False才能发送事件

````Python
schedules.Schedule(
    # fire every hour
    clocks=[clocks.IntervalClock(timedelta(hours=1))],
    # but only on weekdays
    filters=[filters.is_weekday],
    # and only at 9am or 3pm
    or_filters=[
        filters.between_times(pendulum.time(9), pendulum.time(9)),
        filters.between_times(pendulum.time(15), pendulum.time(15)),
    ],
    # and not in January
    not_filters=[filters.between_dates(1, 1, 1, 31)]
)
````

### 调整

通过调整，调度计划可以修改时钟发出的日期，并传入多个过滤器：

 - 添加（为日期添加间隔）
 - **next_weekday**（将日期延迟到下一个工作日）

````Python
schedules.Schedule(
    # fire every day
    clocks=[clocks.IntervalClock(timedelta(days=1))],

    # filtered for month ends
    filters=[filters.is_month_end],

    # and run on the next weekday
    adjustments=[adjustments.next_weekday]
    )
````

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/schedules.html)
- [联系译者](https://github.com/listen-lavender)

