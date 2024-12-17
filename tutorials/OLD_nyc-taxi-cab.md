---
标题: 物联网分析与监控
摘要: 使用 TimescaleDB 分析物联网数据。
关键词: [物联网，分析，监控]
---

# 物联网简介：纽约市出租车

用例：物联网分析与监控

在本教程中，你将学习：

1.  如何开始使用TimescaleDB
2.  如何使用TimescaleDB分析和监控物联网传感器数据

<!--- 此链接不再有效，已删除。LKB 2023-05-11

数据集：<Tag type="download">[nyc_data.tar.gz]()</Tag>

-->

预计完成时间：25分钟。

### 先决条件

要完成本教程，你需要对结构化查询语言（SQL）有基本了解。虽然教程会带你逐步了解每个SQL命令，但如果你之前接触过SQL将会有所帮助。

### 访问Timescale

有多种方式使用Timescale来跟随本教程。**所有连接信息和数据库命名**都假设你连接到**Timescale**，这是我们的托管、全托管数据库即服务。[注册免费30天试用账户][cloud-signup]，无需信用卡。一旦你确认账户并登录，进入下面的**背景**部分。

如果你想跟随本地或现场安装的教程，可以按照[安装TimescaleDB][install-timescale]的说明操作。安装完成后，你需要创建一个教程数据库并安装**Timescale**扩展。

使用命令行的`psql`，创建一个名为`nyc_data`的数据库并安装扩展：

```sql
CREATE DATABASE tsdb;
\c tsdb
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;
```

现在你可以开始本地跟随教程了！

### 背景

纽约市是超过830万人的家园。在本教程中，我们使用TimescaleDB分析和监控纽约黄色出租车的数据，以识别提高效率和减少温室气体排放的方法。我们执行的分析类似于许多问题领域中的数据科学组织用来规划升级、设定预算、分配资源等的分析。

在本教程中，你将完成三个任务：

*   **任务1：准备就绪 [5-15分钟]** 你将学习如何设置和连接到一个*TimescaleDB*实例，并使用*psql*从本地终端加载CSV文件中的数据。
*   **任务2：分析 [10分钟]** 你将学习如何使用TimescaleDB和*PostgreSQL*分析时间序列数据集。
*   **任务3：监控 [10分钟]** 你将学习如何使用TimescaleDB监控物联网设备。你还将学习如何结合使用TimescaleDB和其他PostgreSQL扩展，如*PostGIS*，查询地理空间数据。

### 任务1：准备就绪

对于本教程，我们使用来自[纽约市出租车和豪华轿车委员会][NYCTLC]（NYC TLC）的黄色出租车数据。NYC TLC是负责许可和监管纽约市黄色出租车和其他租赁车辆的机构。这些车辆以在五个行政区运送纽约人和游客而闻名。

NYC TLC拥有超过20万辆许可车辆，每天完成约100万次行程。他们已经公开了他们的出租车使用数据。而且，因为几乎所有这些数据都是时间序列数据，适当的分析需要一个专门构建的时间序列数据库。我们使用TimescaleDB的独特功能来完成本教程中的任务。

#### 下载和加载数据

让我们从下载数据集开始。为了（下载）时间和（你的机器上的）空间考虑，我们只获取2016年1月的数据，这个数据集包含约1100万条记录！

这个下载包含两个文件：

1.  `nyc_data.sql` - 一个设置必要表格的SQL文件
2.  `nyc_data_rides.csv` - 一个包含行程数据的CSV文件

你可以从下面的链接下载文件：

<!--- 此链接不再有效，已删除。LKB 2023-05-10

<Tag type="download">[nyc_data.tar.gz]()</Tag>

-->

#### 连接到TimescaleDB

要连接到数据库，你需要确保命令行上安装了`psql`实用程序。按照[设置psql命令行实用程序][setup-psql]的说明操作。

接下来，找到你的`host`、`port`和`password`。

之后，通过在终端输入以下命令连接到你的TimescaleDB实例，确保将{大括号}替换为你的真实密码、主机名和端口号。

