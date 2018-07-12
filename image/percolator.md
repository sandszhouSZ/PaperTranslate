# 利用分布式事务和通知机制的大规模增量索引处理

# 1 摘要

当新爬取的文档加入系统时，需要持续的对当前已有的海量存储文档进行更新。这项利用小量的、独立增量数据对大量数据进行更新的任务是数据处理的一类典型例子。这项任务的技术选型取决于底层存储数据的基础设施。$数据库$在存储量以及吞吐上显然无法满足这类任务:Google's 索引系统在数千台机器上存储了数十PB达到数据并且每天处理数十亿计的更新。$MapReduce以及其他批处理系统$由于注重效率更适合创建大批量并发执行的任务，也不适合处理各别的小的更新。

我们搭建了Percolator，一个面向$增量更新大数据集合$的系统，目前已经线上部署并作为Google新的网页搜索索引。通过代替基于索引系统的批量索引系统，采用基于增量处理的Percolator，每天处理同样数据量的数据，可以将Google搜索结果中文档的平均年龄减少50%。

# 1  介绍

考虑下这个任务：创建一个$web文档索引$以便为用户搜索查询提供服务。索引系统首先需要爬取网上所有的网页，然后处理并维护一组不变的索引。比如，如果同样一份内容通过多个URLS爬取到，只有最高PageRank的URL才会出现在索引中。每个连接都是倒置的($即文本 -> 连接$)以便保证每个出链的锚文本和链接指向的网页关联起来。 连接倒置必须能处理重复：指向同一个份数据的的多个副本界面应该跳转到PageRank最高的那个副本界面。

这是一个批处理任务，可以描述为一系列MapReduce操作：一个用于集群重复，一个用于连接倒置，etc。由于MapReduce限制了计算的并发能力，因此保证不变性非常容易: 所有的文本只有结束一个步骤才能开始下一个步骤。比如，当索引系统正在将当前最高PgeRank值的URL写到倒置链接，我们不必担心PageRank的实时变化；前一个MapReduce步骤已经确定了每个页面的PageRank。

当新爬取了一小部分网页，现在考虑如何更新索引。仅仅在这些新的网页上运行MapReduce并不明智，比如，在这些新网页和其他网页有些链接，MapReduce必须在整个存储集群上运行，所以需要在这部分增量网页和旧的网页上执行MapReduce。如果提供足够的计算资源，MapReduce的可扩展性能很容易的实现这个目标，实际上Google之前的网页搜索索引就是这样实现的。但是，重新执行所有的网页丢弃了之前已经执行结束的工作结果，导致整体延迟和总存储量的大小而不是增量数据的大小成正比。

索引系统可以将索引库存储在DBMS中，在更新特定的文档时通过事务来保证不变量。 但是，当前的DBMS不能处理过于庞大的数据：Google的索引系统在数千台机器上存储了数十PB的数据。分布式存储系统比如Bittable的确可以处理如此大量的存储但是却没有提供一种在并发更新场景下能保证数据不变量的手段。

一种处理网页搜索索引的理想的数据处理系统应该对$增量处理$做特殊优化；应该允许保持非常大量文档的存储的同时在新文档爬取时能高效的进行更新。该系统将能并发处理大量小的更新，一个理想的系统也应该在并发更新时提供一种保持不变量的机制，并且对哪些更新已经被处理进行跟踪。

这篇Paper的剩余部分描述了一种特定增量处理系统：Percolator。Percolator提供数十PB存储的随机访问。允许我们独立的处理新增文档，不用像MapReduce一样需要全局扫描现有存储。为了实现高吞吐，很多节点的多个线程需要并发修改存储，所以Percolator提供ACID兼容的事务机制来满足工程师对存储状态一致性的要求；我们当前实现了$快照隔离语义$。

除了关于并发性的考虑，增量系统的工程师需要对增量计算的状态进行追踪。Percolator提供观察者：当用户自定义列发生变化时，系统会调用特定代码片段并通知给用户。Percolator应用程序按照一系列观察者结构化为一个整体；每个观察者完成一个任务同时通过写BitTable来唤醒下游的观察者来执行新的任务。最开始时，一个外部的进程通过向Bigtabal写初始数据来触发调用链的第一个观察者。

Percolator是专门为增量处理而创建，并不是为了取代大多数已经存在的数据处理方案。$\color{red}结果不能分割为小的更新的$计算(比如对文件排序)更适合采用MapReduce。而且$\color{red}计算应该具有很强的一致性需求$，否则Bigtable已经足够。最后，$\color{red}计算量应该在某些维度(总的数据量、CPU消耗)非常大$；不适合MapReduce或者Bigtable的小的计算可以考虑传统的DBMS系统。

在Google内部，Percolator的第一个应用案例是将新爬取网页融合进实时网页搜索索引中。通过将索引系统切换到增量系统，我们可以在网页爬取后及时更新到现有搜索索引中。新方案将平均文档处理延迟降低了100倍，搜索结果返回的文档的平均年龄下降了50%(搜索结果中网页的年龄除过包含索引外，还包含文档爬取到文档修改之间的时间差)。该系统另外用于将页面渲染成图片；Percolator跟踪网页和其依赖资源的的关联关系，所以当依赖的资源发生变化时，网页也会重新得到处理。

# 2 设计

为了在海量场景下处理增量计算，Percolator提供两个主要的抽象:  $基于随机访问场景的ACID事务$；$观察者，一种组织增量计算的方式$。

一个Percolator系统在集群每台机器上包含三个二进制：$\color{red}一个Percolator worker$，$\color{red}一个Bittable tablet  server$，$\color{red}一个GFS chunkserver$。所有的观察者链接到Percolator worker，该worker扫描Bigtable以查找已更改的列(通知)并且以Percolator worker进程中函数调用的方式调用对应的观察者。观察者通过与Bittable tablet servers进行读/写 RPCS通信的方式执行事务，Bittable tablet servers同样与GFS chunkservers进行读/写 RPCS通信进行交互。该系统依赖两个小的服务： $\color{red}时钟服务$和$\color{red}轻量级锁服务$。 时钟服务提供严格意义上单调递增的时间戳：快照隔离协议的正确操作时序需要用到该属性。percolator worker使用轻量级的锁服务来更有效率的查找脏通知。

从程序员的视角，一个Percolator存储包含少量的表。每个表包含一系列按照行和列索引的“cells”。每个cell包含一个值：一个封装的字节数组。（实际上，为了支持快照隔离，我们将每个cell实现为一系列通过时间戳索引的数值）

Percolator的设计受到大规模和对低延迟不敏感这两个需求的影响。对延迟的容忍可以让我们采用一些简单懒惰的策略：比如当事务在某些节点执行失效延迟清理未释放锁。这种懒惰的、易实现的方式会导致事务提交延迟数十秒。这种延迟在执行OLTP任务的DBMS系统中是不能允许的，但是这在构建网页增量索引的处理系统中是可以容忍的。Percolator没有事务管理的中控节点；而且没有全局死锁检测。这样会增加冲突事务的延迟，但是却允许系统扩展到数千台节点。

## 2.1 Bigtable概览

