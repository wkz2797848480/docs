---
标题: 设置格拉法纳（Grafana）警报
摘要: 使用格拉法纳（Grafana）在出现问题时接收警报。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [格拉法纳（Grafana），警报，集成]
---

# 设置Grafana警报

警报是监控的重要组成部分，因为它们在出现问题并需要我们关注时主动通知我们。这可能包括：

*   当某物崩溃时
*   您消耗了太多资源（例如，内存，CPU）
*   出现中断
*   用户通过支持票报告性能下降

在本教程中，您将学习如何设置Grafana以在出现问题时通过您已经使用的许多通信渠道提醒您。

### 前提条件

要完成本教程，您需要对结构化查询语言（SQL）有初步了解。教程将指导您完成每个SQL命令，但如果您之前见过SQL将会有所帮助。

*   首先，[安装TimescaleDB][install-timescale]。
*   接下来设置Grafana。

安装TimescaleDB和Grafana完成后，配置Grafana以连接到该数据库。如果您对如何使用TimescaleDB的背景感兴趣，请务必遵循完整的教程。

对于本教程，您需要先创建各种Grafana可视化，然后设置警报。使用我们的全套Grafana教程获取有关Grafana的必要背景。在本教程中，我们将简单地通知您创建哪个Grafana可视化以及使用哪个查询。

### 警报原则

在为您的系统设置警报时，请考虑以下事项：

*   仅在需要人为输入时使用警报
*   小心不要过度使用警报。如果工程师过于频繁地收到警报，它可能会失去作用或无法达到其目的。
*   使用与您的场景直接相关的警报：如果您正在监控SaaS产品，请设置站点正常运行时间和延迟的警报。如果您正在监控基础设施，请设置磁盘使用率、高CPU或内存使用率以及API错误的警报。

### Grafana中警报的介绍

除了数据可视化之外，Grafana还提供警报功能，以保持您对异常情况的了解。通过使用Grafana，您无需学习如何使用另一款软件，也无需在后端集成服务。您只需使用您的仪表板。

#### 使用Grafana警报的优缺点

使用Grafana进行警报有一些缺点：

*   您只能在具有时间序列输出的图形可视化上设置警报
*   因此，您不能使用表格输出或任何非时间序列数据

最终，对于大多数情况，这是可以的，因为：
*   您主要处理时间序列数据进行警报
*   您通常可以将任何其他可视化（例如，Gauge或Single Stat）转换为时间序列图

#### Grafana警报的可用数据源

只有某些数据源支持Grafana警报：PostgreSQL、Prometheus和Cloudwatch。TimescaleDB基于PostgreSQL，是Grafana警报的有效数据源。

Grafana警报有两个部分：**警报规则**和**通知渠道**。

#### 警报规则

警报规则是Grafana警报中最重要的部分。规则是您定义的触发警报的条件。Grafana根据计划评估规则，您需要指定评估规则的频率。

用简单的语言来说，规则的例子可能包括：

*   当磁盘使用率超过90%时
*   当平均内存使用率在5分钟内超过90%时（这是一个基于间隔的规则的例子）
*   当设备的温度超出给定范围时（这是一个具有许多不同条件的规则的例子）

#### 通知渠道

通知渠道是一旦触发警报规则就会发送警报的地方。如果您没有通知渠道，那么您的警报只会在Grafana上显示。

包括您的团队可能已经使用的工具：

*   Slack
*   电子邮件
*   OpsGenie
*   PagerDuty

Grafana提供与webhooks、电子邮件以及十多个外部服务的集成。

每当我们创建一个警报时，我们都会将其分配给一个通知渠道，并附带一条消息。在我们的教程中，我们将设置两个常见的通知渠道：Slack和PagerDuty。

#### 警报的生命周期

您可以将警报视为根据与之相关的规则通过不同状态的对象。可能的状态包括：OK、PENDING、ALERTING、NO DATA。

### 警报1：集成TimescaleDB、Grafana和Slack

我们在这个第一个警报中的目标是在Slack中主动通知我们，当我们长时间持续高内存使用时。您可以使用webhooks将Grafana连接到Slack。

#### 第0步：设置您的Grafana可视化

创建一个新的图形可视化。在查询中，连接到现有的Prometheus数据源并输入此查询：

```sql
SELECT
  $__timeGroupAlias("time", 1m),
  avg(value) as "mem_used_percent"
FROM metrics
WHERE
  $__timeFilter("time") AND
  name LIKE  'mem_used_percent'
GROUP BY 1
ORDER BY 1
```

您的图表应该像这样：

