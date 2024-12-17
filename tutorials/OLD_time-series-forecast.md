---
标题: 时间序列预测
摘要: 基于历史数据预测数据集未来可能的值。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [预测，分析]
---

# 时间序列预测

时间序列预测使我们能够基于历史时间序列数据预测数据集可能的未来值。时间序列数据共同表示了系统、过程或行为随时间的变化。当您在一段时间内累积了数百万数据点时，您可以构建模型来预测接下来可能发生的一组值。

时间序列预测可用于：

*   预测下个季度的云基础设施支出
*   预测将来给定股票的价值
*   预测下个季度可能销售的产品单位数
*   预测物联网设备的剩余寿命
*   预测大型假日晚上所需的出租车或拼车司机数量

仅时间序列预测本身就是一个强大的工具。但时间序列数据与商业数据结合可以为任何开发人员带来竞争优势。TimescaleDB 是 PostgreSQL 的时间序列数据库，因此存储在 TimescaleDB 中的时间序列数据可以轻松与另一个关系数据库中的商业数据结合，以便开发更具洞察力的预测，了解您的数据（和业务）随时间的变化。

这个时间序列预测示例展示了如何将 TimescaleDB 与 R、Apache MADlib 和 Python 集成以执行各种时间序列预测方法。它使用了 Hello Timescale 教程中也使用过的纽约市出租车数据。数据集包含了 2016 年 1 月纽约市所有黄色出租车行程的信息，包括接送和下车时间、GPS 坐标和行程的总价格。您可以从这个丰富的数据集中提取一些有趣的洞察，构建时间序列预测模型，并探索各种预测和机器学习工具的使用。

### 设置

先决条件：

*   [安装 TimescaleDB][install]
*   从 Hello Timescale 教程下载并加载数据集
*   在数据库中安装并设置 PostGIS
*   [安装 R][install_r]
*   [安装 Python][install_python]

首先，让我们创建模式并填充表。下载文件 [`forecast.sql`][forecast-sql] 并执行以下命令：

```bash
psql -U postgres -d tsdb -h localhost -f forecast.sql
```

`forecast.sql` 文件包含创建三个 TimescaleDB 超表 `rides_count`、`rides_length` 和 `rides_price` 的 SQL 语句。让我们以如何创建 `rides_count` 表为例。以下是从 `forecast.sql` 中取出的代码片段：

```sql
CREATE TABLE rides_count(
  one_hour TIMESTAMP WITHOUT TIME ZONE NOT NULL,
  count NUMERIC
);
SELECT create_hypertable('rides_count', 'one_hour');

INSERT INTO rides_count
  SELECT time_bucket_gapfill('1 hour', pickup_datetime, '2016-01-01 00:00:00','2016-01-31 23:59:59') AS one_hour,
    COUNT(*) AS count
  FROM rides
  WHERE ST_Distance(pickup_geom, ST_Transform(ST_SetSRID(ST_MakePoint(-74.0113,40.7075),4326),2163)) < 400
    AND pickup_datetime < '2016-02-01'
  GROUP BY one_hour
  ORDER BY one_hour;
```

请注意，您已将 `rides_count` 表设置为 TimescaleDB 超表。这使您能够利用 TimescaleDB 在时间序列数据上更快的插入和查询性能。在这里，您可以看到 PostgreSQL 聚合函数（如 `COUNT`）和各种 PostGIS 函数都像往常一样与 TimescaleDB 一起工作。您可以使用 PostGIS 从原始 `rides` 表中选择接送地点距离 GPS 位置（40.7589, -73.9851）不到 400 米的数据中心点。

数据来自 [NYC Taxi and Limousine Commission][NYCTLC]。对于某些小时缺少数据点。您可以用 0 填补缺失值。要了解更多信息，请参阅 [填补空白][gap_filling] 文档。创建 `rides_length` 和 `rides_price` 也采用了类似的方法。

在您进入接下来的几个部分之前，请检查以下表格是否在您的数据库中。

