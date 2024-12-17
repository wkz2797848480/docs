---
标题: 探索股票市场数据
摘要: 使用结合了 Plotly（绘图库）、Pandas（数据分析库）以及 psycopg2（Python 数据库连接库）的 TimescaleDB（时间序列数据库）来探索股票市场数据集。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [金融，分析，psycopg2，Pandas，Plotly]
标签: [K 线]
---

# 探索股市数据

当您成功收集了1分钟的日内股票数据后，是时候开始有趣地探索这些数据了。

由于数据集的高粒度性，有多种方式可以探索它。例如，您可以逐分钟分析股票价格和交易量。使用TimescaleDB，您还可以使用TimescaleDB聚合函数将记录分桶到自定义的时间间隔（例如，2分钟或15分钟）。

让我们看看如何操作！

## 安装Plotly和Pandas

开始数据探索之前，您需要先安装几个工具：

*   [Pandas][pandas-docs]，用于查询和结构化数据（如果您已完成前几节的步骤，这已经安装好了）
*   [Plotly][plotly-docs]，用于快速创建可视化

**安装这两个工具**

```bash
pip install plotly pandas
```

安装完成后，您需要打开一个新的Python文件，或使用Jupyter笔记本开始探索数据集。

## 建立数据库连接

使用您之前用`psycopg2`创建的配置文件来创建数据库连接对象：

```python
import config, psycopg2
conn = psycopg2.connect(database=config.DB_NAME,
                        host=config.DB_HOST,
                        user=config.DB_USER,
                        password=config.DB_PASS,
                        port=config.DB_PORT)
```

在每个数据探索脚本中，您需要引用这个连接对象才能查询数据库。

## 生成股市洞察

让我们开始分析交易量，然后看看每周的价格点，最后深入研究价格变化。所示查询的结果使用Plotly进行可视化。

<Highlight type="tip">
让这些查询作为您的灵感来源，随意更改，比如分析的`bucket`，`symbol`或查询的其他部分。玩得开心！
</Highlight>

1.  哪些符号有最高的交易量？
2.  Apple的交易量随时间如何变化？
3.  Apple的股价随时间如何变化？
4.  哪些符号有最高的周收益？
5.  FAANG（Facebook、Apple、Amazon、Netflix、Google/Alphabet）随时间的周价格？
6.  Apple、Facebook、Google的周价格变化？
7.  Amazon和Zoom的日价格变化分布？
8.  Apple的15分钟K线图

### 1. 哪些符号有最高的交易量？

让我们生成一个条形图，显示过去14天中最活跃交易的符号：

```python
import plotly.express as px
import pandas as pd
query = """
    SELECT symbol, sum(trading_volume) AS volume
    FROM stocks_intraday
    WHERE time > now() - INTERVAL '{bucket}'
    GROUP BY symbol
    ORDER BY volume DESC
    LIMIT 5
""".format(bucket="14 day")
df = pd.read_sql(query, conn)
fig = px.bar(df, x='symbol', y='volume', title="过去14天最活跃交易的符号")
fig.show()
```

