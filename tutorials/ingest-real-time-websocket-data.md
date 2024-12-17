---
标题: 摄取实时金融网络套接字数据
摘要: 搭建一个数据管道，以便从不同的金融应用程序编程接口（API）获取数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [金融，分析，网络套接字，数据管道]
---

import CreateHypertableStocks from "versionContent/_partials/_create-hypertable-twelvedata-stocks.mdx";
import GraphOhlcv from "versionContent/_partials/_graphing-ohlcv-data.mdx";

# 实时金融WebSocket数据摄入

本教程向您展示如何使用WebSocket连接将实时时间序列数据摄入到TimescaleDB中。本教程建立了一个数据管道，从我们的数据合作伙伴[Twelve Data][twelve-data]摄入实时数据。Twelve Data提供多种不同的金融API，包括股票、加密货币、外汇和ETF。它还支持WebSocket连接，以便您频繁更新数据库。使用WebSockets，您需要连接到服务器、订阅符号，您就可以在交易时间内开始实时接收数据。

完成本教程后，您将建立一个数据管道，将实时金融数据摄入到您的Timescale中。

本教程使用Python和Twelve Data提供的API[包装库][twelve-wrapper]。

## 前提条件

开始之前，请确保您已具备以下条件：

*   注册了[免费Timescale账户][cloud-install]。
*   下载了包含您的Timescale服务凭据的文件，如`<HOST>`、`<PORT>`和`<PASSWORD>`。或者，您可以在服务的`连接信息`部分找到这些详细信息。
*   安装了Python 3。
*   注册了[Twelve Data][twelve-signup]。免费层级非常适合本教程。
*   记下您的Twelve Data[API密钥](https://twelvedata.com/account/api-keys)。

<Collapsible heading="连接到websocket服务器" defaultExpanded={false}>

当您通过websocket连接到Twelve Data API时，您在计算机和websocket服务器之间创建了一个持久连接。
您设置了一个Python环境，并传递了两个参数以创建一个websocket对象并建立连接。

## 设置一个新的Python环境

为这个项目创建一个新的Python虚拟环境并激活它。完成本教程所需的所有软件包都安装在这个环境中。

<Procedure>

### 设置一个新的Python环境

1.  创建并激活一个Python虚拟环境：

    ```bash
    virtualenv env
    source env/bin/activate
    ```

1.  安装Twelve Data Python
    [包装库][twelve-wrapper]
    支持websocket。这个库允许您向API发送请求并维护一个稳定的websocket连接。

    ```bash
    pip install twelvedata websocket-client
    ```

1.  安装[Psycopg2][psycopg2]，以便您可以从Python脚本连接TimescaleDB：

    ```bash
    pip install psycopg2-binary
    ```

</Procedure>

## 创建websocket连接

计算机和websocket服务器之间的持久连接用于在连接保持期间接收数据。您需要传递两个参数以创建一个websocket对象并建立连接。

### WebSocket参数

*   `on_event`

    这个参数需要是一个函数，每当从websocket接收到新的数据记录时调用：

    ```python
    def on_event(event):
        print(event) # 打印出数据记录（字典）
    ```

    这就是您想要实现摄入逻辑的地方，因此每当有新数据可用时，您将其插入数据库。

*   `symbols`

    这个参数需要是一个股票代码符号列表（例如`MSFT`）或加密货币交易对（例如`BTC/USD`）。当使用websocket连接时，您总是需要订阅您想要接收的事件。您可以使用`symbols`参数来做到这一点，或者如果您的连接已经创建，您也可以使用`subscribe()`函数来获取额外符号的数据。

<Procedure>

### 连接到websocket服务器

1.  创建一个名为`websocket_test.py`的新Python文件，并使用`<YOUR_API_KEY>`连接到Twelve Data服务器：

    ```python
       import time
       from twelvedata import TDClient

        messages_history = []

        def on_event(event):
         print(event) # 打印出数据记录（字典）
         messages_history.append(event)

       td = TDClient(apikey="<YOUR_API_KEY>")
       ws = td.websocket(symbols=["BTC/USD", "ETH/USD"], on_event=on_event)
       ws.subscribe(['ETH/BTC', 'AAPL'])
       ws.connect()
       while True:
       print('messages received: ', len(messages_history))
       ws.heartbeat()
       time.sleep(10)
    ```

1.  运行Python脚本：

    ```bash
    python websocket_test.py
    ```

1.  当您运行脚本时，您将从服务器接收到关于您连接状态的响应：

    ```bash
    {'event': 'subscribe-status',
     'status': 'ok',
     'success': [
            {'symbol': 'BTC/USD', 'exchange': 'Coinbase Pro', 'mic_code': 'Coinbase Pro', 'country': '', 'type': 'Digital Currency'},
            {'symbol': 'ETH/USD', 'exchange': 'Huobi', 'mic_code': 'Huobi', 'country': '', 'type': 'Digital Currency'}
        ],
     'fails': None
    }
    ```

    当您与websocket服务器建立连接后，等待几秒钟，您可以看到类似这样的数据记录：

    ```bash
    {'event': 'price', 'symbol': 'BTC/USD', 'currency_base': 'Bitcoin', 'currency_quote': 'US Dollar', 'exchange': 'Coinbase Pro', 'type': 'Digital Currency', 'timestamp': 1652438893, 'price': 30361.2, 'bid': 30361.2, 'ask': 30361.2, 'day_volume': 49153}
    {'event': 'price', 'symbol': 'BTC/USD', 'currency_base': 'Bitcoin', 'currency_quote': 'US Dollar', 'exchange': 'Coinbase Pro', 'type': 'Digital Currency', 'timestamp': 1652438896, 'price': 30380.6, 'bid': 30380.6, 'ask': 30380.6, 'day_volume': 49157}
    {'event': 'heartbeat', 'status': 'ok'}
    {'event': 'price', 'symbol': 'ETH/USD', 'currency_base': 'Ethereum', 'currency_quote': 'US Dollar', 'exchange': 'Huobi', 'type': 'Digital Currency', 'timestamp': 1652438899, 'price': 2089.07, 'bid': 2089.02, 'ask': 2089.03, 'day_volume': 193818}
    {'event': 'price', 'symbol': 'BTC/USD', 'currency_base': 'Bitcoin', 'currency_quote': 'US Dollar', 'exchange': 'Coinbase Pro', 'type': 'Digital Currency', 'timestamp': 1652438900, 'price': 30346.0, 'bid': 30346.0, 'ask': 30346.0, 'day_volume': 49167}
    ```

    每个价格事件为您提供了关于给定交易对的多个数据点，例如交易所的名称和当前价格。您还可以偶尔在响应中看到`heartbeat`事件；这些事件随着时间的推移信号连接的健康状况。
    此时，websocket连接成功地传递数据。

</Procedure>

</Collapsible>

<Collapsible heading="实时数据集" headingLevel={2} defaultExpanded={false}>
    
要将数据摄入到您的Timescale服务中，您需要实现`on_event`函数。
    
websocket连接设置完成后，您可以使用`on_event`函数将数据摄入到数据库中。这是一个数据管道，它将实时金融数据摄入到您的Timescale服务中。

股票交易在周一至周五实时摄入，通常在纽约证券交易所的正常交易时间（上午9:30至下午4:00东部时间）。

<CreateHypertableStocks />

当您将数据摄入到像Timescale这样的事务数据库中时，批量插入数据比逐行插入数据更有效。使用一个事务插入多行可以显著提高您的Timescale数据库的整体摄入能力和速度。

## 内存中批处理

实现批处理的一个常见做法是先将新记录存储在内存中，然后当批处理达到一定大小时，将所有记录从内存中一次性插入数据库。完美的批处理大小并不是通用的，但您可以尝试不同的批处理大小（例如100、1000、10000等），看看哪一个更适合您的用例。使用批处理是将数据从Kafka、Kinesis或websocket连接摄入TimescaleDB时的一个相当常见的模式。

您可以使用Psycopg2在Python中实现批处理解决方案。
您可以在`on_event`函数中实现摄入逻辑，然后将其传递给websocket对象。

这个函数需要：

1.  检查项目是否是数据项，而不是websocket元数据。
1.  调整数据，使其适应数据库模式，包括数据类型和列的顺序。
1.  将其添加到内存批处理中，这是Python中的一个列表。
1.  如果批处理达到一定大小时，插入数据，并重置或清空列表。

## 实时摄入数据

<Procedure>

1.  更新打印当前批处理大小的Python脚本，以便您可以跟踪数据何时从内存中摄入到数据库中。使用您想要摄入数据的Timescale服务的`<HOST>`、`<PASSWORD>`和`<PORT>`详细信息，以及您从Twelve Data获得的API密钥：

    ```python
    import time
    import psycopg2

    from twelvedata import TDClient
    from psycopg2.extras import execute_values
    from datetime import datetime

    class WebsocketPipeline():
        # 超表的名称
        DB_TABLE = "stocks_real_time"

        # 超表中按正确顺序的列
        DB_COLUMNS=["time", "symbol", "price", "day_volume"]

        # 用于批量插入数据的批处理大小
        MAX_BATCH_SIZE=100

        def __init__(self, conn):
            ""“连接到Twelve Data web socket服务器并将数据流式传输
            到数据库。

            参数：
                conn：psycopg2连接对象
            ""“
            self.conn = conn
            self.current_batch = []
            self.insert_counter = 0

        def _insert_values(self, data):
            if self.conn is not None:
                cursor = self.conn.cursor()
                sql = f"""
                INSERT INTO {self.DB_TABLE} ({','.join(self.DB_COLUMNS)}) 
                VALUES %s;"""
                execute_values(cursor, sql, data)
                self.conn.commit()

        def _on_event(self, event):
            ""“每当有新的数据记录从服务器返回时调用此函数。

            参数：
                event （字典）：数据记录
            ""“
            if event["event"] == "price":
                # 数据记录
                timestamp = datetime.utcfromtimestamp(event["timestamp"])
                data = (timestamp, event["symbol"], event["price"], event.get("day_volume"))

                # 将新数据记录添加到批处理
                self.current_batch.append(data)
                print(f"Current batch size: {len(self.current_batch)}")

                # 如果达到最大批处理大小，则摄入数据，然后重置批处理
                if len(self.current_batch) == self.MAX_BATCH_SIZE:
                    self._insert_values(self.current_batch)
                    self.insert_counter += 1
                    print(f"Batch insert #{self.insert_counter}")
                    self.current_batch = []
            def start(self, symbols):
                ""“连接到web socket服务器并开始流式传输实时数据 
                到数据库。

                参数：
                    symbols （符号列表）：股票/加密货币符号列表
                ""“
                td = TDClient(apikey="<YOUR_API_KEY")
                ws = td.websocket(on_event=self._on_event)
                ws.subscribe(symbols)
                ws.connect()
                while True:
                   ws.heartbeat()
                   time.sleep(10)
        onn = psycopg2.connect(database="tsdb", 
                            host="<HOST>", 
                            user="tsdbadmin", 
                            password="<PASSWORD>",
                            port="<PORT>")
    
        symbols = ["BTC/USD", "ETH/USD", "MSFT", "AAPL"]
        websocket = WebsocketPipeline(conn)
        websocket.start(symbols=symbols)
        ```

1.  运行脚本：

    ```bash
    python websocket_test.py
    ```

</Procedure>

您甚至可以创建单独的Python脚本来启动多个websocket连接，用于不同类型的符号，例如一个用于股票，另一个用于加密货币价格。

### 故障排除

如果您看到类似这样的错误消息：

```bash
2022-05-13 18:51:41,976 - ws-twelvedata - ERROR - TDWebSocket ERROR: Handshake status 200 OK
```

那么请检查您是否使用了从Twelve Data获得的正确API密钥。

</Collapsible>

<Collapsible heading="查询数据" defaultExpanded={false}>

要查看OHLCV值，最有效的方法是创建一个连续聚合。您可以创建一个连续聚合来聚合每小时的数据，然后将聚合设置为每小时刷新一次，并聚合最后两小时的数据。

<Procedure>

### 创建连续聚合

1.  连接到包含Twelve Data股票数据集的Timescale数据库`tsdb`。

1.  在psql提示符下，创建连续聚合以每分钟聚合数据：

    ```sql
    CREATE MATERIALIZED VIEW one_hour_candle
    WITH (timescaledb.continuous) AS
        SELECT
            time_bucket('1 hour', time) AS bucket,
            symbol,
            FIRST(price, time) AS "open",
            MAX(price) AS high,
            MIN(price) AS low,
            LAST(price, time) AS "close",
            LAST(day_volume, time) AS day_volume
        FROM stocks_real_time
        GROUP BY bucket, symbol;
    ```

    当您创建连续聚合时，它默认会刷新。

1.  设置一个刷新策略，以在每小时更新连续聚合，如果有新数据在超表中最后两小时可用：

    ```sql
    SELECT add_continuous_aggregate_policy('one_hour_candle',
        start_offset => INTERVAL '3 hours',
        end_offset => INTERVAL '1 hour',
        schedule_interval => INTERVAL '1 hour');
    ```

</Procedure>

## 查询连续聚合

当您设置好连续聚合后，您可以查询它以获取OHLCV值。

<Procedure>

### 查询连续聚合

1.  连接到包含Twelve Data股票数据集的Timescale数据库。

1.  在psql提示符下，使用此查询选择所有`AAPL`过去5小时的OHLCV数据，按时间桶：

    ```sql
    SELECT * FROM one_hour_candle
    WHERE symbol = 'AAPL' AND bucket >= NOW() - INTERVAL '5 hours'
    ORDER BY bucket;
    ```

    查询结果如下：

    ```sql
             bucket         | symbol  |  open   |  high   |   low   |  close  | day_volume
    ------------------------+---------+---------+---------+---------+---------+------------
     2023-05-30 08:00:00+00 | AAPL   | 176.31 | 176.31 |    176 | 176.01 |
     2023-05-30 08:01:00+00 | AAPL   | 176.27 | 176.27 | 176.02 |  176.2 |
     2023-05-30 08:06:00+00 | AAPL   | 176.03 | 176.04 | 175.95 |    176 |
     2023-05-30 08:07:00+00 | AAPL   | 175.95 |    176 | 175.82 | 175.91 |
     2023-05-30 08:08:00+00 | AAPL   | 175.92 | 176.02 |  175.8 | 176.02 |
     2023-05-30 08:09:00+00 | AAPL   | 176.02 | 176.02 |  175.9 | 175.98 |
     2023-05-30 08:10:00+00 | AAPL   | 175.98 | 175.98 | 175.94 | 175.94 |
     2023-05-30 08:11:00+00 | AAPL   | 175.94 | 175.94 | 175.91 | 175.91 |
     2023-05-30 08:12:00+00 | AAPL   |  175.9 | 175.94 |  175.9 | 175.94 |
    ```

</Procedure>

</Collapsible>

<Collapsible heading="在Grafana中可视化OHLCV数据" defaultExpanded={false}>

您可以使用Grafana可视化您使用查询创建的OHLCV数据。
<GraphOhlcv />

</Collapsible>

[candlestick-tutorial]: /tutorials/:currentVersion:/financial-tick-data/
[get-started]: /getting-started/:currentVersion:/
[install-ts]: /getting-started/latest/
[psycopg2]: https://www.psycopg.org/docs/ 
[twelve-data]: https://twelvedata.com 
[twelve-signup]: https://twelvedata.com/pricing 
[twelve-wrapper]: https://github.com/twelvedata/twelvedata-python 
[cloud-install]: /getting-started/:currentVersion:/#create-your-timescale-account