Percolator是建立在分布式的Bigtable系统之上。Bigtable对用户呈现了多维度已排序的map结构：key为<行、列、时间戳>元组。Bigtable提供行级别的查看和更新操作，而且Bigtable的行事务允许每个行可以原子执行read-modify-write操作。Bigtable处理数PB的数据并且海量的节点上运行非常稳定。

一个运行的Bigtable包括一系列分片tablet 节点，每个节点负责提供一些分片tablets数据的服务（每个tablet包含连续的一段key空间）。master协调各个tablet节点（比如让节点加载或者释放tablet）。一个分片tablet存储为Google SSTable格式。SSTable存储在GFS存储系统中；Bigtable依靠底层GFS系统在磁盘异常时保证数据可靠。Bigtable通过将一系列的列的集合映射为局部组来允许用户控制表性能特性。每个局部组的所有列存储在单独的SSTables中，相比全量扫描，只对这些特定列扫描会节省很多时间。

构建于Bigtable之上这个决策确定了Percolator的轮廓。Percolator继承了Bigtable交互的全部接口：数据按照行和列分布，同时Percolator的元数据单独存储在特定的列（参考图5）。Percolator的API和Bigtable的API基本一致：Percolator的库很大程度的包含了Bigtable操作并在此之上封装成Percolator相关的计算接口。实施Percolator的挑战在于提供提供Bigtable没有的特性：$多行之间并发操作的事务性$以及$观察者框架$。

# 2.2 事务 

Percolator通过ACID 快照隔离语义提供跨行、跨表的事务。Percolator的用户通过使用特定的语言(当前为C++) 在其代码中调用Percolator API来完成他们的事务逻辑。表2展示了通过对其内容进行哈希来对集群文档进行聚类的简化版本：

![使用PercolatorApi进行checksum计算](E:\Paper\Google论文集合\论文翻译\使用PercolatorApi进行checksum计算.png)

如果Commit()返回false，表明这个事务出现冲突（这个例子中，是由于两个具有相同内容的URL的同时被处理），其会在稍后进行重试。对Get()和Commit()的调用会阻塞；通过在线程池中同时执行很多事务来达到并发。

在不要求强一致事务保证下，增量处理数据是可行的，事务由于可以追踪的特性让用户更容易分析系统各个时间点的状态，这样可以避免在持久化系统中长期存在不一致的错误。比如，在一个强事务性的网页索引系统，工程师可以做出如下假定：文档内容的哈希和其存储在索引副本的各份数据永远是一致的。没有事务性，一个瞬时宕机会导致一个长期的错误：文档表的一个条目在其备份表中没有对应的URL。事务同时使得建立索引表更加实时，各副本数据一致。上述所有的例子都需要跨行的事务，而不是Bigtable已经提供的单行内的事务。

Percolator利用Bigtable中的时间戳维度对每份数据存储多个版本。多版本被用以提供快照隔离，这保证每个事务可以读到特定时间戳的稳定的版本快照。写操作不同。快照隔离可以解决写-写冲突： 如果事务A和事务B并发写同一个cell，最多只能有一个可以提交成功。快照隔离不保证串行化；尤其是，基于快照隔离的事务受到写倾斜的影响。快照隔离相比串行协议最大的优势是读性能。因为每个时间戳代表了一致性快照，读操作只需要在Bigtable之上执行给定时间戳的查找操作，不用获取锁。图3展示了基于快照隔离的事务之间的关联关系(写事务2无法看到写事务1的写的数据，但事务3可以看到事务1、2的写的数据，事务1和事务2产生并发，两者最终只能成功1个)。

![1](E:\Paper\Google论文集合\论文翻译\1.png)

由于Percolator是作为库实现在client端去和Bigtable交互，而不是直接控制存储的访问，相比传统的PDBMS系统，Percolator面临分布式事务环境中的一些特有的挑战。其他并行数据库将 对系统组件加锁并操作磁盘的功能已经进行集成：每个节点内部协调去访问磁盘的数据，该节点即可以授权对请求加锁也可以拒绝违反锁规则的访问请求。

然而，部署Percolator的任何Client节点都可以执行对Bigtable表中的状态修改：没有一个合适的地方拦截通信并分配锁。这样，Percolator必须显式的维护锁。$\color{red}锁必须在节点异常期间一直存在$；如果一个锁在提交的两个阶段之间丢失，系统会错误的提交两个可能存在冲突的事务。$\color{red}锁服务必须提供高吞吐$；数千台机器将会同时申请锁。$\color{red}锁服务也必须低延迟$；每个Get()操作除了读取数据也要读锁，我们希望最小化延迟。基于这些约束，锁服务需要有多副本（去单），分布式，负载均衡（均分负载），写到一个持久化数据存储中。Bigtable本身符合所有的这些需求 ，所以Percolator将锁存储在相关Bigtable特定的内存列，当访问某一行数据时在Bigtable的行事务中读取锁或者修改锁。

我们现在更加详细的描述事务协议。表6展示了Percolator的伪代码，表4展示了事务执行期间Percolator数据和元数据的布局，这些不同的元数据列被系统按照表5的形式使用。事务的构造需要访问timestamp服务获取一个开始时间戳（line 6），该值决定了Get()操作可以看到的一致性快照。Set()操作会本地buffer（line 7）直到最后进行提交。提交buffer缓冲的数据是一个两阶段步骤，这两个阶段需要通过client进行协调。不同节点间的事务通过Bigtable 分片tablet节点间的行事务来完成。

```
图4注解：
各列的含义见图5注解

1.  时间戳5，Bob和Joe各有10美金和2美金；
	时间戳6，将Bob、Jeo当前已提交数据同步到write列,{6，data列时间戳5锁对应的数据}，其他两列增加默认值
	
2. 	事务开始，创建strartStamp7，Bob账户通过写lock列来锁住Bob’s的帐号。该锁是事务的主锁，同时将数据列更	 新为当前值。此时lock列增加：{7,主锁}，data列增加：{7,最新值}，write列增加默认

3.  Bob账户操作完毕后，开始操作Joe账户，时间戳仍然为startStamp 7，该锁是该事务的副锁，所以锁列为指向主	  锁的引用（可以在事务异常时，知道如何清理残留数据），同时将数据列更新为当前值，此时lock列新增:{7,主锁
	位置}，data列增加:{7,最新值}，write列增加默认
	
4. 	此时到了提交环节，首先删除7的主锁并且在新CommitStamp时间戳8更新write列为当前数据存储的时间戳，此	 时：write列:｛8,数据存储的时间戳｝，data列：空，lock列：空。
	此时Bob行的读操作可以读到最新的数据

5. 	事务对Joe执行类似的操作，即删除lock列startStamp时间点的主锁信息，然后增加新的commitStamp时间点数	
	据，即：write列:(8,数据写入的时间戳)，lock列：空，data列：空。
```



![事务数据和元数据布局](E:\Paper\Google论文集合\论文翻译\事务.png)

![事务1](E:\Paper\Google论文集合\论文翻译\事务1.png)

```
图5注解：
图5的c以及图4的bal标识Bigtable中存储的代表Percolator的列，Percolator包含3列：data，write，lock
其中：

lock列: 表示尚未提交的写事务当前的状态。    内容为：<时间戳，该时间戳写事务中当前主锁的位置>列表
write列：表示当前已经提交成功的数据的位置。 内容为：<时间戳,该时间戳已经交数据的时间戳位置>列表
data列： 表示数据本身。					内容为：<时间戳，该时间戳的数据>列表

notify列：观察者需要执行
ack_O列：观察者O已经执行；存储上次成功运行的开始时间戳
```

