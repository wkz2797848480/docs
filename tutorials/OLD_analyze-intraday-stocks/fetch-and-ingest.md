---
标题: 获取并摄取盘中股票数据
摘要: 将盘中股票数据摄取到 TimescaleDB 数据库中。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [金融，分析，psycopg2，Pandas，Plotly]
标签: [K 线]
---

# 获取并导入日内股票数据

在这一步中：

*   创建配置文件（可选）
*   获取股票数据
*   将数据导入到TimescaleDB中

## 创建配置文件

这是一个可选步骤，但强烈建议您不要将密码或其他敏感信息直接存储在代码中。相反，创建一个配置文件，例如`config.py`，并在其中包含您的数据库连接详情和Alpha Vantage API密钥：

```python
# config.py的示例内容：
DB_USER = 'tsdbadmin'
DB_PASS = 'passwd'
DB_HOST = 'xxxxxxx.xxxxxxx.tsdb.cloud.timescale.com'
DB_PORT = '66666'
DB_NAME = 'tsdb'
APIKEY = 'alpha_vantage_apikey'
```

稍后，每当您需要引用此配置文件中的任何信息时，您需要导入它：

```python
import config
apikey = config.APIKEY
...
```

## 收集股票代码

为了获取日内股票数据，您需要知道您想要分析哪些股票代码。首先，我们来收集一个代码列表，以便我们稍后可以获取它们的数据。通常，您有几个选项可以动态收集股票代码列表：

*   从一个公共网站抓取（[示例代码在这里][scraping-example]）
*   使用具有此功能的API
*   从一个开放仓库下载

为了简化事情，下载这个CSV文件开始：

*   [下载前100个美国股票代码（基于市值）][symbols-csv]

## 从CSV文件中读取代码

将CSV文件下载到项目文件夹后，创建一个名为`ingest_stock_data.py`的新Python文件。确保将此文件添加到与`symbols.csv`文件相同的文件夹中。在此文件中添加以下代码，将`symbols.csv`文件读取到一个列表中：

```python
# ingest_stock_data.py:
import csv
with open('symbols.csv') as f:
    reader = csv.reader(f)
    symbols = [row[0] for row in reader]
    print(symbols)
```

运行此代码：

```bash
python ingest_stock_data.py
```

您应该看到打印出的代码列表：

```bash
['AAPL', 'MSFT', 'AMZN', 'GOOG', 'FB']
```

现在您有了一个股票代码列表，稍后可以使用它向Alpha Vantage API发出请求。

## 获取日内股票数据

### 关于API

Alpha Vantage API提供2年的历史日内股票数据，间隔为1、5、15或30分钟。API输出大量数据到CSV文件中（大约每个代码每天2200行，对于1分钟间隔），因此它将数据集分割成一个月的桶。这意味着对于单个代码的一个请求，您可以获得的最大数据量是一个月。最大历史日内数据量为24个月。要获取最大数量的数据，您需要按月分割您的请求。例如，`year1month1`，`year1month2`等。请注意，每个请求一次只能获取一个代码的数据。

这里有一个示例API端点：

```
https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY_EXTENDED&symbol=IBM&interval=1min&slice=year1month1&apikey=your_apikey 
```