```sql
\dt
             List of relations
 Schema |      Name       | Type  |  Owner
--------+-----------------+-------+----------
 public | payment_types   | table | postgres
 public | rates           | table | postgres
 public | rides           | table | postgres
 public | rides_count     | table | postgres
 public | rides_length    | table | postgres
 public | rides_price     | table | postgres
 public | spatial_ref_sys | table | postgres

(7 rows)
```

### 使用 R 进行季节性 ARIMA

[ARIMA（自回归积分滑动平均）模型][arima] 是时间序列分析中常用的工具，用于更好地了解数据集并预测未来值。ARIMA 模型可以大致分为季节性和非季节性。季节性 ARIMA 模型用于具有固定周期内重复特征的数据集。例如，一周内每小时温度值的数据集具有周期为 1 天的季节性成分，因为温度每天都会在白天上升，晚上下降。相比之下，比特币的价格随时间变化（可能）是非季节性的，因为没有清晰的可观察模式在固定时间段内重复。

本教程使用 R 来分析一周内时代广场出租车接送数量的季节性。

表 `rides_count` 包含了本教程部分所需的数据。`rides_count` 有两个列 `one_hour` 和 `count`。`one_hour` 列是 1 月 1 日至 1 月 31 日每小时的 TimescaleDB `time_bucket`。`count` 列是每个小时期间时代广场的接送数量。

```sql
SELECT * FROM rides_count;
      one_hour       | count
---------------------+-------
 2016-01-01 00:00:00 |   176
 2016-01-01 01:00:00 |   218
 2016-01-01 02:00:00 |   221
 2016-01-01 03:00:00 |   344
 2016-01-01 04:00:00 |   397
 2016-01-01 05:00:00 |   269
 2016-01-01 06:00:00 |   192
 2016-01-01 07:00:00 |   145
 2016-01-01 08:00:00 |   166
 2016-01-01 09:00:00 |   233
 2016-01-01 10:00:00 |   295
 2016-01-01 11:00:00 |   440
 2016-01-01 12:00:00 |   472
 2016-01-01 13:00:00 |   472
 2016-01-01 14:00:00 |   485
 2016-01-01 15:00:00 |   538
 2016-01-01 16:00:00 |   430
 2016-01-01 17:00:00 |   451
 2016-01-01 18:00:00 |   496
 2016-01-01 19:00:00 |   538
 2016-01-01 20:00:00 |   485
 2016-01-01 21:00:00 |   619
 2016-01-01 22:00:00 |  1197
 2016-01-01 23:00:00 |   798
 ...
```

创建两个 PostgreSQL 视图 `rides_count_train` 和 `rides_count_test` 用于训练和测试数据集。

```sql
-- 制作训练数据集
CREATE VIEW rides_count_train AS
SELECT * FROM rides_count
WHERE one_hour <= '2016-01-21 23:59:59';

-- 制作测试数据集
CREATE VIEW rides_count_test AS
SELECT * FROM rides_count
WHERE one_hour >= '2016-01-22 00:00:00';
```

R 有一个 [RPostgres][rpostgres] 包，允许您从 R 连接到数据库。下面的代码建立了与 PostgreSQL 数据库 `nyc_data` 的连接。您可以通过更改 `dbConnect` 的参数连接到不同的数据库。代码的最后一行应该打印出您数据库中的所有表的列表。这意味着您已成功连接，准备从 R 查询数据库。

```r
# 安装并加载 RPostgres 包
install.packages("RPostgres")
library("DBI")

# 创建与 postgres 数据库的连接
con <- dbConnect(RPostgres::Postgres(), dbname = "nyc_data",
      host = "localhost",
      user = "postgres")

# 列出数据库中的表以验证连接
dbListTables(con)
```

您可以在 R 中使用 SQL 代码查询数据库。将查询结果放入 R 数据框中，允许您使用 R 提供的工具分析数据。

