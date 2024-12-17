---
标题: 非同质化代币（NFT）架构设计与数据摄取
摘要: 设计一种架构，用于摄取并存储 NFT 交易数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [加密货币，区块链，金融，分析]
标签: [NFT]
布局组件: [大尺寸的上一篇 / 下一篇按钮]
内容分组: 分析 NFT 数据
---

# NFT模式设计和数据摄入

一个设计得当的数据库模式对于高效存储和分析数据至关重要。本教程使用具有多个支持关系表的NFT时间序列数据。

为了帮助您熟悉NFT数据，以下是一些可以通过此数据集回答的问题：

*   哪些收藏品的交易量最高？
*   给定收藏品或资产的每日交易数量是多少？
*   哪些收藏品以太币（ETH）的交易量最大？
*   哪个账户进行了最多的NFT交易？
*   平均售价和中位数售价之间有何相关性？

所有这些问题的一个共同主题是，大多数洞察都与销售本身或销售聚合有关。因此，您需要创建一个专注于数据的时间序列方面的模式。确保可以连接支持表也很重要，这样您可以更容易地进行涉及时间序列和关系表的查询。TimescaleDB的PostgreSQL基础和全SQL支持使您能够在分析中轻松结合时间序列和关系表。

## 表和字段描述

您需要这些表：

TimescaleDB超表：

*   **nft_sales**：成功的NFT交易

关系表（常规PostgreSQL表）：

*   **assets**：独特的NFT项目
*   **collections**：NFT收藏品
*   **accounts**：NFT交易账户/用户

### nft_sales表

`nft_sales`表包含有关OpenSea平台上成功销售事件的时间序列形式的信息。一行代表一个成功的销售事件。

*   `id`字段是由OpenSea API提供的唯一字段。
*   `total_price`字段是以ETH（或OpenSea上可用的其他加密货币支付符号）支付给NFT的价格。
*   `quantity`字段表示交易中出售的NFT数量（可以超过1）。
*   `auction_type`字段默认为NULL，除非交易是作为拍卖的一部分发生的。
*   `asset_id`和`collection_id`字段可用于连接支持的关系表。

| 数据字段 | 描述 |
|---|---|
| id | OpenSea ID（唯一） |
| time | 销售时间 |
| asset_id | NFT的ID，外键：assets(id) |
| collection_id | 此NFT所属收藏品的ID，外键：collections(id) |
| auction_type | 拍卖类型（'dutch', 'english', 'min_price'） |
| contract_address | 智能合约地址 |
| quantity | 出售的NFT数量 |
| payment_symbol | 支付符号（通常是ETH，取决于NFT铸造的区块链） |
| total_price | 为NFT支付的总价格 |
| seller_account | 卖家账户，外键：accounts(id) |
| from_account | 转账使用的账户，外键：accounts(id) |
| to_account | 接收转账的账户，外键：accounts(id) |
| winner_account | 买家账户，外键：accounts(id) |

### assets表

`assets`表包含有关交易中涉及的资产（NFT）的信息。一行代表OpenSea平台上的一个独特NFT资产。

*   `name`字段是NFT的名称，不是唯一的。
*   `id`字段是主键，由OpenSea API提供。
*   一个资产可以从多个交易中引用（多次交易）。

| 数据字段 | 描述 |
|---|---|
| id | OpenSea ID（PK） |
| name | NFT的名称 |
| description | NFT的描述 |
| contract_date | 智能合约的创建日期 |
| url | NFT的OpenSea URL |
| owner_id | NFT所有者账户的ID，外键：accounts(id) |
| details | 其他额外的数据字段（JSONB） |

### collections表

`collections`表保存有关NFT收藏品的信息。一行代表一个独特的NFT收藏品。
一个收藏品包括多个独特的NFT（在`assets`表中）。

*   `slug`字段是收藏品的唯一标识符。

| 数据字段 | 描述 |
|---|---|
| id | 自增（PK） |
| slug | 收藏品的Slug（唯一） |
| name | 收藏品的名称 |
| url | 收藏品的OpenSea URL |
| details | 其他额外的数据字段（JSONB） |

