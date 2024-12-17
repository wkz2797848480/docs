---
标题: 分析比特币区块链
摘要: 分析比特币区块链。
产品: [云服务]
关键词: [中级，加密货币，区块链，比特币，金融，分析]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 分析比特币区块链
---

# 分析比特币区块链

[区块链][blockchain-def]本质上是一种分布式数据库。区块链中的[交易][transactions-def]是时间序列数据的一个例子。您可以使用Timescale查询区块链上的交易，就像您可能在任何其他数据库中查询时间序列交易一样。

在本教程中，您将使用Timescale超函数来分析比特币区块链上的交易。您可以使用这些指导来查询区块链上的任何类型的数据，包括其他加密货币、智能合约或健康数据。

## 前提条件

开始之前，请确保您已经：

*   注册了[免费的Timescale账户][cloud-install]。
*   []()<Optional />注册了[Grafana账户][grafana-setup]以绘制您的查询结果。

## 本教程的步骤

本教程涵盖：

1.  [设置您的数据集][blockchain-dataset]。
2.  [查询您的数据集][blockchain-analyze]。

## 使用Timescale分析比特币区块链

本教程使用一个比特币样本数据集向您展示如何聚合区块链交易数据，并构建查询以分析聚合信息。本教程中的查询帮助您确定一种加密货币是否具有高额交易费用，显示交易量和费用之间的相关性，或者是否昂贵的挖掘成本。

首先，设置并连接到Timescale数据库，创建表，并使用`psql`将数据加载到表中。如果您已经完成了[初学者区块链教程][blockchain-query]，那么您已经有了加载的数据集，您可以直接跳到查询部分。

然后，您将学习如何使用Timescale超函数对您的数据集进行分析。它引导您创建一系列连续聚合，并查询聚合以分析数据。您还可以使用这些查询在Grafana中绘制输出结果。

[cloud-install]: /getting-started/:currentVersion:/#create-your-timescale-account
[blockchain-dataset]: /tutorials/:currentVersion:/blockchain-analyze/blockchain-dataset/
[blockchain-analyze]: /tutorials/:currentVersion:/blockchain-analyze/analyze-blockchain-query/
[blockchain-query]: /tutorials/:currentVersion:/blockchain-query/beginner-blockchain-query/
[blockchain-def]: https://www.pcmag.com/encyclopedia/term/blockchain 
[transactions-def]: https://www.pcmag.com/encyclopedia/term/bitcoin-transaction 
[grafana-setup]: /use-timescale/:currentVersion:/integrations/observability-alerting/grafana/installation/
