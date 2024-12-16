---
标题: 用于 pg 向量和 pg 向量缩放的 Python 接口
摘要: 在 Python 中使用 pg 向量缩放和 pg 向量
产品: [云服务]
关键词: [人工智能，向量，pg 向量，Timescale 向量，pg 向量缩放，Python]
标签: [人工智能，向量，Python]
---

以下是您提供的Markdown格式文档的中文翻译，保留了所有格式和超链接：

# Python 接口 for pgvector 和 pgvectorscale

您使用 Timescale 上的 pgai 来驱动生产级别的人工智能应用。`timescale_vector` 是您用于与 Timescale 服务上的 pgai 进行程序化交互的 Python 接口。

在开始使用 `timescale_vector` 之前：

- [注册 Timescale 上的 pgai](https://console.cloud.timescale.com/signup?utm_campaign=vectorlaunch&utm_source=docs&utm_medium=direct)：在 Timescale 云数据平台上免费试用 90 天的 pgai。
- [遵循入门教程](https://timescale.github.io/python-vector/tsv_python_getting_started_tutorial.html)：
学习如何在真实世界的数据集上使用 Timescale 上的 pgai 进行语义搜索。

如果您更倾向于使用 LLM 开发或数据框架，请查看 pgai 与 [LangChain](https://python.langchain.com/docs/integrations/vectorstores/timescalevector) 和 [LlamaIndex](https://gpt-index.readthedocs.io/en/stable/examples/vector_stores/Timescalevector.html) 的集成。

## 前提条件

`timescale_vector` 依赖于 `psycopg2` 的源码分发，并遵循 [psycopg2 的最佳实践](https://www.psycopg.org/docs/install.html#psycopg-vs-psycopg-binary)。

在您安装 `timescale_vector` 之前：

* 遵循 [psycopg2 构建前提条件](https://www.psycopg.org/docs/install.html#build-prerequisites)。

## 安装

要使用Python与Timescale上的pgai进行交互：

1. 安装`timescale_vector`：

    ```bash
    pip install timescale_vector
    ```
2. 安装`dotenv`：

    ```bash
    pip install python-dotenv
    ```

    在这些示例中，您使用`dotenv`来传递密钥和键值。

    就是这样，您已经准备好开始了。

## timescale_vector库的基本使用

首先，导入所有必要的库：

```python
from dotenv import load_dotenv, find_dotenv
import os
from timescale_vector import client
import uuid
from datetime import datetime, timedelta
```

加载您的PostgreSQL凭证，最安全的方式是使用`.env`文件：

```python
_ = load_dotenv(find_dotenv(), override=True)
service_url  = os.environ['TIMESCALE_SERVICE_URL']
```

接下来，创建客户端。本教程使用同步客户端。但该库也提供了异步客户端（接口相同，使用的是异步函数）。

客户端构造函数需要三个必需的参数：

| 名称             | 描述                                                                                   |
|----------------|-------------------------------------------------------------------------------------------|
| `service_url`    | Timescale服务URL/连接字符串                                                                 |
| `table_name`     | 用于存储嵌入的表名。将其视为集合名称                                                         |
| `num_dimensions` | 向量中的维度数量                                                                           |

```python
vec  = client.Sync(service_url, "my_data", 2)
```

接下来，为集合创建表：

```python
vec.create_tables()
```
接下来，插入一些数据。数据记录包含：

- 一个UUID，用于唯一标识嵌入
- 关于嵌入的元数据的JSON blob
- 嵌入所代表的文本
- 嵌入本身

因为这些数据包括作为主键的UUIDs，所以应该使用upsert进行摄取。

```python
vec.upsert([\
    (uuid.uuid1(), {"animal": "fox"}, "the brown fox", [1.0,1.3]),\
    (uuid.uuid1(), {"animal": "fox", "action":"jump"}, "jumped over the", [1.0,10.8]),\
])
```

现在，您可以创建一个向量索引来加速相似性搜索：

```python
vec.create_embedding_index(client.TimescaleVectorIndex())
```

然后，您可以查询相似项：

```python
vec.search([1.0, 9.0])
```

    [[UUID('73d05df0-84c1-11ee-98da-6ee10b77fd08'),
      {'action': 'jump', 'animal': 'fox'},
      'jumped over the',
      array([ 1. , 10.8], dtype=float32),
      0.00016793422934946456],
     [UUID('73d05d6e-84c1-11ee-98da-6ee10b77fd08'),
      {'animal': 'fox'},
      'the brown fox',
      array([1. , 1.3], dtype=float32),
      0.14489260377438218]]

有许多搜索选项在下面的“高级搜索”部分有介绍。

下面是一个简单的搜索示例，它使用相似性搜索返回一个项目，并且受到元数据过滤器的限制：

```python
vec.search([1.0, 9.0], limit=1, filter={"action": "jump"})
```

    [[UUID('73d05df0-84c1-11ee-98da-6ee10b77fd08'),
      {'action': 'jump', 'animal': 'fox'},
      'jumped over the',
      array([ 1. , 10.8], dtype=float32),
      0.00016793422934946456]]

返回的记录包含5个字段：

| 名称        | 描述                                                    |
|------------|---------------------------------------------------------|
| id         | 记录的UUID                                              |
| metadata   | 与记录关联的JSON元数据                                    |
| contents   | 被嵌入的文本内容                                          |
| embedding  | 向量嵌入                                                |
| distance   | 查询嵌入与向量之间的距离                                   |

您可以通过简单地使用记录作为字典，用字段名称作为键来访问这些字段：

```python
records = vec.search([1.0, 9.0], limit=1, filter={"action": "jump"})
(records[0]["id"],records[0]["metadata"], records[0]["contents"], records[0]["embedding"], records[0]["distance"])
```

    (UUID('73d05df0-84c1-11ee-98da-6ee10b77fd08'),
     {'action': 'jump', 'animal': 'fox'},
     'jumped over the',
     array([ 1. , 10.8], dtype=float32),
     0.00016793422934946456)

您可以通过ID删除：

```python
vec.delete_by_ids([records[0]["id"]])
```

或者您可以通过元数据过滤器删除：

```python
vec.delete_by_metadata({"action": "jump"})
```

要删除所有记录，请使用：

```python
vec.delete_all()
```

## 高级使用

本节更详细地介绍了Python接口。它涵盖了：

1.  搜索过滤选项 - 如何通过额外的约束缩小您的搜索范围
2.  索引 - 如何加速您的相似性查询
3.  基于时间的分区 - 如何优化基于时间的相似性查询
4.  设置在距离计算中使用的不同距离类型

### 搜索选项

`search`函数非常灵活，允许您以多种方式搜索正确的向量。本节将搜索选项分为3部分进行描述：

1.  基本相似性搜索。
2.  根据关联的元数据过滤搜索。
3.  在启用时间分区时基于时间进行过滤。

以下示例基于此数据：

```python
vec.upsert([\
    (uuid.uuid1(), {"animal":"fox", "action": "sit", "times":1}, "the brown fox", [1.0,1.3]),\
    (uuid.uuid1(),  {"animal":"fox", "action": "jump", "times":100}, "jumped over the", [1.0,10.8]),\
])
```

基本查询如下：

```python
vec.search([1.0, 9.0])
```

    [[UUID('7487af96-84c1-11ee-98da-6ee10b77fd08'),
      {'times': 100, 'action': 'jump', 'animal': 'fox'},
      'jumped over the',
      array([ 1. , 10.8], dtype=float32),
      0.00016793422934946456],
     [UUID('7487af14-84c1-11ee-98da-6ee10b77fd08'),
      {'times': 1, 'action': 'sit', 'animal': 'fox'},
      'the brown fox',
      array([1. , 1.3], dtype=float32),
      0.14489260377438218]]

您可以为返回的项目数量提供限制：

```python
vec.search([1.0, 9.0], limit=1)
```

    [[UUID('7487af96-84c1-11ee-98da-6ee10b77fd08'),
      {'times': 100, 'action': 'jump', 'animal': 'fox'},
      'jumped over the',
      array([ 1. , 10.8], dtype=float32),
      0.00016793422934946456]]

#### 通过元数据缩小搜索范围

通过元数据过滤结果有两种主要方式：
- `filters`用于元数据的相等匹配。
- `predicates`用于元数据的复杂条件。

过滤器在表达能力上有所限制，但性能更好。如果使用场景允许，应该使用过滤器。

#### 使用过滤器进行相等匹配

您可以指定一个匹配元数据的字典，其中所有键必须匹配提供的值（不在过滤器中的键不受限制）：

```python
vec.search([1.0, 9.0], limit=1, filter={"action": "sit"})
```

    [[UUID('7487af14-84c1-11ee-98da-6ee10b77fd08'),
      {'times': 1, 'action': 'sit', 'animal': 'fox'},
      'the brown fox',
      array([1. , 1.3], dtype=float32),
      0.14489260377438218]]

您还可以指定一个过滤器字典列表，如果项目匹配任何一个字典，则返回该项目：

```python
vec.search([1.0, 9.0], limit=2, filter=[{"action": "jump"}, {"animal": "fox"}])
```

    [[UUID('7487af96-84c1-11ee-98da-6ee10b77fd08'),
      {'times': 100, 'action': 'jump', 'animal': 'fox'},
      'jumped over the',
      array([ 1. , 10.8], dtype=float32),
      0.00016793422934946456],
     [UUID('7487af14-84c1-11ee-98da-6ee10b77fd08'),
      {'times': 1, 'action': 'sit', 'animal': 'fox'},
      'the brown fox',
      array([1. , 1.3], dtype=float32),
      0.14489260377438218]]

#### 使用谓词进行更高级的元数据过滤

谓词允许更复杂的搜索条件。例如，您可以在数值上使用大于和小于条件。

```python
vec.search([1.0, 9.0], limit=2, predicates=client.Predicates("times", ">", 1))
```

    [[UUID('7487af96-84c1-11ee-98da-6ee10b77fd08'),
      {'times': 100, 'action': 'jump', 'animal': 'fox'},
      'jumped over the',
      array([ 1. , 10.8], dtype=float32),
      0.00016793422934946456]]

`Predicates`对象由元数据键的名称、操作符和值定义。

支持的操作符有：`==`, `!=`, `<`, `<=`, `>`, `>=`

值的类型决定了要执行的比较类型。例如，传入`"Sam"`（一个字符串）执行字符串比较，而传入`10`（一个整数）执行整数比较，传入`10.0`（一个浮点数）执行浮点数比较。需要注意的是，使用值`"10"`也会执行字符串比较，因此使用正确的类型非常重要。支持的Python类型有：`str`, `int`, 和 `float`。

再举一个字符串比较的例子：

```python
vec.search([1.0, 9.0], limit=2, predicates=client.Predicates("action", "==", "jump"))
```

    [[UUID('7487af96-84c1-11ee-98da-6ee10b77fd08'),
      {'times': 100, 'action': 'jump', 'animal': 'fox'},
      'jumped over the',
      array([ 1. , 10.8], dtype=float32),
      0.00016793422934946456]]

谓词的真正强大之处在于它们也可以使用`&`操作符（用于`AND`语义组合谓词）和`|`（用于`OR`语义组合）。所以您可以这样做：

```python
vec.search([1.0, 9.0], limit=2, predicates=client.Predicates("action", "==", "jump") & client.Predicates("times", ">", 1))
```

    [[UUID('7487af96-84c1-11ee-98da-6ee10b77fd08'),
      {'times': 100, 'action': 'jump', 'animal': 'fox'},
      'jumped over the',
      array([ 1. , 10.8], dtype=float32),
      0.00016793422934946456]]

仅供参考，下一个示例显示了由于谓词而没有返回结果的情况：

```python
vec.search([1.0, 9.0], limit=2, predicates=client.Predicates("action", "==", "jump") & client.Predicates("times", "==", 1))
```

    []

还有一个示例，其中谓词被定义为变量，并使用括号进行分组：

```python
my_predicates = client.Predicates("action", "==", "jump") & (client.Predicates("times", "==", 1) | client.Predicates("times", ">", 1))
vec.search([1.0, 9.0], limit=2, predicates=my_predicates)
```

    [[UUID('7487af96-84c1-11ee-98da-6ee10b77fd08'),
      {'times': 100, 'action': 'jump', 'animal': 'fox'},
      'jumped over the',
      array([ 1. , 10.8], dtype=float32),
      0.00016793422934946456]]

还有用于组合多个谓词的`AND`语义的语法糖。您可以向`Predicates`传入多个3元组：

```python
vec.search([1.0, 9.0], limit=2, predicates=client.Predicates(("action", "==", "jump"), ("times", ">", 10)))
```

    [[UUID('7487af96-84c1-11ee-98da-6ee10b77fd08'),
      {'times': 100, 'action': 'jump', 'animal': 'fox'},
      'jumped over the',
      array([ 1. , 10.8], dtype=float32),
      0.00016793422934946456]]

## 按时间过滤搜索

当使用`time-partitioning`（见下文）时，您可以非常高效地按时间过滤搜索。时间分区将基于UUID的ID中嵌入的时间戳与嵌入相关联。首先，创建一个具有时间分区的集合并插入一些数据（一个来自2018年1月的项目，另一个来自2019年1月）：

```python
tpvec = client.Sync(service_url, "time_partitioned_table", 2, time_partition_interval=timedelta(hours=6))
tpvec.create_tables()

specific_datetime = datetime(2018, 1, 1, 12, 0, 0)
tpvec.upsert([\
    (client.uuid_from_time(specific_datetime), {"animal":"fox", "action": "sit", "times":1}, "the brown fox", [1.0,1.3]),\
    (client.uuid_from_time(specific_datetime+timedelta(days=365)),  {"animal":"fox", "action": "jump", "times":100}, "jumped over the", [1.0,10.8]),\
])
```

然后，您可以通过指定`uuid_time_filter`来使用时间戳进行过滤：

```python
tpvec.search([1.0, 9.0], limit=4, uuid_time_filter=client.UUIDTimeRange(specific_datetime, specific_datetime+timedelta(days=1)))
```

    [[UUID('33c52800-ef15-11e7-be03-4f1f9a1bde5a'),
      {'times': 1, 'action': 'sit', 'animal': 'fox'},
      'the brown fox',
      array([1. , 1.3], dtype=float32),
      0.14489260377438218]]

[`UUIDTimeRange`](https://timescale.github.io/python-vector/vector.html#uuidtimerange) 
可以指定`start_date`或`end_date`或两者（如上例所示）。
仅指定`start_date`或`end_date`将使另一端不受限制。

```python
tpvec.search([1.0, 9.0], limit=4, uuid_time_filter=client.UUIDTimeRange(start_date=specific_datetime))
```

    [[UUID('ac8be800-0de6-11e9-889a-5eec84ba8a7b'),
      {'times': 100, 'action': 'jump', 'animal': 'fox'},
      'jumped over the',
      array([ 1. , 10.8], dtype=float32),
      0.00016793422934946456],
     [UUID('33c52800-ef15-11e7-be03-4f1f9a1bde5a'),
      {'times': 1, 'action': 'sit', 'animal': 'fox'},
      'the brown fox',
      array([1. , 1.3], dtype=float32),
      0.14489260377438218]]

您可以选择使用`start_inclusive`和`end_inclusive`参数来定义起始和结束日期是否包含在内。将`start_inclusive`设置为true将使用`>=`操作符进行比较，而将其设置为false则应用`>`操作符。默认情况下，起始日期是包含在内的，而结束日期是不包含的。
一个例子：

```python
tpvec.search([1.0, 9.0], limit=4, uuid_time_filter=client.UUIDTimeRange(start_date=specific_datetime, start_inclusive=False))
```

    [[UUID('ac8be800-0de6-11e9-889a-5eec84ba8a7b'),
      {'times': 100, 'action': 'jump', 'animal': 'fox'},
      'jumped over the',
      array([ 1. , 10.8], dtype=float32),
      0.00016793422934946456]]

请注意，当使用`start_inclusive=False`选项时，结果会有所不同，因为第一行具有由`start_date`指定的确切时间戳。

使用上述描述的`filter`和`predicates`参数，也可以很容易地集成时间过滤器，使用特殊的保留关键字名称，使其看起来时间戳是您的元数据的一部分。这在与其他系统集成时很有用，这些系统只想指定一组过滤器（通常这些是“自动检索器”类型的系统）。保留关键字名称是`__start_date`和`__end_date`用于过滤器，`__uuid_timestamp`用于谓词。下面有一些例子：

```python
tpvec.search([1.0, 9.0], limit=4, filter={ "__start_date": specific_datetime, "__end_date": specific_datetime+timedelta(days=1)})
```

    [[UUID('33c52800-ef15-11e7-be03-4f1f9a1bde5a'),
      {'times': 1, 'action': 'sit', 'animal': 'fox'},
      'the brown fox',
      array([1. , 1.3], dtype=float32),
      0.14489260377438218]]

```python
tpvec.search([1.0, 9.0], limit=4,
             predicates=client.Predicates("__uuid_timestamp", ">", specific_datetime) & client.Predicates("__uuid_timestamp", "<", specific_datetime+timedelta(days=1)))
```

    [[UUID('33c52800-ef15-11e7-be03-4f1f9a1bde5a'),
      {'times': 1, 'action': 'sit', 'animal': 'fox'},
      'the brown fox',
      array([1. , 1.3], dtype=float32),
      0.14489260377438218]]

### 索引

索引可以加速对数据的查询。默认情况下，系统会创建索引以通过UUID和元数据查询您的数据。

为了加速基于嵌入的相似性搜索，您需要创建额外的索引。

请注意，如果不使用索引进行查询，您总是得到确切的结果，但查询速度慢（它必须读取您存储的所有数据才能进行每次查询）。使用索引，您的查询速度会快几个数量级，但结果是近似的（因为目前没有已知的技术可以提供精确的索引）。

幸运的是，Timescale提供了3种优秀的近似索引算法：StreamingDiskANN、HNSW和ivfflat。

以下是这些算法之间的权衡：

| 算法          | 构建速度 | 查询速度 | 更新后需要重建 |
|--------------|---------|---------|-------------|
| StreamingDiskAnn | 快       | 最快     | 否           |
| HNSW        | 快     | 较快     | 否           |
| ivfflat      | 最快    | 最慢     | 是           |

您可以看到[基准测试](https://www.timescale.com/blog/how-we-made-postgresql-the-best-vector-database/)在博客上。

在大多数用例中，您应该使用StreamingDiskANN索引。这可以通过以下方式创建：

```python
vec.create_embedding_index(client.TimescaleVectorIndex())
```

索引是为特定的距离度量类型创建的。因此，在创建索引和查询期间在客户端设置相同的距离度量非常重要。详见下文的`距离类型`部分。

这些索引每个都有一个构建时选项集，用于控制创建索引时的速度/准确性权衡，以及一个额外的查询时选项，用于在特定查询期间控制准确性。库为所有这些选项使用了智能默认值。如何手动调整这些选项的详细信息如下。

<!-- vale Google.Headings = NO -->
#### StreamingDiskANN索引
<!-- vale Google.Headings = YES -->

StreamingDiskANN索引是一种基于图的算法，使用了[DiskANN](https://github.com/microsoft/DiskANN)算法。您可以在[博客](https://www.timescale.com/blog/how-we-made-postgresql-the-best-vector-database/)上了解更多关于其发布的消息。

要创建此索引，请运行：

```python
vec.create_embedding_index(client.TimescaleVectorIndex())
```

上述命令使用智能默认值创建索引。您可以调整许多参数来调整准确性/速度权衡。

您可以在索引构建时设置的参数有：

| 参数名称       | 描述                                                                                                                                                  | 默认值  |
|----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|---------|
| `num_neighbors` | 设置每个节点的最大邻居数。值越高，准确性增加，但图遍历会变慢。                                                                                              | 50      |
| `search_list_size` | 这是在构建期间使用的贪婪搜索算法中的S参数。值越高，图质量提高，但索引构建速度变慢。                                                                              | 100     |
| `max_alpha`     | 是算法中的alpha参数。值越高，图质量提高，但索引构建速度变慢。                                                                                                | 1.0     |

要设置这些参数，您可以运行：

```python
vec.create_embedding_index(client.TimescaleVectorIndex(num_neighbors=50, search_list_size=100, max_alpha=1.0))
```

您还可以在查询时设置一个参数来控制准确性与查询速度的权衡。该参数在`search()`函数中使用`query_params`参数设置。您可以设置`search_list_size`（默认值：100）。这是在查询时图搜索中考虑的额外候选数量。值越高，查询准确性提高，但查询速度变慢。

您可以在搜索期间如下指定此值：

```python
vec.search([1.0, 9.0], limit=4, query_params=TimescaleVectorIndexParams(search_list_size=10))
```

要删除索引，请运行：

```python
vec.drop_embedding_index()
```

#### pgvector HNSW索引

Pgvector提供了一个基于流行的[HNSW算法](https://arxiv.org/abs/1603.09320)的基于图的索引算法。

要创建此索引，请运行：

```python
vec.create_embedding_index(client.HNSWIndex())
```

上述命令使用智能默认值创建索引。您可以调整许多参数来调整准确性/速度权衡。

您可以在索引构建时设置的参数有：

| 参数名称       | 描述                                                                                                                                                                                                                                                            | 默认值  |
|----------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------|
| `m`            | 表示每层的最大连接数。可以将这些连接视为在图构建期间为每个节点创建的边。增加m可以提高准确性，但也会增加索引构建时间和大小。                                                                                                                                 | 16      |
| `ef_construction` | 表示构建图时动态候选列表的大小。它影响索引质量和构建速度之间的权衡。增加`ef_construction`可以在牺牲索引构建时间更长的代价下获得更准确的搜索结果。 | 64      |


为了设置这些参数，您可以运行：

```python
vec.create_embedding_index(client.HNSWIndex(m=16, ef_construction=64))
```

您还可以设置一个参数来控制查询时的准确性与查询速度的权衡。这个参数是在`search()`函数中使用`query_params`参数设置的。您可以设置`ef_search`（默认值：40）。这个参数指定了在搜索期间使用的动态候选列表的大小。值越高，查询准确性越好，但查询速度会变慢。

您可以在搜索时指定这个值，如下所示：

```python
vec.search([1.0, 9.0], limit=4, query_params=HNSWIndexParams(ef_search=10))
```

要删除索引，请运行：

```python
vec.drop_embedding_index()
```

#### pgvector ivfflat索引

Pgvector提供了一种基于聚类的索引算法。[博客文章](https://www.timescale.com/blog/nearest-neighbor-indexes-what-are-ivfflat-indexes-in-pgvector-and-how-do-they-work/)详细描述了它的工作原理。它提供了最快的索引构建速度，但查询速度是最慢的任何索引算法。

要创建这个索引，请运行：

```python
vec.create_embedding_index(client.IvfflatIndex())
```

注意：*ivfflat永远不应该在空表上创建*，因为它需要聚类数据，这只有在索引首次创建时才会发生，而不是在新行插入或修改时。此外，如果您的表经常进行修改，您需要定期重建此索引以保持良好的准确性。详见[博客文章](https://www.timescale.com/blog/nearest-neighbor-indexes-what-are-ivfflat-indexes-in-pgvector-and-how-do-they-work/)。

Pgvector ivfflat有一个`lists`索引参数，该参数会根据您的表中的行数自动设置一个智能的默认值。如果您知道您将有一个不同的表大小，您可以指定用于计算`lists`参数的记录数，如下所示：

```python
vec.create_embedding_index(client.IvfflatIndex(num_records=1000000))
```

您也可以直接设置`lists`参数：

```python
vec.create_embedding_index(client.IvfflatIndex(num_lists=100))
```

您还可以设置一个参数来控制查询时的准确性与查询速度的权衡。这个参数是在`search()`函数中使用`query_params`参数设置的。您可以设置`probes`。这个参数指定了在查询期间搜索的聚类数量。建议将此参数设置为`sqrt(lists)`，其中lists是上面在索引创建期间使用的`num_list`参数。值越高，查询准确性越好，但查询速度会变慢。

您可以在搜索时指定这个值，如下所示：

```python
vec.search([1.0, 9.0], limit=4, query_params=IvfflatIndexParams(probes=10))
```

要删除索引，请运行：

```python
vec.drop_embedding_index()
```

### 时间分区

在许多用例中，当您有许多嵌入时，时间是与嵌入相关联的重要组件。例如，在嵌入新闻故事时，您经常根据时间和相似性进行搜索（例如，过去一周与比特币相关的故事或2016年11月关于克林顿的故事）。

然而，传统上，根据两个组件“相似性”和“时间”进行搜索对于近似最近邻（ANN）索引来说是一个挑战，这使得相似性搜索索引的效果降低。

解决这个问题的一个方法是按时间分区数据，并在每个分区上单独创建ANN索引。然后，在搜索期间，您可以：

- 第1步：过滤不匹配时间谓词的分区。
- 第2步：在所有匹配的分区上执行相似性搜索。
- 第3步：合并第2步中每个分区的所有结果，重新排名，并按时间过滤结果。

第1步通过一次性过滤掉大量数据，使搜索更加高效。

Timescale-vector使用TimescaleDB的hypertables支持时间分区。要使用此功能，只需在创建客户端时指定每个分区的时间长度：

```python
from datetime import timedelta
from datetime import datetime
```

```python
vec = client.Async(service_url, "my_data_with_time_partition", 2, time_partition_interval=timedelta(hours=6))
await vec.create_tables()
```

然后，在插入数据时，ID使用UUID v1，UUID的时间组件指定了嵌入的时间。例如，要为当前时间创建一个嵌入，只需执行：

```python
id = uuid.uuid1()
await vec.upsert([(id, {"key": "val"}, "the brown fox", [1.0, 1.2])])
```

要为过去的特定时间插入数据，请使用`uuid_from_time`函数创建UUID

```python
specific_datetime = datetime(2018, 8, 10, 15, 30, 0)
await vec.upsert([(client.uuid_from_time(specific_datetime), {"key": "val"}, "the brown fox", [1.0, 1.2])])
```

然后您可以通过在搜索调用中指定`uuid_time_filter`来查询数据：

```python
rec = await vec.search([1.0, 2.0], limit=4, uuid_time_filter=client.UUIDTimeRange(specific_datetime-timedelta(days=7), specific_datetime+timedelta(days=7)))
```

### 距离度量

默认使用余弦距离来衡量一个嵌入与给定查询的相似度。除了余弦距离，还支持欧几里得/L2距离。距离类型是在创建客户端时使用`distance_type`参数设置的。例如，要使用欧几里得距离度量，您可以创建客户端：

```python
vec  = client.Sync(service_url, "my_data", 2, distance_type="euclidean")
```

`distance_type`的有效值是`cosine`和`euclidean`。

重要的是要注意，您应该在创建索引和执行查询的客户端上使用一致的距离类型。这是因为一个索引仅对一种特定的距离度量有效。

请注意，StreamingDiskANN索引目前仅支持余弦距离。

