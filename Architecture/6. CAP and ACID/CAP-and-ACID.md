## CAP and ACID

[CAP Theorem](http://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed) 假定在分布式系统中，任何时候都只能保证三个属性(一致性，可用性和分区容错性，即consistency, availability, and partition-tolerance)中的两个。由于分区是不可避免的，而可用性对于一类部署至关重要，因此 Aerospike 3.x 系统将可用性只与一致性之上。因此，Aerospike系统可以归类为 AP，并且可以按以下工作以提供非常高的性能和可用性：

- 在每个子系统中优先考虑可用性而不是一致性。
- 利用 Aerospike 的高垂直规模(high vertical scale)（每个节点100万 TPS 和数 TB 的容量），可以确保较小的集群大小（1 到 100 个节点）。
- 确保高吞吐量下的低延迟 transaction，并在节点故障和滚动升级期间保持数据可用。
- 提供一些简单的冲突解决支持，以确保在集群更改事件期间，新的数据可以 "赢得" 较旧的更改。

在 Aerospike 3中，Aerospike仅支持可用性。在这些情况下，任何分区的服务器 set 都将声明对所有数据的完全所有权。当集群分区被 resolved，会发生冲突，导致数据丢失。

在 Aerospike 4中，引入了 Strong Consistency。使用此算法，集群拆分和分区可以仔细管理集群的那个部分仍然可用，从而避免了任何可能发生冲突的写入。

Aerospike 4 在主键的基础上保持一致性。它通过将写入提交到具有独立硬件组件的多个物理服务器上提供耐用性。为了获得更大的耐用性，可能需要进行写操作以刷新到持久存储。

[ACID](https://en.wikipedia.org/wiki/ACID) ，但是，通常还意味着另外两个功能：
- 多记录隔离和一致性
- 查询隔离，因此多记录查询在写入时将保持一致

Aerospike不提供这两个功能。

有关更多信息，请参见有关 [strong consistency](https://docs.aerospike.com/docs/guide/consistency.html) 的这些详细信息。