```bash
psql -x "postgres://tsdbadmin:{YOUR_PASSWORD_HERE}@{YOUR_HOSTNAME_HERE}:{YOUR_PORT_HERE}/tsdb?sslmode=require"
```

你应该看到以下连接消息：

```bash
=require
psql (12.4)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

tsdb=>
```

要验证TimescaleDB是否已安装，运行`\dx`命令列出你的PostgreSQL数据库安装的所有扩展。你应该看到类似以下输出：

```sql
                  List of installed extensions
| Name        | Version | Schema     | Description                                  |
|-------------|---------|------------|----------------------------------------------|
| plpgsql     | 1.0     | pg_catalog | PL/pgSQL procedural language                 |
| timescaledb | 1.6.0   | public     | Enables scalable inserts and complex queries |
```

#### 定义你的数据模式

如上所述，NYC TLC从其车队的每辆车收集特定行程数据，每天生成数百万次行程的数据。

他们收集每次行程的以下数据：

*   接客日期和时间（作为时间戳）
*   接客地点（纬度和经度）
*   下车日期和时间（作为时间戳）
*   下车地点（纬度和经度）
*   行程距离（英里）
*   车费（美元）
*   乘客数量
*   费率类型（例如标准、机场等）
*   支付类型（现金、信用卡等）

为了有效存储这些数据，我们将需要三个表：

1.  一个名为`rides`的[超表][hypertables]，存储每次行程的所有上述数据。
2.  一个常规Postgres表`payment_types`，将支付类型映射到它们的英文描述。
3.  一个常规Postgres表`rates`，将数字费率代码映射到它们的英文描述。

`nyc_data.sql`脚本定义了我们三个表的模式。该脚本自动配置你的TimescaleDB实例，创建适当的`rides`、`payment_types`和`rates`表。

在下面的命令中，确保用你的TimescaleDB实例的信息替换大括号中的内容，就像你之前做的那样。同时注意，这个命令包括了为你自动创建的Timescale数据库。如果你在本地运行数据库，请根据需要替换数据库名称。

```bash
psql -x "postgres://tsdbadmin:{YOUR_PASSWORD_HERE}@{|YOUR_HOSTNAME_HERE}:{YOUR_PORT_HERE}/tsdb?sslmode=require" < nyc_data.sql
```

或者，你可以从`psql`命令行手动运行每个脚本。这个第一个脚本创建了一个名为`rides`的表，用于存储行程数据。注意，我们还在创建一些索引，以帮助本教程后续的查询：

```sql
CREATE TABLE "rides"(
    vendor_id TEXT,
    pickup_datetime TIMESTAMP WITHOUT TIME ZONE NOT NULL,
    dropoff_datetime TIMESTAMP WITHOUT TIME ZONE NOT NULL,
    passenger_count NUMERIC,
    trip_distance NUMERIC,
    pickup_longitude  NUMERIC,
    pickup_latitude   NUMERIC,
    rate_code         INTEGER,
    dropoff_longitude NUMERIC,
    dropoff_latitude  NUMERIC,
    payment_type INTEGER,
    fare_amount NUMERIC,
    extra NUMERIC,
    mta_tax NUMERIC,
    tip_amount NUMERIC,
    tolls_amount NUMERIC,
    improvement_surcharge NUMERIC,
    total_amount NUMERIC
);
SELECT create_hypertable('rides', 'pickup_datetime', 'payment_type', 2, create_default_indexes=>FALSE);
CREATE INDEX ON rides (vendor_id, pickup_datetime DESC);
CREATE INDEX ON rides (rate_code, pickup_datetime DESC);
CREATE INDEX ON rides (passenger_count, pickup_datetime DESC);
```

这个脚本创建了`payment_types`表，并预配置了出租车可以接受的支付类型：

```sql
CREATE TABLE IF NOT EXISTS "payment_types"(
    payment_type INTEGER,
    description TEXT
);
INSERT INTO payment_types(payment_type, description) VALUES
(1, 'credit card'),
(2, 'cash'),
(3, 'no charge'),
(4, 'dispute'),
(5, 'unknown'),
(6, 'voided trip');
```

