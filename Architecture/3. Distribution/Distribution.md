## Distribution

我们为必须 24/7 可用且可靠地处理大数据的应用程序设计了 Aerospike 数据库。这有几个含义：

- 自动数据位置检测无需担心数据位置。Aerospike客户端会自动检测数据位置，并确保在单个跃点中处理请求。您的应用程序可以将数据库视为存储在单个服务器上。Aerospike smart client 可处理集群分发。
- 自动集群平衡添加容量时，只需将一个节点添加到集群中，集群便会重新平衡以包括新节点。吞吐量和性能随容量的增加而线性增加。
- 无单点故障节点上的 SSD 可以发生故障，节点可以发生故障或离线进行维护或升级，或者整个数据中心可以在不影响可靠性的情况下发生故障。

可靠地管理集群是 Aerospike 数据库的核心，因此非常中心。Aerospike 使用以下功能实现了此目的：

- [Data Distribution](https://docs.aerospike.com/docs/architecture/data-distribution.html) : 强大的分区功能可确保数据分布均匀，避免热点并自动平衡数据，而无需人工干预。
- [Clustering](https://docs.aerospike.com/docs/architecture/clustering.html) : Aerospike 集群数据库自动检测故障并修复。
- **Replication :** Aerospike的功能包括以复制能力，以避免单节点故障：
    - **Intra Cluster Replication**
    - [Rack Aware Replication](https://docs.aerospike.com/docs/architecture/rack-aware.html)
    - [Cross-Datacenter Replication](https://docs.aerospike.com/docs/architecture/xdr.html)