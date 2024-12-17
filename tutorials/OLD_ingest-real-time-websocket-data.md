---
标题: 摄取实时金融网络套接字数据
摘要: 设置数据管道，以便从不同的金融应用程序编程接口获取数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [金融，分析，网络套接字，数据管道]
---

# 摄入实时金融websocket数据

本教程向您展示如何使用websocket连接将实时时间序列数据摄入到TimescaleDB中。教程设置了数据管道，从我们的数据中心合作伙伴[Twelve Data][twelve-data]摄入实时数据。Twelve Data提供了许多不同的金融API，包括股票、加密货币、外汇、ETF等。如果您希望频繁更新数据库，它还支持websocket连接。使用websockets，您需要连接到服务器，订阅符号，您就可以在市场交易时间内开始实时接收数据。

完成本教程后，您将拥有一个设置好数据管道，将实时金融数据摄入到您的TimescaleDB实例中。

本教程使用Python和Twelve Data提供的API[包装库][twelve-wrapper]。

## 前提条件

开始之前，请确保您已具备以下条件：

*   正在本地或云端运行的TimescaleDB实例。更多信息，请[查看安装选项][install-ts]。
*   已安装Python 3。
*   已注册[Twelve Data][twelve-signup]。免费层级非常适合本教程。

## 设置新的Python环境

为这个项目创建一个新的Python虚拟环境并激活它。完成本教程所需的所有包都安装在这个环境中。

<Procedure>

### 设置新的Python环境

1. 创建并激活Python虚拟环境：

    ```bash
    virtualenv env
    source env/bin/activate
    ```

2. 安装Twelve Data Python[包装库][twelve-wrapper]，支持websocket。这个库使您能够轻松地向API发送请求并保持稳定的websocket连接。

    ```bash
    pip install twelvedata websocket-client
    ```

3. 安装[Psycopg2][psycopg2]，以便您可以从Python脚本连接TimescaleDB：

    ```bash
    pip install psycopg2-binary
    ```

</Procedure>

## 创建websocket连接

当您通过websocket连接到Twelve Data API时，您在计算机和websocket服务器之间创建了一个持久连接。这个持久连接可以用来接收数据，只要连接保持，您就可以持续接收数据。您需要传递两个参数来创建websocket对象并建立连接。

### Websocket参数

*   `on_event`

    这个参数需要是一个函数，每当从websocket接收到新的数据记录时就调用它：

    ```python
    def on_event(event):
        print(event) # 打印出数据记录（字典）
    ```

    这就是您想要实现数据摄入逻辑的地方，所以每当有新数据可用时，您就将其插入数据库。

*   `symbols`

    这个参数需要是一个股票代码列表（例如，`MSFT`）或加密货币交易对（例如，`BTC/USD`）。当使用websocket连接时，您总是需要订阅您想要接收的事件。您可以通过使用`symbols`参数来做到这一点，或者如果您的连接已经创建，您也可以使用`subscribe()`函数来获取额外符号的数据。

<Procedure>

### 连接到websocket服务器

1. 创建一个名为`websocket_test.py`的新Python文件，并使用包装库连接到Twelve Data服务器：

    ```python
    # websocket_test.py:
    from twelvedata import TDClient
    
    def on_event(event):
     print(event) # 打印出数据记录（字典）
    
    td = TDClient(apikey="TWELVE_DATA_APIKEY")
    ws = td.websocket(symbols=["BTC/USD", "ETH/USD"], on_event=on_event)
    ws.connect()
    ws.keep_alive()
    ```

    确保将您的API密钥作为参数传递给`TDClient`对象。

2. 现在运行Python脚本：

    ```bash
    python websocket_test.py
    ```

3. 运行脚本后，您会立即从服务器接收到关于您连接状态的响应：

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

</Procedure>

<Highlight type="note">
为了无限期保持websocket连接活跃，请使用包装库的`keep_alive()`函数。它确保连接将保持活动状态，直到被终止。如果您不添加这行代码，连接可能会立即中断。
</Highlight>