```r
# 查询数据库并将结果输入到 R 数据框中
# 2016/01/01 - 2016/01/21 的训练数据集
count_rides_train_query <- dbSendQuery(con, "SELECT * FROM rides_count_train;")
count_rides_train <- dbFetch(count_rides_train_query)
dbClearResult(count_rides_train_query)
head(count_rides_train)
             one_hour count
1 2016-01-01 00:00:00   176
2 2016-01-01 01:00:00   218
3 2016-01-01 02:00:00   221
4 2016-01-01 03:00:00   344
5 2016-01-01 04:00:00   397
6 2016-01-01 05:00:00   269

# 2016/01/22 - 2016/01/31 的测试数据集
count_rides_test_query <- dbSendQuery(con, "SELECT * FROM rides_count_test")
count_rides_test <- dbFetch(count_rides_test_query)
dbClearResult(count_rides_test_query)
head(count_rides_test)
             one_hour count
1 2016-01-22 00:00:00   702
2 2016-01-22 01:00:00   401
3 2016-01-22 02:00:00   247
4 2016-01-22 03:00:00   169
5 2016-01-22 04:00:00   140
6 2016-01-22 05:00:00   100
```

为了将数据输入到 ARIMA 模型中，您必须先将数据框转换为 R 中的时间序列对象。[`xts`][r-xts] 是一个包，允许您轻松地进行此操作。您还可以将时间序列对象的频率设置为 168。这是因为预计接送数量将以每周固定的模式波动，一周有 168 小时，换句话说，每个季节周期有 168 个数据点。如果您想将数据建模为每天 1 天的季节性，可以将频率参数更改为 24。

```r
# 安装并加载 xts 包
install.packages("xts")
library("xts")

# 将数据框转换为时间序列
xts_count_rides <- xts(count_rides_train$count, order.by = as.POSIXct(count_rides_train$one_hour, format = "%Y-%m-%d %H:%M:%S"))

# 将系列的频率设置为每周 24 * 7
attr(xts_count_rides, 'frequency') <- 168
```

R 中的 [`forecast`][r-forecast] 包提供了一个有用的函数 `auto.arima`，它自动为数据集找到最佳的 ARIMA 参数。将参数 D 设置为 1，捕捉模型的季节性，以强制函数找到季节性模型。这个计算可能需要一段时间（在这个数据集中，大约需要五分钟）。一旦计算完成，您可以将 `auto.arima` 函数的输出保存到 `fit` 中，并获取已创建的 ARIMA 模型的摘要。

```r
# 安装并加载预测包所需的 ARIMA
install.packages("forecast")
library("forecast")

# 使用 auto.arima 自动获取最佳拟合的 arima 模型参数
fit <- auto.arima(xts_count_rides[,1], D = 1, seasonal = TRUE)

# 查看拟合的摘要
summary(fit)
Series: xts_count_rides[, 1]
ARIMA(4,0,2)(0,1,0)[168] with drift

Coefficients:
         ar1      ar2     ar3     ar4      ma1     ma2   drift
      2.3211  -1.8758  0.3959  0.1001  -1.7643  0.9444  0.3561
s.e.  0.0634   0.1487  0.1460  0.0588   0.0361  0.0307  0.0705

sigma^2 estimated as 5193:  log likelihood=-1911.21
AIC=3838.42   AICc=3838.86   BIC=3868.95

Training set error measures:
                     ME     RMSE      MAE       MPE     MAPE      MASE
Training set -0.2800571 58.22306 33.15943 -1.783649 7.868031 0.4257707
                    ACF1
Training set -0.02641353
```

最后，ARIMA 模型可以用来预测未来的值。`h` 参数指定了预测的步数。

```r
# 使用 arima 模型预测未来值，h 指定了要预测的读数数量
fcast <- forecast(fit, h=168)
fcast
         Point Forecast      Lo 80     Hi 80         Lo 95     Hi 95
4.000000       659.0645  566.71202  751.4169  517.82358229  800.3053
4.005952       430.7339  325.02891  536.4388  269.07209741  592.3956
4.011905       268.1259  157.28358  378.9682   98.60719504  437.6446
4.017857       228.3024  116.08381  340.5210   56.67886523  399.9260
4.023810       200.7340   88.25064  313.2174   28.70554423  372.7625
4.029762       140.5758   28.04128  253.1103  -31.53088134  312.6824
4.035714       196.1703   83.57555  308.7650   23.97150358  368.3690
4.041667       282.6171  169.80545  395.4288  110.08657346  455.1476
4.047619       446.6713  333.28115  560.0614  273.25604289  620.0865
4.053571       479.9449  365.53618  594.3537  304.97184340  654.9180
...
```

