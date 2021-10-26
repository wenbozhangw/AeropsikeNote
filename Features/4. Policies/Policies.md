## Policies

Aerospike 允许以极大的灵活性进行读写。使用 Aerospike 客户端 `policy`，您可以创建乐观并发(optimistic concurrency)的 read-modify-write 模式，控制生存时间(time to live)，并仅在先前不存在记录(或相反)时才写入记录。 

这种类型的操作非常快，因为 generation 和 time-to-live 等信息存储在主键索引(primary key index)中。检索数据对象不需要额外的工作。

这些策略会影响数据库操作和客户端操作。许多策略用于向服务器发送适当的线路协议命令(wire protocol commands)。其他策略（如 maxRetries）会影响客户端操作。

每个客户端都存在这些策略，并且 API 略有不同。在了解您的应用程序需要哪些策略后，请参阅客户端特定文档以了解准确的语法。

---

### Set Default Client Policies

可以为每个 AerospikeClient 实例创建默认客户端策略。以下示例演示如何在 Java 客户端中设置策略默认值。有关特定于语言的示例，请参阅您的客户端的文档。

```java
// Set client default policies.
ClientPolicy clientPolicy = new ClientPolicy();
clientPolicy.readPolicyDefault.replica = Replica.MASTER_PROLES;
clientPolicy.readPolicyDefault.consistencyLevel = ConsistencyLevel.CONSISTENCY_ALL;
clientPolicy.readPolicyDefault.socketTimeout = 100;
clientPolicy.readPolicyDefault.totalTimeout = 100;
clientPolicy.writePolicyDefault.commitLevel = CommitLevel.COMMIT_ALL;
clientPolicy.writePolicyDefault.socketTimeout = 500;
clientPolicy.writePolicyDefault.totalTimeout = 500;

// Connect to the cluster.
AerospikeClient client = new AerospikeClient(clientPolicy, new Host("seed1", 3000));
```

---

### Set Per-Transaction Client Policies

要在每个 transaction 的基础上设置策略，请将所需的策略设置传递给单个 API 调用。例如，要使用 `master` 提交级别执行写入：
```java
// Make a copy of the client's default write policy.
WritePolicy policy = new WritePolicy(client.writePolicyDefault);

// Change commit level.
policy.commitLevel = ConsistencyLevel.COMMIT_MASTER;

// Write record with modified write policy.
client.put(policy, key, bins);
```

如果 transaction 调用中指定的策略不为空，则该策略将覆盖在客户端连接级别定义的相应策略。如果 transaction 策略为空，则将使用相应的默认客户端策略。

---

### Server Policies

您可以使用命名空间级别的动态可更改服务器配置参数，来覆盖服务器上客户端选择的每个 transaction 数据一致性级别。

参考 reference 页面中的一致性覆盖配置参数 ： [read-consistency-level-override](https://docs.aerospike.com/docs/reference/configuration#read-consistency-level-override) 和 [write-commit-level-override](https://docs.aerospike.com/docs/reference/configuration#write-commit-level-override) 。

---

### Policy Definitions

以下部分描述了 Aerospike Java 客户端策略。其他客户端使用类似的结构。

#### Replica

副本（`Policy.replica`）指定客户端在单条记录操作期间将访问哪个副本：
- `SEQUENCE` (default) —— 首先尝试包含 key 的主分区(master partition)的节点。如果连接失败，所有命令都会尝试包含副本分区(replicated partition)的节点。如果达到 socketTimeout，读取也会尝试包含副本分区的节点，但写入保留在主节点(master node)上。
- `MASTER` —— 使用包含 key 的主分区的节点。
- `MASTER_PROLES` —— 在包含 key 的主分区和副本分区的节点之间以轮询的方式分布读取。写操作总是使用包含 key 的主分区的节点。
- `RANDOM` —— 以轮询的方式读取分布到集群中的所有节点。写操作总是使用包含 key 的主分区的节点。

默认情况下，所有客户端读取首先被定向到主副本(master replica)（`Replica.SEQUENCE`），但是，您可能希望将读取分散到所有可用副本（例如，读取热键的性能影响可以沿着副本因子的顺序降低）。设置复制策略为 `Replica.MASTER_PROLES`  在 master 和 proles 之间分发读取。

#### Data Consistency Level

一致性级别（`Policy.consistencyLevel`）指定服务器内部需要访问多少个副本，以确定最近的记录值，并将其返回给客户端：

- `CONSISTENCY_ONE` (default) —— 返回前读取单个副本。
- `CONSISTENCY_ALL` —— 返回前读取所有副本。

读取记录（包括读取和操作函数）时的默认客户端行为是仅读取一个副本（`ConsistencyLevel.CONSISTENCY_ONE`）。

在集群 reconfiguration 期间，读取单个副本可能不会返回最新写入的版本。如果您希望服务器提供 "duplicate resolution"，即联系 replicas 并找到最新版本，包括更新 master 的 copy，请将一致性级别策略设置为 `ConsistencyLevel.CONSISTENCY_ALL`。

由于读取所有副本而导致的潜在性能下降仅在集群 reconfiguration 期间才显著。

#### Linearize Read

如果启用，线性化读取策略(`Policy.linearizeRead`)会强制对支持强一致性模式的服务器命名空间进行线性化读取。

#### Send Key

如果启用，发送 key (`Policy.sendKey`) 除了读取和写入的 hash digest 外，还会发送 user-defined key。如果 key 是在写入时发送的，key 将与服务器上的记录一起存储，并在扫描和查询时返回给客户端。

#### Socket Timeout

socket timeout (`Policy.socketTimeout`) 指定处理数据库命令时的 socket 空闲超时（以毫秒为单位）。

如果 socketTimeout 不为零并且 socket 已空闲至少 socketTimeout，则检查 maxRetries 和 totalTimeout。如果未超过 maxRetries 和 totalTimeout，则重试 transaction。

如果 socketTimeout 和 totalTimeout 都非零且 socketTimeout > totalTimeout，则 socketTimeout 将设置为 totalTimeout。

如果 socketTimeout 为零，则没有 socket 空闲限制。

#### Total Timeout

总超时(`Policy.totalTimeout`)以毫秒为单位指定总 transaction 超时。

totalTimeout 在客户端上进行跟踪，并与 wire protocol 中的 transaction 一起发送到服务器。客户端很可能会先超时，但服务器也可以使 transaction 超时。

如果 totalTimeout 不为零并且在 transaction 完成之前达到 totalTimeout，则 transaction 将因超时异常而中止。

#### Max Retries

Max retries(`Policy.maxRetries`)指定中止当前 transaction 之前的最大重试次数。初次尝试不计为重试。

如果超过 maxRetries，transaction 将因超时异常而中止。

不应该重试非幂等的数据库写入（例如 add()），因为如果客户端使先前的 transaction 尝试超时，则写入操作可能会执行多次。对于将 maxRetries 设置为零的非幂等写入，使用不同的 WritePolicy 非常重要。

读取默认值：2（初始尝试 + 2 次重试 = 3 次尝试）

write/query/scan 的默认值：0（不重试）

#### Sleep Between Retries

重试之间的休眠时间(`Policy.sleepBetweenRetries`)是重试之间休眠的毫秒数。输入零跳过休眠。当 maxRetries 为零时，将忽略此字段。此字段在异步模式下也会被忽略。

睡眠仅在连接错误和服务器超时时发生，这表明节点已关闭且集群正在重组。当客户端的 socketTimeout 到期时，睡眠不会发生。

当节点关闭时，读取不必休眠，因为集群在机损重组期间不会关闭读取。读取的默认值为零。

