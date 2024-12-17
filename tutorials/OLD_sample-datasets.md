---
标题: 示例数据集
摘要: 下载这些示例数据集，以便开始探索 TimescaleDB。
---

# 样本数据集

Timescale 创建了几个样本数据集，以帮助您开始使用 TimescaleDB。这些数据集在数据库大小、时间间隔数量和分区字段的值的数量上各不相同。

每个 gzip 归档包含一个 `.sql` 文件，用于在数据库中创建必要的超表，以及几个包含要复制到这些表中的数据的 `.csv` 文件。这些文件假设您导入它们的数据库已经[设置了 TimescaleDB 扩展][installation]。

**设备操作**：这些数据集包括从移动设备收集的 CPU、内存和网络等指标。点击名称下载。

对于更多细节和示例用法，参见[设备操作数据集](#device-ops-datasets)。

**天气**：这些数据集包括来自各种位置的温度和湿度数据。点击名称下载。

对于更多细节和示例用法，参见[天气数据集](#weather-datasets)。

## 导入
简要来说，导入步骤如下：

1.  设置带有 TimescaleDB 的数据库。
2.  解压缩归档。
3.  通过 `psql` 导入 `.sql` 文件以创建超表。
4.  通过 `psql` 从 `.csv` 文件导入数据。

每个数据集的名称格式为 `[数据集]_[size].tar.gz`。
例如，`devices_small.tar.gz` 是数据集 `devices` 和大小 `small`。
每个数据集包含一个名为 `[数据集].sql` 的 `.sql` 文件和几个命名为 `[数据集]_[size]_[table].csv` 的 CSV 文件。

例如，如果您想导入 `devices_small` 数据集，它从 `devices.sql` 创建两个表（`device_info` 和一个名为 `readings` 的超表）。
因此，有两个 CSV 文件：`devices_small_readings.csv` 和 `devices_small_device_info.csv`。
所以，要将此数据集导入名为 `devices_small` 的 TimescaleDB 数据库：

```bash
# (1) 解压缩归档
tar -xvzf devices_small.tar.gz

# (2) 将 .sql 文件导入数据库
psql -U postgres -d tsdb < devices.sql

# (3) 从 .csv 文件导入数据到数据库
psql -U postgres -d tsdb -c "\COPY readings FROM devices_small_readings.csv CSV"
psql -U postgres -d tsdb -c "\COPY device_info FROM devices_small_device_info.csv CSV"
```

数据现在已准备好使用。

<Highlight type="tip">
PostgreSQL 中的标准 `COPY` 命令是单线程的。要加速导入较大的样本数据集，您可以使用[并行导入工具](https://github.com/timescale/timescaledb-parallel-copy) 代替。
</Highlight>

```bash
# 访问您的数据库（例如：`tsdb`）
psql -U postgres -h localhost -d tsdb
```

## 设备操作数据集

导入这些数据集之一（`devices_small`、`devices_med`、`devices_big`）后，您将拥有一个名为 `device_info` 的普通 PostgreSQL 表和一个名为 `readings` 的超表。
`device_info` 表包含有关每个设备的静态元数据，例如操作系统名称和制造商。
`readings` 超表跟踪来自每个设备的数据，例如 CPU 活动或内存水平。
超表作为单个表暴露，因此您可以像普通 SQL 表一样查询它们并与元数据进行连接，如本节示例查询所示。

#### 模式

```sql
表 "public.device_info"
列       | 类型 | 修饰符
-------------+------+-----------
device_id    | text |
api_version  | text |
manufacturer | text |
model        | text |
os_name      | text |
```

```sql
表 "public.readings"
列              |       类型       | 修饰符
--------------------+------------------+-----------
time                | bigint           |
device_id           | text             |
battery_level       | double precision |
battery_status      | text             |
battery_temperature | double precision |
bssid               | text             |
cpu_avg_1min        | double precision |
cpu_avg_5min        | double precision |
cpu_avg_15min       | double precision |
mem_free            | double precision |
mem_used            | double precision |
rssi                | double precision |
ssid                | text             |
索引：
  "readings_device_id_time_idx" btree (device_id, "time" DESC)
  "readings_time_idx" btree ("time" DESC)
```

#### 示例查询

使用 `devices_med` 数据集

**充电设备的最新 10 个电池温度读数**

```sql
SELECT time, device_id, battery_temperature
FROM readings
WHERE battery_status = 'charging'
ORDER BY time DESC LIMIT 10;

time                   | device_id  | battery_temperature
-----------------------+------------+---------------------
2016-11-15 23:39:30-05 | demo004887 |                99.3
2016-11-15 23:39:30-05 | demo004882 |               100.8
2016-11-15 23:39:30-05 | demo004862 |                95.7
2016-11-15 23:39:30-05 | demo004844 |                95.5
2016-11-15 23:39:30-05 | demo004841 |                95.4
2016-11-15 23:39:30-05 | demo004804 |               101.6
2016-11-15 23:39:30-05 | demo004784 |               100.6
2016-11-15 23:39:30-05 | demo004760 |                99.1
2016-11-15 23:39:30-05 | demo004731 |                97.9
2016-11-15 23:39:30-05 | demo004729 |                99.6
(10 rows)
```

**电池电量低于 33% 且不在充电状态的最繁忙设备（1 分钟平均值）**

```sql
SELECT time, readings.device_id, cpu_avg_1min,
battery_level, battery_status, device_info.model
FROM readings
JOIN device_info ON readings.device_id = device_info.device_id
WHERE battery_level < 33 AND battery_status = 'discharging'
ORDER BY cpu_avg_1min DESC, time DESC LIMIT 5;

time                   | device_id  | cpu_avg_1min | battery_level | battery_status |  model
-----------------------+------------+--------------+---------------+----------------+---------
2016-11-15 23:30:00-05 | demo003764 |        98.99 |            32 | discharging    | focus
2016-11-15 22:54:30-05 | demo001935 |        98.99 |            30 | discharging    | pinto
2016-11-15 19:10:30-05 | demo000695 |        98.99 |            23 | discharging    | focus
2016-11-15 16:46:00-05 | demo002784 |        98.99 |            18 | discharging    | pinto
2016-11-15 14:58:30-05 | demo004978 |        98.99 |            22 | discharging    | mustang
(5 rows)
```

```sql
SELECT date_trunc('hour', time) "hour",
min(battery_level) min_battery_level,
max(battery_level) max_battery_level
FROM readings r
WHERE r.device_id IN (
    SELECT DISTINCT device_id FROM device_info
    WHERE model = 'pinto' OR model = 'focus'
) GROUP BY "hour" ORDER BY "hour" ASC LIMIT 12;

hour                   | min_battery_level | max_battery_level
-----------------------+-------------------+-------------------
2016-11-15 07:00:00-05 |                17 |                99
2016-11-15 08:00:00-05 |                11 |                98
2016-11-15 09:00:00-05 |                 6 |                97
2016-11-15 10:00:00-05 |                 6 |                97
2016-11-15 11:00:00-05 |                 6 |                97
2016-11-15 12:00:00-05 |                 6 |                97
2016-11-15 13:00:00-05 |                 6 |                97
2016-11-15 14:00:00-05 |                 6 |                98
2016-11-15 15:00:00-05 |                 6 |               100
2016-11-15 16:00:00-05 |                 6 |               100
2016-11-15 17:00:00-05 |                 6 |               100
2016-11-15 18:00:00-05 |                 6 |               100

(12 rows)
```

---

## 天气数据集

导入这些数据集之一（`weather_small`、`weather_med`、`weather_big`）后，您将注意到一个名为 `locations` 的普通 PostgreSQL 表和一个名为 `conditions` 的超表。
`locations` 表包含有关每个位置的元数据，例如其名称和环境类型。
`conditions` 超表跟踪来自这些位置的温度和湿度读数。
由于超表作为单个表暴露，您可以像普通 SQL 表一样查询它们并与元数据进行连接，如本节示例查询所示。

#### 模式

```sql
表 "public.locations"
列      | 类型 | 修饰符
------------+------+-----------
device_id   | text |
location    | text |
environment | text |
```

```sql
表 "public.conditions"
列      |           类型           | 修饰符
------------+--------------------------+-----------
time        | timestamp with time zone | not null
device_id   | text                     |
temperature | double precision         |
humidity    | double precision         |
索引：
"conditions_device_id_time_idx" btree (device_id, "time" DESC)
"conditions_time_idx" btree ("time" DESC)
```

#### 示例查询

使用 `weather_med` 数据集。

**最后十个读数**

```sql
SELECT * FROM conditions c ORDER BY time DESC LIMIT 10;

time                   |     device_id      |    temperature     |      humidity
-----------------------+--------------------+--------------------+--------------------
2016-12-06 02:58:00-05 | weather-pro-000000 |  84.10000000000034 |  83.7000000000053
2016-12-06 02:58:00-05 | weather-pro-000001 | 35.999999999999915 |  51.7999999999994
2016-12-06 02:58:00-05 | weather-pro-000002 |  68.90000000000006 |  63.09999999999999
2016-12-06 02:58:00-05 | weather-pro-000003 |  83.70000000000041 |  84.69999999999989
2016-12-06 02:58:00-05 | weather-pro-000004 |  83.10000000000039 |  84.00000000000051
2016-12-06 02:58:00-05 | weather-pro-000005 |  85.10000000000034 |  81.70000000000017
2016-12-06 02:58:00-05 | weather-pro-000006 |  61.09999999999999 | 49.800000000000026
2016-12-06 02:58:00-05 | weather-pro-000007 |   82.9000000000004 |  84.80000000000047
2016-12-06 02:58:00-05 | weather-pro-000008 | 58.599999999999966 |               40.2
2016-12-06 02:58:00-05 | weather-pro-000009 | 61.000000000000014 | 49.399999999999906
(10 rows)
```

**来自“outside”位置的最后 10 个读数**

```sql
SELECT time, c.device_id, location,
trunc(temperature, 2) temperature, trunc(humidity, 2) humidity
FROM conditions c
INNER JOIN locations l ON c.device_id = l.device_id
WHERE l.environment = 'outside'
ORDER BY time DESC LIMIT 10;

time                   |     device_id      |   location    | temperature | humidity
-----------------------+--------------------+---------------+-------------+----------
2016-12-06 02:58:00-05 | weather-pro-000000 | field-000000  |       84.10 |    83.70
2016-12-06 02:58:00-05 | weather-pro-000001 | arctic-000000 |       35.99 |    51.79
2016-12-06 02:58:00-05 | weather-pro-000003 | swamp-000000  |       83.70 |    84.69
2016-12-06 02:58:00-05 | weather-pro-000004 | field-000001  |       83.10 |    84.00
2016-12-06 02:58:00-05 | weather-pro-000005 | swamp-000001  |       85.10 |    81.70
2016-12-06 02:58:00-05 | weather-pro-000007 | field-000002  |       82.90 |    84.80
2016-12-06 02:58:00-05 | weather-pro-000014 | field-000003  |       84.50 |    83.90
2016-12-06 02:58:00-05 | weather-pro-000015 | swamp-000002  |       85.50 |    66.00
2016-12-06 02:58:00-05 | weather-pro-000017 | arctic-000001 |       35.29 |    50.59
2016-12-06 02:58:00-05 | weather-pro-000019 | arctic-000002 |       36.09 |    48.80
(10 rows)
```

**“field”位置的每小时平均值、最小值和最大温度**

```sql
SELECT date_trunc('hour', time) "hour",
trunc(avg(temperature), 2) avg_temp,
trunc(min(temperature), 2) min_temp,
trunc(max(temperature), 2) max_temp
FROM conditions c
WHERE c.device_id IN (
    SELECT device_id FROM locations
    WHERE location LIKE 'field-%'
) GROUP BY "hour" ORDER BY "hour" ASC LIMIT 24;

hour                   | avg_temp | min_temp | max_temp
-----------------------+----------+----------+----------
2016-11-15 07:00:00-05 |    73.80 |    68.00 |    79.09
2016-11-15 08:00:00-05 |    74.80 |    68.69 |    80.29
2016-11-15 09:00:00-05 |    75.75 |    69.39 |    81.19
2016-11-15 10:00:00-05 |    76.75 |    70.09 |    82.29
2016-11-15 11:00:00-05 |    77.77 |    70.79 |    83.39
2016-11-15 12:00:00-05 |    78.76 |    71.69 |    84.49
2016-11-15 13:00:00-05 |    79.73 |    72.69 |    85.29
2016-11-15 14:00:00-05 |    80.72 |    73.49 |    86.99
2016-11-15 15:00:00-05 |    81.73 |    74.29 |    88.39
2016-11-15 16:00:00-05 |    82.70 |    75.09 |    88.89
2016-11-15 17:00:00-05 |    83.70 |    76.19 |    89.99
2016-11-15 18:00:00-05 |    84.67 |    77.09 |    90.00
2016-11-15 19:00:00-05 |    85.64 |    78.19 |    90.00
2016-11-15 20:00:00-05 |    86.53 |    78.59 |    90.00
2016-11-15 21:00:00-05 |    86.40 |    78.49 |    90.00
2016-11-15 22:00:00-05 |    85.39 |    77.29 |    89.30
2016-11-15 23:00:00-05 |    84.40 |    76.19 |    88.70
2016-11-16 00:00:00-05 |    83.39 |    75.39 |    87.90
2016-11-16 01:00:00-05 |    82.40 |    74.39 |    87.10
2016-11-16 02:00:00-05 |    81.40 |    73.29 |    86.29
2016-11-16 03:00:00-05 |    80.38 |    71.89 |    85.40
2016-11-16 04:00:00-05 |    79.41 |    70.59 |    84.40
2016-11-16 05:00:00-05 |    78.39 |    69.49 |    83.60
2016-11-16 06:00:00-05 |    78.42 |    69.49 |    84.40
(24 rows)
```

[installation]: /getting-started/latest/


