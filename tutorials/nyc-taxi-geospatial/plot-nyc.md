---
标题: 绘制地理空间时间序列数据教程 —— 查询数据
摘要: 查询地理空间时间序列数据。
产品: [云服务]
关键词: [教程，地理信息系统（GIS），地理空间，学习]
标签: [教程，中级]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 绘制纽约市出租车地理空间数据
---

# 查询数据

当您的数据集加载完成后，您可以开始构建一些查询来发现数据背后的含义。在本节中，您将学习如何将纽约出租车数据集与[PostGIS][postgis]的地理空间数据结合起来，以回答以下问题：

*   [2016年元旦有多少行程是从时代广场出发的？](#2016年元旦有多少行程是从时代广场出发的)
*   [在曼哈顿，哪些行程行驶了超过5英里？](#在曼哈顿，哪些行程行驶了超过5英里)

## 为PostGIS设置您的数据集

要回答这些地理空间问题，您需要纽约出租车数据集中的行程计数数据，但您还需要一些地理空间数据来确定哪些行程是从哪里出发的。Timescale与所有其他PostgreSQL扩展兼容，因此您可以使用[PostGIS][postgis]扩展按时间和地点切片数据。

加载扩展后，您需要修改超表，使其准备好进行地理空间查询。`rides`表包含接客纬度和经度的列，但需要将其转换为几何坐标，以便与PostGIS良好协作。

<Procedure>

### 为PostGIS设置您的数据集

1. 连接到包含纽约出租车数据集的Timescale数据库。
2. 在psql提示符下，添加PostGIS扩展：

    ```sql
    CREATE EXTENSION postgis;
    ```

    您可以通过运行`\dx`命令检查PostGIS是否正确安装，它会出现在扩展列表中。
3. 修改超表，添加用于接客和落客位置的几何列：

    ```sql
    ALTER TABLE rides ADD COLUMN pickup_geom geometry(POINT,2163);
    ALTER TABLE rides ADD COLUMN dropoff_geom geometry(POINT,2163);
    ```

4. 将纬度和经度点转换为几何坐标，以便与PostGIS良好协作。这可能需要一些时间，因为它需要更新这两个列中的所有数据：

    ```sql
    UPDATE rides SET pickup_geom = ST_Transform(ST_SetSRID(ST_MakePoint(pickup_longitude,pickup_latitude),4326),2163),
       dropoff_geom = ST_Transform(ST_SetSRID(ST_MakePoint(dropoff_longitude,dropoff_latitude),4326),2163);
    ```

</Procedure>

## 2016年元旦有多少行程是从时代广场出发的？

当您的数据库为PostGIS数据设置好后，您可以构建一个查询，返回2016年元旦从时代广场出发的行程数量，以30分钟为间隔。

<Procedure>

### 找出2016年元旦从时代广场出发的行程数量

<Highlight type="note">
时代广场位于(40.7589,-73.9851)。
</Highlight>

1. 连接到包含纽约出租车数据集的Timescale数据库。
2. 在psql提示符下，使用此查询选择2016年1月1日从时代广场400米范围内接客的所有行程，并返回每30分钟的行程计数：

    ```sql
    SELECT time_bucket('30 minutes', pickup_datetime) AS thirty_min,
        COUNT(*) AS near_times_sq
    FROM rides
    WHERE ST_Distance(pickup_geom, ST_Transform(ST_SetSRID(ST_MakePoint(-73.9851,40.7589),4326),2163)) < 400
    AND pickup_datetime < '2016-01-01 14:00'
    GROUP BY thirty_min
    ORDER BY thirty_min;
    ```

3. 返回的数据看起来像这样：

    ```sql
         thirty_min      | near_times_sq
    ---------------------+---------------
     2016-01-01 00:00:00 |            74
     2016-01-01 00:30:00 |           102
     2016-01-01 01:00:00 |           120
     2016-01-01 01:30:00 |            98
     2016-01-01 02:00:00 |           112
    ```

</Procedure>

## 在曼哈顿，哪些行程行驶了超过5英里？

这个查询非常适合在地图上绘制。它查看了在曼哈顿城市内行驶超过5英里的行程。

在这个查询中，您希望返回超过5英里的行程，但还包括距离，以便您可以使用不同的视觉效果来可视化更长的距离。查询还包括一个`WHERE`子句来应用地理空间边界，寻找在时代广场2公里范围内的行程。最后，在`GROUP BY`子句中，提供`trip_distance`和位置变量，以便Grafana可以正确地绘制数据。

<Procedure>

### 找出在曼哈顿行驶超过5英里的行程

1. 连接到包含纽约出租车数据集的Timescale数据库。
2. 在psql提示符下，使用此查询在曼哈顿找到超过5英里的行程：

    ```sql
    SELECT time_bucket('5m', rides.pickup_datetime) AS time,
           rides.trip_distance AS value,
           rides.pickup_latitude AS latitude,
           rides.pickup_longitude AS longitude
    FROM rides
    WHERE rides.pickup_datetime BETWEEN '2016-01-01T01:41:55.986Z' AND '2016-01-01T07:41:55.986Z' AND
      ST_Distance(pickup_geom,
                  ST_Transform(ST_SetSRID(ST_MakePoint(-73.9851,40.7589),4326),2163)
      ) < 2000
    GROUP BY time,
             rides.trip_distance,
             rides.pickup_latitude,
             rides.pickup_longitude
    ORDER BY time
    LIMIT 500;
    ```

3. 返回的数据看起来像这样：

    ```sql
            time         | value |      latitude      |      longitude
    ---------------------+-------+--------------------+-------------------
     2016-01-01 01:40:00 |  0.00 | 40.752281188964844 | -73.975021362304688
     2016-01-01 01:40:00 |  0.09 | 40.755722045898437 | -73.967872619628906
     2016-01-01 01:40:00 |  0.15 | 40.752742767333984 | -73.977737426757813
     2016-01-01 01:40:00 |  0.15 | 40.756877899169922 | -73.969779968261719
     2016-01-01 01:40:00 |  0.18 | 40.756717681884766 | -73.967330932617188
     ...
    ```

4. [](#)<Optional /> 在Grafana中可视化这一点，创建一个新的面板，并选择`Geomap`可视化。选择纽约出租车数据集作为您的数据源，并输入前一步的查询。在`Format as`部分，选择`Table`。您的世界地图现在在纽约上方显示一个点，放大以查看可视化效果。
5. [](#)<Optional /> 要使这个可视化更有用，改变行程显示的方式。在选项面板中，`Data layer`下，添加一个名为`Distance traveled`的图层，并选择`markers`选项。在`Color`部分，选择`value`。您还可以在这里调整符号和大小。
6. [](#)<Optional /> 选择一个颜色方案，以便不同行程长度以不同颜色显示。在选项面板中，`Standard options`下，将`Color scheme`更改为有用的`by value`范围。这个例子使用了`Blue-Yellow-Red (by value)`选项。

    <img
    class="main-content__illustration"
    src="https://assets.timescale.com/docs/images/grafana-postgis.webp" 
    width={1375} height={944}
    alt="在Grafana中按距离可视化出租车行程"
    />

</Procedure>

[postgis]: http://postgis.net/