当您与websocket服务器建立连接后，等待几秒钟，您就可以看到实际的数据记录，如下所示：

```bash
{'event': 'price', 'symbol': 'BTC/USD', 'currency_base': 'Bitcoin', 'currency_quote': 'US Dollar', 'exchange': 'Coinbase Pro', 'type': 'Digital Currency', 'timestamp': 1652438893, 'price': 30361.2, 'bid': 30361.2, 'ask': 30361.2, 'day_volume': 49153}
{'event': 'price', 'symbol': 'BTC/USD', 'currency_base': 'Bitcoin', 'currency_quote': 'US Dollar', 'exchange': 'Coinbase Pro', 'type': 'Digital Currency', 'timestamp': 1652438896, 'price': 30380.6, 'bid': 30380.6, 'ask': 30380.6, 'day_volume': 49157}
{'event': 'heartbeat', 'status': 'ok'}
{'event': 'price', 'symbol': 'ETH/USD', 'currency_base': 'Ethereum', 'currency_quote': 'US Dollar', 'exchange': 'Huobi', 'type': 'Digital Currency', 'timestamp': 1652438899, 'price': 2089.07, 'bid': 2089.02, 'ask': 2089.03, 'day_volume': 193818}
{'event': 'price', 'symbol': 'BTC/USD', 'currency_base': 'Bitcoin', 'currency_quote': 'US Dollar', 'exchange': 'Coinbase Pro', 'type': 'Digital Currency', 'timestamp': 1652438900, 'price': 30346.0, 'bid': 30346.0, 'ask': 30346.0, 'day_volume': 49167}
```

每个价格事件都会给您提供关于给定交易对的多个数据点，例如交易所的名称和当前价格。您还可以偶尔在响应中看到`heartbeat`事件；这些事件随着时间的推移信号连接的健康状况。

此时，websocket连接工作正常，数据持续流动。您需要实现`on_event`函数，以便数据被摄入到TimescaleDB中。

## 将websocket数据摄入到TimescaleDB中

现在websocket连接已设置好，您可以使用`on_event`函数将数据摄入到数据库中。

当您将数据摄入到像TimescaleDB这样的事务数据库中时，批量插入数据比逐行插入数据更有效。使用一个事务插入多行可以显著提高您的TimescaleDB实例的整体摄入能力和速度。

### 内存中批处理

实现批处理的常见做法是先将新记录存储在内存中，然后当批处理达到一定大小时，一次性将所有记录从内存插入到数据库中。完美的批处理大小并不是通用的，但您可以尝试不同的批处理大小（例如，100、1000、10000等），看看哪个更适合您的用例。使用批处理是从一个Kafka、Kinesis或websocket连接摄入数据到TimescaleDB时的一个相当常见的模式。

现在您可以看到如何使用Psycopg2在Python中实现批处理解决方案。

### 使用Psycopg2实现批处理

记得在`on_event`函数中实现摄入逻辑，然后将其传递给websocket对象。

这个函数需要：

1.  检查项目是否是数据项，而不是websocket元数据。
2.  调整数据，使其适应数据库模式，包括数据类型和列的顺序。
3.  将其添加到内存批处理中，这是Python中的一个列表。
4.  如果批处理达到一定大小，则插入数据并重置或清空列表。

以下是完整实现：

```python
from psycopg2.extras import execute_values
conn = psycopg2.connect(database="tsdb", 
                        host="host", 
                        user="tsdbadmin", 
                        password="passwd",
                        port="66666")
columns = ["time", "symbol", "price", "day_volume"]
current_batch = []
MAX_BATCH_SIZE = 100
def _on_event(self, event):
    if event["event"] == "price":
        # 数据记录
        timestamp = datetime.utcfromtimestamp(event["timestamp"])
        data = (timestamp, event["symbol"], event["price"], event.get("day_volume"))

        # 将新数据记录添加到批处理
        current_batch.append(data)
            
        # 摄入数据如果最大批处理大小已达到，则重置批处理
        if len(current_batch) == MAX_BATCH_SIZE:
            cursor = conn.cursor()
            sql = f"""
            INSERT INTO {DB_TABLE} ({','.join(columns)}) 
            VALUES %s;"""
            execute_values(cursor, sql, data)
            conn.commit()
            current_batch = []
```

