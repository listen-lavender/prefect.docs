在我们使用Prefect之前，先回顾一下一个典型的实际生产中的ETL工作流。

> 
> 跟随终端演示：
> 
> 教学代码一览
> ````bash
> git clone --depth 1 https://github.com/PrefectHQ/prefect.git
> cd prefect/examples/tutorial
> pip install -r requirements.txt
> ````
> 执行实例
> ````bash
> python 01_etl.py
> ````
> 

## Aircraft ETL(航班数据ETL应用)样例

在本教程中，我们试图获取和存储有关航班实时数据的信息，以用于将来的分析。未来的分析需要从多个来源提取，清理和合并数据。这种业务场景，我们使用实时航班数据（位置信息）和其他关联数据（机场位置，班次，航线计划信息）。

在引入任何Prefect概念镶嵌业务之前，让我们从解决问题的参考实现开始：

````Python
import aircraftlib as aclib

dulles_airport_position = aclib.Position(lat=38.9519444444, long=-77.4480555556)
area_surrounding_dulles = aclib.bounding_box(dulles_airport_position, radius_km=200)

# Extract: fetch data from multiple data sources
ref_data = aclib.fetch_reference_data()
raw_aircraft_data = aclib.fetch_live_aircraft_data(area=area_surrounding_dulles)

# Transform: clean the fetched data and add derivative data to aid in the analysis
live_aircraft_data = []
for raw_vector in raw_aircraft_data:
    vector = aclib.clean_vector(raw_vector)
    if vector:
        aclib.add_airline_info(vector, ref_data.airlines)
        live_aircraft_data.append(vector)

# Load: save the data for future analysis
db = aclib.Database()
db.add_live_aircraft_data(live_aircraft_data)
db.update_reference_data(ref_data)
````

上述代码的优点是易于阅读。但是，它的简单性与缺点的作用相当。首先，工作流程是严格线性的：

![Aircraft ETL](aircraftetl.png)

这导致错过计算机会：

 - 两个**Extract**步骤可以并行以节省时间
 - 同样的，两个**Load**步骤可以并行以节省时间

此外，不必要的线性流程带来了第二个问题：工作流中的task失败现在会影响业务不相关代码：

 - 如果获取实时航班数据失败，但是我们已经获取了更新的其他关联数据，我们将简单地丢弃关联数据
 - 如果在转换为结构化数据时出现意外问题，则关联数据会丢失
 - 如果数据转换成功但存储数据库不可用，则数据会丢失，并且工作流程将需要从头开始

这些不良表现说明该脚本的设计欠佳。这里涉及的缺失保障处理实践是我们在Prefect中称之为“业务负向工程实践”的一种，或者是必须采取人工编码预防措施（侵入业务代码），以确保工作流运行良好。上面的代码几乎没有这些预防措施。

> 
> 接下来
> 
> 接下来，我们将以Aircraft ETL为例，并使用Prefect改进工作流行为。
> 

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/tutorial/01-etl-before-prefect.html)
- [联系译者](https://github.com/listen-lavender)

