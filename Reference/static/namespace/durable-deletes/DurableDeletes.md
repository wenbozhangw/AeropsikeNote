## durable deletes

默认情况下，Aerospike会以一种非常有效的方式删除数据，仅通过删除已删除对象的主索引条目就可以立即回收内存。这样做的缺点是，在冷启动名称空间时，这些已删除对象的先前持久化版本可能会重新出现。为了解决此问题，Aerospike在企业版
3.10 中引入了 [durable deletes](https://www.aerospike.com/docs/guide/durable_deletes.html);

可以将持久删除指定为基本操作的策略。可以在一下调用中提供持久删除策略：

- Write (仅适用于删除最后一个bin，导致记录删除)
- Delete
- Operate (仅适用于删除最后一个bin，导致记录删除)
- UDF (仅适用于当UDF执行导致删除记录，通过调用删除，或移除最后一个bin)

### Durable Delete and Tombstone

发出持久删除后，将写入一个 Tombstone。Tombstone 写入与记录更新的相似之处在于：

- 它继续占据索引条目，与其他记录索引条目一样。
- 它与记录的先前副本一起保留在磁盘上。
- 它具有与任何其他记录相同的元数据。

    - last-update-time - 就像通常更新一样。
    - generation - 像正常更新一样递增。

- 它按照命名空间上指定的 replication factor 进行复制。
- 迁移方式与迁移当前记录的方式相同。
- 解决冲突的方法与 data-records 相同。

Tombstone 被写为**无过期**。

由于 "Durable Delete" 取决于准确的时钟，因此使时钟在集群中同步非常重要。

### Tombstone on cold start

Tombstone 仅仅是没有任何 bin 的记录。

- 它包含所有元数据，包含 key。
- 它不包含 LDT 标志。

冷启动时，将扫描磁盘以重建内存索引树中的记录。比较记录的版本。最近 last-update-time（出现`抢七`使用generation判断）将会被重新读回。如果对于持久删除的记录，逻辑删除只是参与比较的一个版本，可以防止记录的任何较旧版本返回。如果逻辑删除是最新版本，它将被重新加载到索引中。

### Tombstone Management
与数据记录类似，删除 tombstone 时也将对其进行回收。删除后，它们将从内存索引中删除，并且磁盘上的副本符合碎片整理的条件。索引的内存可以立即重用。存储空间可根据磁盘碎片整理的时间重新使用。

一种特殊的背景机制 (`Tomb-Raider`) 用于删除不再需要的 tombstone。删除 tombstone 的条件如下：

- 磁盘上没有大于该记录时间的副本。

  - 这种情况可确保冷启动不会恢复任何较旧的副本。
  
- Tombstone 的 last-update-time 在 当前时间减去配置的 `tomb-raider-eligible-age` 之前。

  - 这种情况会阻止与集群分离 `tomb-raider-eligible-age` 秒的节点重新加入时会引入较旧的副本。
  
- 该节点并不等待任何传入的迁移。
- XDR已成功运送该墓碑（如果使用XDR 5.0及更高版本）。

如果满足所有条件，则将回收墓碑。

实际的后台线程大致分为以下步骤：

- 遍历索引以将所有 tombstone 标记为可移除 (cenotaphs) 的候选者。
- 扫描每个磁盘块以获取记录，为每个记录取消标记 cenotaph。
- 再次遍历索引。剩下的所有 cenotaph 都是永久删除的候选人。

对于非持久化的命名空间，删除 tombstone 是独立的，并且仅需要一个索引迭代即可删除 tombstone。

冷启动还删除了不需要的 tombstone，这是磁盘读取的一部分：

- 所有 tombstone 在初次访问时都被标记为可移除（cenotaphs）的候选者。
- 如果读取了 tombstone 覆盖的后续活着的记录，cenotaphs 将不会被标记，并且 tombstone 会留下。
- 否则，在冷启动结束时，将会删除所有 cenotaphs。

以下配置可用于控制 `Tomb-raider` 的行为：
- `tomb-raider-period` - 两次运行之间的最短时间（以秒为单位），默认为 1 天 (86400)。
- `tomb-raider-eligible-age` - 即使发现可以安全删除，保留 tombstone 的秒数，默认为 1 天 (86400)。
- `tomb-raider-sleep` - （仅存储）- 在磁盘上的大块读取之前休眠的微秒数，默认值为 1000 微秒 （1ms）。


### Expired and Evicted Records

过期和逐出的记录不会生成 tombstones。这是理想的行为：

- 它允许最大的资源容量用于数据记录，而不是 tombstone。
- 如果资源容量增加（例如，通过增加节点上的内存容量），则有可能在冷启动时恢复非持久性已删除记录。

在其他情况下，收回的记录可能会返回：

- 复制副本的冷启动逐出时间比主节点浅。在这种情况下，当主节点离开，并且副本冷启动时，副本记录可能会恢复。

tombstone 没有设置到期时间，因此不符合驱逐条件。tombstone有其自己的删除机制。

### XDR 
XDR 始终提供持久删除。

默认情况下，XDR 装载客户端 delete。可以使用 `ignore-expunges true`参数禁用装载客户端删除。
默认情况下，XDR 不会从 Namespace Supervisor (nsup) 进程中删除该结果，成为"nsup deletes"。可以使用 `ship-nsup-deletes true` 参数启用装载 nsup delete。
从 XDR 5.0 开始，由于没有摘要日志，XDR 创建 xdr_tombstone 来运送非持久删除。当 XDR 的 LST (Last Ship Time) 大于 XDR tombstone 的 LUT(Last Update time)时，换句话说，当它们成功发送后，这些 xdr tombstone 回收。
有关其他定义和参数，请参阅 [Configure XDR](https://www.aerospike.com/docs/operations/configure/cross-datacenter/index.html) 。


### Scan, Batch
Scan/Batch 跳过返回 tombstone 的记录。


### Conflict Resolution Policy
冲突解决策略会影响整个集群状态更改的持久删除行为。为了保证持久删除的正确传播，冲突解决策略应为 "last-update-time"。对于对网络分区敏感的用例，建议考虑配置`strong-consistency`.


### Caveats

#### Capacity Sizing, Tuning Requirement

启用持久删除时需要注意的关键警告是对集群大小的影响。先前已根据集群中活动记录的数量确定了集群的大小。tombstone 占用 RAM 和磁盘空间，它们永远不会过期，因此永远也无法被驱逐。这在日志和统计信息中报告。建议与 Aerospike Solutions Architects一起打开用例和规模讨论，以讨论用例以及使用持久删除如何影响集群的规模。大小影响如下：

- Index space = 64 bytes + optional key size
- Disk space = 128 bytes + optional key size

持久删除的另一个潜在影响是， Tomb Raider 打扫磁盘时，SSD 上的大块读取增加。如果大小不正确，这可能会将延迟引入集群，这可能会影响处理操作。

#### Tombstone Reporting
- 日志行已更新，以报告节点命名空间内 tombstone 的状态：
```log
Aug 24 2016 22:13:50 GMT: INFO (info): (ticker.c:336) {test} objects: all 4344 master 2209 prole 2135
Aug 24 2016 22:13:50 GMT: INFO (info): (ticker.c:372) {test} tombstones: all 2378 master 1293 prole 1085
```

- 此外，以下命名空间统计信息可用于跟踪 tombstone :
```log
$ asinfo -v "namespace/test" -l | grep tomb
tombstones=2378
master_tombstones=1293
prole_tombstones=1085
tomb-raider-eligible-age=86400
tomb-raider-period=10000
```

#### Client Server Compatibility

- 默认客户端策略不是持久删除，以保持向后兼容。
- 必须增强客户端应用程序以使用持久删除功能。
- 具有持久删除策略的新客户端针对旧版本的服务器进行写操作将只会忽略持久删除策略。

#### Community/Enterprise Compatibility
如果企业版驱动器具有 tombstone，将其降级为社区版，则在读取 tombstone 时冷启动将失败。需要清理驱动器才能成功。

引入了一个新的错误代码：

`AS_PROTO_RESULT_FAIL_ENTERPRISE_ONLY` - 如果针对社区版服务器发送了持久删除策略。