---
标题: 分析非同质化代币（NFT）销售数据
摘要: 学习如何从最大的 NFT 交易市场收集、存储并分析 NFT 销售数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [加密货币，区块链，金融，分析]
标签: [NFT]
布局组件: [大尺寸的上一篇 / 下一篇按钮]
内容分组: 分析 NFT 数据
---

# 分析非同质化代币（NFT）销售数据

本教程是一个逐步指南，用于收集、存储和分析来自最大NFT市场[OpenSea][opensea]的NFT（非同质化代币）销售数据。

NFT，像许多与区块链和加密货币相关的数据一样，起初可能看起来复杂，但在本教程中，我们将带您从零开始成为NFT领域的英雄，并为您提供分析NFT趋势的基础。

本教程向您展示如何：

*   为NFT交易设计模式
*   摄入时间序列NFT数据及其他相关的关系数据
*   使用PostgreSQL和TimescaleDB查询数据集，以解锁数据中的洞察

## NFT入门套件

本教程是[Timescale NFT入门套件][starter-kit]的一部分，旨在帮助您开始分析NFT数据，并激发您构建自己更复杂的项目。

NFT入门套件包含：

*   数据摄入脚本，从OpenSea收集历史数据并将其摄入到TimescaleDB
*   样本数据集，如果您不想等待太长时间来摄入数据，可以快速开始
*   用于存储NFT销售、资产、收藏和账户的模式
*   预加载样本NFT数据的本地TimescaleDB数据库
*   在[Apache Superset][superset]和[Grafana][grafana]中预构建的仪表板和图表，用于可视化您的数据分析
*   作为您自己分析起点的查询

要开始，请克隆NFT入门套件[Github仓库][starter-kit]并按照本教程操作。

## 完成本教程。赚取一个NFT！

因为我们和您一样喜欢NFT，我们创建了[Time Travel Tigers][eon-collection]，一个关于我们Timescale吉祥物Eon的限量版20个NFT的集合！前20个完成本教程的人可以免费获得限量版NFT！

认领您的NFT很简单。您所要做的就是完成下面的教程，在[这个表格][nft-form]中回答问题，我们将把限量版的Eon NFT发送到您的ETH地址（对您没有任何费用！）。

您可以在[OpenSea][eon-collection]上实时查看Time Travel Tigers系列中的所有NFT。

1.  NFT模式设计和摄入
2.  分析NFT交易

## 前提条件

*   OpenSea API密钥（[从这里申请][opensea-key]）
*   TimescaleDB（[安装选项][install-ts]）
*   Psql或任何其他PostgreSQL客户端（例如DBeaver或PgAdmin）

[eon-collection]: https://opensea.io/collection/time-travel-tigers-by-timescale 
[grafana]: https://grafana.com 
[install-ts]: /getting-started/latest/
[nft-form]: https://docs.google.com/forms/d/e/1FAIpQLSdZMzES-vK8K_pJl1n7HWWe5-v6D9A03QV6rys18woGTZr0Yw/viewform?usp=sf_link 
[nft-wiki]: https://en.wikipedia.org/wiki/Non-fungible_token 
[opensea-key]: https://docs.opensea.io/reference/api-keys 
[opensea]: https://opensea.io/ 
[starter-kit]: https://github.com/timescale/nft-starter-kit 
[superset]: https://superset.apache.org