这个脚本创建了`rates`表，并预配置了出租车可以收取的费率类型：

```sql
CREATE TABLE IF NOT EXISTS "rates"(
    rate_code   INTEGER,
    description TEXT
);
INSERT INTO rates(rate_code, description) VALUES
(1, 'standard rate'),
(2, 'JFK'),
(3, 'Newark'),
(4, 'Nassau or Westchester'),
(5, 'negotiated fare'),
(6, 'group ride');
```

你可以通过在`psql`命令行运行`\dt`命令来确认这些脚本的成功。你应该看到以下内容：

```sql
           List of relations
 Schema |     Name      | Type  |  Owner
--------+---------------+-------+----------
 public | payment_types | table | postgres
 public | rates         | table | postgres
 public | rides         | table | postgres
(3 rows)
```

#### 将行程数据加载到TimescaleDB

接下来，让我们将出租车数据上传到你的TimescaleDB实例中。数据在名为`nyc_data_rides.csv`的文件中，我们将其加载到`rides`超表中。为此，我们将使用以下`psql` `\copy`命令。

>:WARNING: PostgreSQL `\COPY`命令是单线程的，不支持将批量插入分批到多个事务中。有了近1100万行数据，这个导入可能需要10分钟或更长时间，具体取决于你的互联网连接。

```sql
\COPY rides FROM nyc_data_rides.csv CSV;
```

一个更快的替代方案是Timescale提供的[Parallel COPY命令][parallel-copy]，它用GoLang编写，可以多线程导入CSV文件，显著提高导入速度。设置`--workers` <= CPUs（或者如果CPU支持超线程，则为CPUs x 2）。**确保替换你的连接字符串、数据库名称和文件位置。**

```bash
timescaledb-parallel-copy --connection {CONNECTION STRING} --db-name {DATABASE NAME} --table rides --file {PATH TO `nyc_data_rides.csv`} --workers 4 --truncate --reporting-period 30s
```

使用这个Parallel Copy命令，你可以每30秒获得一次导入进度的更新。

导入完成后，你可以通过运行以下命令来验证你的设置：

```sql
SELECT * FROM rides LIMIT 5;
```

如果你看到类似于以下结果，恭喜你成功完成了任务1！

>:TIP: 你可以通过在`psql`命令行上使用`\x`标志来切换展开显示的开关。下面的输出是在展开显示开启时你将看到的。

```sql
-[ RECORD 1 ]---------+--------------------
vendor_id             | 1
pickup_datetime       | 2016-01-01 00:00:01
dropoff_datetime      | 2016-01-01 00:11:55
passenger_count       | 1
trip_distance         | 1.20
pickup_longitude      | -73.979423522949219
pickup_latitude       | 40.744613647460938
rate_code             | 1
dropoff_longitude     | -73.992034912109375
dropoff_latitude      | 40.753944396972656
payment_type          | 2
fare_amount           | 9
extra                 | 0.5
mta_tax               | 0.5
tip_amount            | 0
tolls_amount          | 0
improvement_surcharge | 0.3
total_amount          | 10.3
-[ RECORD 2 ]---------+--------------------
vendor_id             | 1
pickup_datetime       | 2016-01-01 00:00:02
dropoff_datetime      | 2016-01-01 00:11:14
passenger_count       | 1
trip_distance         | 6.00
pickup_longitude      | -73.947151184082031
pickup_latitude       | 40.791046142578125
rate_code             | 1
dropoff_longitude     | -73.920768737792969
dropoff_latitude      | 40.865577697753906
payment_type          | 2
fare_amount           | 18
extra                 | 0.5
mta_tax               | 0.5
tip_amount            | 0
tolls_amount          | 0
improvement_surcharge | 0.3
total_amount          | 19.3
```

### 任务2：分析

假设纽约市出租车和豪华轿车委员会设定了一个关键目标，即通过研究过去的出租车乘客历史和行为，计划未来，以在2024年之前减少20%的温室气体排放。鉴于每天乘坐出租车的人数，他们相信这将有助于实现目标。

