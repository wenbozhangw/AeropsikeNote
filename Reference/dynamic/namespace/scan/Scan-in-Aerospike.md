## FAQ - Scans in Aerospike

### Note
Aerospike 4.7版对Aerospike扫描子系统进行了重大更改。该常见问题解答涵盖了4.7版之前的Aerospike扫描子系统；对于自4.7版以来的版本，请参阅 [ FAQ - Scans in versions 4.7 and above](https://discuss.aerospike.com/t/faq-scans-in-versions-4-7-and-above/7199) 。

---

### Synopsis
Scan 是 Aerospike 中常用的操作之一。该知识库涵盖了有关 Aerospike 中 scan 的一些常见问题，以在您的用例用有效地使用它们。

---

### -What are the different kinds of scans in Aerospike ?

在Aerospike中可以一次看到4种 scan 类型：

#### 1) Basic Scans :

 - 简单的扫描命名空间或者 set 中的所有记录。
 - 以下命令是使用AQL进行基本扫描的示例：
   - `aql> select * from <ns>.<set>`
 - Basic scans 由客户端启动，并且它们具有返回给客户端的 open socket。Basic scans 可以将输出返回给客户端。如果客户端超时或无法接收正在进行扫描的服务器返回的数据，它可能会则色有问题的 network socket，并最终有暂时停止扫描子系统的风险。

#### 2) Aggregation Scans : 

 - Aggregation scans 与 Basic scans 类似，但是希望它们通过 Stream UDF 返回一些操作后结果作为输出。
 - Aggregation scans 由客户端启动，并且它们具有返回到客户端的 open socket。
 - Aggregation scan 可以将输出返回给客户端。对于可能被卡住的扫描，类似的 disclaimer 也适用于 Aggregation scans .

#### 3) UDF Background Scans :

 - UDF Background scans 会遍历 set 或 namespace 中的所有记录，并将 UDF 应用于记录。
 - UDF Background scans 没有打开客户端套接字。它们已断开连接，并且不将输出返回给客户端。
 - 您必须监视扫描以检查进度。

#### 4) Sindex builder Scans : 
 - 这仅在创建二级索引时发生，因为需要读取记录才能填充存储在内存中的二级索引。
 - 这是作为客户端创建的二级索引的一部分启动的。

---

### - How does the scan subsystem work ?

#### Versions 3.6.0 and above

在服务器端，扫描子系统具有一组专用线程，这些线程集合由节点上的所有扫描共享。服务器端的扫描线程池由动态配置（`scan-threads`）控制。`scan-threads`的默认值为 4，最大值为 32。`scan-threads` 是 basic scans 的唯一限制。如果将此值设置得太高，则与扫描相关的活动可能会干扰其他读取或写入 transaction。如果设置得太低，扫描可能会失败。

扫描是交错排列的。一个线程从扫描队列中取出具有分区 ID 的 Job，然后扫描该分区。可以同时运行多个扫描，无需争用以轮循方式选择 Job。每个扫描一次扫描一个没有争用的分区。

在节点上扫描仅处理主分区。如果分区不是主分区，则扫描会跳过该分区。

Note : UDF background scans 会生成内部 UDF transactions。UDF transactions 转到 transaction system。UDF background scans 由 `scan-max-udf-transactions` 配置控制。`scan-max-udf-transactions` 的默认值为 32。可以动态更改。当 UDF transactions 达到此值时，transactions 将休眠。当 UDF transactions 数量下降到 `scan-max-udf-transactions` 值以下时，transaction 将恢复。请参阅， [scan udf throttling guide](https://discuss.aerospike.com/t/how-to-throttle-scan-udfs/5377/1) 。

例如，让我们考虑以下设置为 32 的 `scan-max-udf-transactions`。对于 UDF scan，扫描线程会将 32 个 transactions 排入 transaction queue。当尝试将第 33 个 transaction 放入队列时，如果前 32 个 transactions 中没有一个完成，则扫描线程将在重新尝试该 transaction 之前进入休眠状态。

#### Versions prior to 3.6.0

在旧的扫描中，扫描等待轮流进行。在第一次扫描完成之前，无法开始第二次扫描。第三次扫描等待第二次扫描，依此类推。这是一个巨大的限制，因为缓慢的较大扫描会轻易耗尽资源并停止后续扫描。

---

### - How many parallel scans can be run against a cluster ? 

问题：可以对一个群集运行多少个并行扫描？

扫描是通过客户端（例如 Java，Python甚至AQL）触发的。对于可以从客户端请求的扫描数量没有设置限制。但是，在服务器端，这由配置 `scan-max-active` 控制，该配置在每个节点的默认情况下为100。如果正在运行的扫描次数等于 `scan-max-active` 的值，并且您又提交了一次扫描，则数据库将返回以下错误 : "AS_PROTO_RESULT_FAIL_FORBIDDEN" 。

---

### - I have a 20-node cluster and I issued a scan from the client side. But, why do I see active scans on 16 of the nodes in the cluster?

问题：我有一个20个节点的集群，并且从客户端发出了扫描。但是，为什么我在集群中的16个节点上看到活动的扫描？

默认情况下，客户端可以发出的并行扫描请求数为 16（由客户端上的线程池大小配置，并在 AQL 上使用 "-z" 选项配置）。要一次在 16 个以上的节点上运行扫描，您需要更新客户端策略( E.g. [API in Java](https://www.aerospike.com/apidocs/java/com/aerospike/client/policy/ClientPolicy.html#threadPool) )。

C 客户端上的扫描使用固定大小的同步线程池（默认大小 = 16）来并行发出节点扫描。当扫描回调返回 false 时，当前所有 16 个节点扫描都将停止，但是某些节点扫描可能尚未开始。不幸的是，4.3.11之前的 C 客户端版本会启动这些节点的扫描，然后再分析第一条记录后检查是否终止。这是低效的。在 C 客户端版本 4.3.11 和更高版本中，在开始任何新节点扫描之前，将进行终止检查。

---

### - Is a scan taskID configurable ?

扫描 taskID 是在内部随机生成的，当前不可配置。

---

### - How do I set priority for a scan if I have multiple scans ?

问题：如果有多个扫描，如何设置扫描的优先级？

#### Version 3.6.0 and above

要以不同的优先级运行扫描，可以更改相关 scan jon 的 set-priority，以使优先级作业在不同的队列中管理，但仍由相同的扫描线程执行。

```
asinfo -v 'jobs:module=scan;cmd=set-priority;trid=<jobid>;value=3'
```

每个扫描作业位于三个优先级队列（High, Medium, and Low）之一中：

```
|     High      |    Medium     |      Low      |
|  - - - - - -  |  - - - - - -  | - - - - - - - |
| Partition IDX | Partition IDY | Partition IDX |
| Partition IDY | Partition IDZ | Partition IDY |
```

如果高优先级队列为空，则从中等优先级队列扫描工作。如果中等队列为空，则从低等优先级队列进行扫描工作。扫描优先级仅在一次扫描多个扫描作业时才重要。

当同时发出并具有相同优先级时，对较小命名空间的扫描不一定比对较大命名空间的扫描更快。这是因为扫描将以循环方式从不同的挂起作业中拾取分区，因此进度将被交错。

#### Version prior to 3.6.0

有关设置优先级的详细信息，请参见 [managing scan](https://www.aerospike.com/docs/operations/manage/scans#manage) 。

---

### - I issued 150 scans which completed, but why do I see only 100 when I check statistics on the Aerospike server ?

问题：我发出了150次扫描，这些扫描已经完成，但是为什么我在Aerospike服务器上检查统计信息时却只看到100次？

`scan-max-done` 控制为监控而保留的已完成扫描的数量。默认值为100。它存储扫描信息以供以后查看。完成的扫描包含有关何时开启扫描，何时完成，完成扫描多次时间，发现的内容以及通过网络发送多少字节的信息。

---

### My set of 10 records takes over 5 minutes on each node. How can this simple operation be so slow?

问题：我的10条记录集在每个节点上花费了5分钟以上。这个简单的操作怎么会这么慢呢？

**Note** : 在版本 5.6 和更高版本中，可以利用 `enable-index` 配置参数为为要高效扫描的集合创建特殊索引。

扫描 set 将必须扫描整个命名空间，因此，无论 set 中是否有 1 条记录，还是 set 覆盖了整个命名空间，扫描都将遍历命名空间中的每条记录（索引部分），然后再进行返回与 set 匹配的记录。

这是因为 set 信息与每个记录一起存储在它们的索引中以及记录本身（基于配置的内存或磁盘）中。当对命名空间中的 set 发出扫描时，必须逐个分区地缩小整个命名空间，以便针对每个条目检查其是否属于要扫描的集合。减少分区意味着完成遍历它。

Note : 减少分区需要获取一个锁，这可能会减慢同一个分区内新记录的创建和记录的删除。在版本 3.11 和更高版本中，在 `partition-tree-locks` 下为每个分区引入额外的锁可以大大减少此类争用。

根据您的用例，您可能希望基于此模型进行不同的建模。您可以在本文中找到一些相关信息: [Query versus Scan](https://discuss.aerospike.com/t/query-versus-scan/4054) 。

此外，除了分区缩减阶段（对于每个命名空间扫描而言）相同之外，我们还有另外的步骤来获取记录并将它们返回给客户端。该步骤可能需要更长的时间，具体取决于集合的大小，存储子系统的保持情况以及客户端使用要返回的内容的速度。因此，并非所有扫描都需要相同的时间。

---

### - Would the partition reduction phase for a namespace with 100 sets that were 1GB each be the same as a namespace with 100 sets that were 100GB each? Would a namespace that has 10 sets of 100GB each actually reduce faster than 100 sets that are 10GB each?

问题：具有 100 个分布为 1GB 的 set 的命名空间的分区所建阶段是否与具有 100 个分布为 100GB 的 set 的命名空间相同 ？ 具有 10 set 每个 100GB 的命名空间是否会比 100 个 set 每个 10GB 减少的速度更快 ? 

 - 如果命名空间的总记录数相同且集合的大小相同，则在两种情况下扫描都将花费很长时间。
 - 如果并行发出多个扫描，则在扫描线程数有限制的情况下，每组扫描的大小会影响整体扫描速度。

---

### When a scan is issued on a larger set size (for example 100,000,000 records), it sees much less CPU usage compared to a scan issued on a smaller set size (for example 1,000,000 records). This is observed on a namespace where the data is stored on disk and index is stored in memory.

问题 ： 当以较大的 set 大小（例如 100,000,000 条记录） 发出扫描时，与以较小的 set 大小 (例如 1,000,000 条记录) 发出扫描相比，看到的 CPU 使用率要低得多。在命名空间中可以观察到这一点，在命名空间中数据存储在磁盘上，索引存储在内存中。

在扫描期间，扫描线程逐个分区缩小（遍历）索引分区，以找到与 set 匹配的记录。当找到与该记录 set 匹配的记录时，扫描线程将从磁盘中获取记录（处分在 `read-page-cache` 中(如果开启), 或 `post-write-queue` 中找到该记录 ）并在继续进一步减少索引之前将其保存在缓冲区中。当为该命名空间扫描每个节点拥有的所有主分区时，这个过程就完成了。

对于较大的 set，期望在减少索引的同时找到更多匹配的记录，因此会更频繁地暂停以从磁盘中获取数据。这就解释了为什么考虑到频繁涉及的 I/O 操作，将无法连续利用 CPU (或更精确地说，是与扫描线程匹配的内核数)的原因。

在 set 较小的情况下，由于发现的记录要少得多，因此扫描线程将主要减少索引，这会占用大量 CPU。这将导致充分利用多个 CPU 内核（与扫描线程的数量匹配）。扫描较小的 set 所花费的时间当然比扫描较大的 set 所花费的时间快的多。

**Note :** 在更新本文时，Aerospike 正在开发一项新功能，以允许扫描 set 不必减少它们所属的命苦的整个索引。

---

### Why do I get a timeout when I scan a set of 10 records when my set of 100 million records works fine and does not timeout?

问题 : 当我的1亿条记录可以正常工作并且没有超时时，为什么在扫描10条记录时出现超时？

```log
aql> select * from namespace.set
Error: (9) Timeout: timeout=1000 iterations=1 failedNodes=0 failedConns=0
```
 - AQL 超时，因为它在 1 秒钟内为获取任何数据。现在，对于较小的 set ，这可能比较大的 set 更有可能，因为并非所有分区都减少了来自较小 set 的记录。增加超时时间可能在这会有用。

---

### How do I decide on a scan timeout?

问题 ： 如何确定扫描超时？

足够的扫描超时时间取决于影响扫描性能的因素 —— 磁盘 I/O ，并行正在进行的扫描次数，客户端和服务器之间的网络以及使用情况（对于特定情况有效的超时时间是多长时间）。因此，如果命名空间的比例增加，则需要调整超时。服务器端还提供了其他配置调整 (`scan-threads`, `scan-max-active`)，这些调整在增加超时之前可能会有所帮助，但是您可能不想对集群中的所有资源进行扫描。为了防止干扰其他 read/write transactions 。

---

### How do I decide on a retry setting for a scan?

问题 ：如何确定扫描的重试设置？

对于任何扫描 Job，请始终在客户端策略中将重试设置为 0。重试不太可能起作用，并且可能导致返回重复的记录。详情， [ Why do I get a warning - “job with trid X already active” when issuing a scan?](https://discuss.aerospike.com/t/why-do-i-get-a-warning-job-with-trid-x-already-active-when-issuing-a-scan/7189) 。

---

### Note

 - Guide to Scans: https://www.aerospike.com/docs/guide/scan.html 
 - Operational information for Scans: https://www.aerospike.com/docs/operations/manage/scans