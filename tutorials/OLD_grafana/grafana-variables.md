---
标题: 使用格拉法纳（Grafana）变量
摘要: 学习如何利用格拉法纳（Grafana）变量协助终端用户筛选并定制可视化内容。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [格拉法纳（Grafana），可视化，分析]
---

# 使用Grafana变量

Grafana变量使您的仪表板的最终用户能够过滤和自定义可视化效果。

### 前提条件

要完成本教程，您需要对结构化查询语言（SQL）有初步了解。教程将指导您完成每个SQL命令，但如果您之前接触过SQL将会有所帮助。

*   首先，[安装TimescaleDB][install-timescale]。
*   接下来，设置Grafana。

当您的TimescaleDB和Grafana安装完成后，摄取纽约出租车教程中的数据，并将Grafana配置为连接到该数据库。如果您对如何使用TimescaleDB感兴趣，请确保按照完整的教程操作。

### 创建变量

我们的目标是创建一个变量，该变量基于用于乘车的支付类型来控制显示的行程类型。

在`payment_types`表中，我们可以看到有几种类型的支付方式：

```
 payment_type | description
--------------+-------------
            1 | credit card
            2 | cash
            3 | no charge
            4 | dispute
            5 | unknown
            6 | voided trip
(6 rows)
```

Grafana包括许多类型的变量，Grafana中的变量就像编程语言中的变量一样。我们定义一个变量，然后可以在我们的查询中引用它。

#### 定义一个新的Grafana变量

要创建一个新变量，请转到您的Grafana仪表板设置，导航到侧边栏中的“变量”选项，然后点击“添加变量”按钮。

在这种情况下，我们使用“查询”类型，您的变量定义为SQL查询的结果。

在“常规”部分，我们将变量命名为`payment_type`并给它一个`查询`类型。
然后，我们给它分配一个标签“Payment Type”，这是它在下拉菜单中显示的方式。

选择您的数据源并提供查询：

```sql
SELECT payment_type FROM payment_types;
```

打开“多值”和“包含所有选项”。这使您的仪表板用户可以选择多个支付类型。我们的配置应该像这样：

![定义Grafana变量](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/grafana_define_variable.png)

点击“添加”保存您的变量。

#### 在Grafana面板中使用变量

让我们编辑我们在Grafana地理空间查询教程中创建的WorldMap面板。首先您会注意到，现在我们已经为这个仪表板定义了一个变量，面板的左上角现在有一个该变量的下拉菜单。

我们可以使用这个变量通过SQL中的`WHERE`子句来过滤我们的查询结果。检查`rides.payment_type`是否在变量数组中，我们已经将其命名为`$payment_type`。

让我们这样修改我们之前的查询：

```sql
SELECT time_bucket('5m', rides.pickup_datetime) AS time,
       rides.trip_distance AS value,
       rides.pickup_latitude AS latitude,
       rides.pickup_longitude AS longitude
FROM rides
WHERE $__timeFilter(rides.pickup_datetime) AND
  ST_Distance(pickup_geom,
              ST_Transform(ST_SetSRID(ST_MakePoint(-73.9851,40.7589),4326),2163)
  ) < 2000 AND
  rides.payment_type IN ($payment_type)
GROUP BY time,
         rides.trip_distance,
         rides.pickup_latitude,
         rides.pickup_longitude
ORDER BY time
LIMIT 500;
```

现在我们可以使用下拉菜单根据使用的支付类型过滤我们的行程：

![使用变量过滤Grafana Worldmap的结果](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/grafana_worldmap_query_with_variable.png)

#### 使用Grafana变量构建动态面板

我们已经看到了如何在查询中使用Grafana变量。您还可以使用Grafana变量构建动态面板。在我们的例子中，我们已经允许人们根据出租车行程的`payment_type`选择数据。我们还可以为选择的每个`payment_type`自动创建一个图表面板，以便我们可以并排查看这些查询。

首先，让我们创建一个新的图表面板，使用`$payment_type`变量。这是您的查询：