在这个教程中，我们将历史出租车行程数据分析限制在2016年1月的所有NYC TLC出租车行程。你可以想象，在更广泛的场景中，你可能想要检查几年的行程数据。

#### 2016年1月每天发生了多少行程？

首先探索的问题很简单：*2016年1月每天发生了多少行程？*

由于TimescaleDB支持完整的SQL，所有需要的只是一条简单的SQL查询，以计算行程数量并按它们发生的日期进行分组/排序，如下所示：

```sql
-- 2016年1月前5天每天的总行程数是多少
SELECT date_trunc('day', pickup_datetime) as day, COUNT(*) FROM rides GROUP BY day ORDER BY day;
```

有了这些信息，我们知道每天发生了*多少*行程，我们可以确定一周和一个月中发生最多行程的日期。你的结果应该像这样：

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
 2016-01-08 00:00:00 | 392070
 2016-01-09 00:00:00 | 405825
 2016-01-10 00:00:00 | 351788
 2016-01-11 00:00:00 | 342651
 2016-01-12 00:00:00 | 367390
 2016-01-13 00:00:00 | 395090
 2016-01-14 00:00:00 | 396473
 2016-01-15 00:00:00 | 401289
 2016-01-16 00:00:00 | 411899
 2016-01-17 00:00:00 | 379156
 2016-01-18 00:00:00 | 341481
 2016-01-19 00:00:00 | 385187
 2016-01-20 00:00:00 | 382105
 2016-01-21 00:00:00 | 399654
 2016-01-22 00:00:00 | 420162
 2016-01-23 00:00:00 |  78133
 2016-01-24 00:00:00 | 159766
 2016-01-25 00:00:00 | 282087
 2016-01-26 00:00:00 | 327655
 2016-01-27 00:00:00 | 359180
 2016-01-28 00:00:00 | 383326
 2016-01-29 00:00:00 | 414039
 2016-01-30 00:00:00 | 435369
 2016-01-31 00:00:00 | 361505
 2017-11-17 00:00:00 |      2
(32 rows)
```

#### 乘客的平均车费是多少？

但这个初步分析是不完整的。我们还需要了解每次行程有多长时间。毕竟，如果我们想要减少对环境的影响，我们可能希望阻止出租车空驶和短途行程。

我们可以通过查看只有一名乘客的行程的日均车费来洞察行程的持续时间。再一次，这是一个简单的SQL查询，带一些条件语句，如下所示：

```sql
-- 对于前7天，只有一名乘客的行程的日均车费是多少？
SELECT date_trunc('day', pickup_datetime)
AS day, avg(fare_amount)
FROM rides
WHERE passenger_count = 1
AND pickup_datetime < '2016-01-08'
GROUP BY day ORDER BY day;
```

>:TIP: 像上面的查询在TimescaleDB上执行速度比在普通PostgreSQL数据库上快20倍，这得益于Timescale的自动时间和空间分区。

你的结果应该像这样：

```sql
         day         |         avg
---------------------+-------------------
 2016-01-01 00:00:00 | 12.5464748850129787
 2016-01-02 00:00:00 | 12.1129878886746750
 2016-01-03 00:00:00 | 12.8262352076841150
 2016-01-04 00:00:00 | 11.9116533573721472
 2016-01-05 00:00:00 | 11.7534235580737452
 2016-01-06 00:00:00 | 11.7824805635293235
 2016-01-07 00:00:00 | 11.9498961299166930
(7 rows)
```

#### 每种费率类型的行程发生了多少次？

当然，车费金额只给我们提供了一定程度的洞察。有些车费是预设的，无论行程长度如何。例如，往返机场的行程是从市内出发的固定费率。所以，让我们检查按行程类型划分的行程分解。这也是一个相当直接的SQL查询：

```sql
-- 这个月每种费率类型的行程发生了多少次？
SELECT rate_code, COUNT(vendor_id) AS num_trips
FROM rides
WHERE pickup_datetime < '2016-02-01'
GROUP BY rate_code
ORDER BY rate_code;
```

运行上述查询后，你将得到以下输出，显示每种费率代码的行程发生了多少次：

```sql
 rate_code | num_trips
