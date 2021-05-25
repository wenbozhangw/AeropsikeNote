## Rack Awareness

Aerospike Rack Awareness 功能使您可以将记录的不同副本（来自主分区）存储在不同的硬件故障组(hardware failure groups)中（例如，如果副本因子为2，则分区的主副本及其副本将存储在不同的硬件故障组上）。这些组由其机架号(`rack-id`)定义。Rackaware 配置只会影响 proles 分区，而不会影响集群中的主分区分配。

仅在 Aerospike Server 3.x 及更高版本中提供了 Rack awareness 功能。如果使用的是先前版本，则必须升级Aerospike。从版本4.0开始，机架感知功能是仅Aerospike Enterprise Edition Server的功能。

默认情况下，副本分区均匀地分布在所有集群节点上。（包含n个节点的集群中）每个节点将大约拥有 1/n 的数据分区作为主分区。这些分区的副本均匀地分布在其他 n-1 个节点之间（总计，每个节点大约拥有 rf/n，其中 rf 是 replication-factor ）。当物理机架发生故障时（或在云部署的情况下，为 Availability Zone），这些节点可能同时包含同一分区的主分区和副本分区，从而可能导致数据不可用。

为了帮助避免此类数据不可用，可以将机架感知功能配置为使用多个机架（由机`rack-id`定义）。

当 replication-factor 小于或等于机架数时，对于每个分区，机架数的副本因子确保每个分区都有一个分区副本。

如果副本因子大于机架数，则对每个分区，所有机架都将具有该分区数据的副本，对于每个分区，一些机架将拥有额外的副本。

如果单个节点发生故障，集群将暂时不平衡，知道该节点恢复位置。这种不平衡不会导致服务器中断。集群继续不间断。节点重新启动后，集群将自动重新平衡。

下面是一些示例来说明此功能当前的运行方式：

- 当配置 3 个机架且副本因子为 3 时，每个机架将接收给定分区的一个副本。
- 如果在这种配置下丢失了机架，则假设未违反 [paxos-single-replica-limit](https://docs.aerospike.com/docs/reference/configuration/#paxos-single-replica-limit) 规定，每个分区仍将具有 3 个副本，第一个副本（主副本）将位于一个机架上，第二个副本（主副本）将位于另一个机架上，而第三个将位于2个机架中的一个机架上，具体取决于继承列表的顺序。

有关如何配置机架感知的更多详细信息，请参阅“Operations”下的 [Rack-aware](https://docs.aerospike.com/docs/operations/configure/network/rack-aware) 部分。