`forecast` 的输出可能难以解读。您可以使用以下代码绘制预测值：

```r
# 绘制预测值
plot(fcast, include = 168, main="时代广场出租车接送数量随时间变化", xlab="日期", ylab="接送数量", xaxt="n", col="red", fcol="blue")
ticks <- seq(3, 5, 1/7)
dates <- seq(as.Date("2016-01-15"), as.Date("2016-01-29"), by="days")
dates <- format(dates, "%m-%d %H:%M")
axis(1, at=ticks, labels=dates)
legend('topleft', legend=c("Observed Value", "Predicted Value"), col=c("red", "blue"), lwd=c(2.5,2.5))

# 绘制测试数据集中观察到的值
count_rides_test$x <- seq(4, 4 + 239 * 1/168, 1/168)
count_rides_test <- subset(count_rides_test, count_rides_test$one_hour < as.POSIXct("2016-01-29"))
lines(count_rides_test$x, count_rides_test$count, col="red")
```

<img class="main-content__illustration" src="http://assets.iobeam.com/images/docs/rides_count.png"  alt="Rides Count Graph" />

在这些数据的图形化中，围绕预测线（蓝色）的灰色区域是预测区间，或预测的不确定性，而红线是实际观察到的接送数量。1 月 23 日星期六的接送数量为零，因为这段时间缺少数据。

您可能会发现 1 月 22 日的预测与观测值惊人地匹配，但随后几天的预测高估了。显然，模型已经捕捉到了数据的季节性，因为您可以看到预测的接送数量在凌晨 1 点急剧下降，然后从大约早上 6 点再次上升。与早晨相比，下午的接送数量显著增加，午餐时间左右有轻微的下降，下午 6 点左右有一个明显的高峰，人们可能在下班后乘坐出租车回家。

虽然这些发现没有揭示任何完全出乎意料的事情，但分析结果验证了您的预期仍然是有价值的。必须指出的是，ARIMA 模型并不完美，这从 1 月 25 日的异常预测中可以看出。创建的 ARIMA 模型使用前一周的数据进行预测。2016 年 1 月 18 日是马丁·路德·金日，因此当天的接送分布与标准星期一略有不同。此外，假期可能也影响了周围几天的乘客行为。模型没有捕捉到由各种假期引起的这种异常数据，这在得出结论之前必须注意。简单地删除这种异常数据，例如只使用 1 月份的前两周，可能会导致更准确的预测。这表明了解数据背后的上下文的重要性。

尽管 R 提供了丰富的统计模型库，但在执行计算之前需要将数据导入 R。对于更大的数据集，这可能成为将所有数据传输到 R 进程的瓶颈（R 进程本身可能会耗尽内存并开始交换）。因此，让我们看看一种将计算转移到数据库并提高性能的替代方法。

### 使用 Apache MADlib 的非季节性 ARIMA

[MADlib][madlib] 是一个开源的数据库内数据分析库，提供了广泛的流行机器学习方法和各种补充统计工具。

MADlib 支持许多在 R 和 Python 中可用的机器学习算法。通过在数据库内执行这些机器学习算法，可能足以高效地处理整个数据集，而不是将较小的样本拉到外部程序中。

按照他们的文档中概述的步骤安装 MADlib：
[MADlib 安装指南][madlib_install]。

在 `nyc_data` 数据库中设置 MADlib：

```bash
/usr/local/madlib/bin/madpack -s madlib -p postgres -c postgres@localhost/nyc_data install
```

<Highlight type="warning">
此命令可能因您安装 MADlib 的目录以及您的 PostgreSQL 用户、主机和数据库的名称而有所不同。
</Highlight>

