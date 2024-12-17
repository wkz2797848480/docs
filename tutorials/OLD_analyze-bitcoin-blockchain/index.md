---
标题: 分析比特币区块链
摘要: 学习如何存储并分析你的比特币区块链数据以发现趋势。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [加密货币，区块链，比特币，金融，分析]
布局组件: [大尺寸的上一篇 / 下一篇按钮]
内容分组: 分析比特币区块链
---

# 分析比特币区块链

区块链数据是时间序列数据。你可以使用TimescaleDB来摄取、存储和分析区块链交易数据。本教程重点分析比特币，但你可以将同样的原则和TimescaleDB功能应用于任何区块链数据，包括以太坊、索拉纳等。

<Highlight type="note">
本教程向您展示了在区块链领域进行自主研究的一种方法。从数据中得出的任何结论仅作为示例。它们的目的是帮助您了解TimescaleDB功能，并激发您自己的数据分析和结论。要阅读我们从分析5年的比特币交易数据中得出的结论，请[查看我们的博客文章](https://www.timescale.com/blog/analyzing-the-bitcoin-blockchain-looking-behind-the-hype-with-postgresql/)。
</Highlight>

## 你将学到什么

本教程教你如何在TimescaleDB中摄取和分析区块链数据。

## 前提条件

开始之前，请确保你拥有：

*   本地或云端运行的TimescaleDB实例。更多信息，请参见[安装选项][install-timescale]
*   [`psql`][psql-install]、DBeaver或任何其他PostgreSQL客户端

<Highlight type="note">
启动新的TimescaleDB实例并完成本教程的最简单方法是[注册一个免费的Timescale账户](http://console.cloud.timescale.com/signup)（无需信用卡）。
</Highlight>

[install-timescale]: /getting-started/latest/
[psql-install]: /use-timescale/:currentVersion:/integrations/query-admin/about-psql

