## Namespace Configuration

可以配置和调整 Aerospike 的命名空间，以适合各种使用情况。在本节中，我们将比较并提供最常见的命名空间配置变量的配置方法。有关命名空间参数的完整列表，请参见 [configuration reference](https://docs.aerospike.com/docs/reference/configuration) 。

配置命名空间的最低要求是提供命名空间命名。这将通过该名称创建一个具有 4GB 内存容量的命名空间。配置将类似于以下内容，带注释的参数指示默认值。

```
namespace <namespace-name> {
    # memory-size 4G           # 4GB内存用于索引和数据
    # replication-factor 2     # 对于多个节点，保留2份数据
    # high-water-memory-pct 60 # 如果容量4GB的60%超过则逐出非零TTL数据
    # stop-writes-pct 90       # 如果容量超过4GB的90%，停止写入
    # default-ttl 0            # 来自不提供TTL的客户端的写操作将默认为0或永不过期
    # storage-engine memory    # 数据只存储在内存中
}
```

在以下各节中，我们将更深入地研究命名空间和存储配置，并提供配置示例，使您能够快速入门。

对于未运行最新群集协议的版本 ([introduced in versions 3.13](https://docs.aerospike.com/docs/operations/upgrade/cluster_to_3_13/index.html)) ，添加或删除命名空间需要完全关闭群集。在服务器的这些旧版本上，在整个群集中的aerospike.conf中保持相同的命名空间顺序也很重要。

Aerospike Enterprise Edition Server版本的群集中最多有32个命名空间，而Aerospike Community Edition Server版本（4.0版开始）中的群集中最多有2个命名空间。

### Where to Next ?

- 配置 [Storage Engine](https://docs.aerospike.com/docs/operations/configure/namespace/storage) ，该存储引擎确定是否以及在何处保留记录。
- 配置 [Data Retention Policy](https://docs.aerospike.com/docs/operations/configure/namespace/retention) ，该策略确定在最初写入记录后将其保留多长时间。
- 配置 [Data Durability Policy](https://docs.aerospike.com/docs/operations/configure/namespace/durability) ，该策略确定在集群中保留多少条记录的复制副本 (replica copies) 。
- 或者返回 [Configure Page](https://docs.aerospike.com/docs/operations/configure) 。