现在您可以使用 MADlib 的库来分析出租车数据集。在这里，您可以训练一个 ARIMA 模型来预测在给定时间从肯尼迪机场到时代广场的行程价格。

让我们看看 `rides_price` 表。`trip_price` 列是在每个小时期间从肯尼迪机场到时代广场的行程的平均价格。由于在某个小时期间没有行程而被遗漏的数据点用前一个值填充。这是通过本教程前面提到的 [gap filling][gap_filling] 完成的。

```sql
SELECT * FROM rides_price;
      one_hour       |    trip_price
---------------------+------------------
 2016-01-01 00:00:00 |            58.34
 2016-01-01 01:00:00 |            58.34
 2016-01-01 02:00:00 |            58.34
 2016-01-01 03:00:00 |            58.34
 2016-01-01 04:00:00 |            58.34
 2016-01-01 05:00:00 |            59.59
 2016-01-01 06:00:00 |            58.34
 2016-01-01 07:00:00 | 60.3833333333333
 2016-01-01 08:00:00 |          61.2575
 2016-01-01 09:00:00 |           58.435
 2016-01-01 10:00:00 |           63.952
 2016-01-01 11:00:00 | 59.9576923076923
 2016-01-01 12:00:00 |           60.462
 2016-01-01 13:00:00 |            61.65
 2016-01-01 14:00:00 |           58.342
 2016-01-01 15:00:00 |          59.8965
 2016-01-01 16:00:00 | 61.6468965517241
 2016-01-01 17:00:00 |           58.982
 2016-01-01 18:00:00 |         64.28875
 2016-01-01 19:00:00 | 60.8433333333333
 2016-01-01 20:00:00 |        61.888125
 2016-01-01 21:00:00 | 61.4064285714286
 2016-01-01 22:00:00 |  61.107619047619
 2016-01-01 23:00:00 | 57.9088888888889
```

您也可以为训练和测试数据集创建两个表。您可以在这里创建表而不是视图，因为您需要在稍后的时间序列预测分析中向这些数据集添加列。

```sql
-- 制作训练数据集
SELECT * INTO rides_price_train FROM rides_price
WHERE one_hour <= '2016-01-21 23:59:59';

-- 制作测试数据集
SELECT * INTO rides_price_test FROM rides_price
WHERE one_hour >= '2016-01-22 00:00:00';
```

现在您可以使用 [MADlib 的 ARIMA][madlib_arima] 库来对您的数据集进行预测。

MADlib 尚未提供自动寻找 ARIMA 模型最佳参数的方法。因此，ARIMA 模型的非季节性阶数是使用 R 的 `auto.arima` 函数获得的，就像您在前一节中使用季节性 ARIMA 获得的那样。以下是 R 代码：

```r
# 连接数据库并获取记录
library("DBI")
con <- dbConnect(RPostgres::Postgres(), dbname = "nyc_data",
      host = "localhost",
      user = "postgres")
rides_price_train_query <- dbSendQuery(con, "SELECT * FROM rides_price_train;")
rides_price_train <- dbFetch(rides_price_train_query)
dbClearResult(rides_price_train_query_query)

# 将数据框转换为时间序列
library("xts")
xts_rides_price <- xts(rides_price_train$trip_price, order.by = as.POSIXct(rides_price_train$one_hour, format = "%Y-%m-%d %H:%M:%S"))
attr(xts_rides_price, 'frequency') <- 168

# 使用 auto.arima() 计算阶数
library("forecast")
fit <- auto.arima(xts_rides_price[,1])

# 查看拟合的摘要
summary(fit)
Series: xts_rides_price[, 1]
ARIMA(2,1,3)

Coefficients:
         ar1      ar2      ma1     ma2      ma3
      0.3958  -0.5142  -1.1906  0.8263  -0.5791
s.e.  0.2312   0.1593   0.2202  0.2846   0.1130

sigma^2 estimated as 11.06:  log likelihood=-1316.8
AIC=2645.59   AICc=2645.76   BIC=2670.92

Training set error measures:
                    ME    RMSE      MAE         MPE    MAPE      MASE
Training set 0.1319955 3.30592 2.186295 -0.04371788 3.47929 0.6510487
                     ACF1
Training set -0.002262549
```