-----------+-----------
         1 |  10626315
         2 |    225019
         3 |     16822
         4 |      4696
         5 |     33688
         6 |       102
        99 |       216
(7 rows)
```

虽然这是技术上正确的，但你可能希望使用更易于阅读的内容。为了做到这一点，我们可以使用SQL join的力量，并将这些结果与`rates`表的内容结合起来，如下所示：

```sql
-- 每种费率类型的行程发生了多少次？
-- 将rides与rates join以获取更多关于rate_code的信息
SELECT rates.description, COUNT(vendor_id) AS num_trips,
  RANK () OVER (ORDER BY COUNT(vendor_id) DESC) AS trip_rank FROM rides
  JOIN rates ON rides.rate_code = rates.rate_code
  WHERE pickup_datetime < '2016-02-01'
  GROUP BY rates.description
  ORDER BY LOWER(rates.description);
```

>:TIP: 这是一个简单的例子，说明了强大的观点：通过允许超表和常规PostgreSQL表之间的JOIN，TimescaleDB允许你将时间序列数据与关系或业务数据结合起来，以发现强大的洞察。

你的结果应该像这样，将`rates`表中的信息与你之前运行的查询结合起来：

```sql
      description      | num_trips | trip_rank
-----------------------+-----------+------------
 group ride            |       102 |          6
 JFK                   |    225019 |          2
 Nassau or Westchester |      4696 |          5
 negotiated fare       |     33688 |          3
 Newark                |     16822 |          4
 standard rate         |  10626315 |          1
(6 rows)
```

#### 肯尼迪和纽瓦克机场行程分析

从你计算费率类型的工作中，NYC TLC注意到，前往约翰·肯尼迪国际机场（JFK）和纽瓦克国际机场（EWR）的行程分别是第二和第四最受欢迎的行程类型。鉴于机场行程的受欢迎程度和随之而来的碳足迹，纽约市认为改善机场公共交通可能是一个改进领域——减少城市交通和与机场行程相关的整体碳足迹。

在实施任何计划之前，他们希望你更仔细地检查前往JFK（代码2）和纽瓦克（代码3）的行程。对于每个机场，他们希望了解以下信息，针对1月份：

*   前往该机场的行程数量
*   平均行程持续时间（即下车时间 - 接客时间）
*   平均行程费用
*   平均小费
*   最小、最大和平均行程距离
*   平均乘客数量

为了做到这一点，我们可以运行以下查询：

```sql
-- 对于每个机场：行程数量、平均行程持续时间、平均费用、平均小费、平均距离、最小距离、最大距离、平均乘客数量
SELECT rates.description, COUNT(vendor_id) AS num_trips,
   AVG(dropoff_datetime - pickup_datetime) AS avg_trip_duration, AVG(total_amount) AS avg_total,
   AVG(tip_amount) AS avg_tip, MIN(trip_distance) AS min_distance, AVG (trip_distance) AS avg_distance, MAX(trip_distance) AS max_distance,
   AVG(passenger_count) AS avg_passengers
 FROM rides
 JOIN rates ON rides.rate_code = rates.rate_code
 WHERE rides.rate_code IN (2,3) AND pickup_datetime < '2016-02-01'
 GROUP BY rates.description
 ORDER BY rates.description;
