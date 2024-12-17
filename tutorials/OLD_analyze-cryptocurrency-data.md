---
标题: 分析加密货币市场数据
摘要: 使用 TimescaleDB 分析时间序列加密货币数据集。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [加密货币，金融，分析]
---


# 分析加密货币市场数据

本教程是一个逐步指南，介绍了如何使用TimescaleDB分析时间序列加密货币数据集。本教程中的指导用于创建[这个分析4100多个加密货币][crypto-blog]。

教程涵盖以下步骤：

1. 设计我们的数据库模式
2. 使用公开可用的加密货币定价数据创建数据集
3. 将数据集加载到TimescaleDB中
4. 在TimescaleDB中查询数据

如果您不想运行创建数据库模式或数据集的脚本，可以直接[跳转到TimescaleDB部分](#load-the-dataset-into-timescaledb)。

您还可以下载本教程的资源：

*   模式创建脚本：<Tag type="download" >[schema.sql](https://github.com/timescale/examples/blob/master/crypto_tutorial/schema.sql)</Tag> 
*   数据集创建脚本：<Tag type="download" >[crypto_data_extraction.py](https://github.com/timescale/examples/blob/master/crypto_tutorial/crypto_data_extraction.py)</Tag> 
*   数据集：<Tag type="download" >[Crypto Currency Dataset September 2019](https://github.com/timescale/examples/tree/master/crypto_tutorial/Cryptocurrency%20dataset%20Sept%2016%202019)</Tag>  （请注意，这些数据来自2019年9月。如果您需要最新数据，请按照本教程第2部分的步骤操作）

## 前提条件

要完成本教程，您需要对结构化查询语言（SQL）有初步了解。教程将指导您完成每个SQL命令，但如果您之前使用过SQL将会有所帮助。

首先，请[安装TimescaleDB][install-timescale]。安装完成后，您可以开始摄入或创建样本数据。

本教程直接引入第二个教程，涵盖如何将Timescale与Tableau结合使用来可视化时间序列数据。

## 设计数据库模式

当您有一个新数据库运行时，您需要一些数据插入其中。在获取分析数据之前，您需要定义您想要执行查询的数据类型。

在本分析中，我们有两个主要目标：

*   探索比特币和以太坊的价格，以不同法定货币计价，随时间变化。
*   探索不同加密货币的价格，以比特币计价，随时间变化。

您可能想要问的一些问题：

*   比特币的美元价格随时间如何变化？
*   以太坊的南非兰特价格随时间如何变化？
*   比特币的交易量在韩元计价随时间如何增加或减少？
*   过去两周交易量最大的加密货币是什么？
*   比特币最赚钱的是哪一天？
*   过去三个月最赚钱的新币种有哪些？

了解您想要询问的数据问题有助于指导您的模式定义。

这些需求导致我们设计了四个表。我们需要三个TimescaleDB超表，分别称为`btc_prices`、`crypto_prices`和`eth_prices`，以及一个关系表，称为`currency_info`。

`btc_prices`和`eth_prices`超表包含自2010年以来比特币在17种不同法定货币的价格数据。这是比特币表，但以太坊表非常相似：

|字段|描述|
|-|-|
|`time`|价格记录的特定日期时间戳，默认为00:00:00+00|
|`opening_price`|当天首次交易的硬币价格|
|`highest_price`|当天交易的最高价格|
|`lowest_price`|当天交易的最低价格|
|`closing_price`|当天最后交易的价格|
|`volume_btc`|当天以加密货币价值交换的量，以BTC计|
|`volume_currency`|当天转换为相应法定货币的交易量|
|`currency_code`|对应非BTC价格/交易量的法定货币|

最后，`currency_info`表将货币代码映射为其英文名称：

|字段|描述|
|-|-|
|`currency_code`|货币的2-7个字符缩写。在其他超表中使用|
|`currency`|货币的英文名称|

当您在数据库中为表建立了模式后，您可以制定`create_table` SQL语句来实际创建所需的表：

```sql
-- 加密货币分析的模式
DROP TABLE IF EXISTS "currency_info";
CREATE TABLE "currency_info"(
   currency_code   VARCHAR (10),
   currency        TEXT
);

-- btc_prices表的模式
DROP TABLE IF EXISTS "btc_prices";
CREATE TABLE "btc_prices"(
   time            TIMESTAMP WITH TIME ZONE NOT NULL,
   opening_price   DOUBLE PRECISION,
   highest_price   DOUBLE PRECISION,
   lowest_price    DOUBLE PRECISION,
   closing_price   DOUBLE PRECISION,
   volume_btc      DOUBLE PRECISION,
   volume_currency DOUBLE PRECISION,
   currency_code   VARCHAR (10)
);

-- crypto_prices表的模式
DROP TABLE IF EXISTS "crypto_prices";
CREATE TABLE "crypto_prices"(
   time            TIMESTAMP WITH TIME ZONE NOT NULL,
   opening_price   DOUBLE PRECISION,
   highest_price   DOUBLE PRECISION,
   lowest_price    DOUBLE PRECISION,
   closing_price   DOUBLE PRECISION,
   volume_crypto   DOUBLE PRECISION,
   volume_btc      DOUBLE PRECISION,
   currency_code   VARCHAR (10)
);

-- eth_prices表的模式
DROP TABLE IF EXISTS "eth_prices";
CREATE TABLE "eth_prices"(
   time            TIMESTAMP WITH TIME ZONE NOT NULL,
   opening_price   DOUBLE PRECISION,
   highest_price   DOUBLE PRECISION,
   lowest_price    DOUBLE PRECISION,
   closing_price   DOUBLE PRECISION,
   volume_eth      DOUBLE PRECISION,
   volume_currency DOUBLE PRECISION,
   currency_code   VARCHAR (10)
);

-- 创建超表以提高性能的Timescale特定语句
SELECT create_hypertable('btc_prices', 'time');
SELECT create_hypertable('eth_prices', 'time');
SELECT create_hypertable('crypto_prices', 'time');
```

请注意，有三个`create_hypertable`语句是TimescaleDB特定的语句。超表是跨时间间隔的单个连续表的抽象，因此您可以使用标准SQL对其进行查询。有关超表的更多信息，请参见[Timescale文档][hypertable-docs]和这篇[博客文章][hypertable-blog]。

## 创建待分析的数据集

现在您已经定义了想要的数据，可以构建包含这些数据的数据集。您可以编写一个小型Python脚本来从[CryptoCompare][cryptocompare]提取数据到四个CSV文件中，称为`coin_names.csv`、`crypto_prices.csv`、`btc_prices.csv`和`eth_prices.csv`。

要从CryptoCompare获取数据，您需要[获得一个API密钥][cryptocompare-apikey]。对于此分析，免费密钥就足够了。

脚本由五部分组成：

*   导入完成数据提取所需的Python库
*   用硬币名称列表填充`currency_info`表
*   获取历史上比特币（BTC）在4198种其他加密货币中的价格，并填充`crypto_prices`表
*   获取不同法定货币中的比特币历史价格以填充`btc_prices`
*   获取不同法定货币中的以太坊历史价格以填充`eth_prices`

以下是完整的Python脚本，您也可以<Tag type="download" >[下载](https://github.com/timescale/examples/blob/master/crypto_tutorial/crypto_data_extraction.py)</Tag> 

```python
#####################################################################
#1. 导入库并设置API密钥
#####################################################################
import requests
import json
import csv
from datetime import datetime

apikey = 'YOUR_CRYPTO_COMPARE_API_KEY'
#附加到URL字符串末尾
url_api_part = '&api_key=' + apikey

#####################################################################
#2. 填充所有硬币名称列表
#####################################################################
#从cryptocompare API获取硬币列表的URL
URLcoinslist = 'https://min-api.cryptocompare.com/data/all/coinlist&#39; 

#获取带有符号的加密货币列表
res1 = requests.get(URLcoinslist)
res1_json = res1.json()
data1 = res1_json['Data']
symbol_array = []
cryptoDict = dict(data1)

#写入CSV
with open('coin_names.csv', mode = 'w') as test_file:
   test_file_writer = csv.writer(test_file,
                                 delimiter = ',',
                                 quotechar = '"',
                                 quoting=csv.QUOTE_MINIMAL)
   for coin in cryptoDict.values():
     if day.get('time') == None:
       continue #跳过此项
       name = coin['Name']
       symbol = coin['Symbol']
       symbol_array.append(symbol)
       coin_name = coin['CoinName']
       full_name = coin['FullName']
       entry = [symbol, coin_name]
       test_file_writer.writerow(entry)
print('Done getting crypto names and symbols. See coin_names.csv for result')

#####################################################################
#3. 填充每种加密货币在BTC中的历史价格
#####################################################################
#注意：由于我们要为4000多个硬币填充数据，此部分可能需要一段时间才能运行
#进度计数器
progress = 0
num_cryptos = str(len(symbol_array))
for symbol in symbol_array:
   # 获取该货币的数据
   URL = 'https://min-api.cryptocompare.com/data/histoday?fsym=&#39;  +
         symbol +
         '&tsym=BTC&allData=true' +
         url_api_part
   res = requests.get(URL)
   res_json = res.json()
   data = res_json['Data']
   # 将所需字段写入csv
   with open('crypto_prices.csv', mode = 'a') as test_file:
       test_file_writer = csv.writer(test_file,
                                     delimiter = ',',
                                     quotechar = '"',
                                     quoting=csv.QUOTE_MINIMAL)
       for day in data:
           rawts = day['time']
           ts = datetime.utcfromtimestamp(rawts).strftime('%Y-%m-%d %H:%M:%S')
           o = day['open']
           h = day['high']
           l = day['low']
           c = day['close']
           vfrom = day['volumefrom']
           vto = day['volumeto']
           entry = [ts, o, h, l, c, vfrom, vto, symbol]
           test_file_writer.writerow(entry)
   progress = progress + 1
   print('Processed ' + str(symbol))
   print(str(progress) + ' currencies out of ' +  num_cryptos + ' written to csv')
print('Done getting price data for all coins. See crypto_prices.csv for result')
```

[crypto-blog]: https://blog.timescale.com/blog/cryptocurrency-time-series-analysis-timescaledb/
[install-timescale]: https://www.timescale.com/download/
[hypertable-docs]: https://docs.timescale.com/latest/how-to-build/hypertables/
[hypertable-blog]: https://blog.timescale.com/blog/introducing-hypertables/
[cryptocompare]: https://www.cryptocompare.com/
[cryptocompare-apikey]: https://min-api.cryptocompare.com/pricing/
```

#####################################################################
#4. 填充不同法定货币中的BTC价格
#####################################################################
# 我们想要查询的法定货币列表
# 你可以扩展这个列表，但CryptoCompare网站上没有
# 一个全面的法定货币列表
fiatList = ['AUD', 'CAD', 'CNY', 'EUR', 'GBP', 'GOLD', 'HKD',
'ILS', 'INR', 'JPY', 'KRW', 'PLN', 'RUB', 'SGD', 'UAH', 'USD', 'ZAR']

# 进度计数器
progress2 = 0
for fiat in fiatList:
   # 获取该法定货币中比特币的价格数据
   URL = 'https://min-api.cryptocompare.com/data/histoday?fsym=BTC&tsym=&#39;  +
         fiat +
         '&allData=true' +
         url_api_part
   res = requests.get(URL)
   res_json = res.json()
   data = res_json['Data']
   # 将所需字段写入csv
   with open('btc_prices.csv', mode = 'a') as test_file:
       test_file_writer = csv.writer(test_file,
                                     delimiter = ',',
                                     quotechar = '"',
                                     quoting=csv.QUOTE_MINIMAL)
       for day in data:
           rawts = day['time']
           ts = datetime.utcfromtimestamp(rawts).strftime('%Y-%m-%d %H:%M:%S')
           o = day['open']
           h = day['high']
           l = day['low']
           c = day['close']
           vfrom = day['volumefrom']
           vto = day['volumeto']
           entry = [ts, o, h, l, c, vfrom, vto, fiat]
           test_file_writer.writerow(entry)
   progress2 = progress2 + 1
   print('processed ' + str(fiat))
   print(str(progress2) + ' currencies out of  17 written')
print('Done getting price data for btc. See btc_prices.csv for result')

#####################################################################
#5. 填充不同法定货币中的ETH价格
#####################################################################
# 进度计数器
progress3 = 0
for fiat in fiatList:
   # 获取该法定货币中以太坊的价格数据
   URL = 'https://min-api.cryptocompare.com/data/histoday?fsym=ETH&tsym=&#39;  +
         fiat +
         '&allData=true' +
         url_api_part
   res = requests.get(URL)
   res_json = res.json()
   data = res_json['Data']
   # 将所需字段写入csv
   with open('eth_prices.csv', mode = 'a') as test_file:
       test_file_writer = csv.writer(test_file,
                                     delimiter = ',',
                                     quotechar = '"',
                                     quoting=csv.QUOTE_MINIMAL)
       for day in data:
           rawts = day['time']
           ts = datetime.utcfromtimestamp(rawts).strftime('%Y-%m-%d %H:%M:%S')
           o = day['open']
           h = day['high']
           l = day['low']
           c = day['close']
           vfrom = day['volumefrom']
           vto = day['volumeto']
           entry = [ts, o, h, l, c, vfrom, vto, fiat]
           test_file_writer.writerow(entry)
   progress3 = progress3 + 1
   print('processed ' + str(fiat))
   print(str(progress3) + ' currencies out of  17 written')
print('Done getting price data for eth. See eth_prices.csv for result')
```

运行脚本后，您应该得到四个.csv文件：

```bash
python crypto_data_extraction.py
```

## 将数据集加载到TimescaleDB中

开始之前，您需要一个[TimescaleDB的运行安装][install-timescale]。

### 设置模式

现在您在开始时的所有辛苦工作将派上用场，您可以使用您创建的SQL脚本来设置TimescaleDB实例。如果您不想自己输入SQL脚本，您也可以下载<Tag type="download">[schema.sql](https://github.com/timescale/examples/blob/master/crypto_tutorial/schema.sql)</Tag>。

登录到TimescaleDB实例。找到您的`host`、`port`和`password`，然后连接到数据库：

```bash
psql -x "postgres://tsdbadmin:{YOUR_PASSWORD_HERE}@{YOUR_HOSTNAME_HERE}:{YOUR_PORT_HERE}/defaultdb?sslmode=require"
```

在`psql`命令行中，创建一个数据库。我们称之为`crypto_data`：

```sql
CREATE DATABASE crypto_data;
\c crypto_data
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;
```

从命令提示符，您可以像这样将模式创建脚本应用到数据库：

```bash
psql -x "postgres://tsdbadmin:{YOUR_PASSWORD_HERE}@{|YOUR_HOSTNAME_HERE}:{YOUR_PORT_HERE}/crypto_data?sslmode=require" < schema.sql
```

输出应该看起来像这样：

```sql
NOTICE:  00000: table "currency_info" does not exist, skipping
LOCATION:  DropErrorMsgNonExistent, tablecmds.c:1057
DROP TABLE
Time: 78.384 ms
CREATE TABLE
Time: 87.011 ms
NOTICE:  00000: table "btc_prices" does not exist, skipping
LOCATION:  DropErrorMsgNonExistent, tablecmds.c:1057
DROP TABLE
Time: 77.094 ms
CREATE TABLE
Time: 79.815 ms
NOTICE:  00000: table "crypto_prices" does not exist, skipping
LOCATION:  DropErrorMsgNonExistent, tablecmds.c:1057
DROP TABLE
Time: 78.430 ms
CREATE TABLE
Time: 78.430 ms
NOTICE:  00000: table "eth_prices" does not exist, skipping
LOCATION:  DropErrorMsgNonExistent, tablecmds.c:1057
DROP TABLE
Time: 77.410 ms
CREATE TABLE
Time: 80.883 ms
    create_hypertable
-------------------------
 (1,public,btc_prices,t)
(1 row)

Time: 83.154 ms
    create_hypertable
-------------------------
 (2,public,eth_prices,t)
(1 row)

Time: 84.650 ms
     create_hypertable
----------------------------
 (3,public,crypto_prices,t)
(1 row)

Time: 81.864 ms
```

现在当您使用`psql`重新登录到TimescaleDB实例时，您可以运行`\dt`命令并看到表已正确创建：

```sql
             List of relations
 Schema |     Name      | Type  |   Owner
--------+---------------+-------+-----------
 public | btc_prices    | table | tsdbadmin
 public | crypto_prices | table | tsdbadmin
 public | currency_info | table | tsdbadmin
 public | eth_prices    | table | tsdbadmin
(4 rows)
```

### 摄入数据

现在您已经用期望的模式创建了表，剩下的就是将您创建的.csv文件中的数据插入到表中。

确保您使用`psql`登录到TimescaleDB，以便您可以依次运行以下每个命令：

```sql
\COPY btc_prices FROM 'btc_prices.csv' CSV;
\COPY eth_prices FROM 'eth_prices.csv' CSV;
\COPY crypto_prices FROM 'crypto_prices.csv' CSV;
\COPY currency_info FROM 'coin_names.csv' CSV;
```

<Highlight type="important">
数据摄入可能需要一段时间，这取决于您的互联网连接速度。
</Highlight>

您可以通过运行一个简单的SQL命令来验证摄入是否成功，例如：

```sql
SELECT * FROM btc_prices LIMIT 5;
```

您应该得到类似这样的输出：

```sql
-[ RECORD 1 ]---+-----------------------
time            | 2013-03-11 00:00:00+00
opening_price   | 60.56
highest_price   | 60.56
lowest_price    | 60.56
closing_price   | 60.56
volume_btc      | 0.1981
volume_currency | 12
currency_code   | AUD
-[ RECORD 2 ]---+-----------------------
time            | 2013-03-12 00:00:00+00
opening_price   | 60.56
highest_price   | 60.56
lowest_price    | 41.38
closing_price   | 47.78
volume_btc      | 47.11
volume_currency | 2297.5
currency_code   | AUD
-[ RECORD 3 ]---+-----------------------
time            | 2013-03-07 00:00:00+00
opening_price   | 181.15
highest_price   | 273.5
lowest_price    | 237.4
closing_price   | 262.87
volume_btc      | 33.04
volume_currency | 8974.45
currency_code   | CNY

-[ RECORD 4 ]---+-----------------------
time            | 2013-03-07 00:00:00+00
opening_price   | 32.31
highest_price   | 35.03
lowest_price    | 26
closing_price   | 31.57
volume_btc      | 13321.61
volume_currency | 425824.38
currency_code   | EUR
-[ RECORD 5 ]---+-----------------------
time            | 2013-03-11 00:00:00+00
opening_price   | 35.7
highest_price   | 37.35
lowest_price    | 35.4
closing_price   | 37.15
volume_btc      | 3316.09
volume_currency | 121750.98
currency_code   | EUR

Time: 224.741 ms
```

## 查询和分析数据

在教程开始时，我们定义了一些要回答的问题。
自然，这些问题的每个答案都以SQL查询的形式存在。现在您的数据库已正确设置，数据已捕获并摄入，您可以找到一些答案：

例如，**比特币的美元价格随时间如何变化？**

```sql
SELECT time_bucket('7 days', time) AS period,
      last(closing_price, time) AS last_closing_price
FROM btc_prices
WHERE currency_code = 'USD'
GROUP BY period
ORDER BY period
```

**BTC的日回报随时间如何变化？哪些天的回报最差和最好？**

```sql
SELECT time,
      closing_price / lead(closing_price) over prices AS daily_factor
FROM (
  SELECT time,
         closing_price
  FROM btc_prices
  WHERE currency_code = 'USD'
  GROUP BY 1,2
) sub window prices AS (ORDER BY time DESC)
```

**比特币在不同法定货币中的交易量随时间如何变化？**

```sql
SELECT time_bucket('7 days', time) AS period,
      currency_code,
      sum(volume_btc)
FROM btc_prices
GROUP BY currency_code, period
ORDER BY period
```

**以太坊（ETH）的BTC价格随时间如何变化？**

```sql
SELECT
   time_bucket('7 days', time) AS time_period,
   last(closing_price, time) AS closing_price_btc
FROM crypto_prices
WHERE currency_code='ETH'
GROUP BY time_period
ORDER BY time_period
```

**不同法定货币中的ETH价格随时间如何变化？**

```sql
SELECT time_bucket('7 days', c.time) AS time_period,
      last(c.closing_price, c.time) AS last_closing_price_in_btc,
      last(c.closing_price, c.time) * last(b.closing_price, c.time) FILTER (WHERE b.currency_code = 'USD') AS last_closing_price_in_usd,
      last(c.closing_price, c.time) * last(b.closing_price, c.time) FILTER (WHERE b.currency_code = 'EUR') AS last_closing_price_in_eur,
      last(c.closing_price, c.time) * last(b.closing_price, c.time) FILTER (WHERE b.currency_code = 'CNY') AS last_closing_price_in_cny,
      last(c.closing_price, c.time) * last(b.closing_price, c.time) FILTER (WHERE b.currency_code = 'JPY') AS last_closing_price_in_jpy,
      last(c.closing_price, c.time) * last(b.closing_price, c.time) FILTER (WHERE b.currency_code = 'KRW') AS last_closing_price_in_krw
FROM crypto_prices c
JOIN btc_prices b
   ON time_bucket('1 day', c.time) = time_bucket('1 day', b.time)
WHERE c.currency_code = 'ETH'
GROUP BY time_period
ORDER BY time_period
```

**过去14天哪些加密货币的交易量最大？**

```sql
SELECT 'BTC' AS currency_code,
       sum(b.volume_currency) AS total_volume_in_usd
FROM btc_prices b
WHERE b.currency_code = 'USD'
AND now() - date(b.time) < INTERVAL '14 day'
GROUP BY b.currency_code
UNION
SELECT c.currency_code AS currency_code,
       sum(c.volume_btc) * avg(b.closing_price) AS total_volume_in_usd
FROM crypto_prices c JOIN btc_prices b ON date(c.time) = date(b.time)
WHERE c.volume_btc > 0
AND b.currency_code = 'USD'
AND now() - date(b.time) < INTERVAL '14 day'
AND now() - date(c.time) < INTERVAL '14 day'
GROUP BY c.currency_code
ORDER BY total_volume_in_usd DESC
```

**哪些加密货币的日回报最高？**

```sql
WITH
   prev_day_closing AS (
SELECT
   currency_code,
   time,
   closing_price,
   LEAD(closing_price) OVER (PARTITION BY currency_code ORDER BY TIME DESC) AS prev_day_closing_price
FROM
    crypto_prices
)
,    daily_factor AS (
SELECT
   currency_code,
   time,
   CASE WHEN prev_day_closing_price = 0 THEN 0 ELSE closing_price/prev_day_closing_price END AS daily_factor
FROM
   prev_day_closing
)
SELECT
   time,
   LAST(currency_code, daily_factor) AS currency_code,
   MAX(daily_factor) AS max_daily_factor
FROM
   daily_factor
GROUP BY
   time
```

[crypto-blog]: https://blog.timescale.com/blog/analyzing-bitcoin-ethereum-and-4100-other-cryptocurrencies-using-postgresql-and-timescaledb/ 
[cryptocompare-apikey]: https://min-api.cryptocompare.com 
[cryptocompare]: https://www.cryptocompare.com 
[hypertable-blog]: https://blog.timescale.com/blog/when-boring-is-awesome-building-a-scalable-time-series-database-on-postgresql-2900ea453ee2/
[hypertable-docs]: /use-timescale/:currentVersion:/hypertables
[install-timescale]: /getting-started/latest/