当然，您可以继续按照上一节季节性 ARIMA 的步骤使用 R 进行分析。不幸的是，MADlib 尚未提供自动寻找 ARIMA 模型阶数的方法。

然而，对于更大的数据集，您可以采取加载数据子集到 R 中以计算模型参数的方法，然后使用 MADlib 训练模型。您可以使用本教程中概述的选项组合，利用不同工具的优势并弥补弱点。

使用 R 找到的 ARIMA(2,1,3) 参数，您可以使用 MADlib 的 `arima_train` 和 `arima_forecast` 函数。

```sql
-- 训练 arima 模型并预测从肯尼迪机场到时代广场的行程价格
DROP TABLE IF EXISTS rides_price_output;
DROP TABLE IF EXISTS rides_price_output_residual;
DROP TABLE IF EXISTS rides_price_output_summary;
DROP TABLE IF EXISTS rides_price_forecast_output;

SELECT madlib.arima_train('rides_price_train', -- 输入表
      'rides_price_output', -- 输出表
      'one_hour', -- 时间戳列
      'trip_price', -- 时间序列列
      NULL, -- 分组列
      TRUE, -- 包含均值
      ARRAY[2,1,3] -- 非季节性阶数
      );

SELECT madlib.arima_forecast('rides_price_output', -- 模型表
                        'rides_price_forecast_output', -- 输出表
                        240 -- 步数（10 天）
                        );
```

让我们检查训练有素的 ARIMA 模型为第二天预测了哪些值。

```sql
SELECT * FROM rides_price_forecast_output;
 steps_ahead | forecast_value
-------------+----------------
           1 |  62.3175746635
           2 |  62.7126520845
           3 |  62.8920386424
           4 |  62.7550446339
           5 |   62.606406819
           6 |  62.6197088842
           7 |  62.7032173055
           8 |  62.7292577943
           9 |  62.6956015822
          10 |  62.6685763075
...
```

模型似乎表明，从肯尼迪机场到时代广场的行程价格在一天之内保持相对恒定。MADlib 还提供了各种统计函数来评估模型。

```sql
ALTER TABLE rides_price_test ADD COLUMN id SERIAL PRIMARY KEY;
ALTER TABLE rides_price_test ADD COLUMN forecast DOUBLE PRECISION;

UPDATE rides_price_test
SET forecast = rides_price_forecast_output.forecast_value
FROM rides_price_forecast_output
WHERE rides_price_test.id = rides_price_forecast_output.steps_ahead;

SELECT madlib.mean_abs_perc_error('rides_price_test', 'rides_price_mean_abs_perc_error', 'trip_price', 'forecast');

SELECT * FROM rides_price_mean_abs_perc_error;
 mean_abs_perc_error
---------------------
  0.0423789161532639
(1 row)
```

之前，您必须设置 `rides_price_test` 表的列以适应 MADlib 的 `mean_abs_perc_error` 函数的格式。有多种方法可以评估模型预测值的质量。在这种情况下，您计算了平均绝对百分比误差，得到了 4.24%。

您从中学到了什么？非季节性 ARIMA 模型预测从机场到曼哈顿的行程价格保持恒定在 62 美元，并且在测试数据集上表现良好。与一些在高峰时段有高峰定价的拼车应用（如 Uber）不同，黄色出租车的价格几乎整天保持恒定。

从技术角度来看，您已经看到了 TimescaleDB 如何与 PostGIS 和 MADlib 等其他 PostgreSQL 扩展无缝集成。这意味着 TimescaleDB 用户可以轻松利用 PostgreSQL 生态系统的强大功能。

### 使用 Python 的 Holt-Winters

[Holt-Winters][holt-winters] 模型是时间序列分析和预测中另一个广泛使用的工具。它只能用于季节性时间序列数据。Holt-Winters 模型使用简单指数平滑来做出未来预测。因此，对于时间序列数据，预测是通过取过去值的加权平均值来计算的，更近的数据点比之前的点具有更大的权重。Holt-Winters 被认为是比 ARIMA 更简单的模型，但在时间序列预测中哪个模型更优越没有明确的答案。建议为特定数据集创建两种模型并比较性能，以找出哪个更适合。