```

这产生了以下输出：

```sql
-[ RECORD 1 ]-----+-------------------
description       | JFK
num_trips         | 225019
avg_trip_duration | 00:45:46.822517
avg_total         | 64.3278115181384683
avg_tip           | 7.3334228220728027
min_distance      | 0.00
avg_distance      | 17.2602816651038357
max_distance      | 221.00
avg_passengers    | 1.7333869584346211
-[ RECORD 2 ]-----+-------------------
description       | Newark
num_trips         | 16822
avg_trip_duration | 00:35:16.157472
avg_total         | 86.4633688027582927
avg_tip           | 9.5461657353465700
min_distance      | 0.00
avg_distance      | 16.2706122934252764
max_distance      | 177.23
avg_passengers    | 1.7435501129473309
```

根据你的分析，你可以确定：

*   JFK的行程数量是纽瓦克的13倍。这通常导致前往和离开JFK的道路拥堵，尤其是在高峰时段。他们决定探索这些地区的道路交通改善，以及增加从机场出发的公共交通（例如公交车、地铁、火车等）。
*   每个机场的行程平均乘客数量相同（每次行程约1.7名乘客）。
*   行程距离大致相同，为16-17英里。
*   JFK的行程费用便宜约30%，这可能是因为新泽西隧道和高速公路收费。
*   纽瓦克的行程比JFK短22%（10分钟）。

这些数据不仅对城市规划者有用，对机场旅客和纽约市旅游局等旅游组织也有用。例如，旅游组织可以建议成本意识较强的旅客，如果他们不想支付84美元的纽瓦克行程费用，可以改乘公共交通，如从宾州车站出发的新泽西交通列车（成人票价15.25美元）。同样，他们可以建议前往JFK机场的旅客，如果他们担心交通拥堵，可以乘坐地铁和机场快线，只需7.50美元。

此外，你还可以为从纽约市出发的旅客提供关于选择哪个机场的建议。例如，根据上述数据，我们可以建议那些认为自己可能赶时间且不介意支付额外费用的旅客，考虑从纽瓦克出发而不是JFK。

如果你已经走到了这一步，那么你就成功完成了任务2，现在对如何使用TimescaleDB分析时间序列数据有了基本的了解！

### 任务3：监控

我们还可以利用出租车行程的时间序列数据来监控行程的当前状态。

>:WARNING: 更现实的设置将涉及创建一个数据管道，直接从汽车中流式传输传感器数据到TimescaleDB。然而，我们使用2016年1月的数据来说明无论设置如何都适用的基本原理。

#### 2016年第一天每5分钟发生了多少次行程？

2016年1月1日。纽约市的乘客庆祝了新年前夕，使用出租车前往和离开他们新年的第一次聚会。

你可能首先想知道最近发生了多少次行程。我们可以通过计算2016年第一天完成的行程数量，以5分钟为间隔来近似得出这个数字。

虽然计算发生多少次行程很容易，但在PostgreSQL中按5分钟时间间隔分割数据并没有那么容易。因此，我们需要使用类似于下面的查询：

```sql
-- 每5分钟的行程数量的普通Postgres查询
SELECT
  EXTRACT(hour from pickup_datetime) as hours,
  trunc(EXTRACT(minute from pickup_datetime) / 5)*5 AS five_mins,
  COUNT(*)
FROM rides
WHERE pickup_datetime < '2016-01-02 00:00'
GROUP BY hours, five_mins;
```

可能不是立即清楚为什么上述查询返回按5分钟桶分割的行程，让我们更仔细地检查它，使用示例时间为08:49:00。

在上述查询中，我们首先提取行程发生的时间：

```sql
EXTRACT(hour from pickup_datetime) as
```

所以对于08:49，我们`hours`的结果将是8。然后我们需要计算`five_mins`，给定时间戳最接近的5分钟倍数。为此，我们计算行程开始的分钟数除以5的商。然后我们截断结果，取该商的底数。之后，我们将截断的商乘以5，本质上找到最接近的5分钟桶，该分钟最接近：

```sql
trunc(EXTRACT(minute from pickup_datetime) / 5)*5 AS five_mins
```

所以对于08:49的时间，我们的结果将是`trunc(49/5)*5 = trunc(9.8)*5 = 9*5 = 45`，所以这个时间将在45&nbsp;min桶中。在提取小时和时间落入哪个5分钟间隔之后，我们按`hours`和`five_mins`间隔对结果进行分组。哇，对于一个概念上简单的问题来说，这是很多步骤！

在时间序列分析中，按任意时间间隔分割是常见的，但在普通PostgreSQL中有时可能不太方便。幸运的是，TimescaleDB有许多定制的SQL函数，使时间序列分析变得快速简单。例如，`time_bucket`是PostgreSQL `date_trunc`函数的更强大的版本。它允许使用任意时间间隔，而不是`date_trunc`提供的标准天、分钟、小时。

所以当使用TimescaleDB时，上述复杂的查询变成了一个更简单的SQL查询，如下所示：

```sql
-- 2016年第一天每5分钟发生了多少次行程？
-- 使用TimescaleDB的“time_bucket”函数
SELECT time_bucket('5 minute', pickup_datetime) AS five_min, count(*)
FROM rides
WHERE pickup_datetime < '2016-01-02 00:00'
GROUP BY five_min
ORDER BY five_min;
```

你的查询结果应该开始像这样：

```sql
      five_min       | count
