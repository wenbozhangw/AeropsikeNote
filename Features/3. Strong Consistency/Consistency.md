## Consistency

Strong Consistency 保证对单个记录的所有写入都将按特定 order (sequentially)应用，并且写入不会被重新排序或跳过，即它们不会丢失。

在所有形式的强一致性中，写入不会丢失 — 在同时硬件故障的范围内。有关哪些硬件故障可能导致数据丢失的确切说明，请参阅 [Consistency Architecture Guide](https://docs.aerospike.com/docs/architecture/consistency.html) 。

Aerospike 保证数据不会丢失，但有一下三个例外：

1. 节点进程暂停超过 27 秒，或集群节点之间的时钟偏差超过 27 秒。
2. 如果未启用 `commit-to-device`，则同时(在 `flush-max-ms` 内)不干净或不受信任的服务器将关闭。
3. 在数据可以复制之前出现多个硬件存储故障。

在每种情况下，Aerospike 都试图提供最佳可用性并尽量减少潜在问题。

在始终偏差的情况下，Aerospike 的 gossip 集群协议会持续健康偏差量，如果偏差变大就会发出警告 — 并在数据丢失发生之前禁用集群。

在服务器崩溃或关闭的情况下，Aerospike 会在第一次失败的情况下自动快速复制和重新平衡数据。如果故障迅速发生，则数据部分可能会被标记位 "dead"，并需要操作员干预。这是为了允许只读使用数据，或允许重新加载最近的修改。

---

### Configuration for Strong Consistency

在整个命名空间上启用 Strong Consistency。 [The Consistency management documentation](https://docs.aerospike.com/docs/operations/configure/consistency/index.html) 解释了如何配置命名空间和 roster。 

---

### Managing Strong Consistency

管理强一致性比管理可用命名空间更复杂。

[This section](https://docs.aerospike.com/docs/operations/manage/consistency/index.html) 介绍添加和删除节点、安全启动和停止服务器等管理概念。

---

### Client Requirements

只有 Aerospike Java client 4.1.2, C client 4.3.5, C# 3.5.3, and Python 3.0.1 版本支持强一致性。

但是，使用不支持 "strong-consistency" 的先前客户端也可以运行；您可能会获得过时的读取，并且可能会由于重传而产生一致性违规（丢失中间状态）。如果您使用旧客户端版本，则不会丢失写入，但可能无法获得会话一致性(Session Consistency)或线性化（linearizability）。

稍早于所属客户端的版本可能具有正确的 API，但是有各种关于返回代码和陈旧节点状态的bug。请使用规定的版本或更好的版本。

请参阅后续章节，了解与强一致性相关的新 API 功能。

---

### Upgrading from AP to SC namespaces

通常，不支持从 AP 更改为 SC 命名空间。在更改过程中可能会出现一致性违规的情况。建议创建一个新的命名空间，然后进行备份和恢复（"forklift"升级过程）。

---

### Using Strong Consistency

一般来说，作为程序员的变化很少。首先，您只知道数据是安全的。

但是，有一些新功能，通过客户端 Policy 对象进行管理，以及错误代码含义的差异。

#### Linearizable Reads

'Policy' 对象上存在一个新字段。为了获得 Linearizable 读取，您必须将 `linearizableRead` 字段设置为 `true`。如果将此字段设置为在 non-SC 配置的命名空间上读取，读取将失败。如果您不在 SC 命名空间上设置此字段，您将获得会话一致性(Session Consistency)。如果您在 4.0 之前的服务器请求上设置此字段，则该字段将被忽略。

`Policy` 对象可以在初始 AerospikeClient 构造函数调用中设置。默认情况下，所有使用构造函数的 AerospikeClient 对象的后续调用都将使用该值。否则，您应该在单个操作上使用此构造的 Policy 对象来指示完全线性化(Linearized)读取还是会话读取一致性(Session Consistency)。

多个 Policy 对象 — BatchPolicy、WritePolicy、QueryPolicy 和 ScanPolicy — 从 Policy 对象继承。其中，新的 'linearizableRead' 字段仅对默认对象和继承的 BatchPolicy 对象有意义，它应用于批处理操作中的所有元素。

#### InDoubt Errors

添加了一个名为 `InDoubt` 的所有错误返回的新字段。这是为了表示肯定未应用或可能已应用写入的差异。

在大多数数据库 API（如 SQL 标准）中，诸如 TIMEOUT 之类的故障 "known" 是不确定的，但没有特定的标志来表示哪些错误是不确定的。通常的做法是读取数据库以确定是否已应用写入。

例如，如果客户端驱动程序在联系服务器之前超时，则客户端应用程序可以确定 transaction 没有被应用；如果客户端已连接到服务器并发送数据但未通过 TCP 收到响应，则客户端不确定是否已经应用写入。Aerospike API 的改进允许客户端指出哪些故障肯定没有被应用。

我们相信这个标志对于高级程序员很有用，并减少在高压力情况下需要从数据库读取错误的情况。

#### Data Unreachable Errors

在 SC 中，会出现由于网络分区或其它中断而导致数据不可用的时期。如果数据不可用或变得不可用，客户端可能会出现多种错误。

这些错误在 Aerospike 中已经存在，但在以强一致性运行时有重要区别。以下是一些主要客户端库的错误列表：

- [Java Client Error Code](https://www.aerospike.com/apidocs/java/com/aerospike/client/ResultCode.html) .
- [C# Client Error Code](https://www.aerospike.com/apidocs/csharp/html/AllMembers_T_Aerospike_Client_ResultCode.htm) .
- [C Client Error Code](https://www.aerospike.com/apidocs/c/dc/d42/as__status_8h.html) .

当集群分区或多个硬件故障导致数据不可用时，可以看到四种不同的错误（此处使用 Java 客户端错误代码）： `PARTITION_UNAVAILABLE`、`INVALID_NODE_ERROR`、`TIMEOUT`、`CONNECTION_ERROR`。

对于这些情况，其他客户端可能有不同的错误代码。有关详细信息，请参阅客户端库特定的错误代码表。

当服务器确定其集群没有正确的数据时，服务器返回错误 `PARTITION_UNAVAILABLE`。具体来说，客户端可以连接到一个集群，并且最初已经收到一个包含该分区数据的分区表。在这种情况下，客户端向收到的最近的服务器发送请求。如果该服务器是不再具有数据可用性的集群的一部分，则将返回 `PARTITION_UNAVAILABLE` 错误。

错误 `INVALID_NODE_ERROR` 由客户端在客户端没有节点发送请求的情况下生成。这在初始化时期发生在指定的 "seed node" 地址不正确时，当客户无法连接到种子列表中的特定节点，因此永远不会受到完整的 partition map 时，以及当从服务器收到的 partition map 不包含相关分区时，就会发生这种情况。

如果 roster 配置错误，或服务有 `dead_paritions` 或 `unavailable_partition`（并且需要维护或 re-clustering），也会发生 `INVALID_NODE_ERROR`。如果您收到此错误，请使用上述步骤验证集群中的数据是否可用。

其他两个错误 `TIMEOUT` 和 `CONNECTION_ERROR`，将在网络分区(network partition)发生或正在发生的情况下返回。客户端必须有一个 partition map 和一个节点来发送请求，但不能连接到该节点，或者已经连接但随后发生故障。`CONNECTION_ERROR`可能会随着网络分区一直存在，只有在修复了分区或操作人员干预更改了集群 rosters 才会消失。

---

### Use of AQL with Strong Consistency

AQL工具通常由开始使用 Aerospike 的开发人员和管理员使用。关于 AQL 的 SC 使用，您应该了解几个命令。为了使用 AQL，请确保您运行的是 Aerospike Tools 3.15.3.2 或更高版本。

#### AQL Durable Delete

为了从 SC 命名空间中删除数据，您几乎肯定希望使用持久删除(durable delete)。默认情况下不允许清除删除(expunge delete)（以一致性为代价立即恢复存储），并且会导致 FORBIDDEN 错误。但是，这些删除是 AQL 中的默认值。

为了让 AQL 生成持久删除：
```
aql> set DURABLE_DELETE true
```

这将导致使用删除标志(delete flag)执行同一 AQL 会话中的后续删除。

#### AQL Linearize

为了使用 Linearize 标志执行读取（即 AQL 中的 SELECT） :
```
aql> set LINEARIZE_READ true
```

所有后续的主键选择命令都将使用线性化读取策略执行。

请注意，Linearize 仅支持主键请求，而不支持本文档中提到的查询和扫描，因此仅适用于使用 "PK = " SELECT 表单。

#### AQL Error codes

由于 AQL 经常在初识使用中用于诊断和验证配置，请理解错误代码之间的区别：

`PARTITION_UNAVAILABLE`、`INVALID_NODE_ERROR`、`TIMEOUT` 和 `CONNECTION_ERROR`。

上面对这些错误代码进行了深入描述。

---

### Consistency Guarantees

#### Strong Consistency

强一致性保证规定对单个记录的所有写入都将按特定 order（sequentially），并且不会重新排序或跳过写入。

特别是，确认为已提交的写入已被应用，并且与同一记录的其他写入相比，它们存在于 transaction timeline 中。即使面对网络故障(network failures)和中断（outages）以及分区(partitions)，这种保证也适用。被指定为 "timeouts"（或来自客户端 API 的 "InDoubt"）的写入可能会或可能不会被应用，但如果它们已经被应用，它们只会被观察到。

Aerospike 的强一致性保证是针对每条记录的，并且不涉及 multi-record transaction 语义。每个记录的写入或更新将是原子的(atomic)和隔离的(isolated)，并且使用混合时钟(bybrid clock)保证排序。

Aerospike 提供完整的 Linearizable 模式，它再所有可以观察数据的客户端之间提供单一的线性视图，以及更实用的 Session Consistency 模式，它保证单个进程看到更新的顺序集(sequential set)。这两种读取策略可以基于 read-by-read 进行选择，从而允许需要更高保证的 transaction 支付额外的同步代价，下面详述。

在 "timeout" 返回值的情况下 —— 这可能是由于网络拥塞而产生的，在任何 Aerospike 问题之外 —— 写入保证完全写入，或者根本不写入；永远不会出现部分写入的情况（即，永远不会出现至少一个副本被写入但不是所有副本都被写入的情况）。如果在所有副本之间复制(replicate)写入 transaction 失败，记录将处于 "un-replicated" 状态，强制在记录上的任何后续 transaction（读取或写入）之前进行 "re-replication" transaction 。

Strong Consistency 是在每个命名空间的基础上配置的。将命名空间从一种模式切换到另一种模式是不切实际的 —— 创建新的命名空间和迁移数据是推荐的方法。

#### Linearizability

在并发编程中，一个操作（或一组操作）是原子的、可线性化的、不可分割的或不可中断的，如果它再系统的其余部分看来，是瞬时发生的 —— 或者可以立即用于读取。

所有访问都以相同的 order (sequentially) 被所有并行进程看到。该保证由 Aerospike 有效 master 强制执行，并导致 in-transaction "health check"，以确定其他服务器的状态。

如果应用(applied)写入并观察到客户端应用了写入，则不会观察到记录的先前版本。启用此客户端模式后，"global consistency" —— 指连接到集群的所有客户端 —— 将在给定时间看到单个记录的状态视图。

这种模式需要在每次读取时进行额外的同步，从而导致了性能损失。这些同步数据包不会查找或读取单个记录，而是简单地验证单个分区的存在和健康状况。

#### Session Consistency

这种模式在其他数据库中被称为 "Monotonic reads, monotonic writes, read-your-writes, write-follows-reads"，是最实用的强一致性模式。

与 Linearizable 模型不同，会话一致性仅限于客户端会话 —— 在这种情况下，它是单个客户端系统上的 Aerospike 集群对象，除非使用共享内存在进程之间共享集群状态。

会话一致性非常适用于涉及设备或用户会话的所有场景，因为它保证 `monotonic reads, monotonic writes, and read your own writes (RYW)` guarantees.

会话一致性为会话提供可预测的一致性和最大读取吞吐量，同时提供最低延迟写入和读取。

---

### Performance Considerations

当与以下设置一起使用时，强一致性模式(Strong Consistency mode)在性能上类似于可用性模式(Availability mode)：

1. Replication factor two,
2. Session Consistency.

当 replication factor 大于 2 时，写入会导致额外的 "replication advise"数据包到执行副本(acting replicas)。当 master 不等待响应时，额外的网络数据包将在系统上造成负载。

当启用 Linearizability 读取问题时，在读取期间 master 必须向每个代理副本发送请求。这些 "regime check" 数据包 —— 不会进行完整读取 —— 会导致额外的延迟和数据包负载，并降低性能。

---

### Availability Considerations

尽管 Aerospike 允许使用两个数据副本(copies)进行操作，但故障情况下的可用性需要在 master promotion 期间有一个副本(replica)，然后再故障期间中需要三个副本(copies)（the copy that failed, the new master, and the prospective replica）。如果没有这第三个潜在副本(potential copy)，分区可能仍然不可用。因此，具有两个数据副本(copies)的双节点集群在拆分中将不可用。

---

### Exceptions to Consistency Guarantees

请注意以下可能导致一致性或持久性问题的操作环境。

#### Clock discontinuity

超过大约 27 秒的时钟中断(clock discontinuity)可能会导致写入丢失。在这种情况下，数据以 27 秒时钟分辨率(clock resolution)之外的时间值写入存储器。由于存储的时钟值的精度，随后的数据合并可能会选择此未提交的写入，覆盖其他成功完成的记录版本。

在不连续性小于 27 秒的情况下，内部混合始终的分辨率将选择正确的记录。

有机时钟(organic clock)偏斜不是问题，集群的心跳机制检测漂移(drift)（默认情况下，偏斜15秒），和日志警告。如果偏斜(skew)变得极端（默认为 20 秒），节点将拒绝写入，返回 "forbidden" 错误代码，从而防止违反一致性。

由于四个特定原因，往往会发生这种类型的超时：

1. 管理员错误，操作员执行将时钟设置到遥远的将来或遥远的过去
2. 出现故障的时间同步组件
3. 虚拟机休眠
4. Linux 进程暂停或 Docker 容器暂停

请务必避免这些问题。

#### Clock discontinuity due to virtual machine live migrations

虚拟机和虚拟化环境可以实现安全的强一致性部署。

但是，有两个特定的配置问题是危险的。

实时迁移(**Live migrations**)，是将虚拟机从一个物理机箱透明地移动到另一个物理机箱的过程，会导致某些危险。时钟不连续地移动，网络缓冲区(network buffers)可以保留在较低级别的缓冲区(lower level buffers)中并在以后应用。

如果可以肯定地将实时迁移限制在 27 秒以内，则可以保证强一致性。但是，更好的操作过程是安全地停止 Aerospike，移动虚拟机，然后重新启动 Aerospike。

第二种情况涉及进程暂停，这也用于容器环境。这些会产生类似的危险，不应执行。使用这些功能的操作原因很少，

#### UDFs

UDF 系统会起作用，但是，目前，UDF 读取不会被线性化，并且 UDF 写入以某些方式失败可能会导致不一致。

#### Non-durable deletes, expiration and data eviction

非持久删除(non-durable deletes)，包括数据过期和逐出(data expiration and eviction)不一致。这些删除具有释放 DRAM 和存储空间的必要特征，因此即使具有强一致性也可能需要它们。

这些不生成持久 "tombstones" 的数据库删除可能违反一致性保证。在这些情况下，您可能会看到数据返回。出于这个原因，我们通常需要在 SC 配置中禁用逐出和过期，但是，由于存在有效用例，我们允许手动覆盖。

但是，在某些情况下，即使对于 SC 命名空间，也可能需要过期和非持久删除，并且对于架构师、程序员和操作人员来说可能是安全的。

这些通常是一些对象正在被修改，而 "old" 对象正在过期和删除的情况。不能写入可能会过期或删除的对象。

例如，对一个对象（代表金融交易） —— 的写入可能会持续几秒钟，然后该对象可能会在数据库中存在数天而没有写入。该对象可以安全地过期，因为写入和过期之间的时间段非常长。

如果您确定您的用例是合适的，您可以启用非持久删除，请在您的 `/etc/aerospike/aerospike.conf` 中使用以下配置设置添加命名空间选择：
```
strong-consistency-allow-expunge true
```

持久删除会生成 tombstone，完全支持一致性。

---

### Client version

为了实现 Session Consistency 或 Linearizable 一致性，您必须使用支持这些功能的 Aerospike 客户端版本。在撰写本文时，当前的最小客户端版本为 Java 4.1.2（及更高版本）、C Client 4.3.5 或更高版本、C# 3.5.3 或 Python 3.0.1 或更高版本。

如果您使用其他客户端版本，您可能会看到过时的读取。

---

### Client retransmit

如果您启用 Aerospike 客户端写入重传(retransmission)，您可能会发现某些 test suites 会声称违反一致性。这是因为初始写入(initial write)会导致超时，因此会重新传输，并可能会多次应用。

存在多种危险。

首先，可以多次应用写入。然后，次写入可能会 "surround" 其他写入，从而导致违反一致性。这可以通过使用 "read-modify-write" 模式和指定 generation 以及禁用 retransmission 来避免。

其次，可能会生成不正确的错误代码。例如，可能会正确应用写入，但瞬时网络故障可能会导致重新传输。在第二次写入，磁盘可能已满，从而产生特定的，而不是 "InDoubt" 错误 —— 尽管 transaction 已被应用。此类错误无法使用 "read-modify-write" 模式来解决。

我们强烈建议禁用客户端超时功能，并根据应用程序的需要重新传输。虽然这看起来像是额外的工作，但在调试时纠正错误代码的好处是无价的。

---

### Secondary Index requests (scan and query)

如果执行扫描或查询，则可能会返回陈旧读(stale reads)和脏读(dirty reads)。为了提高性能，Aerospike 目前将返回 master 上 "In Doubt" 且尚未完全提交的数据。

此冲突只存在于查询和扫描中，并将在后续版本中予以纠正。

---

### Durability Exceptions

#### Storage hardware failure

检测到一些持久性异常，将该分区标记为管理接口(administrative interfaces)中的 `dead_partition`。

当整个 roster 中的所有节点都可用并连接时，就会发生这种情况，但某些分区数据不可用。要允许分区在此接受读取和写入，管理员必须通过执行上面的 "revive" 命苦来覆盖错误，或者使服务器与丢失的数据联机。

当强一致性与没有后备存储(no backing store)的 data in memory 一起使用时，或者当后备存储被擦除或丢失时，就会发生这些情况。

#### Incorrect Roster Management

通过 replication-factor 或一次减少多个 roster 节点可能会导致记录数据丢失。必须遵循从集群中安全添加和删除的过程。请严格遵守有关 roster 管理的操作程序。

#### Partial storage erasure

在某些数量的扇区(sectors)或驱动器部分已被操作员擦除的情况下，Aerospike将无法注意到故障。replication-factor 或更多节点上的部分数据擦除（例如恶意用户或驱动器故障）可能会擦除记录并逃避检测。

#### Simultaneous server restarts

默认情况下，Aerospike最初将数据写入缓冲区，并在所有必须的服务器节点确认接收数据并将其放入持久队列时考虑写入的数据。

Aerospike 尝试从系统中删除所有其他写入队列。推荐的 Aerospike 硬件配置使用 RAW 设备模式，该模式禁用所有操作系统 page cache 和大多数设备缓存(device cache)。我们建议禁用任何硬件设备写入缓存，这必须以特定于设备的方式完成。

如果服务器出现故障，Aerospike将立即开始复制数据(replicating data)的过程。如果复制快速发送，则不会丢失任何写入。

为了防止在多次快速故障(multiple repid failures)期间写入队列中的数据丢失，请启用其他地方注明的 `commit-to-device`。默认算法会限制丢失的数据并提供最高级别的性能，但如果需要更高级别的保证，可以在单个基于存储的命名空间上启用此功能。

在文件系统或带有写入缓冲区的设备用作存储的情况下，`commit-to-device`可能无法防止这种基于缓冲区的数据丢失。

---

### OutSide

- [分布式一致性模型分类](https://www.cnblogs.com/yylingyao/p/12691664.html)