您可以使用 Python 分析一天中不同时间段从金融区到时代广场所需的时间。

您需要安装这些 Python 包：

```bash
pip install psycopg2
pip install pandas
pip install numpy
pip install statsmodels
```

数据的格式与前两节非常相似。`rides_length` 表中的 `trip_length` 列是在给定时间段内从金融区到时代广场的平均行程长度。

```sql
SELECT * FROM rides_length;
     three_hour      |   trip_length
---------------------+-----------------
 2016-01-01 00:00:00 | 00:21:50.090909
 2016-01-01 03:00:00 | 00:17:15.8
 2016-01-01 06:00:00 | 00:13:21.666667
 2016-01-01 09:00:00 | 00:14:20.625
 2016-01-01 12:00:00 | 00:16:32.366667
 2016-01-01 15:00:00 | 00:19:16.921569
 2016-01-01 18:00:00 | 00:22:46.5
 2016-01-01 21:00:00 | 00:17:22.285714
 2016-01-02 00:00:00 | 00:19:24
 2016-01-02 03:00:00 | 00:19:24
 2016-01-02 06:00:00 | 00:12:13.5
 2016-01-02 09:00:00 | 00:17:17.785714
 2016-01-02 12:00:00 | 00:20:56.785714
 2016-01-02 15:00:00 | 00:24:41.730769
 2016-01-02 18:00:00 | 00:29:39.555556
 2016-01-02 21:00:00 | 00:20:09.6
...
```

您也可以为训练和测试数据集创建两个 PostgreSQL 视图。

```sql
-- 制作训练数据集
CREATE VIEW rides_length_train AS
SELECT * FROM rides_length
WHERE three_hour <= '2016-01-21 23:59:59';

-- 制作测试数据集
CREATE VIEW rides_length_test AS
SELECT * FROM rides_length
WHERE three_hour >= '2016-01-22 00:00:00';
```

Python 有一个 [`psycopg2`][python-psycopg2] 包，允许您在 Python 中查询数据库：

```python
import psycopg2
import psycopg2.extras

# 建立连接
conn = psycopg2.connect(dbname='nyc_data', user='postgres', host='localhost')

# 游标对象允许查询数据库
# 创建服务器端游标以防止记录在显式提取之前被下载
cursor_train = conn.cursor('train', cursor_factory=psycopg2.extras.DictCursor)
cursor_test = conn.cursor('test', cursor_factory=psycopg2.extras.DictCursor)

# 执行 SQL 查询
cursor_train.execute('SELECT * FROM rides_length_train')
cursor_test.execute('SELECT * FROM rides_length_test')

# 从数据库获取记录
ride_length_train = cursor_train.fetchall()
ride_length_test = cursor_test.fetchall()
```

您现在可以操纵数据以输入 Holt-Winters 模型。

```python
import pandas as pd
import numpy as np

# 将记录变成 pandas 数据框
ride_length_train = pd.DataFrame(np.array(ride_length_train), columns = ['time', 'trip_length'])
ride_length_test = pd.DataFrame(np.array(ride_length_test), columns = ['time', 'trip_length'])

# 将数据框列的类型转换为 datetime 和 timedelta
ride_length_train['time'] = pd.to_datetime(ride_length_train['time'], format = '%Y-%m-%d %H:%M:%S')
ride_length_test['time'] = pd.to_datetime(ride_length_test['time'], format = '%Y-%m-%d %H:%M:%S')
ride_length_train['trip_length'] = pd.to_timedelta(ride_length_train['trip_length'])
ride_length_test['trip_length'] = pd.to_timedelta(ride_length_test['trip_length'])

# 将数据框的索引设置为时间戳
ride_length_train.set_index('time', inplace = True)
ride_length_test.set_index('time', inplace = True)

# 将 trip_length 转换为秒的数值
ride_length_train['trip_length'] = ride_length_train['trip_length']/np.timedelta64(1, 's')
ride_length_test['trip_length'] = ride_length_test['trip_length']/np.timedelta64(1, 's')
```

