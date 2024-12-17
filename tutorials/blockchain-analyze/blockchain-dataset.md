---
标题: 分析比特币区块链 —— 设置数据集
摘要: 设置数据集，以便能够分析比特币区块链。
产品: [云服务]
关键词: [中级，加密货币，区块链，比特币，金融，分析]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 分析比特币区块链
---

import CreateAndConnect from "versionContent/_partials/_cloud-create-connect-tutorials.mdx";
import CreateHypertableBlockchain from "versionContent/_partials/_create-hypertable-blockchain.mdx";
import AddDataBlockchain from "versionContent/_partials/_add-data-blockchain.mdx";
import GrafanaConnect from "versionContent/_partials/_grafana-connect.mdx";

# 设置数据库

本教程使用的数据集包含了过去五天的比特币区块链数据，存储在一个名为`transactions`的超表中。

<Collapsible heading="创建Timescale服务并连接到您的服务" defaultExpanded={false}>

<CreateAndConnect/>

</Collapsible>

<Collapsible heading="数据集" defaultExpanded={false}>

数据集每天更新，包含最近五天的数据，通常是大约150万笔比特币交易。数据包括每笔交易的详细信息，包括交易价值以[satoshi][satoshi-def]计，即比特币的最小单位。它还指出某笔交易是否是区块中的第一笔交易，称为[coinbase][coinbase-def]交易，这包括矿工挖矿得到的奖励。

<CreateHypertableBlockchain />

<AddDataBlockchain />

</Collapsible>

<Collapsible heading="连接到Grafana" defaultExpanded={false}>

本教程中的查询适合在Grafana中进行图形化展示。如果您想要可视化查询结果，请将您的Grafana账户连接到比特币区块链数据集。

<GrafanaConnect />

</Collapsible>

[satoshi-def]: https://www.pcmag.com/encyclopedia/term/satoshi 
[coinbase-def]: https://www.pcmag.com/encyclopedia/term/coinbase-transaction
