---
标题: 摄取实时金融网络套接字数据 - 设置数据集
摘要: 设置数据集，以便能够查询金融逐笔数据来分析价格变化。
产品: [云服务]
关键词: [金融，分析，网络套接字，数据管道]
标签: [教程，中级]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 摄取实时金融网络套接字数据
---

import CreateAndConnect from "versionContent/_partials/_cloud-create-connect-tutorials.mdx";
import CreateHypertable from "versionContent/_partials/_create-hypertable-twelvedata-stocks.mdx";
import CreateHypertableStocks from "versionContent/_partials/_create-hypertable-twelvedata-stocks.mdx";
import GrafanaConnect from "versionContent/_partials/_grafana-connect.mdx";

# 设置数据库

本教程使用的数据集包含了前100个最活跃交易符号的每秒股票交易数据，存储在一个名为 `stocks_real_time` 的超表中。它还包括一个单独的公司符号和公司名称表，存储在一个常规的 PostgreSQL 表中，名为 `company`。

<Collapsible heading="创建Timescale服务并连接到您的服务" defaultExpanded={false}>

<CreateAndConnect/>

</Collapsible>

<Collapsible heading="连接到websocket服务器" defaultExpanded={false}>

当您通过websocket连接到Twelve Data API时，您在计算机和websocket服务器之间创建了一个持久连接。
您设置了一个Python环境，并传递了两个参数来创建一个websocket对象并建立连接。

## 设置一个新的Python环境

为这个项目创建一个新的Python虚拟环境并激活它。完成本教程所需的所有包都安装在这个环境中。

<Procedure>

### 设置一个新的Python环境

1. 创建并激活一个Python虚拟环境：

    ```bash
    virtualenv env
    source env/bin/activate
    ```

1. 安装Twelve Data Python
    [wrapper library][twelve-wrapper]
    支持websocket。这个库允许您向API发送请求并保持一个稳定的websocket连接。

    ```bash
    pip install twelvedata websocket-client
    ```

1. 安装[Psycopg2][psycopg2]以便您可以从Python脚本连接TimescaleDB：

    ```bash
    pip install psycopg2-binary
    ```

</Procedure>

## 创建websocket连接

计算机和websocket服务器之间的持久连接用于只要连接保持就接收数据。您需要传递两个参数来创建一个websocket对象并建立连接。

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

1. 创建一个名为`websocket_test.py`的新Python文件，并使用`<YOUR_API_KEY>`连接到Twelve Data服务器：

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

1. 运行Python脚本：

    ```bash
    python websocket_test.py
    ```

1. 当您运行脚本时，您会从服务器接收到关于您连接状态的响应：

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

    当您与websocket服务器建立连接后，等待几秒钟，您可以看到像这样的数据记录：

    ```bash
    {'event': 'price', 'symbol': 'BTC/USD', 'currency_base': 'Bitcoin', 'currency_quote': 'US Dollar', 'exchange': 'Coinbase Pro', 'type': 'Digital Currency', 'timestamp': 1652438893, 'price': 30361.2, 'bid': 30361.2, 'ask': 30361.2, 'day_volume': 49153}
    {'event': 'price', 'symbol': 'BTC/USD', 'currency_base': 'Bitcoin', 'currency_quote': 'US Dollar', 'exchange': 'Coinbase Pro', 'type': 'Digital Currency', 'timestamp': 1652438896, 'price': 30380.6, 'bid': 30380.6, 'ask': 30380.6, 'day_volume': 49157}
    {'event': 'heartbeat', 'status': 'ok'}
    {'event': 'price', 'symbol': 'ETH/USD', 'currency_base': 'Ethereum', 'currency_quote': 'US Dollar', 'exchange': 'Huobi', 'type': 'Digital Currency', 'timestamp': 1652438899, 'price': 2089.07, 'bid': 2089.02, 'ask': 2089.03, 'day_volume': 193818}
    {'event': 'price', 'symbol': 'BTC/USD', 'currency_base': 'Bitcoin', 'currency_quote': 'US Dollar', 'exchange': 'Coinbase Pro', 'type': 'Digital Currency', 'timestamp': 1652438900, 'price': 30346.0, 'bid': 30346.0, 'ask': 30346.0, 'day_volume': 49167}
    ```

    每个价格事件都会给您提供关于给定交易对的多个数据点，例如交易所的名称和当前价格。您还可以偶尔在响应中看到`heartbeat`事件；这些事件随着时间的推移信号连接的健康状况。
    此时，websocket连接已成功地传递数据。