![Bittable列中代表Percolator列的以C命名](E:\Paper\Google论文集合\论文翻译\Bittable列中代表Percolator列的以C命名.png)

```
伪代码如下所示：

 class Transaction ｛
 
	struct Write 
	{ 	//
		Row row; 
		Column col; 
		string value; 
	};
	
	vector<Write> writes_;
	int start_ts_;

	//初始化开始时间戳
 	Transaction() : start_ts_(oracle.GetTimestamp()) {}
 	
	void Set(Write w) ｛ writes_.push back(w); }
	
	//获取最近一次提交的数据(有效数据)，可能会删除过期数据。
	bool Get(Row row, Column c, string* value) 
	{
		while (true) 
		｛
			bigtable::Txn T = bigtable::StartRowTransaction(row);

			// Check for locks that signal concurrent writes.
			//有可能lock列包含了中断的锁信息，代表事务未完成，也可能当前有并发写
			if (T.Read(row, c+"lock", [0, start_ts_])) 
			{
				// There is a pending lock; try to clean it and wait
				BackoffAndMaybeCleanupLock(row, c);
				continue;
			}

			// Find the latest write below our start timestamp.
			latest_write = T.Read(row, c+"write", [0, start_ts_]);
			if (!latest_write.found()) 
				return false; // no data
			
			//获取最近一次提交的开始时间戳并读取数据
			int data_ts = latest_write.start_timestamp();
			*value = T.Read(row, c+"data", [data_ts, data_ts]);
			return true;
		｝
	｝
	
	// Prewrite tries to lock cell w, returning false in case of conflict.
	bool Prewrite(Write w, Write primary) 
	｛
		Column c = w.col;
		bigtable::Txn T = bigtable::StartRowTransaction(w.row);

		 // Abort on writes after our start timestamp . . .
		 //如果在预写时发现行中write列已经有提交的更大的写时间戳。
		if (T.Read(w.row, c+"write", [start_ts_ , max])) 
			return false;
			
		// . . . or locks at any timestamp.
		//如果在预写时发现行的lock列中已经有锁，说明正在被其他人写，退出（清理在get时做）
		if (T.Read(w.row, c+"lock", [0, max])) 
			return false;
		
		//写入data列： ｛start_ts_,当前值｝
		T.Write(w.row, c+"data", start_ts_ , w.value);
		//写入lock列：｛start_ts_,主行.主链｝
		// The primary’s location.
		T.Write(w.row, c+"lock", start_ts_ ,fprimary.row, primary.colg); 
		return T.Commit();
	｝
	
	bool Commit()
	{	//commit操作为两阶段：首先预写所有的节点；然后提交
	    --------------------------------------------------------------------
        				预提交
        --------------------------------------------------------------------
		Write primary = writes_[0];
		vector<Write> secondaries(writes_.begin()+1, writes_.end());
		
		//预写主节点
		if (!Prewrite(primary, primary)) 
			return false;
		//对每个从节点，预写
		for (Write w : secondaries)
		{	//预写所有的从节点
			if (!Prewrite(w, primary)) 
				return false;
		}
        --------------------------------------------------------------------
        				提交
        --------------------------------------------------------------------
		int commit_ts = oracle .GetTimestamp();
		// Commit primary first.
		Write p = primary;
		bigtable::Txn T = bigtable::StartRowTransaction(p.row);
		if (!T.Read(p.row, p.col+"lock", [start_ts_, start_ts_]))
		{ 	//确保提交时，主的锁还在
			return false; // aborted while working
		｝
		
		 // 写主行write列：｛commit_ts,数据存储时间戳｝
		 //Pointer to data written at start ts .
		T.Write(p.row, p.col+"write", commit_ts,start_ts_ );
		//清理主行lock列从0-commit_ts为止的lock数据
		T.Erase(p.row, p.col+"lock", commit_ts);
		
		//行事务提交
		if (!T.Commit()) 
			return false; // commit point

		// Second phase: write out write records for secondary cells.
		for (Write w : secondaries)
		{
			bigtable::Write(w.row, w.col+"write", commit_ts, start_ts_);
			bigtable::Erase(w.row, w.col+"lock", commit_ts);
		}
		return true;
	}
} // class Transaction

		Figure 6: Pseudocode for Percolator transaction protocol
```

在commit的第一阶段（prewrite 核心$\color{red}加锁修改数据$），我们尝试锁定被写的所有cell。（为了处理client端的异常，我们随机指定一个锁为主锁；我们稍后会介绍这个机制）。事务通过读待写入的所有cell的元数据来检查是否存在冲突。有两种冲突元数据：如果事务在startTimestamp之后看到另外一个写记录，说明自己过期，需要及时中断；这是快照隔离机制需要防范的一种写-写冲突。如果事务在任何时间戳看到另外lock列的锁标志，也需要及时终端。这是由于另外一个事务在已经提前获取开始时间戳并且预先提交结束后，commit第二个阶段太慢从而导致释放锁过慢，但我们认为这种情况不应该出现，所以我们中断。如果没有出现冲突，我们将该cell的lock列写入锁信息和data列写入最新的数据信息。

如果没有cell冲突，事务将开始提交并且进行到第二阶段($\color{red}解锁使数据可见 $)。第二阶段的开始，客户端从全局时钟获取commit时间戳。然后对于每个cell节点（优先主），客户端释放锁并且使得写入对读可见。这通过则更加write列记录实现。write列记录向读者指示该cell中当前提交的数据时间戳；包含一个到start 时间戳的指针，通过该指针，读请求可以找到实际的数据。一旦主cell上的写对外可见，该事务必须保证提交成功（$\color{red}即如果从提交失败需要不停重试$）。

一个Get()请求首先检查从<0-当前开始时间戳>范围内的的lock列，该范围是事务快照的可见时间戳范围。这段时间范围如果存在锁，说明另外一个事务正在写这个cell，所以此时读事务需要等待写事务完成锁得到释放。如果没有冲突的锁出现，Get()请求读取write列中最新的已提交的数据并且返回。

事务处理在client端失败时会变得比较复杂（而tablet server的异常对系统并没有影响，这是由Bigtable保证在节点异常时，写锁会持续存在$\color{red}即Bigtable保证三份要么都成功要么都不成功$）。如果在提交的过程中Client失败，锁将被遗留下来。Percolator必须清理这些过期的锁，否则这些锁会导致将来的事务永远hang住。Percolator采用了一种懒惰的方式进行清理：当一个事务A发现另一个事务B遗留下来的冲突锁，A可以决定B已经失败并删除该锁。

但是A每次都要精确的判断B的失败非常困难；实际上我们必须保证避免下面的临界区：A在删除B的事务锁的同时B正在进行提交该事务。Percolator通过在每次事务或者删除操作中指派一个cell作为一个同步点来巧妙的解决这个问题。这个cell的锁称之为主锁。事务A和事务B对哪一个lock作为主锁达成一致（主节点的位置写入在其他从cell的cell中的lock列中）。执行清理或者提交需要修改主cell的lock；由于这种修改是在Bigtable的行事务中进行的。所以清理操作和提交操作中只能有一个执行成功。特别的：在B提交之前，必须检查他仍然持久主锁并进行数据更新（见代码主节点提交时的逻辑）（$\color{red}在B将主Cell提交结束后，直接对所有的从执行提交即可，此时无需在检查主锁是否存在,因为主提交成功代表整个事务成功，从Cell必须保证成功$）。在A删除B的锁之前，A必须检查B事务的主锁保证B当前尚未提交成功；如果主锁仍然存在，他就可以安全的删除这个锁。

