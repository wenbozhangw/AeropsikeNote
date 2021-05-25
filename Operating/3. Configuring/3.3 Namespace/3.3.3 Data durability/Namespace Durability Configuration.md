## Namespace Durability Configuration

配置Aerospike命名空间时，一个重要的考虑因素是数据的持久性。Aerospike提供集群内复制和跨集群复制，以保护数据免受服务器故障的影响。

### Intracluster Replication

在 Aerospike 中，`replication-factor` 配置控制驻留在集群中的每个记录的副本数。大多数服务器的 `replication-factor` 为 2，这确保了集群中的所有数据都可以在单节点故障中幸免。副本确实会增加集群的总成本。副本因子为 2 时，集群需要的存储容量是副本因子为 1 的集群的两倍。副本还遭受性能损失，这主要是由于同步复制导致额外的网络延迟。

Aerospike在写入时进行同步复制，以确保整个副本在集群中保持一致，这可能会影响性能。有关异步跨数据中心复制的说明，请参阅 [XDR documentation](https://docs.aerospike.com/docs/operations/configure/cross-datacenter) 。

配置副本因子很简单。下面的示例显示副本因子2：
```
namespace <namespace-name> {
    ...
    replication-factor 2
    ...
}
```

### Cross-Datacenter Replication (XDR)

Aerospike Enterprise Edition包含跨数据中心复制（XDR）功能，可在群集之间复制数据。更多 XDR 配置信息，请参阅 [configuration documentation](https://docs.aerospike.com/docs/operations/configure/cross-datacenter) 。

### Where to Next ?

- 配置 [Storage Engine](https://docs.aerospike.com/docs/operations/configure/namespace/storage) ，该存储引擎确定是否以及在何处保留记录。
- 配置 [Data Retention Policy](https://docs.aerospike.com/docs/operations/configure/namespace/retention) ，该策略确定在最初写入记录后将其保留多长时间。
- 或者返回 [Configure Page](https://docs.aerospike.com/docs/operations/configure) 。