### accounts表

`accounts`表包括至少参与了nft_sales表中一笔交易的账户。
一行代表OpenSea平台上的一个独特账户。

*   `address`从不为空且是唯一的
*   `user_name`除非用户在OpenSea个人资料中提交，否则为NULL

| 数据字段 | 描述 |
|---|---|
| id | 自增（PK） |
| user_name | OpenSea用户名 |
| address | 账户地址，唯一 |
| details | 其他额外的数据字段（JSONB） |

## 数据库模式

本教程中模式使用的数据类型是基于我们对OpenSea API的研究和实际操作经验确定的。首先运行这些SQL命令以创建模式。
或者，您可以从我们的[NFT入门套件GitHub仓库][nft-schema]下载并运行`schema.sql`文件。

```sql
CREATE TABLE collections (
   id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
   slug TEXT UNIQUE,
   name TEXT,
   url TEXT,
   details JSONB
);

CREATE TABLE accounts (
   id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
   user_name TEXT,
   address TEXT UNIQUE NOT NULL,
   details JSONB
);

CREATE TABLE assets (
   id BIGINT PRIMARY KEY,
   name TEXT,
   collection_id BIGINT REFERENCES collections (id), -- 收藏品
   description TEXT,
   contract_date TIMESTAMP WITH TIME ZONE,
   url TEXT UNIQUE,
   img_url TEXT,
   owner_id BIGINT REFERENCES accounts (id), -- 账户
   details JSONB
);

CREATE TYPE auction AS ENUM ('dutch', 'english', 'min_price');
CREATE TABLE nft_sales (
   id BIGINT,
   "time" TIMESTAMP WITH TIME ZONE,
   asset_id BIGINT REFERENCES assets (id), -- 资产
   collection_id BIGINT REFERENCES collections (id), -- 收藏品
   auction_type auction,
   contract_address TEXT,
   quantity NUMERIC,
   payment_symbol TEXT,
   total_price DOUBLE PRECISION,
   seller_account BIGINT REFERENCES accounts (id), -- 账户
   from_account BIGINT REFERENCES accounts (id), -- 账户
   to_account BIGINT REFERENCES accounts (id), -- 账户
   winner_account BIGINT REFERENCES accounts (id), -- 账户
   CONSTRAINT id_time_unique UNIQUE (id, time)
);

SELECT create_hypertable('nft_sales', 'time');

CREATE INDEX idx_asset_id ON nft_sales (asset_id);
CREATE INDEX idx_collection_id ON nft_sales (collection_id);
CREATE INDEX idx_payment_symbol ON nft_sales (payment_symbol);
```

### 模式设计

每个表中的`id`字段是`BIGINT`，因为在PostgreSQL中它的存储大小为8字节（与`INT`的4字节相比），这是必要的，以确保这个值不会溢出。

对于`quantity`字段，我们建议使用数值或十进制（在PostgreSQL中工作方式相同）作为数据类型，因为在一些边缘情况下，我们遇到的交易数量甚至对于BIGINT来说也太大了。

`total_price`需要是`double precision`，因为NFT价格通常包括许多小数位，特别是在以太币（ETH）和类似加密货币的情况下，它们在功能上是无限可分的。

我们为`auction_type`创建了一个`ENUM`，因为这个值只能是'dutch'、'english'或'min_price'，代表用于出售NFT的不同拍卖类型。

我们决定不存储OpenSea API提供的所有数据字段，只存储我们认为有趣或对未来分析有用的那些。但我们仍然想在附近保留所有未使用的数据字段，因此我们在每个关系表中添加了一个`details` JSONB列。这个列包含了关于记录的额外信息。例如，它包括了资产的`background_color`作为字段。

注意：在我们的样本数据集中，我们选择不包括JSONB数据，以保持数据集的易于管理。如果您想要包含完整JSON数据的数据集，您需要直接从OpenSea API获取数据（见下面的步骤）。