现在这些数据可以用来训练从 [`statsmodels`][python-statsmodels] 包导入的 Holt-Winters 模型。您可以预期模式每周重复一次，因此将 `seasonal_periods` 参数设置为 56（一天有八个 3 小时期，一周有七天）。由于季节性变化可能随时间相对恒定，您可以使用加法方法而不是乘法方法，这由 `trend` 和 `seasonal` 参数指定。

```python
from statsmodels.tsa.api import ExponentialSmoothing
fit = ExponentialSmoothing(np.asarray(ride_length_train['trip_length']), seasonal_periods = 56, trend = 'add', seasonal = 'add').fit()
```

您可以使用已训练的模型进行预测并与测试数据集进行比较。

```python
ride_length_test['forecast'] = fit.forecast(len(ride_length_test))
```


现在 `ride_length_test` 有一个列，包含从 1 月 22 日到 1 月 31 日的观测值和预测值。您可以将这些值绘制在一起进行视觉比较：

```python
import matplotlib.pyplot as plt
plt.plot(ride_length_test)
plt.title('从金融区到时代广场的出租车行程长度随时间变化')
plt.xlabel('日期')
plt.ylabel('行程长度（秒）')
plt.legend(['观测', '预测'])
plt.show()
```

<img class="main-content__illustration" src="http://assets.iobeam.com/images/docs/rides_length.png"  alt="Rides Length Graph" />

模型预测从金融区到时代广场的行程长度大致在 16 分钟到 38 分钟之间波动，中午高点，夜间低点。工作日的行程长度明显长于周末（1 月 23、24、30、31）。

从绘制的图形的初始反应是，模型在捕捉总体趋势方面做得相对较好，但有时误差幅度相当大。这可能是由于曼哈顿交通状况的固有不规则性，频繁的道路堵塞、事故和意外的天气条件。此外，正如您使用 R 对季节性 ARIMA 模型进行分析时的情况一样，Holt-Winters 模型也被前一个星期一的马丁·路德·金日的异常数据点所干扰。

### 分析要点

本教程探讨了您可以构建统计模型来分析时间序列数据的不同方法，以及您如何利用 TimescaleDB 与 PostgreSQL 生态系统的全部力量。本教程还探讨了将 TimescaleDB 与 R、Apache MADlib 和 Python 集成。您可以选择您最熟悉的选项，从 TimescaleDB 继承自 PostgreSQL 的众多选择中。ARIMA 和 Holt-Winters 只是您可以用于分析和预测 TimescaleDB 数据库中时间序列数据的众多统计模型和机器学习算法中的两个。

[NYCTLC]: http://www.nyc.gov/html/tlc/html/about/trip_record_data.shtml 
[arima]: https://en.wikipedia.org/wiki/Autoregressive_integrated_moving_average 
[forecast-sql]: http://assets.iobeam.com/sql/forecast.sql 
[gap_filling]: /use-timescale/:currentVersion:/query-data/advanced-analytic-queries/#gap-filling
[holt-winters]: https://otexts.org/fpp2/holt-winters.html 
[install]: /getting-started/latest/
[install_python]: https://www.python.org/downloads/ 
[install_r]: https://www.r-project.org/ 
[madlib]: http://madlib.apache.org/ 
[madlib_arima]: http://madlib.apache.org/docs/latest/group__grp__arima.html 
[madlib_install]: https://cwiki.apache.org/confluence/display/MADLIB/Installation+Guide 
[python-psycopg2]: https://pypi.org/project/psycopg2/ 
[python-statsmodels]: http://www.statsmodels.org/dev/tsa.html 
[r-forecast]: https://cran.r-project.org/web/packages/forecast/forecast.pdf 
[r-xts]: https://cran.r-project.org/web/packages/xts/xts.pdf 
[rpostgres]: https://cran.r-project.org/web/packages/RPostgres/index.html
