> 
> 跟随终端演示：
> 
> ````bash
> cd examples/tutorial
> python 03_parameterized_etl_flow.py
> ````
> 

上一节教程中，我们将Aircraft ETL脚本重构为Prefect flow。但是，**extract_live_data** task是硬编码，只能提取特定区域内的航班数据，在这种情况下，Dulles国际机场周围的半径为200公里：

````Python
@task
def extract_live_data():
    # Get the live aircraft vector data around Dulles airport
    dulles_airport_position = aclib.Position(lat=38.9519444444, long=-77.4480555556)
    area_surrounding_dulles = aclib.bounding_box(dulles_airport_position, radius_km=200)

    print("fetching live aircraft data...")
    raw_aircraft_data = aclib.fetch_live_aircraft_data(area=area_surrounding_dulles)

    return raw_aircraft_data
````

允许从广泛范围获取数据，而不仅仅是在单个机场附近，这是理想方案。一种方法是允许**extract_live_data**采用经度和纬度参数。但是，我们可以走得更远：事实证明，我们的关联数据中有已经机场位置信息，可以利用！

让我们重构我们的Python函数以从关联数据获取用户指定的机场：

````Python
@task
def extract_live_data(airport, radius, ref_data):
    # Get the live aircraft vector data around the given airport (or none)
    area = None
    if airport:
        airport_data = ref_data.airports[airport]
        airport_position = aclib.Position(
            lat=float(airport_data["latitude"]), long=float(airport_data["longitude"])
        )
        area = aclib.bounding_box(airport_position, radius)

    print("fetching live aircraft data...")
    raw_aircraft_data = aclib.fetch_live_aircraft_data(area=area)

    return raw_aircraft_data
````

假如你感到好奇，**area=None**将不会获取所有已知航班的实时数据，而不管其所在的区域如何。

如何在Prefect flow中控制这些函数参数呢？通过使用prefect.Parameter：

````Python
from prefect import Parameter

# ...task definitions...

with Flow("Aircraft-ETL") as flow:
    airport = Parameter("airport", default="IAD")
    radius = Parameter("radius", default=200)

    reference_data = extract_reference_data()
    live_data = extract_live_data(airport, radius, reference_data)

    transformed_live_data = transform(live_data, reference_data)

    load_reference_data(reference_data)
    load_live_data(transformed_live_data)
````

就像task一样，直到调用**flow.run()**时，才使用默认值（如果提供的话）或传递给**.run()**的覆盖值设定参数：

````Python
# Run the Flow with default airport=IAD & radius=200
flow.run()

# ...default radius and a different airport!
flow.run(airport="DCA")
````

最后，请注意我们的执行图已改变：现在获取实时数据取决于获取到的关联数据：

接下来，如果一个task失败了会怎样，以及我们如何制定出现问题时采取的措施?

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/tutorial/03-parameterized-flow.html)
- [联系译者](https://github.com/listen-lavender)
