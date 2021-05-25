## Primary Index

#### Contents

 - [Index Metadata](#index-metadata)
 - [Index Persistence](#index-persistence)
    - [Fast Restart Feature](#fast-restart-feature)
    
The primary key index 是 distributed hash table 技术与每个服务器中的 distributed tree structure 的混合。命名空间中的整个 keyspace 都通过健壮的哈希函数分成多个分区。总共 4096 个分区平均分布在集群节点上。有关 hashing 和 partitioning 的详细信息，请参见 [data distribution](https://docs.aerospike.com/docs/architecture/data-distribution.html) 。

Aerospike 使用了 red-black in-memory structure，称为 *sprig*。对于每个分区，可以有可配置数量的 sprigs 。 配置正确数量的 sprigs 是在内存开销和优化的并行访问之间进行权衡的。

The primary index 位于 20 byte hash 上，称为 digest of the specified primary key。虽然这会扩展某些记录的 key 的大小（例如，只有 8 个字节的整数 key），但是这是有益的，因为无论输入 key 的大小或分布如何，代码操作都是可预测的。

当一台服务器发生故障时，另一台服务器上的索引立即可用。如果发生故障的服务器继续关闭，则开始重新平衡数据，并在新节点上构建负载的索引。

---

### <span id="index-metadata"> Index Metadata </span>

当前，每个索引条目需要 64 个字节。除 20 字节摘要外，以下元数据也存储在索引中。
 - **Generation count :** 跟踪所有写入记录；用于解决 conflicting updates。
 - **Expiration time or TTL :** 跟踪 key 的过期时间。eviction 子系统使用此元数据。
 - **Last Update Time :** 跟踪对 key 的最后一次写入（Citrusleaf epoch）. 用于冷启动期间的 conflict resolution，迁移期间的 conflict resolution（取决于您配置的设置），predicate filtering，incremental backup scans，truncate and truncate-namespace 命令。

---

### <span id="index-persistence"> Index Persistence </span>

主索引从数据本身派生，并且可以根据该数据重建，这取决于 [fast restart](https://docs.aerospike.com/docs/operations/manage/aerospike/fast_start/index.html) 的配置设置。

#### <span id="fast-restart-feature"> Fast Restart Feature </span>

为了以最少的停机时间进行快速集群升级，Aerospike 具有 [fast restart](https://docs.aerospike.com/docs/operations/manage/aerospike/fast_start/index.html) 功能。 快速重启从 Linux shared memory segment 分配索引内存。对于计划的关闭和重新启动（例如，对于升级），在重新启动时，服务器将重新连接到共享内存段并激活主索引，而无需对存储进行数据扫描。