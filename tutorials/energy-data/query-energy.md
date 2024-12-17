---
标题: 能源消耗数据教程 —— 查询数据
摘要: 能源消耗数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [教程，查询]
标签: [教程，初学者]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 分析能源消耗数据
---

# 查询数据

当您加载了数据集后，可以开始构建一些查询来发现数据告诉您什么。
本教程使用[Timescale hyperfunctions][about-hyperfunctions]来构建在标准PostgreSQL中不可能的查询。

在本节中，您将学习如何构建查询，以回答这些问题：

*   [按小时的能耗](#按小时的能耗是多少)
*   [按工作日的能耗](#按一周的哪一天能耗是多少)
*   [按月的能耗](#每月的能耗是多少)

## 按小时的能耗是多少？

当您为能耗数据设置好数据库后，可以构建一个查询，以找到典型一天中每小时的能耗中位数和最大值。

<Procedure>

### 查找每小时消耗多少千瓦时的能源

1. 连接到包含能耗数据集的Timescale数据库。
1. 在psql提示符下，使用Timescale Toolkit功能计算第五十百分位数或中位数。然后使用标准PostgreSQL max函数计算消耗的最大能源：

    ```sql
    WITH per_hour AS (
    SELECT
    time,
    value
    FROM kwh_hour_by_hour
    WHERE "time" at time zone 'Europe/Berlin' > date_trunc('month', time) - interval '1 year'
    ORDER BY 1
    ), hourly AS (
     SELECT
          extract(HOUR FROM time) * interval '1 hour' as hour,
          value
     FROM per_hour
    )
    SELECT
        hour,
        approx_percentile(0.50, percentile_agg(value)) as median,
        max(value) as maximum
    FROM hourly
    GROUP BY 1
    ORDER BY 1;
    ```

1. 您得到的数据看起来像这样：

    ```sql
          hour   |       median       | maximum
        ----------+--------------------+---------
         00:00:00 | 0.5998949812512439 |     0.6
         01:00:00 | 0.5998949812512439 |     0.6
         02:00:00 | 0.5998949812512439 |     0.6
         03:00:00 | 1.6015944383271534 |     1.9
         04:00:00 | 2.5986701108275327 |     2.7
         05:00:00 | 1.4007385207185301 |     3.4
         06:00:00 | 0.5998949812512439 |     2.7
         07:00:00 | 0.6997720645753496 |     0.8
         08:00:00 | 0.6997720645753496 |     0.8
         09:00:00 | 0.6997720645753496 |     0.8
         10:00:00 | 0.9003240409125329 |     1.1
         11:00:00 | 0.8001143897618259 |     0.9
    ```

</Procedure>

## 按一周的哪一天能耗是多少？

您还可以检查周末与工作日之间的能耗差异。

<Procedure>

### 查找工作日的能耗

1. 连接到包含能耗数据集的Timescale数据库。
1. 在psql提示符下，使用此查询找到工作日与周末之间的消耗差异：

    ```sql
    WITH per_day AS (
     SELECT
       time,
       value
     FROM kwh_day_by_day
     WHERE "time" at time zone 'Europe/Berlin' > date_trunc('month', time) - interval '1 year'
     ORDER BY 1
    ), daily AS (
        SELECT
           to_char(time, 'Dy') as day,
           value
        FROM per_day
    ), percentile AS (
        SELECT
            day,
            approx_percentile(0.50, percentile_agg(value)) as value
        FROM daily
        GROUP BY 1
        ORDER BY 1
    )
    SELECT
        d.day,
        d.ordinal,
        pd.value
    FROM unnest(array['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat']) WITH ORDINALITY AS d(day, ordinal)
    LEFT JOIN percentile pd ON lower(pd.day) = lower(d.day);

    ```

1. 您得到的数据看起来像这样：

    ```sql
        day | ordinal |       value
    -----+---------+--------------------
     Mon |       2 |  23.08078714975423
     Sun |       1 | 19.511430831944395
     Tue |       3 | 25.003118897837307
     Wed |       4 |   8.09300571759772
     Sat |       7 |
     Fri |       6 |
     Thu |       5 |
    ```

</Procedure>

## 每月的能耗是多少？

您可能还希望检查每月发生的能耗。

<Procedure>

### 查找每年每个月的能耗

1. 连接到包含能耗数据集的Timescale数据库。
1. 在psql提示符下，使用此查询找到每年的每个月的消耗：

    ```sql
     WITH per_day AS (
     SELECT
       time,
       value
     FROM kwh_day_by_day
     WHERE "time" > now() - interval '1 year'
     ORDER BY 1
    ), per_month AS (
       SELECT
          to_char(time, 'Mon') as month,
           sum(value) as value
       FROM per_day
      GROUP BY 1
    )
    SELECT
       m.month,
       m.ordinal,
       pd.value
    FROM unnest(array['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec']) WITH ORDINALITY AS m(month, ordinal)
    LEFT JOIN per_month pd ON lower(pd.month) = lower(m.month)
    ORDER BY ordinal;
    ```

1. 您得到的数据看起来像这样：

    ```sql
        month | ordinal |       value
        -------+---------+-------------------
        Jan   |       1 |
        Feb   |       2 |
        Mar   |       3 |
        Apr   |       4 |
        May   |       5 | 75.69999999999999
        Jun   |       6 |
        Jul   |       7 |
        Aug   |       8 |
        Sep   |       9 |
        Oct   |      10 |
        Nov   |      11 |
        Dec   |      12 |
    ```

1. [](#)<Optional /> 在Grafana中可视化这一点，创建一个新的面板，并选择`Bar Chart`可视化。选择能耗数据集作为您的数据源，并输入前一步的查询。在`Format as`部分，选择`Table`。

1. [](#)<Optional /> 选择一个配色方案，以便不同的消耗以不同的颜色显示。在选项面板中，`Standard options`下，将`Color scheme`更改为有用的`by value`范围。


   <img src="https://assets.timescale.com/docs/images/grafana-energy.webp" width="1375" height="944" alt="在Grafana中可视化能耗" />

</Procedure>

[about-hyperfunctions]: https://docs.timescale.com/use-timescale/latest/hyperfunctions/about-hyperfunctions/