![在Grafana中将内存使用百分比可视化为时间序列数据](https://assets.iobeam.com/images/docs/screenshots-for-grafana-alerts-tutorial/alert1_query_results_5m.png)

#### 第1步：定义Grafana警报规则

点击可视化中的'Bell'图标以导航到警报部分。我们将定义我们的警报，以便在平均内存消耗连续5分钟超过90%时通知我们。

将规则的频率设置为每分钟评估一次。这意味着图表每分钟被轮询一次，以确定是否应该发送警报。

然后将评估周期设置为五分钟。这配置Grafana以五分钟的窗口查看警报。

您无法更改查询的'When'部分，但您可以将'Is Above'阈值设置为90。换句话说，当内存使用率超过90%时，您将收到警报。

使用其余配置的默认值。您的配置应该像这样：

![在PostgreSQL中为内存使用百分比定义Grafana警报](https://assets.iobeam.com/images/docs/screenshots-for-grafana-alerts-tutorial/alert1_alert_configuration.png)

#### 第2步：为Grafana警报配置Slack

在大多数情况下，您希望构建一个分层警报系统，其中较不严重的警报发送到较不侵入的渠道（例如Slack），而更关键的警报发送到高注意力渠道（例如打电话或发短信给某人）。

让我们从配置Slack开始。要设置Slack，您需要您的Slack管理员为您提供发布到频道的webhook URL。您可以[按照这些说明][slack-webhook-instructions]获取此信息。

要配置通知渠道，请转到主仪表板中的'Bell'图标。它位于屏幕的最左侧。点击'Notification Channels'选项。在通知渠道屏幕上，点击'Add channel'。

在结果表单中，设置您的Slack频道的名称。这在您的Grafana实例的下拉菜单中显示，因此选择一些描述性的内容，其他Grafana实例的用户可以立即识别。

选择'Slack'作为类型，并切换'Include image'和'Send reminders'。输入您的Slack管理员提供的Webhook URL，并选择一个对您的Slack实例用户描述性的用户名。如果您想在Slack中的警报帖子中@提及某人或某个组，您可以在'Mention'字段中这样做。

您的配置应该像这样：

![为警报设置Slack和Grafana](https://assets.iobeam.com/images/docs/screenshots-for-grafana-alerts-tutorial/alert1_setup_slack.png)

而且，您应该能够向您的Slack实例发送测试消息。

#### 第3步：将您的Grafana警报连接到Slack

现在回到您的图形可视化，并选择'Alert'标签。在'Notifications'部分，点击'Send to'旁边的'+'图标，并选择您刚刚创建的Slack通知渠道。同时为您的Slack帖子提供一个消息。

此时，您的警报已配置完成。如果您想测试它，随时更改您输入的'Is Above'字段中的“90”值，并将其更改为低于当前阈值的值。它应该在大约五分钟内触发类似这样的通知：

![为警报设置Slack和Grafana](https://assets.iobeam.com/images/docs/screenshots-for-grafana-alerts-tutorial/alert1_slack_messages.png)

### 警报2：集成TimescaleDB、Grafana和PagerDuty

PagerDuty是管理中型大型团队的支持和事件响应的流行选择。本节中的许多步骤与Slack部分的步骤相似。使用PagerDuty，您需要使用PagerDuty API的直接集成来设置警报。

#### 第0步：设置您的Grafana可视化

在本节中，您监控数据库，以防您意外地用完磁盘空间。这是您希望立即通知某人的警报类型。

我们的图形可视化的查询如下：

```sql
SELECT
  $__timeGroupAlias("time", 1m),
  avg(value) AS "% disk used"
FROM metrics
WHERE
  $__timeFilter("time") AND
  name LIKE 'disk_used_percent'
GROUP BY 1
ORDER BY 1
```

#### 第1步：为Grafana警报配置PagerDuty

要将PagerDuty连接到Grafana，您将需要[集成密钥][pagerduty-integration-key]来监控您正在监控的服务。请注意，这与PagerDuty所称的PagerDuty API密钥不同。

再次前往主仪表板，选择'Bell'图标并选择'Notification channels'。添加一个渠道，输入一个描述性的名称，并选择'Pager Duty'类型。提供您的集成密钥。

#### 第2步：定义Grafana警报规则

创建关于磁盘使用的规则与我们之前关于内存使用的规则
类似，只是磁盘使用只能增加。因此，我们不需要像我们对Slack那样提供'For'时间段。所以，在这种情况下，将您的警报设置为每分钟检查一次，持续时间为零分钟。

在'When'子句中，选择`last()`，`query(A, 1m, now)`，并在'Is Above'字段中输入'80'。

在'Notifications'部分中选择您的PagerDuty渠道，并提供一个描述性的消息。

### 警报3：集成TimescaleDB、Grafana和其他通知平台

Grafana支持许多通知平台，包括：

*   DingDing
*   Discord
*   电子邮件
*   Google Hangouts
*   Hipchat
*   Kafka
*   Line
*   Microsoft Teams
*   OpsGenie
*   PagerDuty
*   Prometheus Alertmanager
*   Pushover
*   Slack
*   Telegram
*   Threema
*   VictorOps
*   Webhooks

与所有这些平台集成的步骤与您用于Slack（webhooks）和PagerDuty（API或集成密钥）的步骤类似。

### 总结

通过遵循所有TimescaleDB + Grafana教程，完善您的Grafana知识。

[install-timescale]: /getting-started/latest/
[pagerduty-integration-key]: https://support.pagerduty.com/docs/services-and-integrations 
[slack-webhook-instructions]: https://slack.com/help/articles/115005265063-Incoming-Webhooks-for-Slack
