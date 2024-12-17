---
标题: 设置 TimescaleDB 与格拉法纳（Grafana）
摘要: 使用格拉法纳（Grafana）在 TimescaleDB 托管服务上可视化您的数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [格拉法纳（Grafana），可视化，分析]
---

import GrafanaConnect from "versionContent/_partials/_grafana-connect.mdx";

# 设置TimescaleDB和Grafana

本教程使用TimescaleDB的托管服务来设置您的数据库，并设置Grafana。

## 为Grafana创建新服务

您需要登录到您的Timescale托管服务账户来创建一个新的服务来运行Grafana。

<Procedure>

### 为Grafana创建新服务

1.  登录到您的TimescaleDB托管服务账户，并点击“创建新服务”。
2.  在“选择您的服务”部分，点击“TimescaleDB Grafana - 指标仪表板”：
    ![选择Grafana服务](https://assets.timescale.com/docs/images/mst-selectservice-grafana.png)
3.  在“选择您的云服务提供商”和“选择您的云服务区域”部分，选择您喜欢的提供商和区域，或接受默认值。
4.  在“选择您的服务计划”部分，点击“Dashboard-1”。
5.  在“提供您的服务名称”部分，为您的新服务输入一个名称。在这个例子中，我们使用了`grafana-tutorial`。
    ![命名Grafana服务](https://assets.timescale.com/docs/images/mst-nameservice-grafana.png)
6.  当您对选择满意时，点击“创建服务”返回到“服务”视图，同时您的服务正在创建。状态指示器显示“重建”时服务正在创建。当指示器变为绿色并显示“运行中”时，服务即可使用。这通常需要几分钟，但不同的云服务可能有所不同。您可以点击列表中的服务名称查看更多信息并进行更改。
    ![构建Grafana服务](https://assets.timescale.com/docs/images/mst-buildservice-grafana.png)

</Procedure>

## 登录到您的Grafana服务

当您的服务构建完成后，您可以登录并设置您的数据服务。

### 登录到您的MST Grafana服务

1.  在TimescaleDB托管服务的“服务”视图中，点击您的新Grafana服务的名称。
2.  在服务详情页面，记下您的服务用户名和密码，并点击“服务URI”字段中的链接以打开Grafana：
    ![构建Grafana服务](https://assets.timescale.com/docs/images/mst-buildservice-grafana.png)
3.  使用您的服务凭据登录Grafana。

<GrafanaConnect />

<!---
我意识到这部分内容目前可能不太适合放在这里，但我接下来会回到这个教程，并适当地重写它。我保证！--LKB 2023-02-28
-->