查看[Alpha Vantage API](https://www.alphavantage.co/documentation/) 文档了解更多信息。

### 创建函数

让我们首先在您之前创建的Python脚本`ingest_stock_data.py`中创建一个函数。这个函数获取一个代码和一个月份的数据。该函数以这两个值为参数：

*   `symbol`：您想要获取数据的股票代码（例如，亚马逊的"AMZN"）。
*   `month`：一个介于1-24之间的整数值，表示您想要获取哪个月份的数据。

将以下代码添加到`ingest_stock_data.py`文件中：

```python
import config
import pandas as pd

def fetch_stock_data(symbol, month):
    """获取单个股票代码（1分钟间隔）的历史日内数据

    参数：
        symbol (string)：股票代码

    返回：
        蜡烛图数据（元组列表）
    """
    interval = '1min'

    # API要求您按月分割请求
    # 如"year1month1"，"year1month2"，...，"year2month1"等...
    slice = "year1month" + str(month) if month <= 12 else "year2month1" + str(month)

    apikey = config.APIKEY

    # 用代码、切片、间隔和API密钥制定正确的API端点
    CSV_URL = 'https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY_EXTENDED&symbol={symbol}&interval={interval}&slice={slice}&apikey={apikey}'.format(symbol=symbol, slice=slice, interval=interval, apikey=apikey)

    # 直接将CSV文件读取到pandas dataframe中
    df = pd.read_csv(CSV_URL)

    # 在dataframe中添加一个新的代码列
    # 这是必要的，因为API不返回代码值
    df['symbol'] = symbol

    # 将时间列转换为datetime对象
    # 这是必要的，以便我们稍后可以无缝地将数据插入数据库
    df['time'] = pd.to_datetime(df['time'], format='%Y-%m-%d %H:%M:%S')

    # 重命名和重新排序列以匹配数据库模式
    df = df.rename(columns={'time': 'time',
                            'open': 'price_open',
                            'high': 'price_high',
                            'low': 'price_low',
                            'close': 'price_close',
                            'volume': 'trading_volume'})
    df = df[['time', 'symbol', 'price_open', 'price_close', 'price_low', 'price_high', 'trading_volume']]

    # 将dataframe转换为准备导入的元组列表
    return [row for row in df.itertuples(index=False, name=None)]
```

运行此脚本：

```bash
python ingest_stock_data.py
```

这个函数从Alpha Vantage API下载数据并为其准备数据库导入。在函数定义后添加这段代码以测试其工作情况：

```python
def test_stock_download():
    test_stock_data = fetch_stock_data("MSFT", 1)
    print(test_stock_data)
test_stock_download()
```

运行脚本：

```bash
python ingest_stock_data.py
```

您应该看到一个巨大的元组列表打印出来，每个元组包含时间戳值和价格数据（蜡烛图）：

```text
[
(Timestamp('2022-05-23 04:04:00'), 255.8, 255.95, 255.8, 255.8, 771, 'MSFT'),
(Timestamp('2022-05-23 04:01:00'), 255.01, 256.5, 255.01, 256.5, 1235, 'MSFT')
...
]
```

将来运行脚本时，如果不需要调用`test_stock_download()`，则将其移除。

## 将数据导入到TimescaleDB

当您的`fetch_stock_data`函数工作正常，并且您可以从API获取蜡烛图时，您可以将其插入数据库。

为了加快导入速度，请使用[pgcopy][pgcopy-docs]而不是逐行导入数据。TimescaleDB作为PostgreSQL的扩展包，这意味着所有您熟悉和喜爱的PostgreSQL工具已经可以与TimescaleDB一起工作。

### 使用pgcopy快速导入数据

安装psycopg2和pgcopy，以便您可以连接到数据库并导入数据。

**安装psycopg2**

```bash
pip install psycopg2-binary
```

**安装pgcopy**

```bash
pip install pgcopy
```

在`ingest_stock_data.py`脚本底部添加以下代码：

**使用pgcopy导入**

```python
from pgcopy import CopyManager
import config, psycopg2

# 建立数据库连接
conn = psycopg2.connect(database=config.DB_NAME,
                        host=config.DB_HOST,
                        user=config.DB_USER,
                        password=config.DB_PASS,
                        port=config.DB_PORT)

# 数据库中的列名（pgcopy需要它作为参数）
COLUMNS = ('time', 'symbol', 'price_open', 'price_close', 'price_low', 'price_high', 'trading_volume')

# 遍历代码列表
for symbol in symbols:

    # 指定一个时间范围（最多24个月）
    time_range = range(1, 2) # （最后1个月）

    # 遍历指定的时间范围
    for month in time_range:

        # 获取给定代码和月份的股票数据
        # 使用您之前创建的函数
        stock_data = fetch_stock_data(symbol, month)

        # 创建一个复制管理器实例
        mgr = CopyManager(conn, 'stocks_intraday', COLUMNS)

        # 插入数据并提交事务
        mgr.copy(stock_data)
        conn.commit()
```

这将开始逐个代码，逐月导入数据。如果您想要下载更多数据，可以修改`time_range`。

```
time               |symbol|price_open|price_close|price_low|price_high|trading_volume|
-------------------+------+----------+-----------+---------+----------+--------------+
2022
-06-21 22:00:00|AAPL  |    135.66|      135.6|   135.55|    135.69|         14871|
2022-06-21 21:59:00|AAPL  |    135.64|     135.64|   135.64|    135.64|           567|
2022-06-21 21:58:00|AAPL  |    135.67|     135.67|   135.67|    135.67|          1611|
2022-06-21 21:57:00|AAPL  |    135.66|      135.7|   135.66|     135.7|           972|
2022-06-21 21:56:00|AAPL  |  135.6401|   135.6401| 135.6401|  135.6401|           441|
2022-06-21 21:54:00|AAPL  |    135.66|     135.66|   135.66|    135.66|           550|
2022-06-21 21:53:00|AAPL  |    135.66|     135.66|   135.66|    135.66|           269|
2022-06-21 21:52:00|AAPL  |    135.67|     135.67|   135.67|    135.67|          2298|
2022-06-21 21:50:00|AAPL  |    135.67|     135.63|   135.63|    135.67|          1086|
2022-06-21 21:49:00|AAPL  |    135.65|     135.65|   135.65|    135.65|           458|
2022-06-21 21:48:00|AAPL  |    135.64|     135.64|   135.64|    135.64|          1006|
2022-06-21 21:47:00|AAPL  |    135.63|     135.63|   135.63|    135.63|           512|
2022-06-21 21:45:00|AAPL  |    135.61|     135.61|   135.61|    135.61|           441|
2022-06-21 21:44:00|AAPL  |    135.62|     135.62|   135.62|    135.62|           526|
```

<Highlight type="tip">
获取和导入日内数据可能需要一段时间，所以如果您想快速看到结果，可以减少月份数量，或限制代码数量。
</Highlight>

这是`ingest_stock_data.py`的最终版本：

```python
# ingest_stock_data.py:
import csv
import config
import pandas as pd
from pgcopy import CopyManager
import psycopg2

with open('symbols.csv') as f:
    reader = csv.reader(f)
    symbols = [row[0] for row in reader]
    print(symbols)
    
def fetch_stock_data(symbol, month):
    """获取单个股票代码（1分钟间隔）的历史日内数据

    参数：
        symbol (string)：股票代码

    返回：
        蜡烛图数据（元组列表）
    """
    interval = '1min'

    # API要求您按月分割请求
    # 如"year1month1"，"year1month2"，...，"year2month1"等...
    slice = "year1month" + str(month) if month <= 12 else "year2month1" + str(month)

    apikey = config.APIKEY

    # 用代码、切片、间隔和API密钥制定正确的API端点
    CSV_URL = 'https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY_EXTENDED&symbol={symbol}&interval={interval}&slice={slice}&apikey={apikey}'.format(symbol=symbol, slice=slice, interval=interval, apikey=apikey)

    # 直接将CSV文件读取到pandas dataframe中
    df = pd.read_csv(CSV_URL)

    # 在dataframe中添加一个新的代码列
    # 这是必要的，因为API不返回代码值
    df['symbol'] = symbol

    # 将时间列转换为datetime对象
    # 这是必要的，以便我们稍后可以无缝地将数据插入数据库
    df['time'] = pd.to_datetime(df['time'], format='%Y-%m-%d %H:%M:%S')

    # 重命名和重新排序列以匹配数据库模式    
    df = df.rename(columns={'open': 'price_open',
                            'high': 'price_high',
                            'low': 'price_low',
                            'close': 'price_close',
                            'volume': 'trading_volume'})
    df = df[['time', 'symbol', 'price_open', 'price_close', 'price_low', 'price_high', 'trading_volume']]
    
    # 将dataframe转换为准备导入的元组列表
    return [row for row in df.itertuples(index=False, name=None)]

# 建立数据库连接
conn = psycopg2.connect(database=config.DB_NAME,
                        host=config.DB_HOST,
                        user=config.DB_USER,
                        password=config.DB_PASS,
                        port=config.DB_PORT)

# 数据库中的列名（pgcopy需要它作为参数）
COLUMNS = ('time', 'symbol', 'price_open', 'price_close', 'price_low', 'price_high', 'trading_volume')

# 遍历代码列表
for symbol in symbols:

    # 指定一个时间范围（最多24个月）
    time_range = range(1, 2) # （最后1个月）

    # 遍历指定的时间范围
    for month in time_range:

        # 获取给定代码和月份的股票数据
        # 使用您之前创建的函数
        stock_data = fetch_stock_data(symbol, month)
        print(stock_data)

        # 创建一个复制管理器实例
        mgr = CopyManager(conn, 'stocks_intraday', COLUMNS)

        # 插入数据并提交事务
        mgr.copy(stock_data)
        conn.commit()

```

[pgcopy-docs]: https://pgcopy.readthedocs.io/en/latest/ 
[scraping-example]: https://github.com/timescale/examples/blob/master/ 
[symbols-csv]: https://assets.timescale.com/docs/downloads/symbols.csv
