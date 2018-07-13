# Bigtable：一种面向结构化数据的分布式式存储系统

# 前沿
Bigtable是一种设计于处理海量结构化数据的分布式存储系统：这些数据分布于成千上万台通用服务器上。Google的很多项目都是用Bigtable来进行底层数据存储,包括：网页索引，Google地图，Goole金融。这些应用对Bigtable提出了不同的需求：包括数据量（从URLS到网页到卫星图片）和延迟需求（从后端批量处理到实时数据服务）。尽管需求多种多样，但是Bigtable已经为所有的Google的产品成功的提供了灵活的、高性能的解决方案。这篇文章中，我们将描述Bigtable提供的一种简单的数据模型，其为client端提供了关于数据布局和格式的动态灵活的控制，而且我们将对Bigtable的设计和实施细节进行详细描述。

# 1 介绍
过去的两年半时间内，我们成功设计、实施、部署了一种管理结构化数据的分布式存储系统，称之为Bigtable。Bigtable被设计为可以将数据可靠的扩展到PB并且分布在数千台机器上。Bigtable已经实现了下面几个目标：`适用性广泛`，`可扩展性强`，`高性能`,`高可用`。Bigtable至少为Google公司超过6个产品提供存储服务，包括Google分析，Google金融，社交网络服务Orkut，个性化搜索，在线文档编辑Writely,Google地图。这些产品将Bigtable应用于一系列严苛的负载场景，从面向高吞吐的批处理任务到面向终端用户的低延迟服务。这些产品使用的Bigtable集群配置千变万化：从几台到成千台节点，最多存储数百PB的数据。

Bigtable在很多方面都酷似数据库：它拥有和数据库类似的很多实现策略。并行数据库和内存数据库已经拥有很好的可扩展性和高性能，但是相比较而言，Bigtable提供了不同的接口。Bigtable不提供`完整的关系数据模型`；而是基于数据分布和格式之上为client端提供了一种简单的数据模型去动态修改，而且让client 在基于底层存储之上建立数据的局部特性。数据按照行和列名字进行索引，行和列可以被为任意字符串。尽快client会经常将多种结构化数据或者半结构化数据进行串行化为字符串，但是Bigtable将该数据看作为原始数据（并不理解内部细节）。通过仔细的选择模式client可以控制数据布局。最后，Bigtable模式参数允许client对`是从内存还是磁盘提取数据`进行动态的控制。

第2部分对数据模型进行更详细的描述，第3部分我们对client API进行大致的描述。第4部分简单介绍Bigtable依赖的底层的基础服务。第5部分对Bigtable实现的关键特性进行描述。第6部分描述为了提升Bigtable的性能我们所做的改进。第7部分对Bigtable的性能进行评估。第8部分Google内部的几个服务是如何使用使用Bigtable的。第9部分讲述从Bigtable的设计和运营中我们学到的一些经验。最后，在第10部分描述了相关工作。第11部分得出最后的结论。

# 2 数据模型
Bigtable是一个稀疏的、分布式的、持久化的 多维度有序数组。 该数组通过行<关键字，列关键字，时间戳>进行索引；数组中每个数据是一个原生的字节流。即：(row:string, column:string, time:int64) -> string。

我们在对很多类似Bigtable的系统的使用进行大量调研之后确定这种数据模型。作为驱动我们设计策略的一个具体的例子，假设我们想保留一份`海量网页和其相关数据`用作其他不同的项目。让我们将这个特殊的表称之为：Webtable。在Webtable中，我们可以使用URL作为行主键，网页的不同的特性作为列名并且将网页的内容保存在`contents`中：当他们被抓取时选择时间戳之下的列，如表1所示：