```sql
SELECT
  --1--
  time_bucket('5m', pickup_datetime) AS time,
  --2--
  COUNT(*)
FROM rides
WHERE $__timeFilter(pickup_datetime)
AND rides.payment_type IN ($payment_type)
GROUP BY time
ORDER BY time
```

现在，让我们使这个面板动态化，以便我们在下拉菜单中检查的每个变量都有一个单独的面板。首先，通过更改面板标题以包含变量名称。转到面板的“常规”标签，并更改“标题”为以下内容：

```bash
{$payment_type} Taxi Rides
```

在“重复”部分，选择您想要基于哪个变量生成动态面板的变量。在这种情况下，是`payment_type`。您可以让您的动态面板垂直或水平生成。在这种情况下，选择重复面板，每行2个，水平排列：

![在Grafana中创建动态面板](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/grafana_create_dynamic_panels.png)

保存并刷新您的仪表板。使用下拉菜单选择一些支付类型。您的仪表板应该像这样：

![Grafana中的动态面板](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/grafana_dynamic_panels.png)

#### 改进Grafana过滤器以提高可读性

但是我们的过滤器并不是很吸引人。我们不知道'1'意味着什么。幸运的是，当我们设置纽约出租车数据集时，我们创建了一个`payment_types`表（我们之前查询过）。`payment_types.description`字段有每个支付代码更易于阅读的解释，例如'credit card'、'cash'等。这些可读的描述是我们想要在下拉菜单中显示的。

点击“仪表板设置”（Grafana可视化右上角的“齿轮”图标）。选择左侧的“变量”选项卡，然后点击`$payment_types`变量。修改您的查询以检索`description`并将其存储在`__text`字段中，检索`payment_type`并将其存储在`__value`字段中，如下所示：

```sql
SELECT description AS "__text", payment_type AS "__value" FROM payment_types
```

您的配置现在应该像这样：

![修改我们的Grafana变量以提高可读性](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/grafana_modify_variable.png)

无需更改WorldMap可视化本身的查询。
当变量显示时，分配为`__text`的任何数据库列将被使用，而分配给`__value`的实际值将被Grafana在进行查询时使用。

正如您所看到的，变量可以在查询中使用，就像您在任何编程语言中使用变量一样。

#### 使用Grafana变量构建动态面板

我们已经看到了如何在查询中使用Grafana变量。您还可以使用Grafana变量构建动态面板，根据变量选择的值自动创建面板。在我们的例子中，我们已经允许人们根据出租车行程的`payment_type`选择数据。我们希望自动为选择的每个`payment_type`创建一个图表面板，以便我们可以并排查看这些查询。

首先，让我们创建一个新的图表面板，使用`$payment_type`变量。这是您的查询：

```sql
SELECT
  --1--
  time_bucket('5m', pickup_datetime) AS time,
  --2--
  COUNT(*)
FROM rides
WHERE $__timeFilter(pickup_datetime)
AND rides.payment_type IN ($payment_type)
GROUP BY time
ORDER BY time
```

现在，让我们使这个面板动态化，以便我们在下拉菜单中检查的每个变量都有一个单独的面板。首先，通过更改面板标题以包含变量名称。转到面板的“常规”标签，并更改“标题”为以下内容：

```bash
{$payment_type} Taxi Rides
```

在“重复”部分，选择您想要基于哪个变量生成动态面板的变量。在这种情况下，是`payment_type`。您可以让您的动态面板垂直或水平生成。在这种情况下，选择重复面板，每行2个，水平排列：

![在Grafana中创建动态面板](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/grafana_create_dynamic_panels.png)

保存并刷新您的仪表板。使用下拉菜单选择一些支付类型。您的仪表板应该像这样：

![Grafana中的动态面板](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/grafana_dynamic_panels.png)

### 总结

通过遵循所有TimescaleDB + Grafana教程，完成您的Grafana知识。

[install-timescale]: /getting-started/latest/
