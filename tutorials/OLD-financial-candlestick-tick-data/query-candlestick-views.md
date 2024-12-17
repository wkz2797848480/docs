---
标题: 查询K线图视图
摘要: 查询你的数据以创建并可视化K线图聚合数据。
关键词: [金融，分析]
标签: [K线]
---

# 查询蜡烛图视图

在本教程中，你已经创建了存储逐笔数据的模式，并设置了多个蜡烛图视图。在本节中，使用一些示例蜡烛图查询，并看看它们如何在数据可视化中表示。

<Highlight type="note">
本节中的查询是示例查询。与本教程一起提供的[样本数据](https://assets.timescale.com/docs/downloads/crypto_sample.zip) 
定期更新，以获得近实时数据，通常不超过几天。我们的样本查询反映了可能比你通常使用的更长的时间过滤器，因此请随意根据数据的老化修改`WHERE`子句中的时间过滤器，或者当你开始插入更新的逐笔数据时进行修改。
</Highlight>

## 1分钟BTC/USD蜡烛图

从包含1分钟蜡烛图的`one_min_candle`连续聚合开始：

```sql
SELECT * FROM one_min_candle
WHERE symbol = 'BTC/USD' AND bucket >= NOW() - INTERVAL '24 hour'
ORDER BY bucket
```

![1分钟蜡烛图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/candlestick/one_min.png) 

## 1小时BTC/USD蜡烛图

如果你发现1分钟蜡烛图太细粒度，你可以查询包含1小时蜡烛图的`one_hour_candle`连续聚合：

```sql
SELECT * FROM one_hour_candle
WHERE symbol = 'BTC/USD' AND bucket >= NOW() - INTERVAL '2 day'
ORDER BY bucket
```

![1小时蜡烛图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/candlestick/one_hour.png) 

## 1天BTC/USD蜡烛图

要进一步缩小视图，查询包含一天蜡烛图的`one_day_candle`连续聚合：

```sql
SELECT * FROM one_day_candle
WHERE symbol = 'BTC/USD' AND bucket >= NOW() - INTERVAL '14 days'
ORDER BY bucket
```

![1天蜡烛图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/candlestick/one_day.png) 

## BTC与ETH 1天价格变化差异折线图

你可以计算并可视化两个符号之间的价格变化差异。在之前的示例中，你看到了如何通过比较开盘价和收盘价来实现这一点。但是，如果你想将今天的收盘价与昨天的收盘价进行比较呢？这里有一个示例，展示了如何使用[`LAG()`][lag]窗口函数在已有的蜡烛图视图上实现这一点：

```sql
SELECT *, ("close" - LAG("close", 1) OVER (PARTITION BY symbol ORDER BY bucket)) / "close" AS change_pct
FROM one_day_candle
WHERE symbol IN ('BTC/USD', 'ETH/USD') AND bucket >= NOW() - INTERVAL '14 days'
ORDER BY bucket
```

![BTC对比ETH](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/candlestick/pct_change.png) 

[lag]: https://www.postgresqltutorial.com/postgresql-lag-function/

