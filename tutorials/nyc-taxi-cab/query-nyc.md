---
标题: 查询时间序列数据教程 —— 查询数据
摘要: 查询时间序列数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [教程，查询]
标签: [教程，初学者]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容_分区: 分析纽约市出租车数据
---

# 查询数据

当您的数据集加载完成后，您可以开始构建一些查询来发现数据背后的含义。在本节中，您将学习如何编写回答以下问题查询：

*   [每天有多少行程发生？](#每天发生多少次行程)
*   [平均车费金额是多少？](#平均车费金额是多少)
*   [每种费率类型的行程各有多少？](#每种费率类型的行程各有多少)
*   [前往和离开机场的行程是哪种类型？](#前往和离开机场的行程是哪种类型)
*   [2016年元旦发生了多少行程？](#2016年元旦发生了多少行程)

## 每天发生多少次行程？

此数据集包含2016年1月的行程数据。要找出每天发生了多少行程，您可以使用`SELECT`语句。在这种情况下，您想要计算每天的总行程数，并按日期以列表形式显示它们。

<Procedure>

### 找出每天发生多少次行程

1. 连接到包含纽约出租车数据集的Timescale数据库。
2. 在psql提示符下，使用此查询选择2016年1月第一周的所有行程，并返回每天的行程计数：

    ```sql
    SELECT date_trunc('day', pickup_datetime) as day,
    COUNT(*) FROM rides
    WHERE pickup_datetime < '2016-01-08'
    GROUP BY day
    ORDER BY day;
    ```

    查询结果如下所示：

    ```sql
             day         | count
    ---------------------+--------
     2016-01-01 00:00:00 | 345037
     2016-01-02 00:00:00 | 312831
     2016-01-03 00:00:00 | 302878
     2016-01-04 00:00:00 | 316171
     2016-01-05 00:00:00 | 343251
     2016-01-06 00:00:00 | 348516
     2016-01-07 00:00:00 | 364894
     ```

</Procedure>

## 平均车费金额是多少？

您可以在`SELECT`查询中包含一个函数，以确定每位乘客支付的平均车费。

<Procedure>

### 找出平均车费金额

1. 连接到包含纽约出租车数据集的Timescale数据库。
2. 在psql提示符下，使用此查询选择2016年1月第一周的所有行程，并返回每天的平均车费：

    ```sql
    SELECT date_trunc('day', pickup_datetime)
    AS day, avg(fare_amount)
    FROM rides
    WHERE pickup_datetime < '2016-01-08'
    GROUP BY day
    ORDER BY day;
    ```

    查询结果如下所示：

    ```sql
             day         |         avg
    ---------------------+-------------------
     2016-01-01 00:00:00 | 12.8569325028909943
     2016-01-02 00:00:00 | 12.4344713599355563
     2016-01-03 00:00:00 | 13.0615900461571986
     2016-01-04 00:00:00 | 12.2072927308323660
     2016-01-05 00:00:00 | 12.0018670885154013
     2016-01-06 00:00:00 | 12.0002329017893009
     2016-01-07 00:00:00 | 12.1234180337303436
    ```

</Procedure>

## 每种费率类型的行程各有多少？

纽约市的出租车对不同类型的行程使用了一系列不同的费率类型。例如，前往机场的行程从市内任何地点都按统一费率收费。本节展示了如何构建一个查询，显示每种不同费率类型的行程数量。它还使用了`JOIN`语句以更富有信息的方式呈现数据。

<Procedure>

### 找出每种费率类型的行程数量

1. 连接到包含纽约出租车数据集的Timescale数据库。
2. 在psql提示符下，使用此查询选择2016年1月第一周的所有行程，并返回每种费率代码的总行程数：

    ```sql
    SELECT rate_code, COUNT(vendor_id) AS num_trips
    FROM rides
    WHERE pickup_datetime < '2016-01-08'
    GROUP BY rate_code
    ORDER BY rate_code;
    ```

    查询结果如下所示：

    ```sql
     rate_code | num_trips
    -----------+-----------
             1 |   2266401
             2 |     54832
             3 |      4126
             4 |       967
             5 |      7193
             6 |        17
            99 |        42
    ```

</Procedure>

输出是正确的，但不太容易阅读，因为您可能不知道不同的费率代码代表什么。然而，数据集中的`rates`表包含了每个代码的人类可读描述。您可以在查询中使用`JOIN`语句连接`rides`和`rates`表，并在结果中呈现两个表的信息。

<Procedure>

### 显示每种费率类型的行程数量

1. 连接到包含纽约出租车数据集的Timescale数据库。
2. 在psql提示符下，复制此查询以选择2016年1月第一周的所有行程，连接`rides`和`rates`表，并返回每种费率代码的总行程数，以及费率代码的描述：

    ```sql
    SELECT rates.description, COUNT(vendor_id) AS num_trips
    FROM rides
    JOIN rates ON rides.rate_code = rates.rate_code
    WHERE pickup_datetime < '2016-01-08'
    GROUP BY rates.description
    ORDER BY LOWER(rates.description);
    ```

    查询结果如下所示：

    ```sql
          description      | num_trips
    -----------------------+-----------
     group ride            |        17
     JFK                   |     54832
     Nassau or Westchester |       967
     negotiated fare       |      7193
     Newark                |      4126
     standard rate         |   2266401
    ```

</Procedure>

## 前往和离开机场的行程是哪种类型？

数据集中有两个主要机场：约翰·F·肯尼迪机场（JFK）由费率代码2代表；纽瓦克机场（EWR）由费率代码3代表。

关于前往和离开这两个机场的行程信息，对于城市规划以及像纽约旅游局这样的组织来说非常有用。

本节展示了如何构建一个查询，返回只前往新的主要机场的行程信息。

<Procedure>

### 找出前往和离开机场的行程类型

1. 连接到包含纽约出租车数据集的Timescale数据库。
2. 在psql提示符下，使用此查询选择2016年1月第一周所有前往和离开JFK和纽瓦克机场的行程，并返回该机场的行程数量、平均行程持续时间、平均行程费用和平均乘客数量：

    ```sql
    SELECT rates.description,
        COUNT(vendor_id) AS num_trips,
        AVG(dropoff_datetime - pickup_datetime) AS avg_trip_duration,
        AVG(total_amount) AS avg_total,
        AVG(passenger_count) AS avg_passengers
    FROM rides
    JOIN rates ON rides.rate_code = rates.rate_code
    WHERE rides.rate_code IN (2,3) AND pickup_datetime < '2016-01-08'
    GROUP BY rates.description
    ORDER BY rates.description;
    ```

    查询结果如下所示：

    ```sql
     description | num_trips | avg_trip_duration |      avg_total      |   avg_passengers
    -------------+-----------+-------------------+--------------------+-------------------
     JFK         |     54832 | 00:46:44.614222   | 63.7791311642836300 | 1.8062080536912752
     Newark      |      4126 | 00:34:45.575618   | 84.3841783809985458 | 1.8979641299079011
    ```

</Procedure>

## 2016年元旦发生了多少行程？

纽约市以时代广场的新年落球庆祝活动而闻名。成千上万的人聚集在此迎接新年，然后分散到城市各处：去他们最喜欢的酒吧，与朋友共进餐，或回家。本节展示了如何构建一个查询，返回2016年1月1日的出租车行程数量，以30分钟为间隔。

在PostgreSQL中，按30分钟时间间隔分割数据并不特别容易。为此，您需要使用`TRUNC`函数计算行程开始的分钟数除以30的商，然后截断结果以取该商的底数。当您得到这个结果后，您可以将截断的商乘以30。

在您的Timescale数据库中，您可以使用`time_bucket`函数将数据分割成时间间隔。

<Procedure>

### 找出2016年元旦发生了多少行程

1. 连接到包含纽约出租车数据集的Timescale数据库。
2. 在psql提示符下，使用此查询选择2016年1月1日的所有行程，并返回每30分钟的行程计数：

    ```sql
    SELECT time_bucket('30 minute', pickup_datetime) AS thirty_min, count(*)
    FROM rides
    WHERE pickup_datetime < '2016-01-02 00:00'
    GROUP BY thirty_min
    ORDER BY thirty_min;
    ```

    查询结果开始如下所示：

    ```sql
         thirty_min      | count
    ---------------------+-------
     2016-01-01 00:00:00 | 10920
     2016-01-01 00:30:00 | 14350
     2016-01-01 01:00:00 | 14660
     2016-01-01 01:30:00 | 13851
     2016-01-01 02:00:00 | 13260
     2016-01-01 02:30:00 | 12230
     2016-01-01 03:00:00 | 11362
    ```

</Procedure>