当client在提交的第二阶段中途crashes时，事务将超过提交点（最少已经写成功主cell）却并没有完全成功。我们必须在这些事务上执行前滚操作。一个事务如果遇到锁问题可以通过检查主锁来区分出两种场景：如果主锁write列已经被替换为新的记录，该事务已经提交成功，这个锁必须向前滚动，否则需要回滚（这是由于我们总是先commit主，我们能保证当主节点没有commit时回滚是安全的）。向前滚动，该事务需要模拟正常的流程 执行删除lock列锁同时增加write列指向数据记录。

由于删除主锁是一个同步操作，一个client清除其他存活client的施加锁是安全的；可是，由于回滚操作会强制事务中断，这样会导致整体性能下降。所以，$\color{red}一个事务只有在怀疑主锁所属的节点已经挂掉或者阻塞时才会尝试清空该锁$。Percolator使用一种简单的机制来判断另一个事务是否有效。运行的worker会给chunbby 锁服务写入一个token来表明自己属于这个系统。其他wokers可以根据这个token是否存在来推断这个worker是否存活（进程退出时会自动删除token）。为了处理worker虽然存活但是没有正常工作，我们给lock列写入一个wall time。这样即使wokers的token有效但是该lock包含的wall time太旧，该lock也会被删除。为了处理耗时比较长的commit操作，worker会在提交的过程中周期性的更新其wall time。

```
备注：
为了防止事务回滚可能导致增量更新的性能下架，需要尽量量化出哪些场景确实需要回滚。

这里主要分处理三种Case：
1. 	事务B所在的进程挂掉：			此时可以通过分布式锁服务chubby的令牌环token是否存在来判断。
2. 	事务B进程存在，但是长期阻塞：	  此时可以通过墙上时钟wall time来判断；
3.	事务B进程存在，但是commit本身耗时太长：	事务B周期更新其wall time；

实际上，感觉1必要性不高，完全可以通过2取代。进程挂掉时锁状态的表现和进程长期阻塞一致的。
```

## 2.3 时间戳

oracle timestamp时钟服务用以提供严格单调递增时间戳。由于一次事务操作需要访问时钟服务两次，这个服务必须有很好的可扩展性。oracle服务周期性的分配出一段连续的时间戳并将其最大值写入可靠存储中。这样其可以通过内存操作来对外提供一段时间戳的分配并满足严格有序。如果oracle服务重启，当前时间戳从持久化中保存的最大的timestamp开始。为了节省RPC开销，每个Percolator worker通过维持一个到oracle的RPC连接实现跨事务的时间戳批量请求。当oracle服务变得过载，批处理可以自然的降低负载。批处理可以增加oracle服务的可伸缩性但是不会对时间戳保序产生影响。我们的单台oracle服务单节点可以对外提供2百万词/s的时间戳分配。

事务协议使用严格递增的时间戳来保证Get()操作获得 截止当 前时间戳为止所有提交的写事务的最新数据。让我们看下这是如何保证的，考虑一个事务R获得开始时间戳为：T~R~，事务W在时间戳T~W~进行事务提交，并且存在T~W~ < T~R~; 此时R事务可以看到W事务写的结果。由于 $\color{blue}事务T加锁$< T~W~ < T~R~  < $\color{blue}事务R读取数据$, 我们可以推断出: oracle 时钟服务分配时间戳T~W~是在分配时间戳T~R~之前或者在同一批。所以，事务W请求T~W~在R收到T~R~之前。我们知道事务R只有在获得开始时间戳T~R~之后才可以开始读，同时事务W在请求commite时间戳T~W~之前已经将锁写入到lock列。因此，上面的原则保证了：事务W写完所有行的锁之后R才开始读取数据；事务R的Get()操作将会看到事务W操作的记录或者W操作的锁。任何一种场景事务R都会阻塞直到锁得到释放。无论那种情况，事务W的写都会被事务R的读可见。

## 2.4 通知

事务让用户在保证不变量的同时对Bigtable表进行修改。但用户也需要有一种方式来触发并且执行事务。在Percolator中，当Bigtable内容发生变化时会触发用户编写的特定编码（observers）从而执行特定的事务逻辑，技术上，我们将所有的观察者代码片段集成在Percolator二进制代码中，该逻辑和系统的tablet服务一起运行。每个观察者注册一个函数和Percolator的若干列。Percolator在数据写入到任何行的那些列时会调用其注册函数。

Percolator应用封装为一系列observers。每个observer完成一个任务并且为下游的observer创建新的任务。在我们的索引系统中，一个MapReduce通过执行$\color{blue}装载事务$将爬取的文档加载到Percolator，这将导致$\color{blue}文档处理事务$来对文档建立索引（分析、提取链接）。文档处理事务触发更多的事务比如$\color{blue}聚集事务$.反过来，聚集事务将触发将发生变化的文档集合导出到服务系统中的$\color{blue}导出事务$。

 通知和数据库的触发机制或者活动数据库的事件比较类似，但数据库的触发机制不是被用来维护数据库不变量。而且，被触发的observer在被触发时在独立的事务中执行。所以触发的写和被触发的observer的写之间没有原子性。通知机制用来帮助对增量计算进行结构化而不是帮助保证数据完整性。

相比较复杂的语义上有重叠的触发器机制，这种语义上的不同和意图导致observer的表现非常容易理解。Percolator应用包含很少的observers -- Google索引系统有大概10个观察者。每个观察者都是显式的在worker二进制的main函数中进行构造。这使得多个观察者观察同一个列成为可能，但是我们尽量避免这个特性以便于当一个特定的列被写入后触发一个明确的observer。用户不必担心通知的无限循环，但是Percolator并没有为此而额外做更多工作；用户一般会创建一系列observer来避免无限循环。

我们提供一个保证：对每个被观察的列，当该列每次变化时最多只有一个关联observer的事务可以提交。相反的情况并非如此。但是对同一列的多个写将可能导致只触发对应的observer一次。我们称之为消息奔溃，通过分摊多个通知的响应处理这样有助于避免多余的计算。比如http://google.com网页被周期性重新加载处理就已经足够有效并不需要没有发现有个指向该网页的链接时都要加载处理http://google.com。

为了提供这种语义级别的通知机制，对每个观察者而言，其注册的列都有一个伴随的“确认”列，包含该观察者最近一次运行的开始时间戳。当这个被观察者注册的列写入了一项数据，Percolator开始一个处理这个通知的事务。这个事务读取被注册的列和伴随其的“确认”列。如果被注册的列在伴随列标记的时间戳之后被写入，我们执行observer并且将确认列同步到最新写入的开始时间戳。否则表示observer已经执行过 ，所以我们没有必要再次执行。注意到如果Percolator如果对伴随列同时启动两个事务，但是一个事务会由于其在确认列的操作存在冲突而被终止。我们保证了最多一个观察者可以对每次通知进行最终的commit。

