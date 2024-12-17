---
标题: 在工具中对部件进行动词化
摘要: 通过对部件进行动词化，利用该工具达成某个成果。
关键词: [名词，动词，教程]
标签: [名词，名词]
---

<!-- markdown-link-check-disable -->

# 在工具中使用小部件

本教程的单段描述。确保在一两句话中涵盖教程的内容，包括预期的学习成果。例如：

```txt
本教程向您展示如何高效存储原始金融tick数据，创建不同的K线视图，并使用OHLCV格式在TimescaleDB中查询聚合数据。
```

## 前提条件

在开始之前，请确保您拥有：

*   本地或云端运行的TimescaleDB实例。
  有关更多信息，请参见[安装选项][install-docs]。
*   [`psql`][psql]或任何其他PostgreSQL客户端。

## 本教程中的步骤

本教程中子页面的编号列表。请记住，这是课程内容，因此这些步骤必须按顺序进行：

1.  [设置您的数据集][tutorial-dataset]
2.  [查询您的数据集][tutorial-query]
3.  [更多尝试的内容][tutorial-advanced]

## 关于小部件和工具

本节收集与教程相关的所有概念信息，以及在整个过程中使用的工具。它回答了“这是什么？”的问题。本节不应包含任何程序，但如果用于解释功能，可以包含代码示例。以逻辑的方式划分此页面，从简单概念开始，逐步过渡到更复杂的概念。谨慎使用图表和截图，并确保它们增加价值。尽量保持本节简洁，通过链接到其他地方存在的更长材料来实现。

例如：

```txt
K线图在金融领域中用于可视化资产价格的变化。每个K线代表一个时间框架，例如1分钟、1小时或类似的时间，并显示该时间段内资产价格的变化。
```

在页面底部包含参考风格的链接。

[install-docs]: install/:currentVersion:/
[psql]: timescaledb/:currentVersion:/how-to-guides/connecting/
[tutorial-dataset]: timescaledb/tutorials/_template/_dataset-tutorial
[tutorial-query]: timescaledb/tutorials/_template/_query-template
[tutorial-advanced]: timescaledb/tutorials/_template/_advanced-tutorial

