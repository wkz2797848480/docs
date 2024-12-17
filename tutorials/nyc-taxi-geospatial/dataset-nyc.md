---
标题: 绘制地理空间时间序列数据教程 —— 设置数据集
摘要: 设置数据集，以便能够查询地理空间时间序列数据。
产品: [云服务]
关键词: [教程，地理信息系统（GIS），地理空间，学习]
标签: [教程，中级]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 绘制纽约市出租车地理空间数据
---

import CreateAndConnect from "versionContent/_partials/_cloud-create-connect-tutorials.mdx";
import CreateHypertableNyc from "versionContent/_partials/_create-hypertable-nyctaxis.mdx";
import AddDataNyc from "versionContent/_partials/_add-data-nyctaxis.mdx";
import GrafanaConnect from "versionContent/_partials/_grafana-connect.mdx";

# 设置数据库

本教程使用的数据集包含了纽约黄色出租车网络的历史数据，存储在一个名为 `rides` 的超表中。它还包括一个单独的支付类型和费率表，分别存储在常规的 PostgreSQL 表中，名为 `payment_types` 和 `rates`。

<Collapsible heading="创建Timescale服务并连接到您的服务" defaultExpanded={false}>

<CreateAndConnect/>

</Collapsible>

<Collapsible heading="数据集" defaultExpanded={false}>

本教程使用的数据集由纽约市出租车和豪华轿车委员会 [NYC TLC][nyc-tlc] 提供，包含了纽约黄色出租车网络的历史数据。

<CreateHypertableNyc />

<AddDataNyc />

</Collapsible>

<Collapsible heading="连接到Grafana" defaultExpanded={false}>

本教程中的查询适合在 Grafana 中可视化。如果您想要可视化查询结果，请将您的 Grafana 账户连接到纽约出租车数据集。

<GrafanaConnect />

</Collapsible>

[nyc-tlc]: https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page
