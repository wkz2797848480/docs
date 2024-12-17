---
标题: 查询比特币区块链
摘要: 查询比特币区块链。
产品: [云服务]
关键词: [初学者，加密货币，区块链，比特币，金融，分析]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 查询比特币区块链
---

# 查询比特币区块链

[区块链][blockchain-def]本质上是一种分布式数据库。区块链中的[交易][transactions-def]是时间序列数据的一个例子。您可以使用Timescale查询区块链上的交易，就像您在任何其他数据库中查询时间序列交易一样。

在本教程中，您将使用Timescale导入、存储和分析比特币区块链上的交易。您可以使用这些技能查询区块链上的任何数据，包括其他加密货币、智能合约或健康数据。

## 前提条件

开始之前，请确保您已经：

*   注册了[免费的Timescale账户][cloud-install]。

## 本教程的步骤

本教程涵盖：

1.  [设置您的数据集][blockchain-dataset]
2.  [查询您的数据集][blockchain-query]
3.  [附加内容：高效存储数据][blockchain-compress]

## 使用Timescale查询比特币区块链

本教程使用一个比特币样本数据集向您展示如何构建区块链数据的查询。您在本教程中执行的查询用于确定加密货币是否按预期表现，绘制货币价值随时间变化的图表，以及比较不同货币。

它首先教您如何设置并连接到Timescale数据库，创建表，并使用`psql`将数据加载到表中。

然后，您将学习如何对数据集进行分析。它引导您使用PostgreSQL查询获取信息，包括查找区块链上最新的交易，以及使用聚合函数收集交易信息。

完成本教程后，您可以使用相同的数据集完成[高级区块链教程][analyze-blockchain]，该教程展示了如何使用Timescale超函数分析区块链数据。

[cloud-install]: /getting-started/:currentVersion:/#create-your-timescale-account
[blockchain-dataset]: /tutorials/:currentVersion:/blockchain-query/blockchain-dataset/
[blockchain-query]: /tutorials/:currentVersion:/blockchain-query/beginner-blockchain-query/
[blockchain-compress]: /tutorials/:currentVersion:/blockchain-query/blockchain-compress/
[blockchain-def]: https://www.pcmag.com/encyclopedia/term/blockchain 
[transactions-def]: https://www.pcmag.com/encyclopedia/term/bitcoin-transaction 
[analyze-blockchain]: /tutorials/:currentVersion:/blockchain-analyze/