---------------------+-------
 2016-01-01 00:00:00 |   703
 2016-01-01 00:05:00 |  1482
 2016-01-01 00:10:00 |  1959
 2016-01-01 00:15:00 |  2200
 2016-01-01 00:20:00 |  2285

```

#### 元旦早晨从时代广场400米范围内出发的行程有多少，在30分钟的桶中？

纽约市以其在时代广场举行的年度新年落球庆典而闻名。数千人聚集在一起迎接新年的到来，然后回家，去他们最喜欢的酒吧，或参加新年的第一次聚会。

这对分析很重要，因为你想在高峰时段了解出租车需求，而在纽约，没有比新年第一天的时代广场地区更高峰的时段了。

要回答这个问题，你可能会首先想到使用我们在上一节中的`time_bucket`来计算30分钟间隔内发起的行程。但我们没有的信息是：我们如何确定哪些行程开始于*时代广场附近*？

这需要我们利用`rides`超表中的接客纬度和经度列。要使用接客地点，我们需要为我们的超表准备好进行地理空间查询。

好消息是，TimescaleDB与所有其他PostgreSQL扩展兼容，对于地理空间数据，我们将使用[PostGIS][postgis]。这使我们能够以TimescaleDB的速度和规模按时间和地点切片数据！

```sql
-- 地理空间查询 - TimescaleDB + POSTGIS - 按时间和地点切片
-- 在数据库中安装扩展
CREATE EXTENSION postgis;
```

然后，在`psql`中运行`\dx`命令以验证PostGIS是否已正确安装。你应该在扩展列表中看到PostGIS扩展，如下所示：

```sql
                                        List of installed extensions
     Name     | Version |   Schema   |                             Description
 -------------+---------+------------+---------------------------------------------------------------------
  plpgsql     | 1.0     | pg_catalog | PL/pgSQL procedural language
  postgis     | 2.5.1   | public     | PostGIS geometry, geography, and raster spatial types and functions
  timescaledb | 1.6.0   | public     | Enables scalable inserts and complex queries for time-series data
 (3 rows)
 ```

现在，我们需要修改我们的表以使用PostGIS。首先，我们将为行程接客和落客位置添加几何列：

```sql
-- 为我们的（lat,long）点创建几何列
ALTER TABLE rides ADD COLUMN pickup_geom geometry(POINT,2163);
ALTER TABLE rides ADD COLUMN dropoff_geom geometry(POINT,2163);
```

接下来，我们需要将纬度和经度点转换为几何坐标，以便与PostGIS良好协作：

>:WARNING: 下一个查询可能需要几分钟。在一次UPDATE语句中更新两个列，如下所示，减少了更新`rides`表中所有行所需的时间。

```sql
-- 生成几何点并写入表
UPDATE rides SET pickup_geom = ST_Transform(ST_SetSRID(ST_MakePoint(pickup_longitude,pickup_latitude),4326),2163),
   dropoff_geom = ST_Transform(ST_SetSRID(ST_MakePoint(dropoff_longitude,dropoff_latitude),4326),2163);