```
备注：
当被注册的列（即数据列）执行写入时，会唤醒注册在该列的观察者线程事务。

该事务在执行时会首先检查注册列（数据咧）和伴随列的时间戳，
	case1：如果最近的开始时间戳一致，则说明observer已经执行过，无需再次执行，减少了额外的执行。
	case2: 会执行observer逻辑并且同步时间戳。

假定被注册的列（即数据列）执行写入时，突然唤醒了注册在该列的多个观察者线程事务
虽然observer逻辑会并发执行，但是由于其在确认列的操作存在冲突，最终只能有一个成功。

这样就保证了每次唤醒最多只能触发一个观察者提交成功
```

为了实现通知，Percolator需要有效的发现脏的cells以及相关的需要被触发运行的observers。由于通知非常稀疏导致这种搜索非常复杂：我们的表有万亿的cells，但是，$\color{blue}如果系统和应用负载保持一致$，将仅仅有数百万的通知。而且，observer代码分布式的遍布在大量的客户端进程上，这意味着这种对$\color{blue}脏cells的查找必须分布式$的。

为了确定脏的cells，Percolator管理了一个特殊的称之为“通知”的Bigtable列。对每个脏的cell包含了一个条目。当一个事务将数据写入到注册的数据列，同时会设置对应的(notify)通知列。workers在通知列中进行分布式的查找来确定脏的cells。当一个observer事务被触发并且该事务提交，我们会删除该notify列。由于notify列仅仅作为Bigtable的一个列(并不是Percolator列)，他并不没有事务的特性而且只作为扫描检查确认列来确定是否要执行observer事务的线索。

为了让这种扫描足有有效，Percolator将notify列存在在Bigtable一个单独的局部组中，这样扫描所有的列需要读取仅仅数百万的脏的cells，而不是数十亿的所有的数据cells。每个Percolator工作线程有若干个线程负责扫描。worker通过在Bigtable中随机选则一个tablet分片，然后从tablet分片中跳出一个随机的key，最后从那个key的位置开始扫描表。当worker选择出表的一部分后将其分配给每个线程。由于每个线程都在扫描Bigtable表中的随机的一段，我们担心两个工作线程并发运行同一行的observers。虽然由于事务通知机器的特性这并不会导致正确性问题，但是这是低效的。为了避免这种情况，每个worker在扫描一行前从一个轻量级锁服务中获得一把锁。锁服务由于是建议锁所以并不需要持久化这些状态，因此伸缩性非常好。

随机扫描阶段需要注意到下面的问题：$\color{blue}聚集效应$。当第一次部署执行时我们需要明确扫描线程可能会倾向于聚集在Bigtable表中的少量区域。有效的减少了扫描的并行性。这种现象在公共的交通系统中比较常见，称之为"队列"或者"巴士聚集"并且在一个公共汽车执行缓慢（由于交通或者装载慢）。由于每个站台的乘客数一直在增长，装载慢变动更加严重，更加减慢了公共汽车的速度。同时，这个缓慢巴士后面的任何巴士由于其在每个站台只需状态更少的乘客其速度却提升。这将导致多个巴士同时到达某一站。我们的扫描线程表现非常类似：一个执行observers的线程会慢下来，同时后面的线程却快速的跳过后面的行，但是由于聚集线程在tablets server重载部分过多，导致后面的线程无法超越前面的线程。 为了解决这个问题，我们用了一种公共交通系统无法实现的方式改造了系统：当一个扫描线程发现其和另一个线程扫描同一个行，他在Bigtable表中再重新选择一个位置继续扫描。继续拿交通运输作为对比，巴士（扫描线程）如果在发现距离前一个巴士过近时可以将自己瞬移到另外一个随机的站台（表的位置）就可以避免聚集。

最终，过往经验让我们发明了一个轻量级但是弱语义的通知机制。我们发现当具有相同内容的很多网页通过执行时，每个事务将会触发重新处理相同的集群从而导致冲突。这让我们设计了一种无需事务冲突却能通知cell的方法。我们通过只写Bigtable的notify列实现这种弱化的通知。为了保留Percolator其他部分的事务语义。我们重新限制这种弱通知机制到特定类型的不可写列，只能通知。这种弱化的语义也意味一个弱化通知可能触发多个观察者可能运行并且提交（虽然系统尽量保证最小化并发）。这已经变为管理冲突的重要的特性；如果一个observer频繁的在某个热点上出现冲突，通过一个非原子性的通知将其拆分为两个关联observers会非常有效

## 2.5 讨论

相比MapReduce类型的系统，Percolator的一个不足在于每个工作单元的RPCs数量。MapReduce可以从GFS服务执行高吞吐量读请求并且获得所有的Web网页。Percolator为了处理一个单独的文档需要执行大概50个独立的Bigtable操作。

其他RPCS的一个来源发生于commit操作。当写一个锁，我们必须进行一个read-modify-write操作，这需要连啊个个Bigtables RPC交互: 一个读用于发现冲突锁或者冲突写；另一个写用于添加新的锁。为了节省这种开销，我们修改了Bigtable的API，增加了一个带条件修改接口可以在一个RPCS中完成read-modify-write操作。许多注定集中在一个tablet server的有条件的修改也可以被批量到一个单独的RPC中，这样可以更近一步节省交互的RPCS。我们通过延迟锁操作若干秒来将操作汇聚成批量以便进行批量化。由于会被并行的获取，这将导致每个事务延迟很短时间。我们通过更大的并行化对额外的延迟进行补偿。批量化会增加冲突发生的可能性，但是在我们的低竞争环境中这个问题并不明显。

我们在从table读数据时也执行类似的批量化：每次读取操作可以延迟特定时间以便有机会在同一个tablet server上聚集更多的读请求。虽然延迟了每次读，很大概率增加了事务的延迟。最终的优化减轻了这个影响，但是：预取利用了读取一行数据的两个或者更多的数据在延迟上实质和读取一个数据一样的事实。在任意情况下，Bigtable必须从文件系统中读取整个SSTable块并进行压缩。Percolator尝试预测，每次一个列被读取，该行的哪些列将会在稍后在同一事务中被读取。这种预测基于历史的行为。预取结合tiems缓存将Bigtable读文件爱你系统的次数降低了10倍。

在Percolator实现的早期版本中，我们决定让所有的API调用 阻塞，通过每个机器上运行数千个线程来提供足够的并发性来实现比较好的CPU利用率。我们选择每个请求对应一个线程的模型相比时间驱动的模型主要为了让应用程序写起来简单，强制用户每次从Bigtable表中获取数据元素时存储中间状态将使得应用开发非常困难。我们的采用每请求对应一个线程的经验整体上看是正面的：应用代码足够简单，我们在多核节点上获取了足够的利用率，而且crash调试由于完善的栈追踪机制非常简单。我们遇到了比我们担心更少的应用层竞争条件。这种实现最大不足在于基于Linux的内核以及Google的高线程基础设施中下的可扩展性。我们内部的内核开发团队正在部署程序来修复内核相关问题。

#3 评估