确保您使用`execute_values()`或Psycopg2中允许一次事务插入多条记录的其他函数。

在您实现了`on_event`函数后，您的Python脚本可以连接到websocket服务器并实时摄入数据。

## 完整代码示例

清理过的Python脚本，打印出现批次大小，以便您可以看到数据何时从内存中被摄入到TimescaleDB中：

```python
from twelvedata import TDClient
import psycopg2
from psycopg2.extras import execute_values
from datetime import datetime

class WebsocketPipeline():
    # 超表的名称
    DB_TABLE = "prices_real_time"

    # 超表中按正确顺序的列
    DB_COLUMNS=["time", "symbol", "price", "day_volume"]

    # 用于批量插入数据的批处理大小
    MAX_BATCH_SIZE=100
    
    def __init__(self, conn):
        """连接到Twelve Data websocket服务器并流式传输
        数据到数据库。
        
        参数：
            conn：psycopg2连接对象
        """
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
        """每当从服务器返回新的数据记录时，此函数被调用。

        参数：
            event（字典）：数据记录
        """
        if event["event"] == "price":
            # 数据记录
            timestamp = datetime.utcfromtimestamp(event["timestamp"])
            data = (timestamp, event["symbol"], event["price"], event.get("day_volume"))

            # 将新数据记录添加到批处理
            self.current_batch.append(data)
            print(f"Current batch size: {len(self.current_batch)}")
            
            # 摄入数据如果最大批处理大小已达到，则重置批处理
            if len(self.current_batch) == self.MAX_BATCH_SIZE:
                self._insert_values(self.current_batch)
                self.insert_counter += 1
                print(f"Batch insert #{self.insert_counter}")
                self.current_batch = []
    

    def start(self, symbols):
        """连接到websocket服务器并开始流式传输实时数据
        到数据库。

        参数：
            symbols（符号列表）：股票/加密货币符号列表
        """
        td = TDClient(apikey="TWELVE_DATA_APIKEY")
        ws = td.websocket(on_event=self._on_event)
        ws.subscribe(symbols)
        ws.connect()
        ws.keep_alive()

conn = psycopg2.connect(database="tsdb", 
                        host="host", 
                        user="tsdbadmin", 
                        password="passwd",
                        port="66666")
    
symbols = ["BTC/USD", "ETH/USD", "MSFT", "AAPL"]
websocket = WebsocketPipeline(conn)
websocket.start(symbols=symbols)
```

运行脚本：

```python
python websocket_test.py
```

您甚至可以创建单独的Python脚本来启动多个websocket连接，用于不同类型的符号（例如，一个用于股票，另一个用于加密货币价格）。

如果您看到类似这样的错误消息：

```bash
2022-05-13 18:51:41,976 - ws-twelvedata - ERROR - TDWebSocket ERROR: Handshake status 200 OK
```

那么请检查您是否使用了从Twelve Data获得的正确API密钥。

继续进行我们的其他教程之一，了解如何在摄入后高效存储和分析您的数据：

*   [使用OHLCV（K线）格式在TimescaleDB中存储金融行情数据][candlestick-tutorial]
*   [TimescaleDB入门][get-started]

[candlestick-tutorial]: /tutorials/:currentVersion:/financial-tick-data/
[get-started]: /getting-started/:currentVersion:/
[install-ts]: /getting-started/latest/
[psycopg2]: https://www.psycopg.org/docs/
[twelve-data]: https://twelvedata.com
[twelve-signup]: https://twelvedata.com/pricing
[twelve-wrapper]: https://github.com/twelvedata/twelvedata-python
