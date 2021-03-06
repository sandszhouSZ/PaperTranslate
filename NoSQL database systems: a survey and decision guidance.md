# NoSQL database systems: a survey and decision guidance
[英文原文](https://vsis-www.informatik.uni-hamburg.de/getDoc.php/publications/555/NoSQL%20database%20systems%20-%20a%20survey%20and%20decision%20guidance.pdf)

# 摘要
今天，数据正在以史无前例的规模被生成并且被消费。这种情况催生了面向持续增上的数据和请求的基于"NoSQL"的可扩展数据管理系统。但是，现有的系统的异构性和多样性使得在面对一个给定的应用上下文进行数据存储选型非常困难。因此，这篇文章从上层视角进行概括：我们并不打算深入对比每个系统的详细的实现特性，我们提供了一个  将功能和非功能的需求与NoSQL数据库中的技术和算法结合起来的比较分类模型。这个NoSQL工具箱使得我们可以根据一个简单的决策树来帮助从业者和研究者根据关键的应用需求过滤出潜在的备选系统。

# 1 介绍
通过几十年的发展，传统的关系型数据库管理系统(RDBMS) 对存储和查询结构化数据提供了涉及：`强一致、事务保证、很高级别的数据可靠性、稳定性等方面 `一套非常强悍的机制。近些年来，一些应用领域处理的有效的数量量已经变得如此巨大以至于通过创痛的数据库方案无法被存储以及处理。社交网络中的用户生成内容，大型网络传感器采集的数据 仅仅是大数据的两个典型的样例。一类可以用来处理大数据的新型的数据存储系统被统一称为：NoSQL数据库，其中很多系统通过牺牲查询能力和一致性保障提供相比关系型数据库更好的水平可扩展和高可用能力。因为任何有状态服务只能和其底层的数据存储具有一样的可扩展和容错能力，这种折衷对于面向服务的计算和作为服务的模型来讲非常关键，

目前有数十个NoSQL数据库系统,由于实现细节变化很快并且系统特性也随着时间变化不断演进，这些系统很难分清出处理其各自适合什么场景? 怎么处理失败? 甚至他们有哪些不同。这篇文章中，我同通过面向应用概念而不是系统特性并且结合NoSQL数据系统的自身需求、实现这些需求的技术需求以及过程中的折衷来对NoSQL的格局进行一个概要展示。我们主要聚焦在：`key-value，文档和宽-列存储`，这几种存储NoSQL类别从数据可扩展性管理上已经覆盖了绝大多数相关技术和设计决策。

第2部分，我们通过key-value存储、文档存储、宽-列存储等存储模型以及设计思想（CAP 以及PACELC）上safety-liveness的折衷两个维度来对NoSQL数据库系统进行分类来提供一种通用分类。
然后在第3部分我们更详细的调研使用到的技术并且描述需求如何和技术相关联。
第4部分通过应用我们的模型我们给出了一个数据库系统核心特征的广泛概述。
最终第5部分总结出根据应用需求来选择合适的NoSQL系统的一个简单和抽象的决策模型。

# 2 整体系统分类
为了对不同的NoSQL系统实现细节进行抽象，我们使用整体分类标准将数据存储分门别类。这一部分，我们使用两个主要分类标准：数据模型 以及 CAP理论。

## 2.1 数据模型分类标准
### 2.1.1  key-value 存储
一个key-value存储包含一些列具有不同key值的key-value序对。由于这个简单的结构，系统只支持get和put操作。存储的数据相对数据库是透明的特性，纯粹的key-value存储不支持CRUD（create、read、update、delete）之外的其他操作。key-value存储因此被认为：没有语义模式（schemaless）：存储数据的结构默认由应用程序在上层进行编码(schema-on-read)并且无法通过一个数据定义语言显示定义（schema-on-write）。
```
思考：关键词：无模式、只支持CRUD操作、value为编码后字节流上层理解
目前见到的key-value形式的存储：
磁盘介质：tdb、tssd
内存介质：tair、dcache、redis
```

这种数据模型的显著的优势就是简单性。非常简单的抽象使得数据分区和查询非常容易，这样数据库系统也能实现低延迟和高吞吐。但是，如果应用需要更复杂的操作，比如：范围查询，这种数据模型将不再有效。表1展示用户账户数据和设置是如何在key-value存储系统中存储中。由于比简单查询更复杂的查询无法被支持。数据只能在应用层通过解码分析才能抽取出像：是否支持cookie这样的信息。
![key-value和document格式的nosql](https://github.com/sandszhouSZ/PaperTranslate/blob/EditBranch/image/Nosql-1-2.png)

### 2.1.2  文档存储
一个文档存储是一个value为半结构化（比如JSON）格式的文档类型的key-value存储。这种相比较原生的key-value的约束给访问数据带来很大的灵活性。应用不仅可以通过ID来获取全部的文档内容，而且可以只获取文档的的部分数据。比如：custor的年龄，并且可以执行聚合（计算技术、排序），query-by-example或者甚至是全文查找。
```
思考：关键词：value为半结构格式的key-value文档，用户的使用可扩展性更高，使用形式更灵活、贴近RDBMS。
目前使用到的存储：
mongoDB，ES搜索
```

### 2.1.3 宽-列存储
宽列存储取名是从 `其经常被用于解释下面的数据模型：拥有很多稀疏列的一个关系表 `得来。但是，从技术上看一个宽-列存储和一个分布式的多层有序map非常类似(比如对于Bigtable：多层体现在：行、列、时间戳 -> data):第一层的key标识表中的row，被称为:行关键字，第二级的key被称为：列关键字。由于没有数据时就不会有列key，这种存储模式使得拥有任意多列非常容易。所以，null值可以不用花费任何开销进行存储。所有的列被分为称为：列族，同一列族的相关列一般都是同时访问，其可以存储在磁盘上临近位置。在磁盘上，宽-列存储并不能colocate每个行的所有数据而是colocate相同行相同列族的数据。所以，一个实体（row）在一个文档存储中不能只通过一次查找就能获取到。但是通过所有的列族的列聚合起来。但是，这种存储布局通常可以保证高度有效的数据压缩同时使得获取一个实体的一部分非常有效。数据通过key的字典序排序，所以通过自己的设计key值可以使得临近的数据的访问基本是物理上临近的。由于所有的行被分割为连续的范围（称为tablet）并且分布到不同的分片节点，行扫描一般只涉及很少的节点所以会非常高效。
```
思考： 关键词：有存储模式，具有为稀疏列的大表，多级key，压缩
目前使用到的存储:
Bigtable、HBase
```
![宽列存储](https://github.com/sandszhouSZ/PaperTranslate/blob/EditBranch/image/Nosql-wide-column.png)
Google为了存储大量网页特别开发Bigtable代表了宽-列存储模型，如图标3。网页表的每一行代表了一个特数据网页。行关键字是URL组件的反转级联组成，而且每个列关键字包含了：列族的名字和列通配符，其通过分好链接在一起。这里有两个列族：内容列族只包含一列用以存储真实的网页数据；anchor列族包含了到每个网页的连接，每个都是单独一列。表中的每个单元（行和列组合对应的数据）都可以通过时间戳或者版本号来被版本化。明确一个时间的多数信息不仅仅通过value而且是是通过key来表达。

## 2.2 一致性-可用性折衷：CAP 和 PACELC
```
思路：
CAP
关键点：CAP存在的前提是：分布式系统，即多个节点可同时接收多个client的read、write请求。离开这个前提讨论将无意义。
CAP理论表明，在出现P时，CAP只能二选一，下面列举作者见到的实际系统的选择：

PC: 一致性优先，在出现网络分区时，保证一致性和可用性，系统将停止服务。比如ChxMaster单Master系统。
PA： 可用性优先，Danamo系统 NRW，写多数即可以，出现分区的节点仍然在对client提供服务。
------------------------------------------------------------------------------------
PACELC：
  如果存在分区场景，需要在可用性(A)和一致性(C)之间权衡
  否则，需要在延迟(L)和一致性(C)之间权衡
```
除了数据如何存储以及如何获取外描述数据库特性的另一个手段就是其提供的一致性。一些数据库被设计来保证强一致和串行化（ACID）（Atmoic、Consistency、Isolation、Duration），另一些数据库更倾向于可用性（BASE： Basically Available，Soft-state，Eventually consistent）。这种折衷被每个分布式数据库系统所继承而且大量的不同的NoSQL系统表明在这两个范式中都有很多实现形态。接下来，我们婵说CAP理论和PACELC理论并根据一致性模型对数据库系统进行分类。

CAP类似于著名的FLP理论，CAP理论由Eric Brewer在PODC 2000年提出并被Gilbert和Lynch证明。由于其给出了分布式系统可能完成的终极上限，其是分布式计算领域真正有影响的不可能理论。CAP理论描述：`a sequentially consistent read/write register【一种数据结构】 that eventually
responds to every request`是不能在一个存在网络分区的异步系统实现的。换句话说，CAP只能保证下面三个特性中的两个：
- 一致性（C）： 读和写一直是原子的执行并且严格一致，换句话说，所有的client端在任何时刻看到的是同一份数据。

- 可用性（A）：系统中的每个非异常节点一直可以接受读和写请求并且最终返回一个有意义的结果，比如不会返回错误信息。

- Partition-tolerance(P): 系统在节点间存在消息丢失或者部分节点异常时仍然对外提供之前一致性保证和可用性。

Brewer主张一个系统在正常的操作中可以既保证可用性和一致性，但是如果出现系统分区，这是不可能的：如果一个系统在存在分区时继续工作，存在一些没有异常的节点会和其他节点丢失联系，所以该节点决定或者继续处理client的请求来保证可用性（AP，最终一致的系统）或者拒绝client的请求来保证一致性（CP）。第一个选择会违背一致性，因为这样会导致无效的读以及冲突的写，同时第二个选择会牺牲可用性。也有系统保证高可用和一致的，但是当存在分区时（CA）全部失败，比如单节点系统。已经证明保证一致性的CAP理论至少是因果一致性，包含一定的数据陈旧。Serializability（事务隔离性的正确标准）并不需要很强的一致性。然而，和一致性类似，操作的串行性在网络分区时也无法实现。

NoSQL系统的根据每个系统的能力分为：AP,CP,CA，这也是被广泛接受的系统分类标准。但是，需要强调的是`CAP理论在正在的操作中是没有任何意义`；只有在遇到网络分区时CAP理论也仅仅高速我们一个系统是更要倾向于可用性还是一致性。 与FLP-理论对比，CAP理论是一个允许任意消息被记录、被丢弃，或者被无限延迟的模型。对网络信道可能存在各种问题的假定（消息经常会异步的并且乱序的到达），一个CAP系统可以用theAttiya, Bar-Noy, Dolev算法（只要大多数节点存活）来保证。

PACELC由Daniel Abadi提出，强调CAP理论在正常操作中 在`延迟和一致性`两方面没法达成一致。尽管相比在异常场景下的可用性-一致性折衷方面CAP在分布式系统的设计有很大的影响。他将 折衷 和 更精确的刻画分布式系统的设计理念 相结合提出了PACELC。从FPACELC,我们了解到，在出现系统分区时，需要可用性和一致性的权衡，否则就是正常的操作，系统只有延迟和一致性的折衷。

这种分类在出现分区场景时基本上提供了（A/C）之间两种可能的选择，并且在正常时候提供了另外两个（L/C）之间的选择。相比CAP理论更加精细。但是，很多系统不能被严格的分配给某一个PACELC子类。并且PACELC分类中的一个子类，比如PC/EL没有任何一个现有的系统可以对应。

# 3 技术
每个取得显著成功的数据库都是为一些特定的应用而设计或者实现了一些显著的系统特性集合。之所以存在如此多不同的数据库系统的一个简单原因是：没有一个系统可以实现所有的有意义的特性。传统的SQL数据库比如PostgreSQL被用来提供全功能包：一个非常灵活的数据模型，包括joins在内的复杂的查询能力支持，全局完整性约束以及事务保证。设计模式的另外一个方面，虽然已经有像Dynamo一样`可以扩展数据和请求并且提供高读写吞吐同时具有很低延迟的key-value存储系统`，但是 却没有除过简单查询之外的功能接口。

本部分，我们强调分布式数据库系统的设计理念，包含：分片、副本、存储管理和查询处理。我们调研了可用的技术并且讨论这些技术如何和数据管理系统的不同的功能和非功能特性结合。为了展示哪种技术适合实现哪类系统特性，我们提供了NoSQL工具箱，表4，表中每个技术和功能和非功能特性通过边连接在一起。
```
设计一个分布式系统的关键是描述清楚：系统面向的应用场景，勾画出功能和非功能特性，这是决定技术选择的关键，说人话就是：
  功能目标决定技术方案，而不是技术方案决定产品。
```
![技术和功能](https://github.com/sandszhouSZ/PaperTranslate/blob/EditBranch/image/NoSQL-%E6%8A%80%E6%9C%AF%E5%92%8C%E5%8A%9F%E8%83%BD.png)

## 3.1 分片
一些分布式关系数据库系统比如Oracle RAC或者IBM的DB2依赖共享磁盘架构(所有的数据库节点共享中心数据存储,比如NAS 或者 SAN)来实现扩容。所以，这些系统在任何时候提供一致性的数据，但是本质上难以扩展。作为对比，本文聚焦的NoSQL数据库系统构建于shared-nothing体系架构之上，意味着每个系统包含拥有私有内存和私有磁盘的通过网络相互通信的节点。这样，高可扩展和数据容量通过在不同的节点间sharding（分区）数据来实现。

目前有3种基本的数据分布技术： 范围分片，哈希分片以及实体组分片。为了使得扫描变得更加高效，数据可以通过 `范围分片`分拆为有序并且连续的数据片段。但是，这种实现需要管理分配的master的协调。为了保证弹性，系统需要及时发现过载的分片并通过将过载的分片及时切分来解决热点。

范围分片被宽-列存储（比如Bigtable，HBase或者Hypertable）和文档存储(比如MongoDB，RethinkDB，Espresso，documentDB)支持。另一种拆分数据的方式是：哈希分片。每个数据项根据其主键的哈希值路由在不同的节点中。只要使用的哈希函数能产生均匀分布，这种方式不需要协作而且也能保证数据最终在各个分片间均匀分布。这种方式最大的缺陷是其只允许查找，遍历非常麻烦。哈希分片主要被用于key-value存储中并且在一些宽-列存储（比如Cassandra和 Azure表）中也存在。

可以根据 serverid = hash(id) mod servers来决定数据项的分片节点。但是，这种哈希模式由于分片节点数量在新节点加入或者离去时会发生变化，此时需要所有的数据项重新分配。所以这种哈希算法这在类似Dynamo、Riak、Cassandra这种允许额外的资源按需添加并且在闲置时剔除的弹性系统中不可行。为了增加的灵活性，弹性系统通常使用一致性哈希技术，该技术中，记录不不是直接分配到server节点而是分配到一个逻辑区间，然后在将该区间映射到所有的分片server中。这样，在系统拓扑发生变化时只有一部分节点需要重新分配。比如，一个弹性系统可以通过将映射到特定节点的所有逻辑分片卸载到其他节点来缩容。之后就可以将该节点关机。一致性哈希在NoSQL系统应用的更多细节可以参考附录。

实体-组分片 是为了实现single-partition transactions路由到co-located data而诞生的一种数据分片模式。这种分片被称为实体-组 并且 被应用显示的声明（比如G-Stor和MegaStor）或者 从事务访问模式中派生（Relation Cloud 以及 Cloud SQL Server）。当一个事务访问跨越多个group的数据，数据的拥有权可以在这两个实体-组之间传递 或者 事务管理者回滚到更昂贵的多节点事务协议。

```
思考：
就分片而言，目前有：
shared-disk 共享磁盘： 关系型数据库；IBM的DB2和ORACLE RAC
range sharding 范围分片： Bigtable，HBase，Hypertable
hash sharding 哈希分片： 一般使用其一致性哈希变种
consistent hash sharding 一致性哈希分片：Dynamo，Riack，Cassandra
entity-group shard实体组哈希： MegaStore，G-Store，Relation Cloud以及Cloud SQL Server
```

## 3.2 副本
就CAP而言，传统的RDBMS经常是运行于单台节点的CA系统：节点异常将导致整个系统不可用。所以系统管理者通过昂贵的但高可用的硬件来确保数据完整性和可用性。作为对比，NoSQL系统比如Dynamo，Bigtable或者Cassandra被设计为面向单节点违法处理掉数据和请求量，因此他们运行于包含上千台节点上。由于异常是不可避免的并且在任意大规模系统中会频繁出现，软件必须在日常运营中能够处理这种问题。在2009年，Google fellow Jeff Dean 指出：Google的一个典型的新集群会由于预期的和非预期的状况在第一年中遇到数千个硬件异常，1000个单节点异常，20个机架异常以及一些网络分区。在更大的云数据中心会有更多的网络分区和停电被上报。副本机制允许系统在遇到这类问题时保持可用性和持久性。但是在不同的节点上存储相同的记录（副本节点） 引入和同步的问题，最终导致在 `数据一致性` 和 `延迟和可用性方面`进行折衷。

Gray根据什么时间点更新将被传播以及哪里的更新将被接受 提出了一个针对不同的副本策略的两级分类法。 在第一级别（when）上有两种可能的选择：`Eager replication(激进副本)方式 `同步的将到来的变化传播给所有的副本之后才对client做出。`Lazy replication`（怠惰副本）只在收到的副本时急应用于持久化，数据采用异步传递。激进副本的一个最大好处数据副本之间是一致的，但是由于需要等到所有副本的响应导致写延迟变高以及可用性受损。惰怠副本更快，这是因为他允许副本多样化；后果是可能返回无效的数据。 在第二级（where），也有两种实现形式：或者`master-slave模式`保证变更智能通过主处理；另外`多主模式`，每个副本都可以接受写。在master-slave模式中，并发控制由于不用复制相比分布式系统不复杂，但是只要master异常整个副本集变得不可用。多主协议需要更复杂的机制去防止或者探测以及处理并发更新。解决这种问题的传统的技术就是`版本化，向量时钟，gossiping协议以及读修复（比如Daynamo）以及收敛或交换数据类型（Riak）`。

基本的，两层分类的所有四种组合理论上都有可能。分布式的关系型系统通常采用： 激进的主-备副本来保证强一致性。激进的多主副本比如Googles的Megastor导致了非常重的额外的用于同步以及可能导致分布式死锁检测的通信交互。NoSQL数据库系统一般采用怠惰副本和Master-slave主备结合（比如HBase以及MongoDB）.
或者采用怠惰副本和多主结合（AP系统比如 Dynamo和Cassandra）。Many NoSQL系统将延迟和一致性的选择权丢给用户。比如，对每一个请求，客户端决定是否等待任何一个返回即返回的低延迟 或者 确定的一致的响应（通过大多数副本或者主节点）来防止无效数据。
```
经验：
激进副本（同步） + 主备复制(源来源于主)           强一致，延迟和可用性低       tssd,lavaDB
激进副本（同步） + 多主复制（源来源于client）     一致性差                    Metastore
怠惰副本（异步） + 主备复制(源来源于主)          一致性和可用性高，            tfs的master， HBase，MongoDB
怠惰副本（异步） + 多主复制（源来源于client）    可用性高                     Dynamo，Cassandra
```

两层分类没有覆盖到的副本一个方面是 副本之间的距离。将数据放在邻近的地方很明显的产生很低的延迟，但是副本距离太近会影响系统可用性；比如两个副本在一个机架上，如果机架问题会导致所有的数据无法访问。但是相比临时不可用更多可能出现的是在一个灾害场景下会导致所有的数据副本丢失。一种可选的降低延迟的技术在Orestes中得到应用，数据通过网页缓存基础设置和缓存一致性协议被缓存到离应用最近的地方

地理位置的副本可以防止系统在异常时出现全部数据丢失，并且改善来自客户端的到分布式系统访问的读延迟。激进的地理分布策略，比如Google公司的Megastor，Spanner，MDCC以及Mencius通过很高的写延迟（基本100ms - 600ms）实现了强一致。 怠惰的地理分布比如Dynamo、PNUTS,Walter，COPS，Cassandra以及Bigtable
最近的修改可能会丢失。但是系统性能更高而且在出现系统分区时可以保持高可用。Charron-Bost在附录中等提供了数据库副本的全面的讨论。

## 33 存储管理
为了性能的最大化，数据库系统需要对存储的媒体进行优化并且持久化。这些通过主存存储（RAM），固态硬盘，以及普通硬盘以及混合形态。和企业级的RDBMS启动不同，分布式的NoSQL数据库尽量避免定制的共享磁盘的架构转而倾向于基于廉价服务器的什么都不共享的集群。存储设备基本可以图形化为"存储金字塔".这里还有一系列透明的cache（比如L1-L3的cpu cache以及磁盘buffer）。这些通过良好工程化的数据库算法被隐含的利用来保证数据局部性。RAM,SSD,HDD存储在成本和性能方面巨大的差异和尽最大的能力发挥他们的能力的策略也是NoSQL数据库如此多样性的一个原因。
![介质](https://github.com/sandszhouSZ/PaperTranslate/blob/EditBranch/image/%E5%90%84%E7%A7%8D%E4%BB%8B%E8%B4%A8.png)
存储管理具有空间维度（存储在哪里）以及 时间维度（何时存储）两个维度。原地更新和追加写是两种空间维度组织数据的技术；内存比如RAM作为数据存储位置同时进行日志记录是从时间维度将主存和持久化存储解耦来对数据真正持久化进行控制。 

在体系时代的结束这篇论文中，Stonebraker发现在传统的RDBMS系统中，只有6.8%的执行时间花费在真正有用的工作上，而其他都时间都消耗在：
- 缓冲管理（34.6%的耗时，比如cache来减少对磁盘的缓慢访问）
- 临界区（14.2%），减少多个线程导致的对共享数据结构的竞争条件
- 锁（16.3%），保证事务的逻辑隔离性
- 手工编码的优化（16.2%）

这意味着通过RAM被用作主存储可以对性能进行很大的提升。缺点是很高的存储花销以及持久化问题 - 一个很小的断点可以使得数据库的状态被破坏。这可以通过两种方式得到解决： 状态可以备份到n个内存存储节点来防止最多n-1个节点的异常（比如HStore，VoltDB） 或者通过日志来得到持久化存储（Redis或者SAPHana）。通过日志，一个随机写模型可以通过 接受的操作和相关的特性（比如redo的信息）被转换为一个顺序操作。在大多数NoSQL系统中，日志的提交规则是尊重，这需要每个写操作只有在被成功写入日志并且日志刷新到持久化存储中才被确认为成功。为了避免单独舒心每次操作导致的HDD硬盘的旋转延迟，log刷盘可以批量执行（组提交），这将显著的增加每次写入的延迟，但是显著的提升了吞吐。

SSD以及基于NAND flash memory的更通用的所有的存储设备 在这几个方面和传统的HDD明显不同： （1） 读和写操作的耗时的非对称性， （2） 没有原地重写-全部的块必须在覆盖写任意一个page之前被擦除。 （3） 首先的编程/擦除次数。 所以，数据库系统存储管理必须将SSD和HDD看为稍慢，持久化的RAM。由于到SSD的随机写相比较顺序写要慢大概一个数量级。另一方面随机读却没有任何性能损失。有一些数据库系统（比如Oracle的 Exadata，Aerospike）对SSD的特性专门做一些工程化实现来提升性能。在HDD盘，相比顺序访问，随机读和写大致都需要花费10-100倍的延迟。日志因为通过顺序写提供了明显的更高的吞吐因此适合SSD和HDD这种介质。

对内存型数据库，原地更新的访问模型是理想的：简化了实现并且到内存的随机写基本和顺序写延迟相差不大，差别狐妖在于pipeling以及CPU-cache层。但是，RDBMS和许多NoSQL系统（比如MongoDB）对于持久化存储也采用原地更新模型。为了处理这种到持久化存储的更慢的随机访问，主存通常被用来做cache并且通过日志来保证持久性。在RDBMS系统中，这是通过一个复杂的buffer池实现。该buffer pool不仅仅对特定的SQL访问模型采用合适的cache替换算法，而且保证ACID语义。NoSQL数据库也有一个简单的buffer pool用来缓存简单的查询并且没有ACID事务。对buffer poll模型的可替代模型留给OS通过虚拟内存（比如MongoDB实现的MMAP存储引擎）。其简化了数据库架构，但是也有 `哪些数据实体或者页面被保存在内存以及何时被替换出去`等方面的更弱控制的缺点。并且read-ahead（预读）和write-behind（写buffer）也被 `依赖文件系统逻辑而不是数据库查找进行操作系统层面的缺失异常处理逻辑` 透明的执行。

Append-only存储（也即是日志结构）通过写的顺序性试图最大化系统吞吐。尽管日志结构化的文件系统已经有一段很长的研究历史，追加写IO直到最近随着被Bigtable应用而重新流行起来。LSM树包含一个内存cache，一个持久化日志和不可变的，周期性写的存储文件。LSM树和不变量比如有序的数组合并树（Sorted Array Merge Tree） 以及 Cache-Oblivious Look-ahead Arrays（COLA） 被应用于很多NoSQL系统（Cassandra，CouchDB,LevelDb，Bitcast，RethinkDB，WiredTiger，RocksDB,influxDB，TokuDB）.通过写一个日志来设计一个数据库来实现最大的写吞吐 总是非常简单，复杂性在于提供一个快速的随机读和顺序读。这需要一个合适的索引结构，该结构或者是copy-on-write数据结构中被持久化的更新（比如CouchDB的COW B-树）或者像一个不可变更数据结构一样被周期性持久化（比如BIgtable类型的系统）。所有的日志结构化树实现都有一个问题是垃圾回收（紧凑）来回收已删除文件的控制的开销非常大。

在基础结构即服务的IAAS的虚拟化环境中，上述讨论的底层存储的很多特性都被隐藏起来。

```
本章的跨度很大，包含：
1. replicas副本的两级分类：when和where，推演出四类存储
2. 存储分布的地理相关性
3. 存储的维度：时间维度（内存操作+日志）和空间维度（原地写，追加写）
4. 几种存储介质极其相关比较，通过分析SSD和磁盘得出： 日志形式适合SSD和磁盘。
   从操作的瓶颈引入内存更适合随机读-写方案的支持
   从而最终引入LSM方案，利用了Mem承载热数据的随机写，以及日志来承载磁盘的顺序写，考察了读索引的重要性以及实现机制（COW B树以及Bigtable类似的结构）
```

## 3.4 查询处理
NoSQL数据库的查询特性主要取决于其分布模型，一致性保证，以及数据模型。`主key查找`，比如通过唯一的ID叉裙数据项，是每个NoSQL系统都支持的，这是由于他即兼容范围分片也兼容哈希分片。`过滤查询`获取某个单独表中的返回符合条件的所有项目（指定项目）。最简单的情况，可以通过过滤全表扫描实现。对哈希分片的数据库这是通过对每个分片实施指定的扫描并对最终的结果进行合并。对于范围分片的系统，基于范围相关属性的某个条件可以在特定的分片执行即可。

为了避免O(n)级别的扫描的效率问题，可以实现二级索引。这可以通过在每个分片内部的`本地二级索引`或者建立于所有分片之上的`全局二级索引`。全局二级索引本身也需要在各个分片之上分布存储，二级索引的一致性保证非常困难并且很难保证一致性协议。实践中，大多数系统仅仅提供最终一致性。（比如MegaStore，Google AppEngine数据存储，DynamoDB）。如果查询谓词和区分原则存在交集时执行本地二级索引的全局查询可以通过在少数的分区实施即可。但是，结果需要通过扫描-采集。比如，一个使用年龄作为范围分片的用户表在一个分片对于年龄的等价性进行判断。但是对于姓名的查询则需要在每个分片进行执行。一种特殊的全局二级索引就是`全文查找`，所有选择条件或者完整的数据项或者 通过数据库-内部的倒排索引建立关系。或者通过外部的搜索平台比如ES或者Solr(Riak凑所，DataStax Cassandra)。

`查询计划`是是对查询进行优化来最大化减少执行延迟的工作，对于合并和joins由于非常抵消并且在应用层代码很难实现，查询计划是非常必要的。关系查询处理的遍历财富和结果在当前的NoSQL系统中被大规模忽视有两个原因：1. key-value和宽-列模型基本上面向CRUD以及基于主键上的扫描操作，导致基本没有查询优化的空间。 2 大部分分布式查询处理的工作都是集中在OLAP场景（在线分析处理）负载更侧重于吞吐而不是延迟，导致关系型数据库上的单节点的查询优化无法轻松的在单分区以及副本数据库上得到应用。但是，大量的查询优化技术(特别是在文档数据库系统中)仍然是一个开放的研究性挑战。

数据库分析可以被本地应用（比如MongoDB，Riak，CouchDB） 或者 通过 外部的分析平台（比如Haddop、Spark、以及Flink）（比如在Cassandra或者HBASE）。NoSQL系统中最为著名的本地批处理分析是MapReduce。除了IO,交互开销以及计划优化非常有限的优化使得这些面向批量以及微批量的实现具有很高的响应延迟。`物理视图`是一个低查询响应时间的可选方案。他们在设计的时候声明并且在发生变化的时候持续更新（比如CouchDB以及Cassandra）。但是，和全局的次要索引一样，视图一致性当系统是分布式的时对快速、高可用的写有非负面影响。只有很少的数据库系统内部原生支持 提取并且查询无限的数据流。近实时分析的管道 或者铜鼓Lambda架构或者通过Kappa架构：前者 对批处理框架（比如Hadoop MapReduce）通过补充流处理（比如Storm）实现。 后者通过完全依赖流处理放弃批量处理来实现。

# 4 系统实例研究
在本部分，我们对主要的key-value，文档类型和宽列模型的存储进行量化的对比。我们参考各个系统的详细设计文档对各个指标进行简要的概括。我们提出了NoSQL工具箱如表4。 其是对数据库系统按照如下3个维度：1 功能需求； 2 非功能需求； 3 技术实现 进行分类的一种抽象描述。我们认为这种分类可以  很好的刻画很多数据库系统并且可以被作为不同数据库系统间非常有意义的对比。表1展示了MongoDB，Redis，HBase，Riak，Cassandra和MySQL在其各自默认配置下相互之间的对比。
![各个产品在功能性、非功能性、技术上的对比](https://github.com/sandszhouSZ/PaperTranslate/blob/EditBranch/image/NoSQL%E5%88%86%E7%B1%BB.png)
一个更加明显的对比在表2中展示。
![更详细的对比](https://github.com/sandszhouSZ/PaperTranslate/blob/EditBranch/image/NoSQL%E4%BA%A7%E5%93%81%E5%AF%B9%E6%AF%941.png)
确定每个特定的系统特性的方法是通过对各种公开的可用的系统文档和系统文献的深度的分析得出的。而且一些特性可能是通过探索开源代码的方式、和开发者私人邮件交流、或者从业者的口头报告和基准测试集的分析得来的。

对比表明SQL和NoSQL数据库是如何完成不同的需求：`RDBMS`系统提供了更强的功能特性，对比而言NoSQL在非功能性方面比如可扩展性、可用性、低延迟、高吞吐方面表现更好。但是即使是NoSQL内部也有很大的不同：`Riak和Cassandra `可以通过配置来实现很多非功能的需求，但是智能提供最总一致性并且无法实现除过数据分析之外更多的功能性能力(对于Cassandra也支持条件更新)。另一方面`MongoDB和HBase`可以提供强一致性以及更多的功能特性比如扫描查询 并且对mongoDB，支持按条件过滤，但是无法保证分区时的读和写高可用并且会有更高的读延迟。`Redis`作为除过MySQL之外唯一的无分片系统面向极限的低延迟高吞吐通过采用`内存数据存储以及异步的主-备分本机制`做了一些列特殊的折衷


# 5 总结
选择一个数据库系统通常意味着相比较其他特性更加倾向于一系列特性（即关键特性）。为了减少选择的困难，我们在表6中提出了一个二叉决策树来将样例应用和潜在的合适的数据库系统进行映射。叶子节点涵盖了从简单缓存（左侧）到大数据分析（右侧）的所有应用。虽然这种问题视角并不完整，但是却明确的给出了特定数据管理系统面向特定问题的解决方案。
![选型决策树](https://github.com/sandszhouSZ/PaperTranslate/blob/EditBranch/image/NoSQL%E5%86%B3%E7%AD%96%E6%A0%91%E9%80%89%E5%9E%8B.png)
首先将树根据 `应用的访问模型`进行拆分：或者分为快速查找， 或者分为更复杂的查询机制。 快速查找应用可以继续通过数据量分为：单机主存即可完全容纳的数据量，一个单节点系统比如Redis或者Memcache或许是其最好解决方案，功能性需求多时可以选择Redis否则选择Memcache更优； 如果数据量超过内存或者单节点也无法存储，可以支持平滑扩容的多节点系统可能更适合。这种情况下最重要的决策是业务更倾向于高可用（AP）还是强一致（CP）。像Cassandra以及Riak可以提供一个永远在线的服务能力，与此相反HBase、MongoDB以及DynamoDB提供了更强的一致性。

树的另一半包含了应用层需要的更复杂的查询类型。这里，也根据数据量分为单机可以处理（HDD-SIZE） 或者分布式存储是必须的（无限的容量）。对于中等数据量的OLTP（实时事务处理）负载，传统的RDBMS或者图数据库比如Neo4J更优，因为其可以提供ACID语义。否则可用性是最为关注的因素的话，分布式系统比如 MongoDB，CouchDB，或者DocumentDB是更优的选择。

如果数据容量超过了单台节点的限制，正确的系统选择依赖于 相关的查询模式：如果复杂的查询需要针对延迟做优化，比如网络社交应用，MongoDB非常有竞争力，这是因为他更倾向于即时查询。HBase和Cassandra在这种场景下也是不错的选择，但是HBase更擅长处理面向高吞吐量的大数据分析以及和Hadoop结合的场景

总体而言，我们确信 我们提出的自顶向下的模型可以非常有效的用于根据核心需要从大量的NoSQL中过滤出合适的系统。NoSQL工具箱更近一步从功能和非功能需求上到通用实现技术等方来对不断发展的NoSQL进行了详细的分类。



# 参考文献
1. Abadi D (2012) Consistency tradeoffs in modern distributed database system design: cap is only part of the story. Computer 45(2):37–
    42
2. Attiya H, Bar-Noy A et al (1995) Sharing memory robustly in message-passing systems. JACM 42(1)
3. Bailis P, Kingsbury K (2014) The network is reliable. Commun ACM 57(9):48–55
4. Baker J, Bond C, Corbett JC et al (2011) Megastore: providing scalable, highly available storage for interactive services. In: CIDR,
    pp 223–234
5. Bernstein PA, Cseri I, Dani N et al (2011) Adapting microsoft sql server for cloud computing. In: 27th ICDE, pp 1255–1263 IEEE
6. Boykin O, Ritchie S, O’Connell I, Lin J (2014) Summingbird: aframework for integrating batch and online mapreduce computations.VLDB      7(13)
7. Brewer EA (2000) Towards robust distributed systems
8. Calder B, Wang J, Ogus A et al (2011) Windows azure storage: a highly available cloud storage service with strong consistency. In:
    23th SOSP. ACM
9. Chang F, Dean J, Ghemawat S et al (2006) Bigtable: a distributed storage system for structured data. In: 7th OSDI, USENIX    
    Association,pp 15–15
10. Charron-Bost B, Pedone F, Schiper A (2010) Replication: theory and practice, lecture notes in computer science, vol. 5959. Springer
11. Cooper BF, Ramakrishnan R, Srivastava U et al (2008) Pnuts: Yahoo!’s hosted data serving platform. Proc VLDB Endow 1(2):1277–1288
12. Corbett JC, Dean J, Epstein M, et al (2012) Spanner: Google’s globally-distributed database. In: Proceedings of OSDI, USENIX
    Association, pp 251–264
13. Curino C, Jones E, Popa RA et al. (2011) Relational cloud: a database service for the cloud. In: 5th CIDR
14. Das S, Agrawal D, El Abbadi A et al (2010) G-store: a scalable  data store for transactional multi key access in the cloud. In: 1st
    SoCC, ACM, pp 163–174
15. Davidson SB, Garcia-Molina H, Skeen D et al (1985) Consistency  in a partitioned network: a survey. SUR 17(3):341–370
16. Dean J (2009) Designs, lessons and advice from building large distributed systems. Keynote talk at LADIS 2009
17. Dean J, Ghemawat S (2008) Mapreduce: simplified data processing on large clusters. COMMUN ACM 51(1)
18. DeC andia G, Hastorun D et al (2007) Dynamo: amazon’s highly available key-value store. In: 21th SOSP, ACM, pp 205–220
19. Fischer MJ, Lynch NA, Paterson MS (1985) Impossibility of distributed consensus with one faulty process. J ACM 32(2):374–382
20. Gessert F, Schaarschmidt M, Wingerath W, Friedrich S, Ritter N (2015) The cache sketch: Revisiting expiration-based caching in the  
    age of cloud data management. In: BTW, pp 53–72
21. Gilbert S, Lynch N (2002) Brewer’s conjecture and the feasibility of consistent, available, partition-tolerant web services. SIGACT     News 33(2):51–59
22. Gray J, Helland P (1996) The dangers of replication and a solution. SIGMOD Rec 25(2):173–182
23. Haerder T, ReuterA(1983) Principles of transaction-oriented database  recovery. ACM Comput Surv 15(4):287–317
24. Hamilton J (2007) On designing and deploying internet-scale services.In: 21st LISA. USENIX Association
25. Hellerstein JM, Stonebraker M, Hamilton J (2007) Architecture of a database system. Now Publishers Inc
26. Herlihy MP,Wing JM (1990) Linearizability: a correctness condition for concurrent objects. TOPLAS 12
27. Hoelzle U, Barroso LA (2009) The Datacenter As a Computer: an introduction to the design of warehouse-scale machines. Morgan
    and Claypool Publishers
28. Hunt P, Konar M, Junqueira FP, Reed B (2010) Zookeeper: waitfree coordination for internet-scale systems. In: USENIXATC.            
    USENIX Association
29. Kallman R, Kimura H, Natkins J et al (2008) H-store: a highperformance, distributed main memory transaction processing
    system. VLDB Endowment
30. Karger D, Lehman E, Leighton T et al (1997) Consistent hashing  and random trees: distributed caching protocols for relieving hot
    spots on the world wide web. In: 29th STOC, ACM
31. Kleppmann M (2016) Designing data-intensive applications. O Reilly, to appear
32. Kraska T, Pang G, FranklinMJet al (2013) Mdcc: Multi-data center  consistency. In: 8th EuroSys, ACM
33. Kreps J (2014) Questioning the lambda architecture. Accessed: 17 Dec 2015
34. Lakshman A, Malik P (2010) Cassandra: a decentralized structured storage system. SIGOPS Oper Syst Rev 44(2):35–40
35. Laney D (2001) 3d data management: Controlling data volume,velocity, and variety. Tech. rep, META Group
36. LloydW, FreedmanMJ, Kaminsky,Met al (2011) Don’t settle for eventual: scalable causal consistency for wide-area storage with
    cops. In: 23th SOSP. ACM
37. Mahajan P, Alvisi L, Dahlin M et al (2011) Consistency, availability,and convergence. University of Texas at Austin Tech Report
    11
38. Mao Y, Junqueira FP, Marzullo K (2008) Mencius: building efficient replicated state machines for wans. OSDI 8:369–384
39. Marz N,Warren J (2015) Big data: principles and best practices of scalable realtime data systems. Manning Publications Co
40. Min C, Kim K, Cho H et al (2012) Sfs: random write considered harmful in solid state drives. In: FAST
41. Özsu MT, Valduriez P (2011) Principles of distributed database systems. Springer Science & Business Media
42. Pritchett D (2008) Base: an acid alternative. Queue 6(3):48–55
43. Qiao L, Surlaker K, Das S et al (2013) On brewing fresh espresso:Linkedin’s distributed data serving platform. In: SIGMOD, ACM,
    pp 1135–1146
44. Sadalage PJ, Fowler M (2013) NoSQL distilled : a brief guide to the emerging world of polyglot persistence. Addison-Wesley,
    Upper Saddle River
45. Shapiro M, Preguica N, Baquero C et al (2011) A comprehensive study of convergent and commutative replicated data types. Ph.D.
    thesis, INRIA
46. ShuklaD,Thota S,RamanKet al (2015) Schema-agnostic indexing with azure documentdb. PVLDB 8(12)
47. SovranY, PowerR,Aguilera MK, Li J (2011) Transactional storage  for geo-replicated systems. In: 23th SOSP, ACM, pp 385–400
48. Stonebraker M, Madden S, Abadi DJ et al (2007) The end of an  architectural era: (it’s time for a complete rewrite). In: 33rd VLDB, 
      pp 1150–1160
49. Wiese L et al (2015) Advanced Data Management: For SQL. Cloud and Distributed Databases. Walter de Gruyter GmbH & Co KG,NoSQL
50. Zhang H, Chen G et al (2015) In-memory big data management and processing: a survey. TKDE
