---
标题: 连接 TimescaleDB 和格拉法纳（Grafana）
摘要: 将 Timescale 连接到格拉法纳（Grafana）以可视化您的数据。
产品: [云服务，管理服务技术（MST）]
关键词: [格拉法纳（Grafana），可视化，分析]
---

# 连接TimescaleDB和Grafana

Grafana内置了Prometheus、PostgreSQL、Jaeger等数据源插件，允许您从兼容的数据库查询和可视化数据。要在Grafana中添加数据源，您必须以具有组织管理角色权限的用户身份登录。

要将Grafana与Timescale连接，请首先安装Grafana。有关安装Grafana的更多信息，请参见[Grafana文档][grafana-install]。

或者，要将您的Grafana服务与TimescaleDB的托管服务连接，请在TimescaleDB的托管服务上创建一个Grafana服务。您可以免费试用30天。

本节向您展示如何在[Grafana][grafana-homepage]中将Timescale配置为数据源。

## 将Timescale配置为数据源

要将Timescale配置为数据源，您需要创建一个服务，然后在Grafana中将Timescale配置为数据源。

<Procedure>

### 创建Timescale服务

1.  登录到[Timescale门户][tsc-portal]。
2.  点击`Create service`。
3.  点击`Download the cheatsheet`。这个`.sql`文件包含了您需要的凭据，以便在Grafana上配置TimescaleDB作为数据源。

</Procedure>

<Procedure>

### 将TimescaleDB配置为数据源

要将TimescaleDB服务与您的Grafana安装一起配置，请登录到Grafana并进行到本程序的第5步。

1.  登录到您的Timescale托管服务账户，并点击您的新Grafana服务的名称。
2.  在服务详情页面，记下您服务的`User`和`Password`字段。
3.  点击`Service URI`字段中的链接以打开Grafana。
4.  使用您的服务凭据登录Grafana。
5.  导航到`Configuration` → `Data sources`。数据源页面列出了Grafana实例先前配置的数据源。
6.  点击`Add data source`以查看所有支持的数据源列表。
7.  在搜索字段中输入`PostgreSQL`并点击`Select`。
8.  配置数据源：
    *   在`Name`字段中，输入您希望TimescaleDB数据集的名称。
    *   在`PostgreSQL Connection`部分，使用创建TimescaleDB服务时下载的`.sql`文件中的`Database`、`User`和`Password`字段。
    *   在`Host`中输入`.sql`文件中的`<HOST>:<PORT>`。
    *   设置`TLS/SSL Mode`为`require`。
    *   在`PostgreSQL details`中启用`TimescaleDB`
9.  点击`Save & test`按钮。如果连接成功，将显示`Database Connection OK`。

</Procedure>

当您在Grafana中将TimescaleDB配置为数据源后，您可以创建使用SQL填充数据的面板。

[grafana-homepage]: https://grafana.com/ 
[tsc-portal]: https://console.cloud.timescale.com/ 
[grafana-install]: https://grafana.com/docs/grafana/latest/installation/

