## ES基础概念

### 什么是ES

此处我们引用ES官网上的介绍

[elastic官网](https://www.elastic.co/cn/what-is/elasticsearch)

Elasticsearch 是一个**分布式**的**免费开源搜索和分析引擎**，适用于包括文本、数字、地理空间、结构化和非结构化数据等在内的所有类型的数据。Elasticsearch 在 **Apache Lucene** 的基础上开发而成，由 Elasticsearch N.V.（即现在的 Elastic）于 2010 年首次发布。Elasticsearch 以其简单的 **REST 风格 API**、**分布式特性**、**速度和可扩展性**而闻名，是 Elastic Stack 的核心组件；Elastic Stack 是一套适用于**数据采集、扩充、存储、分析和可视化**的免费开源工具。人们通常将 Elastic Stack 称为 ELK Stack（代指 Elasticsearch、Logstash 和 Kibana），目前 Elastic Stack 包括一系列丰富的轻量型数据采集代理，这些代理统称为 Beats，可用来向 Elasticsearch 发送数据。

### Elasticsearch 的用途是什么？

Elasticsearch 在速度和可扩展性方面都表现出色，而且还能够索引多种类型的内容，这意味着其可用于多种用例：

- **应用程序搜索**
- 网站搜索
- 企业搜索
- **日志处理和分析**
- **基础设施指标和容器监测**
- **应用程序性能监测**
- 地理空间数据分析和可视化
- 安全分析
- 业务分析

### Elasticsearch 的工作原理是什么？

原始数据会从**多个来源**（包括日志、系统指标和网络应用程序）输入到 Elasticsearch 中。*数据采集*指在 Elasticsearch 中进行*索引*之前解析、标准化并充实这些原始数据的过程。这些数据在 Elasticsearch 中索引完成之后，用户便可针对他们的数据运行复杂的查询，并使用聚合来检索自身数据的复杂汇总。在 Kibana 中，用户可以基于自己的数据创建强大的可视化，分享仪表板，并对 Elastic Stack 进行管理。

### Elasticsearch 索引是什么？

Elasticsearch 索引index ——指相互关联的文档集合。Elasticsearch 会以 JSON 文档的形式存储数据。每个文档都会在一组*键*（字段或属性的名称）和它们对应的值（字符串、数字、布尔值、日期、*数值*组、地理位置或其他类型的数据）之间建立联系。

Elasticsearch 使用的是一种名为***倒排索引***的数据结构，这一结构的设计可以允许十分快速地进行全文本搜索。倒排索引会列出在所有文档中出现的每个特有词汇，并且可以找到包含每个词汇的全部文档。

在索引过程中，Elasticsearch 会存储文档并构建倒排索引，这样用户便可以**近实时**地对文档数据进行搜索。索引过程是在索引 API 中启动的，通过此 API 您既可向特定索引中添加 JSON 文档，也可更改特定索引中的 JSON 文档。

### 为何使用 Elasticsearch？

**Elasticsearch 很快。**由于 Elasticsearch 是在 Lucene 基础上构建而成的，所以在全文本搜索方面表现十分出色。Elasticsearch 同时还是一个近实时的搜索平台，这意味着从文档索引操作到文档变为可搜索状态之间的延时很短，一般只有一秒。因此，Elasticsearch 非常适用于对时间有严苛要求的用例，例如安全分析和基础设施监测。

**Elasticsearch 具有分布式的本质特征。**Elasticsearch 中存储的文档分布在不同的容器中，这些容器称为*分片*，可以进行复制以提供数据冗余副本，以防发生硬件故障。Elasticsearch 的分布式特性使得它可以扩展至数百台（甚至数千台）服务器，并**处理 PB 量级**的数据。

**Elasticsearch 包含一系列广泛的功能。**除了速度、可扩展性和弹性等优势以外，Elasticsearch 还有大量强大的内置功能（例如数据汇总和索引生命周期管理），可以方便用户更加高效地存储和搜索数据。

**Elastic Stack 简化了数据采集、可视化和报告过程。**通过与 Beats 和 Logstash 进行集成，用户能够在向 Elasticsearch 中索引数据之前轻松地处理数据。同时，Kibana 不仅可针对 Elasticsearch 数据提供实时可视化，同时还提供 UI 以便用户快速访问应用程序性能监测 (APM)、日志和基础设施指标等数据。