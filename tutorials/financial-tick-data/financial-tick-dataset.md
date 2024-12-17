---
标题: 分析金融逐笔数据 —— 设置数据集
摘要: 设置数据集，以便能够查询金融逐笔数据来分析价格变化。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [教程，金融，学习]
标签: [教程，初学者]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 分析金融逐笔数据
---

import CreateAndConnect from "versionContent/_partials/_cloud-create-connect-tutorials.mdx";
import CreateHypertable from "versionContent/_partials/_create-hypertable-twelvedata-stocks.mdx";
import AddData from "versionContent/_partials/_add-data-twelvedata-stocks.mdx";

# 设置数据库

本教程使用的数据集包含了前100个最活跃交易符号的每秒股票交易数据，存储在一个名为`stocks_real_time`的超表中。它还包括一个单独的公司符号和公司名称表，存储在一个常规的PostgreSQL表中，名为`company`。

<Collapsible heading="创建Timescale服务并连接到您的服务" defaultExpanded={false}>

<创建和连接/>

</Collapsible>

<Collapsible heading="数据集" defaultExpanded={false}>

数据集每晚更新，包含最近四周的数据，通常大约有800万行数据。股票交易在周一至周五实时记录，通常在纽约证券交易所的正常交易时间内（东部标准时间上午9:30至下午4:00）。

<创建超表 />

<添加数据 />

</Collapsible>

<Collapsible heading="连接到Grafana" defaultExpanded={false}>

本教程中的查询适合在Grafana中进行可视化。如果您想要可视化查询结果，请将您的Grafana账户连接到能源消耗数据集。

<Grafana连接 />

</Collapsible>