![most traded symbols](https://assets.timescale.com/docs/images/tutorials/intraday-stock-analysis/most_traded_symbols.png) 

### 2. Apple的交易量随时间如何变化？

现在让我们尝试一个类似的查询，关注一个符号（例如'AAPL'）的日交易量。

```python
import plotly.express as px
import pandas as pd
query = """
    SELECT time_bucket('{bucket}', time) AS bucket, sum(trading_volume) AS volume
    FROM stocks_intraday
    WHERE symbol = '{symbol}'
    GROUP BY bucket
    ORDER BY bucket
""".format(bucket="1 day", symbol="AAPL")
df = pd.read_sql(query, conn)
fig = px.line(df, x='bucket', y='volume', title="Apple的日交易量随时间变化")
fig.show()
```

![apple trading volume over time](https://assets.timescale.com/docs/images/tutorials/intraday-stock-analysis/apple_trading_volume.png) 

### 3. Apple的股价随时间如何变化？

这个查询返回了Apple随时间变化的周股价：

```python
import plotly.express as px
import pandas as pd
query = """
    SELECT time_bucket('{bucket}', time) AS bucket,
    last(price_close, time) AS last_closing_price
    FROM stocks_intraday
    WHERE symbol = '{symbol}'
    GROUP BY bucket
    ORDER BY bucket
""".format(bucket="7 days", symbol="AAPL")
df = pd.read_sql(query, conn)
fig = px.line(df, x='bucket', y='last_closing_price')
fig.show()
```

![apple price over time](https://assets.timescale.com/docs/images/tutorials/intraday-stock-analysis/apple_price.png) 

### 4. 哪些符号有最高的周收益？

现在生成一个包含最大周收益符号的表格：

```python
import plotly.express as px
import pandas as pd
query = """
    SELECT symbol, bucket, max((closing_price-opening_price)/closing_price*100) AS price_change_pct
    FROM (
        SELECT
        symbol,
        time_bucket('{bucket}', time) AS bucket,
        first(price_open, time) AS opening_price,
        last(price_close, time) AS closing_price
        FROM stocks_intraday
        GROUP BY bucket, symbol
    ) s
    GROUP BY symbol, s.bucket
    ORDER BY price_change_pct {orderby}
    LIMIT 5
""".format(bucket="7 days", orderby="DESC")
df = pd.read_sql(query, conn)
print(df)
```

|symbol |bucket     |price_change_pct |
|-------|-----------|-----------------|
|ZM     |2021-06-07 |24.586495        |
|TSLA   |2021-01-04 |18.280314        |
|BA     |2021-03-08 |17.745225        |
|SNAP   |2021-02-01 |16.149649        |
|TSLA   |2021-03-08 |15.842941        |

`price_change_pct`显示了一周开始和结束之间的价格变化。

`bucket`显示了（周的第一天）。

<Highlight type="tip">
将`orderby`改为"ASC"以查询最大损失。
</Highlight>

### 5. FAANG随时间的周价格？

让我们看看FAANG（Facebook、Apple、Amazon、Netflix、Google/Alphabet）的周股价线图：

```python
import plotly.express as px
import pandas as pd
query = """
    SELECT symbol, time_bucket('{bucket}', time) AS bucket,
    last(price_close, time) AS last_closing_price
    FROM stocks_intraday
    WHERE symbol in {symbols}
    GROUP BY bucket, symbol
    ORDER BY bucket
""".format(bucket="7 days", symbols="('FB', 'AAPL',  'AMZN', 'NFLX', 'GOOG')")
df = pd.read_sql(query, conn)
fig = px.line(df, x='bucket', y='last_closing_price', color='symbol', title="FAANG随时间的周价格")
fig.show()
```

![faang prices](https://assets.timescale.com/docs/images/tutorials/intraday-stock-analysis/faang_prices.png) 

### 6. Apple、Facebook、Google的周价格变化？

直接分析价格点在查看特定符号时很有用，但如果您想比较不同的股票，可能最好看看价格变化。让我们比较Apple、Facebook和Google的价格变化：

```python
import plotly.express as px
import pandas as pd
query = """
   SELECT symbol, bucket, max((closing_price-opening_price)/closing_price) AS price_change_pct
    FROM (
        SELECT
        symbol,
        time_bucket('{bucket}}', time) AS bucket,
        first(price_open, time) AS opening_price,
        last(price_close, time) AS closing_price
        FROM stocks_intraday
        WHERE symbol IN {symbols}
        GROUP BY bucket, symbol
    ) s
    GROUP BY symbol, s.bucket
    ORDER BY bucket
""".format(bucket="7 days", symbols="('AAPL', 'FB', 'GOOG')")
df = pd.read_sql(query, conn)
figure = px.line(df, x="bucket", y="price_change_pct", color="symbol", title="Apple、Facebook、Google的周价格变化")
figure = figure.update_layout(yaxis={'tickformat': '.2%'})
figure.show()
```

![weekly price changes](https://assets.timescale.com/docs/images/tutorials/intraday-stock-analysis/weekly_price_changes.png) 

### 7. Amazon和Zoom的日价格变化分布

现在让我们生成一个散点图，看看Amazon和Zoom的日价格变化分布。分析这些数据可以让您更好地了解个股的波动性以及它们之间的比较。

```python
import plotly.express as px
import pandas as pd
query = """
   SELECT symbol, bucket, max((closing_price-opening_price)/closing_price) AS price_change_pct
    FROM (
        SELECT
        symbol,
        time_bucket('{bucket}', time) AS bucket,
        first(price_open, time) AS opening_price,
        last(price_close, time) AS closing_price
        FROM stocks_intraday

        WHERE symbol IN {symbols}
        GROUP BY bucket, symbol
    ) s
    GROUP BY symbol, s.bucket
    ORDER BY bucket
""".format(bucket="1 day", symbols="('ZM', 'AMZN')")
df = pd.read_sql(query, conn)
figure = px.scatter(df, x="price_change_pct", color="symbol", title="Amazon和Zoom的日价格变化分布")
figure = figure.update_layout(xaxis={'tickformat': '.2%'})
figure.show()
```

![distribution of price changes](https://assets.timescale.com/docs/images/tutorials/intraday-stock-analysis/distribution_price_changes.png) 

### 8. Apple的15分钟K线图

最后，因为这是一个关于股票的教程，让我们为Apple生成一个15分钟的K线图：

对于K线图，您需要导入Plotly的`graph_object`模块。

```python
import pandas as pd
import plotly.graph_objects as go
query = """
    SELECT time_bucket('{bucket}', time) AS bucket,
    FIRST(price_open, time) AS price_open,
    LAST(price_close, time) AS price_close,
    MAX(price_high) AS price_high,
    MIN(price_low) AS price_low
    FROM stocks_intraday
    WHERE symbol = '{symbol}' AND date(time) = date('{date}')
    GROUP BY bucket
""".format(bucket="15 min", symbol="AAPL", date="2021-06-09")
df = pd.read_sql(query, conn)
figure = go.Figure(data=[go.Candlestick(x=df['bucket'],
                   open=df['price_open'],
                   high=df['price_high'],
                   low=df['price_low'],
                   close=df['price_close'],)])
figure.update_layout(title="Apple的15分钟K线图，2021-06-09")
figure.show()
```

<Highlight type="tip">
更改`date`以查看另一天的K线图。
</Highlight>

![candlestick chart apple](https://assets.timescale.com/docs/images/tutorials/intraday-stock-analysis/candlestick.png)