Percolator的性能介于MapReduce和DBMS系统之间。比如，因为Percolaotr是一个分布式系统，他使用一个远远大于传统DBMS系统的资源来处理固定数据的数据；这是基于其可扩展性的代价。相比较MapReduce，Percolator能处理延迟更为敏感的数据，这是基于支持随机查找需要的额外资源的代价。 这些都是难以量化的工程权衡：多少性能损失对于系统能力是不可接受的  和 为了简单增加容量需要购买更多的设备 之间的权衡。或者分层系统提供的开发周期节省 和 相应的系统效率下降 之间如何权衡。

这部分我们将通过对比用类似MapReduce方式的批处理系统来网页增量索引的实践以及Percolator的相关经验来解答一些问题。我们也将利用微基础测试和基于著名的TPC-E基础测试的何曾测试的方式来评估Percolator。这些测试将让我们对Percolator相比较相对Bigtable和DBMS在可扩展性和效率的表现做出评估。

这部分的所有实验是基于Google数据中心的一些节点上的运行获得的。这些节点基于X86处理器并且安装了Linux操作系统；每台机器连接了多干个商业级别SATA磁盘。

## 3.1 与MapReduce对比

我们设计了Percolator来创建Google‘s巨大的“base”索引。该任务之前是通过MapReduce实现的。在之前的系统中，每天我们爬取数十亿个文档然后通过超过100次 MapReduce操作将其和已经存在的文档库集成在一起。结果就是对所有文档建立索引来响应用户的查询。尽快100次 MapRedces操作并不是全局作用于每个文档的关键路径，将系统组织为一系列MapReduce操作意味着每个文档在可以被作为结果返回之前需要花费2-3天时间才能建立完索引。

基于Percolator的索引系统（也即Caffeine）爬取同等数据的文档，但是我们在爬取后会立刻开始建立索引。及时的优势，也是Caffeine系统主要的设计目标就是降低延迟：一个文档插入到索引系统的平均延迟比之前的系统快100倍以上。在系统变得越复杂这种延迟的提升越明显：相比基于Percolator系统添加一个新的集群需要对每个文档都进行一遍额外的查找，而Percolator只需要对存储库做一次扫描。建立新集群阶段也可以通过在一事务完成而一定要用一个新的MapReduce；这种简化使得相比之前MapReduce方案需要超过100个MapReduce操作，Caffeine系统只有大概10个左右Observers，数量大为减少的一个原因。这种组织方式也允许只通过对存储的一部分子集进行处理即可成为可能，而不必向MapReduce一样重新扫描整个存储仓库。

基于现有集群增加新的文档增量系统中不是一点问题没有：为了让系统能跟上输入需要更多的资源，但是相比较批处理这种由于网页在每次MapReduce操作中会在存储集群间到处游荡的策略导致没有足够大资源可以解决延迟而言，Caffeine仍然是一个巨大的提升。由于少量的慢操作导致大部分处理都没有发挥最大并发能力导致在基于批处理的索引系统出现一系列问题，这在Caffeine中是完全不存在的。新系统的低延迟特质也让我们可以模糊介于 大规模、更新缓慢的索引 和 小规模、快速得到更新的索引的严格界限。因为Percolator在每次对文档建立索引时不再处理整个存储集群。我们也可以将规模变得更大：Caffeine‘s目前收集的文档规模是旧系统的3倍以上，并且仅仅受限于磁盘容量。

相比被他替代的系统，同样的爬取速度Caffeine使用大致两倍左右的资源。但是，Caffeine对资源的有效利用率很高。如果我们在两倍的资源之上运行旧的索引系统，我们或者会增加索引大小或者节省延迟为2倍左右。另一方面，如果Caffeine在只有一半的资源上执行，先比老系统，他或许可能无法每天处理同样数据量的文档（但是他处理的每个文档的延迟非常低）。

新系统也非常便于操作。Caffeine基本没有移动的部分：我们运行tablet server，Percolator worker，chunkservers。在旧系统中，将近100个MapReduce操作中每个都需要独立的配置，而且可能失败。而且，相比Percolator对资源非常平滑的使用，MapReduce工作负载的“peaky”特性使得他很难完全利用数据中心的资源。

API代码的简洁性和对存储进行随机查找的能力使得基于Percolator开发新的特性非常容易。对MapReduce而言，随机查找非常尴尬且耗时。另一方面，Caffeine开发者需要考虑并发性，这在MapReduce中并不存在。事务帮助处理并发，但是无法消除引入的复杂性。

对从MapReduce到Percolator的演进带来的收益进行量化，我们创建了从在数十亿文档库中增加新的爬取的文档到和采用Google的索引管道操作类似的删除重复文档方案的综合基准。文档通过3个集群化关键字进行集群化。真实系统中，集群key可能是文档的类似重定向地址或者一致性hash之类的特征。但是在本次实验中，我们从750MB可能的Key的集合中随机均匀的选择。在合成库中平均集群包含3.3个文档。93%的文档在多个集群中存在。keys的聚类逻辑测试的这种分布不会暴漏给真实环境我们看到的非常大的集群中。这些聚类只会影响延迟末端而不是我们展示的结果。在Percolator聚集实践总，每个爬取的文档会立即写入到仓库并被observer可见。observer对每个聚集的key维护了一个索引表并且对文档和索引进行对比来决定是否是重复的。MapReduce对持续到来的文档通过重复的执行3个连续的MapReduce（每个MapReduce处理一个集群Key）来实施集群化。对整个集群执行3个MapReduce处理序列同时任何新爬取的文档进行累加，以便进行下一批操作。

这个时延模拟了按照一定的速度爬取文档。爬取速度和存储集群大小是决定MapReduce或者Percolator的表现的关键因素。我们通过将存储池大小固化调整爬取速度进行分析，表达为每小时爬取占存储的百分比。实际上，每小时只能爬取占存储量很低比例的数据：Web系统中有大概1万亿个网页，一天只能爬取非常少比例的量。当新爬取的量非常少时，我们期望Percolaotr比MapReduce表现的更好，这是因为MapReduce为了将新的文档批量加入集群需要对整个存储池进行操作而Percolator只是按照比例对小量的新爬取的网页进行操作（每个文档需要查找最多3个索引表）。而在爬取速率非常高时，MapReduce表现要比Percolator好。这个交叉之所以会发生是因为相比较对字节的随机查找而言磁盘的流式传输效率最高。在交叉点，Percolator总的耗时和MapReduce的总耗时相等。

我们在240台机器上执行了这项测试，并且测量了文档从爬取到被索引的时延中位数。表格7给出了当爬取速度变化两个处理方式对文档处理的延迟的中位数。当爬取速度非常低，Percolator索引文档的要快于MapReduce。这个场景在最左侧的点展示，此时的爬取速度为文档总大小1%.MapReduce需要接近20min才能完成集群化，因为3个MapReduce就需要20min。这导致了从爬取到被索引需要30min左右：平均一个被爬取的随机的文档需要等待10min才能等待前一批次MapReduce的结束，并开始下一轮的开始。Percolator，另一方面找到了加载文档的新思路，平均可以在2s，大致提升1000倍。2s包含了：找到脏的通知并且执行集群化的事务。需要注意的是：1000倍的延迟是会随着存储量的增加显著的放大。