## 摄入NFT数据

当您创建了数据库和模式后，可以摄入一些数据来操作！您有两种方式来为本教程摄入NFT数据：

*   直接从OpenSea API获取数据
*   下载样本数据并导入

### 直接从OpenSea API获取数据

要从OpenSea API摄入数据，您可以使用入门套件仓库中包含的`opensea_ingest.py`脚本。
该脚本连接到OpenSea API `/events`端点，并从指定时间段获取数据。

<Highlight type="note">
您需要一个OpenSea API密钥来从OpenSea API获取数据。要请求您的密钥，参见[OpenSea API文档](https://docs.opensea.io/reference/request-an-api-key)。 
</Highlight>

<Highlight type="warning">
此程序依赖于OpenSea API。OpenSea API由OpenSea提供和维护。最近，API已经停止工作了一段时间。如果您尝试运行`opensea_ingest.py`脚本时API已更改或无法访问，请尝试下载历史数据文件并导入。您可以使用此数据文件完成教程。
</Highlight>

<Procedure>

### 直接从OpenSea API获取数据

1. 克隆
GitHub上的nft-starter-kit仓库：

    ```bash
    git clone https://github.com/timescale/nft-starter-kit.git 
    cd nft-starter-kit
    ```

1. 创建一个新的Python虚拟环境并安装依赖项：

    ```bash
    virtualenv env && source env/bin/activate
    pip install -r requirements.txt
    ```

1. 替换`config.py`文件中的参数：

    ```python
    DB_NAME="tsdb"
    HOST="YOUR_HOST_URL"
    USER="tsdbadmin"
    PASS="YOUR_PASSWORD_HERE"
    PORT="PORT_NUMBER"
    OPENSEA_START_DATE="2021-10-01T00:00:00" # 示例开始日期（UTC）
    OPENSEA_END_DATE="2021-10-06T23:59:59" # 示例结束日期（UTC）
    OPENSEA_APIKEY="YOUR_OPENSEA_APIKEY" # 需要从OpenSea的文档中请求
    ```

1. 运行Python脚本：

    ```python
    python opensea_ingest.py
    ```

    这将开始分批摄入数据，每次300行：

    ```bash
    Start ingesting data between 2021-10-01 00:00:00+00:00 and 2021-10-06 23:59:59+00:00
    ---
    Fetching transactions from OpenSea...
    Data loaded into temp table!
    Data ingested!
    Data has been backfilled until this time: 2021-10-06 23:51:31.140126+00:00
    ---
    ```

    您可以随时停止摄入过程（Ctrl+C），否则脚本会运行直到给定时间段内的所有交易都被摄入。

</Procedure>

### 下载样本NFT数据

您可以下载并插入包含2021年10月1日至2021年10月7日的NFT销售数据的样本CSV文件。

<Procedure>

### 下载样本NFT数据

1. 下载包含一周样本数据的[CSV文件][sample-data]。
1. 解压ZIP文件：

    ```bash
    unzip nft_sample.zip
    ```

1. 连接到您的数据库：

    ```bash
    psql -x "postgres://host:port/tsdb?sslmode=require"
    ```

    如果您使用的是Timescale，`How to Connect`下的说明提供了一个定制的命令来直接连接到您的数据库。
1. 按此顺序导入CSV文件（总共可能需要几分钟）：

    ```bash
    \copy accounts FROM 001_accounts.csv CSV HEADER;
    \copy collections FROM 002_collections.csv CSV HEADER;
    \copy assets FROM 003_assets.csv CSV HEADER;
    \copy nft_sales FROM 004_nft_sales.csv CSV HEADER;
    ```

</Procedure>

摄入NFT数据后，您可以尝试在数据库上运行一些查询：

```sql
SELECT count(*), MIN(time) AS min_date, MAX(time) AS max_date FROM nft_sales
```
[nft-schema]: https://github.com/timescale/nft-starter-kit/blob/master/schema.sql
[sample-data]: https://assets.timescale.com/docs/downloads/nft_sample.zip
