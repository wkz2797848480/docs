---
标题: 为您的AI应用提供动力  
摘要: Timescale Vector和向量的一般描述  
产品: [云]  
关键字: [ai, 向量, pgvector, pgvectorscale, pgai]  
标签: [ai, 向量]
---

# 用pgai在Timescale上为您的AI应用提供动力

Pgai on Timescale 是一个基于云的解决方案，用于构建使用 PostgreSQL 的搜索、RAG（检索式生成）和 AI 代理。这套工具使您能够部署生产级别的 AI 应用程序，使用 PostgreSQL 作为您的向量数据库，存储向量嵌入、关系数据（例如，相关的元数据）以及基于时间的数据在同一数据库中。

<Highlight type="cloud" header="Start building today" button="Try for free">
Pgai 在 Timescale Cloud 中由三个扩展组成：pgvector、pgvectorscale 和 pgai。pgvector 提供向量数据类型和 HNSW 搜索索引。Pgvectorscale 提供 StreamingDiskANN 索引，以增强嵌入搜索的性能并使向量查询高效。Pgai 允许你直接在数据库内部轻松调用 AI 嵌入和生成模型。这三个扩展默认情况下都会安装在你的 Timescale Cloud 实例中。
</Highlight>
<!-- vale Google.Headings = NO -->

## pgvectorscale ❤️ pgvector

<!-- vale Google.Headings = Yes -->
Pgvector 是 PostgreSQL 中用于向量存储和相似性搜索的流行开源扩展，而 pgvectorscale 为 pgvector 添加了高级索引功能。Pgai on Timescale 提供了这两个扩展，因此您可以使用 pgvector 中已有的所有功能（如 HNSW 和 ivfflat 索引），并且还可以利用 pgvectorscale 中的 StreamingDiskANN 索引来加速向量搜索。

这使得您可以轻松迁移现有的 pgvector 部署，并利用 pgvectorscale 中的额外性能特性。您还可以灵活地创建适合您需求的不同索引类型。有关更多信息，请参见向量搜索索引部分。

## 什么是向量嵌入？

嵌入提供了一种表示数据语义本质的方法，并允许根据数据在意义上的关联程度进行比较。在数据库的背景下，这极其强大：可以想象成是增强版的全文搜索。向量数据库允许存储与数据相关的嵌入，然后搜索与给定查询相似的嵌入。

## 向量嵌入的应用

有许多应用场景中，向量嵌入可以发挥重要作用。

### 语义搜索

通过创建能够理解查询意图和上下文含义的系统，超越传统基于关键词的搜索方法的限制，从而返回更相关的结果。语义搜索不仅仅寻找精确的词语匹配；它把握用户查询背后的深层意图。结果呢？即使搜索词的措辞不同，也能浮现出相关的结果。利用混合搜索的优势，结合词汇和语义搜索方法，为用户提供既丰富又准确的搜索体验。这不仅仅是寻找直接匹配的问题；而是要挖掘上下文和概念上相似的内容，以满足用户需求。

### 推荐系统

想象一个用户对某个特定主题的几篇文章表现出了兴趣。借助嵌入技术，推荐引擎可以深入挖掘这些文章的语义本质，找出数据库中与同一主题共鸣的其他项目。因此，推荐不仅仅停留在标签或类别等表面层次，而是深入内容的核心。

### 检索增强生成(RAG)

通过为大型语言模型（LLMs）如OpenAI的GPT-4、Anthropic的Claude 2以及开源模型如Llama 2提供额外的上下文，可以增强生成性人工智能。当用户提出查询时，会检索相关的数据库内容，并将其用作LLM的额外信息来补充查询。这有助于减少LLM的幻觉现象，因为它确保模型的输出更加基于具体和相关的信息，即使这些信息不是模型原始训练数据的一部分。

### 聚类

嵌入还为聚类数据提供了强大的解决方案。将数据转换为这些向量化形式允许在高维空间中对数据点进行细微的比较通过像K-means或层次聚类这样的算法，数据可以被归类到语义类别中，提供表面属性可能遗漏的洞察。这揭示了固有的数据模式，丰富了探索和决策过程。

## 向量相似性搜索：它是如何工作的

在高层次上，嵌入帮助数据库寻找与给定信息相似的数据（相似性搜索）。这个过程包括几个步骤：

- 首先，为数据创建嵌入并将其插入数据库。这可以在应用程序中进行，也可以在数据库本身中进行。
- 其次，当用户有一个搜索查询（例如，聊天中的问题），该查询随后被转换为嵌入。
- 第三，数据库获取查询嵌入并搜索它存储的最接近匹配（最相似）的嵌入。

在底层，嵌入被表示为一个向量（一系列数字），捕捉数据的本质。为了确定两个数据点之间的相似性，数据库使用向量的数学运算来获得距离度量（通常是欧几里得距离或余弦距离）。在搜索过程中，数据库应该返回那些存储项，其中查询嵌入和存储嵌入之间的距离尽可能小，表明这些项最相似。

## 嵌入模型

pgai on Timescale 与输出向量为2,000维或更少的最受欢迎的嵌入模型一起工作：

- [OpenAI 嵌入模型](https://platform.openai.com/docs/guides/embeddings/)：text-embedding-ada-002 是 OpenAI 推荐的嵌入生成模型。
- [Cohere 表示模型](https://docs.cohere.com/docs/models#representation)：Cohere 提供了许多模型，可以用来从英文或多种语言的文本中生成嵌入。

以下是一些流行的图像嵌入选择：

- [OpenAI CLIP](https://github.com/openai/CLIP): 适用于涉及文本和图像的应用
- [VGG](https://pytorch.org/vision/stable/models/vgg.html)
- [视觉变换器（ViT）](https://github.com/lukemelas/PyTorch-Pretrained-ViT)

[向量搜索索引]: /ai/:currentVersion:/key-vector-database-concepts-for-understanding-pgvector/#vector-search-indexing-approximate-nearest-neighbor-search