当爬取速度逐渐增加，MapReduce的处理同时增大。理想情况下，这和存储的总大小应该是成比例的。实际上，测试的小的MapReduce被文档本身所约束，所以处理时间的增长和爬取速度相关性比较弱。当爬取速度达到6%，只给1TB的数据集增加了150GB的量；处理150GB的数据量只是个噪音。Percaolator的延迟在一定爬取速度下基本不变，然后在爬取速度增加到40%突然显著的增加到无穷。在这个点上，Percolator受限于测试集群的资源导致处理能力跟不上爬取速度，开始将未处理的文档加入到一个无限的queue中。40%是Percolator的性能分解点。MapReduce性能基本不变；最终新爬取的累计的文档会超过MapReduce的处理速度。批量大小在后续运行中没有边界。在特定的配置中，MapReduce支持爬取速度超过存储总大小。

这些结论表明，在我们期望的真实系统中Percolator相比MapReduce而言具有几个数据级的延迟方面的优势（单位数的爬取速度）

![延迟中位数](E:\Paper\Google论文集合\论文翻译\延迟中位数.png)

## 3.2 微基准测试

本部分我们考虑Percolator提供的原子性语义的消耗。本次实验中，我们将Percolator和原生的Bigtable表进行对比。由于Percolator是基于Bigtable之上，Bigtable的任何优化带来的性能提升也会直接传到到Percolator中，所以这里我们只对Bigtable和Percolator的相对性能感兴趣。表8展示了单台Tablet节点上Percolator和原生的Bigtable的性能对比。实验过程中，所有的数据都在tablet的cache系统中缓存同时禁掉Percolator的批量优化功能。

![1531360992385](E:\Paper\Google论文集合\论文翻译\Bigtable对比.png)

和预期一致，Percolator相对Bigtable引入了额外的开销。我们首先对比了两个系统对随机写的表现。就Percolator，我们执行一个事务对一个cell进行写入然后进行提交；这代表了Percolator最差的场景。当执行写入，Percolator在该基准测试中增加了四倍额外的开销。这是由于Percolator需要执行 除了Bigtable单独写一次之外还需要一些额外的操作：为了检查锁而需要的读，为了加锁而需要的写，为了删除锁记录而执行的写。尤其是读操作比写更花费时间并且占用了绝大部分开销。本次测试中，tablet节点成为约束性能的瓶颈。所以获取时间戳的耗时没有被考虑，我们也测试了随机读：业务的每次读在Percolator会在Bigtable转化为一次操作，但是读操作比原生的Bigtable操作某种程度上更加复杂（Percolator除了需要读取数据列之外也需要读取元数据列）。

```
相比原生Bigtable操作，Percolator存在：由于两阶段提交导致读写次数多（3）次。读操作更重两个问题。
```

## 3.3  合成工作

为了测试Percolator更加实际工作负载下的表现。我们基于TPC-E设计了合成测试基准程序，由于TPC-E主要设计目标是OLTP系统，所以对Percolator而言并不是最优的基准测试标准，而且Percolator实现时的一系列权衡对OLTP系统的特性都有影响。TPC-E是一个被广泛认可和理解的基准测试，但是，这让我们对Percolator系统相比较传统的数据库的表现有更好的理解。

TPC-E模拟了经纪公司为用户进行交易，搜索，查询方面的请求。经纪公司向市场交易所提交交易订单来执行交易并且更新经纪人和客户的状态。基准测试对一系列执行的交易进行衡量。平均而言，每个客户500秒执行一次交易，所以基准测试通过增加客户和相关的数据来进行扩展。

TPC-E通常包含3个部分-客户模拟端，市场模拟端，使用SQL逻辑的DBMS系统。由于Percolator是一个运行在Bigtable之上的客户端库，我们的实现将客户和时常模拟端绑定在Percolator的库上来对Bigtable执行操作。Percolator提供低级别的Get/Set/iterator API接口，折合高级别的SQL接口不太一致，所以我们创建索引并且人工执行所有的查询计划。

由于Percolator是一个增量处理系统而不是OLTP系统，我们并不想满足TPC-E的延迟标准。整体看，事务延迟在2-5秒，但是异常情况可能需要好几分钟。异常情况一般是由冲突下的指数退避策略或者Bigtable不可用。最终，我们对TPC-E事务做了微小的调整。在TPC-E中，每笔交易结果会增加经纪人的佣金和交易次数。每个经纪人服务于上百个客户，所以每个经纪人大致5s就会被更新一次，这在Percolator中会导致重复的写冲突。在Percolator中，我们可以通过将写旁路到另一个表中然后周期性对每个经纪人的加过进行聚合来实现这个特性；对基准测试，我们选择省略这种写。

表9展示了当需求增加是Percolator的资源利用。我们通过CPU 核来衡量资源，这是因为CPU是当前的约束因素。我们通过少量几台机器来完成测试，但是我们的Bigtable cell在更大范围集群上共享磁盘资源。结果就是，磁盘带宽并不是系统性能的关键因素。在本试验中，我们通过配置逐步调整客户数量来衡量当前性能表现和使用的cpu核等资源量。性能和资源使用在多个数量级上都是线性增加，从11个核到15000个核。

![TPC-E基准测试](E:\Paper\Google论文集合\论文翻译\TPC-E基准测试.png)

本实验同时提供了相对DBMS而言的Percolator的开销。今天最快的商业TPC-E系统处理使用64个Intel Nehalem核心（每核心两个超线程）的共享内存机器单台可以处理3183 tpsE。我们使用15000核的TPC-E性能达到11200.这种比对非常粗略：Nehalem的核的能力比我们Bigtable中的cpu 单核能力高出20%-30%左右。但是，我们估算对每笔事务Percolator使用了大概30倍的cpu资源。从事务开销的角度看，由于我们的测试集群性比较基础测试的企业其硬件而言使用了更便宜的、商品硬件，因此差距小于30。

实施数据库的传统思路是: 由于操作系统结构像磁盘缓存和调度器都使得实现一个高效的数据库非常困难因此尽量直接使用硬件，即要接近硬件。Percolator中，我们不仅在数据库和硬件之间插入了操作系统，并且插入了多层的软件层和网络连接。传统思路是对的：这种安排会带来开销。这里有大量的开销花费在准备请求，发送他们，在远端节点处理他们。为了展示这些开销，考虑下改变行为的数据库，在DBMS中，这导致了将数据进行内存存储的函数调用并且系统调用强制在RAID阵列的磁盘上写入一条日志。Percolator中，客户端实施事务时需要到Bigtable的很多个RPCS调用，这些修改的提交会被3个chunkserver进行日志记录。最终调用系统调用进行磁盘淘汰。稍后，这些数据会被整理为minor以及major的sstables，每个sstable通过央需要被复制到所有的chunkserver中去。

CPU 膨胀是我们分层的结果。作为交换，我们获得了可扩展性（我们最快的结果，及时不和TPC-E相比，相比现有官方记录也有3倍的提升），而且我们继承了现有系统的好的特性，包括容错。为了展示后者，我们在15个tablet节点上运行基准测试并且性能稳定。表10展示了随着时间变化系统的性能数据。在17：09时由于一个失败导致出现性能下降；我们杀死了1/3的tablet服务。性能在异常后急剧下降，但是在tablet被其他节点加载后逐步恢复。我们让被杀死的tablet server重启，性能逐步返回到最初的值。

