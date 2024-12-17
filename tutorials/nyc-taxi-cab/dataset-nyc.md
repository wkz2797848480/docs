---
标题: 查询时间序列数据教程 —— 设置数据集
摘要: 设置数据集，以便能够查询时间序列数据。
产品: [云服务]
关键词: [初学者，教程，创建，数据集]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 分析纽约市出租车数据
---

import CreateAndConnect from "versionContent/_partials/_cloud-create-connect-tutorials.mdx";
import CreateHypertableNyc from "versionContent/_partials/_create-hypertable-nyctaxis.mdx";
import AddDataNyc from "versionContent/_partials/_add-data-nyctaxis.mdx";
import PreloadedData from "versionContent/_partials/_preloaded-data.mdx";

# 设置数据库

本教程使用的数据集包含了纽约黄色出租车网络的历史数据，存储在一个名为`rides`的超表中。它还包括一个单独的支付类型和费率表，在常规的PostgreSQL表中，分别命名为`payment_types`和`rates`。

<预加载数据 />

<Collapsible heading="创建Timescale服务并连接到您的服务" defaultExpanded={false}>

<创建和连接/>

</Collapsible>

<Collapsible heading="数据集" defaultExpanded={false}>

本教程使用的历史数据来自纽约市出租车和礼宾车委员会[NYC TLC][nyc-tlc]提供的纽约黄色出租车网络。

<创建纽约超表 />

<添加纽约数据 />

</Collapsible>

[nyc-tlc]: https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page