```

最后，我们需要一个额外的信息：时代广场位于（lat，long）（40.7589，-73.9851）。

现在，我们有了回答我们原始问题的所有信息：*元旦早晨在时代广场400米范围内发起的行程有多少，在30分钟的桶中？*

```sql
-- 元旦当天在时代广场400米范围内发起的出租车行程数量，按30分钟桶分组。
-- 请注意：时代广场位于（lat，long）（40.7589，-73.9851）
SELECT time_bucket('30 minutes', pickup_datetime) AS thirty_min, COUNT(*) AS near_times_sq
FROM rides
WHERE ST_Distance(pickup_geom, ST_Transform(ST_SetSRID(ST_MakePoint(-73.9851,40.7589),4326),2163)) < 400
AND pickup_datetime < '2016-01-01 14:00'
GROUP BY thirty_min ORDER BY thirty_min;
```

你应该得到以下结果：

```sql
     thirty_min      | near_times_sq
---------------------+---------------
 2016-01-01 00:00:00 |            74
 2016-01-01 00:30:00 |           102
 2016-01-01 01:00:00 |           120
 2016-01-01 01:30:00 |            98
 2016-01-01 02:00:00 |           112
 2016-01-01 02:30:00 |           109
 2016-01-01 03:00:00 |           163
 2016-01-01 03:30:00 |           181
 2016-01-01 04:00:00 |           214
 2016-01-01 04:30:00 |           185
 2016-01-01 05:00:00 |           158
 2016-01-01 05:30:00 |           113
 2016-01-01 06:00:00 |           102
 2016-01-01 06:30:00 |            91
 2016-01-01 07:00:00 |            88
 2016-01-01 07:30:00 |            58
 2016-01-01 08:00:00 |            72
 2016-01-01 08:30:00 |            94
 2016-01-01 09:00:00 |           115
 2016-01-01 09:30:00 |           118
 2016-01-01 10:00:00 |           135
 2016-01-01 10:30:00 |           160
 2016-01-01 11:00:00 |           212
 2016-01-01 11:30:00 |           229
 2016-01-01 12:00:00 |           244
 2016-01-01 12:30:00 |           230
 2016-01-01 13:00:00 |           235
 2016-01-01 13:30:00 |           238
(28 rows)
```

从上面的图表中，你可以推断出，午夜时分很少有人想乘出租车离开，而在凌晨03:00至05:00之间，许多人乘出租车离开，这时酒吧、俱乐部和其他新年派对已经结束。这对于容量规划、减少空驶车辆和预先布置替代交通工具（如地铁和火车线路的班车）非常有用，这些交通工具的碳足迹较小。

顺便说一句，数据还显示，行程在上午晚些时候开始增加，因为人们前往早餐和其他新年活动。纽约确实是一个不夜城，时代广场是一个很好的反映！

### 结论和后续步骤

在本教程中，你学习了如何开始使用TimescaleDB。

在**任务1**中，你学习了如何设置和连接到TimescaleDB实例，并使用`psql`从CSV文件加载数据。

在**任务2和3**中，你学习了如何使用TimescaleDB进行分析和监控大型数据集。你了解了超表，看到了TimescaleDB如何支持完整的SQL，以及JOIN如何使你能够将时间序列数据与关系或业务数据结合起来。

你还学习了TimescaleDB的特殊SQL函数，如`time_bucket`，以及它们如何使时间序列分析在更少的代码行中成为可能，以及TimescaleDB如何与*PostGIS*等其他扩展兼容，快速按时间和地点查询。

[NYCTLC]: https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page 
[cloud-signup]: https://console.cloud.timescale.com/signup 
[continuous-aggregates]: /getting-started/:currentVersion:/create-cagg/
[hypertables]: /use-timescale/:currentVersion:/hypertables
[install-timescale]: /getting-started/latest/
[migrate]: /use-timescale/:currentVersion:/migration/
[parallel-copy]: https://github.com/timescale/timescaledb-parallel-copy 
[postgis]: http://postgis.net/documentation 
[setup-psql]: /use-timescale/:currentVersion:/integrations/query-admin/about-psql
[time-series-forecasting]: /tutorials/:currentVersion:/time-series-forecast/