</Procedure>

</Collapsible>

<Collapsible heading="实时数据集" headingLevel={2} defaultExpanded={false}>

要将数据摄入到您的Timescale服务中，您需要实现`on_event`函数。

websocket连接设置完成后，您可以使用`on_event`函数将数据摄入到数据库中。这是一个数据管道，将实时金融数据摄入到您的Timescale服务中。

股票交易在周一至周五实时摄入，通常在纽约证券交易所的正常交易时间内（上午9:30至下午4:00 EST）。

<CreateHypertableStocks />

当您将数据摄入到像Timescale这样的事务数据库中时，批量插入数据比逐行插入数据更有效。使用一个事务插入多行可以显著提高您的Timescale数据库的整体摄入能力和速度。

## 内存中批处理

实现批处理的常见做法是先将新记录存储在内存中，然后当批处理达到一定大小时，一次性将所有记录从内存插入到数据库中。完美的批处理大小并不是通用的，但您可以尝试不同的批处理大小（例如，100、1000、10000等），看看哪个更适合您的用例。使用批处理是从一个Kafka、Kinesis或websocket连接摄入数据到TimescaleDB时的一个相当常见的模式。

您可以使用Psycopg2在Python中实现批处理解决方案。
您可以在`on_event`函数中实现摄入逻辑，然后将其传递给websocket对象。

这个函数需要：

1.  检查项目是否是数据项，而不是websocket元数据。
2.  调整数据，使其适应数据库模式，包括数据类型和列的顺序。
3.  将其添加到内存批处理中，这是Python中的一个列表。
4.  如果批处理达到一定大小，则插入数据，并重置或清空列表。

## 实时数据摄入

<Procedure>

1. 更新打印当前批处理大小的Python脚本，以便您可以跟踪数据何时从内存中摄入到数据库中。使用您想要摄入数据的Timescale服务的`<HOST>`、`<PASSWORD>`和`<PORT>`详细信息，以及来自Twelve Data的API密钥：

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
            """连接到Twelve Data websocket服务器并将数据流式传输到数据库。

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

                # 如果达到最大批处理大小，则摄入数据，然后重置批处理
                if len(self.current_batch) == self.MAX_BATCH_SIZE:
                    self._insert_values(self.current_batch)
                    self.insert_counter += 1
                    print(f"Batch insert #{self.insert_counter}")
                    self.current_batch = []
            def start(self, symbols):
                """连接到websocket服务器并开始将实时数据流式传输到数据库。

                参数：
                    symbols（符号列表）：股票/加密货币符号列表
                """
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

1. 运行脚本：

    ```bash
    python websocket_test.py
    ```

</Procedure>

您甚至可以创建单独的Python脚本来启动多个websocket连接，用于不同类型的符号，例如，一个用于股票，另一个用于加密货币价格。

### 故障排除

如果您看到类似这样的错误消息：

```bash
2022-05-13 18:51:41,976 - ws-twelvedata - ERROR - TDWebSocket ERROR: Handshake status 200 OK
```

那么请检查您是否使用了从Twelve Data获得的正确API密钥。

</Collapsible>

<Collapsible heading="连接到Grafana" defaultExpanded={false}>

本教程中的查询适合在Grafana中可视化。如果您想要可视化查询结果，请将您的Grafana账户连接到能耗数据集。

<GrafanaConnect />

</Collapsible>

[twelve-wrapper]: https://github.com/twelvedata/twelvedata-python
[psycopg2]: https://www.psycopg.org/docs/