![1531366785750](E:\Paper\Google论文集合\论文翻译\性能波动.png)

# 4 相关工作

批处理系统比如MapReduce对高效的转换以及整个层面的分析非常适合：这些系统可以同时使用大量的节点去快速的完成处理海量的数据的处理。除过这种可扩展性，在每批只有少量重新时开始一个MapReduce管道会导致无法接受的延迟和浪费。重叠或者流水线相邻的阶段可以减少延迟，但是处理缓慢的分片仍然限制这完成一个流水线的最快时间。Percolator可以避免这个重复扫描的开销。本质上，对加入集群的文档的key建立索引； Stonebraker and DeWitt 在对MapReduce最初的批评就是MapReduce不支持类似索引。

一些对MapReduce的修改的提议通过允许工作者对基准存储仓库可以随机读 同时 仅仅对新到来的工作做做Map映射来减少对存储池进行变更的处理开销。在这些系统中为了实施集群化，我们可以在集群化的每一个阶段维护一个存储池。通过避免重新re-map整个存储池可以允许我们的批处理更少，减少延迟。DryadInc通过 从之前的运行中重用相同部分的计算 同时允许用户配置merge函数来将新的输入和之前遍历的结果进行整合 这两种方式 也对这个问题进行表达异议。这些系统代表了在整个存储池中使用MapReduce 和 任何时间只处理单个文档的Percolator之外的中间路线。

数据库基本满足增量系统的大多数需求：一个RDBMS可以对大容量文档库执行很多独立和并发的修改，并且提供了非常灵活支持计算处理的SQL语言。事实上，Percolator提供了类似数据库的接口：支持事务，遍历，二级索引。Percolator提供分布式事务，这绝对不是一个成熟的DBMS：他缺乏查询语言，比如关系操作如join。Percolator也被设计为处理相比已经存在的并行数据看更大规模的数据，而且对节点异常容忍度更好。和Percolator不一样，数据库系统倾向于注重延迟而不是吞吐，这是由于人们经常对等待数据库查询的返回。

Percolaotr中，数据的组织是非共享型并行数据库的镜像。数据以非共享的形式分布式的部署在一系列廉价服务器上：节点之间智能通过显式的RPC调用进行通信；节点间没有使用共享内存或者磁盘。Percolator使用的数据 被Bigtable按照连续的行分区为tablet并被散布在节点中。这种去聚合的形式镜像自并行数据库。

Percolator的事务管理建立在数据库系统分布式事务的细一些工作之上。Percolator 通过使用两阶段提交技术扩展了分布式系统的多版本时间戳排序技术实现了快照隔离。

Percolator中推动系统逐步变为一个“clean”的状态的的observers角色 和  传统数据库中的物质视角的增量维护 可以进行类比。实践中，当一些索引任务比如通过内容对文档内容聚类工作可以被表达为一种增量视图维护，这种方式非常难以表达一个原生文档转化为一个被索引的文档的方式。

```
Percolator中observers的角色完成类似传统数据库增量视图维护的工作
```

并行数据库的实用程序，比如扩展出类似Percolator的系统，在他们的历史上被质疑很多次。硬件发展趋势一直和并行数据看相背离。CPUS的发展相比磁盘更加快速导致在一个共享内存的主机中很少量的CPU就可以驱动足够多的磁盘去提供足够的服务能力，不用考虑分布式事务的复杂性。TPC-E基准测试的最好的结果是通过连接到SAN的大型共享内存机器实现的。这种趋势开始出现反转，但是，当海量的数据集需要被处理时这种需求将远远超出一个节点的能力。这类数据集需要可以扩展到1000节点之上的分布式的解决方案，但是现有的并行数据库最大可以使用100台左右的机器。Percolator相比并行数据库 通过牺牲一些灵活性和低延迟提供了一个 面向网络数据集的具有足够扩展性的系统。

分布式存储系统比如Bigtable有MapReduce的可扩展性和容错性，但是为存储提供了更加自然的抽象。由于系统可以通过修改存储而不是重写来进行状态变更使得分布式存储系统允许低延迟更新。但是，Percolator不仅仅是一个数据存储系统，也是一个数据转换系统：他提供一种对计算进行格式化来对 数据进行转换的方式。作为对比，像Dynamo，Bigtable，PNUTS之类的系统提供高可用的数据存储但是并没有对应的数据转换机制。这些系统也可以被归纳为NoSQL数据库（MongoDB）：他们都提供高性能和可扩展性想退传统数据库而言，但是只提供很弱的意义。

Percolator通过分布式系统的多行事务扩展了Bigtable，而且提供了observer接口允许应用以通知的形式被格式化的修改数据。我们考虑在Bigtable上直接建立新的索引系统 。但是对没有强一致性的的并发状态修改的需求其复杂性让人生畏。Percolator没有继承Bigtable所有的特性：他跨数据中心的多副本支持非常有限。比如，由于Bigtable跨数据中心的多副本机制仅仅是每个tablet视角的强一直，在一个分布式事务中副本有可能破坏不变量。不像Dynamo或者PNUTS直接响应用户的系统，Percolaotr为了更强的一致性可以接受单数据中心较低的可用性

一些研究系统，类似Percolator，扩展分布式存储系统来提供强一致。Sinfonia为分布式存储提供原子接口。早前发布的Sinfonia版本也提供类似于Percolator的observer的通知机制。Sinfonia和Percolator的不同在于内部应用：Sinfonia被设计用于创建分布式基础设施，而Percolator直接被应用使用（这可能表明为什么Sinfonia的作者丢弃通知机制）。而且，Sinfonia的最小-事务机制的语义有一定约束，不像RDBMS或者Percolator：用户必须配置一系列对比、读、写然后才能提交事务。这种机制对于创建各种各样的基础设施是足够的但是却不能限制应用的使用者。

CloudTPS,类似Percolator，在分布式存储系统（HBASE或者BIgtable之上）建立了ACID-兼容的数据存储。Percolator和CloudTPS不同点在于设计，但是：CloudTPS的事务管理层是被一个称为本地事务管理的中间层负责，其会缓存修改直到最终才写到下层的分布式存储系统。作为对比，Percolator使用客户端，直接和Bigtable交互来协调事务管理。系统的关注点也不同：CloudTPS作为website的后端，对延迟和分区容忍相比Percolator有着很强的关注。

ElasTraS，一个事务数据存储系统，架构上类似于Percolator；ElasTraS事务管理实质上是tablet节点负责。不像Percolator，ElasTraS当动态分布数据集合时提供有限的事务语义，而且不支持结构化计算。

# 5 总结和后续的工作

我们已经创建而且部署了Percolator，而且从2010年5月已经应用于Google的网页索引。这个系统完成了在可接受的资源增加的前提下降低索引单个文档的延迟的目标。

TPC-E结果表明未来工作的重点，我们选择了对廉价服务器进行数量级级别的可扩展的架构，但是我们已经看到这花费了30%额外的负担。我们对探索 这种权衡和描述这个开销的本质非常感兴趣：多少是分布式存储系统必须的，多少是可以被优化的？

# 6 致谢

如果没有很多同事和团队的帮助Percolator不会顺序上线。我们非常感谢索引团队的同时，我们的种子用户，基础团队其他方向的开发者都在不遗余力的优化服务最终满足我们越来越多的需求。