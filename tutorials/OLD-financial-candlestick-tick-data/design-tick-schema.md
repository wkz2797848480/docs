---
标题: 设计架构并摄取逐笔交易数据
摘要: 在 TimescaleDB（时间序列数据库）中摄取并存储实时加密货币或股票数据。
关键词: [金融，分析]
标签: [K 线]
---

# 设计模式并摄入tick数据

本教程向您展示如何在TimescaleDB中存储实时加密货币或股票的tick数据。初始模式提供了仅存储tick数据的基础。一旦您开始存储个别交易，您可以使用基于原始tick数据的TimescaleDB连续聚合来计算K线值。这意味着我们的初始模式不需要特别存储K线数据。

## 模式

此模式使用两个表：

*   **crypto_assets**：一个关系表，用于存储要监控的符号。
   您还可以包括每个符号的额外信息，例如社交链接。
*   **crypto_ticks**：一个时间序列表，用于存储实时tick数据。

**crypto_assets:**

|字段|描述|
|-|-|
|symbol|加密货币对的符号，例如BTC/USD|
|name|对的名称，例如Bitcoin USD|

**crypto_ticks:**

|字段|描述|
|-|-|
|time|时间戳，在UTC时区|
|symbol|来自`crypto_assets`表的加密对符号|
|price|在该时间交易所注册的价格|
|day_volume|给定当天的总成交量（增量）|

创建表：

```sql
CREATE TABLE crypto_assets (
    symbol TEXT UNIQUE,
    "name" TEXT
);

CREATE TABLE crypto_ticks (
    "time" TIMESTAMPTZ,
    symbol TEXT,
    price DOUBLE PRECISION,
    day_volume NUMERIC
);
```

您还需要将时间序列表转换为[hypertable][hypertable]：

```sql
-- 将常规的`crypto_ticks`表转换为7天块的TimescaleDB hypertable
SELECT create_hypertable('crypto_ticks', 'time');
```

这是高效存储TimescaleDB中的时间序列数据的重要步骤。

### 使用TIMESTAMP数据类型

使用`TIMESTAMP WITH TIME ZONE`（`TIMESTAMPTZ`）数据类型存储时间值是最佳实践。这使得使用不同时区查询数据变得更加容易。TimescaleDB在内部以UTC存储`TIMESTAMPTZ`值，并为您的查询进行必要的转换。

## 插入tick数据

创建了hypertable和关系表后，下载包含过去三周的加密资产和tick数据的示例文件。将数据插入到您的TimescaleDB实例中。

<程序>

### 插入示例数据

1.  下载示例`.csv`文件（由[Twelve Data][twelve-data]提供）：<Tag type="download">[crypto_sample.csv](https://assets.timescale.com/docs/downloads/candlestick/crypto_sample.zip)</Tag> 

    ```bash
    wget https://assets.timescale.com/docs/downloads/candlestick/crypto_sample.zip 
    ```

1.  如果需要，解压缩文件并更改目录：

    ```bash
    unzip crypto_sample.zip
    cd crypto_sample
    ```

1.  在`psql`提示符下，将`.csv`文件的内容插入数据库。

    ```bash
    psql -x "postgres://tsdbadmin:{YOUR_PASSWORD_HERE}@{YOUR_HOSTNAME_HERE}:{YOUR_PORT_HERE}/tsdb?sslmode=require"

    \COPY crypto_assets FROM 'crypto_assets.csv' CSV HEADER;
    \COPY crypto_ticks FROM 'crypto_ticks.csv' CSV HEADER;
    ```

</程序>

如果您想要摄入实时市场数据，而不是示例数据，请查看我们的补充教程Ingest real-time financial websocket data to ingest data directly from the [Twelve Data][twelve-data] financial API。

[hypertable]: /use-timescale/:currentVersion:/hypertables/
[twelve-data]: https://twelvedata.com/

