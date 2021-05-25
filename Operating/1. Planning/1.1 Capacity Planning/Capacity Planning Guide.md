## Capacity Planning Guide

对于Aerospike数据库，可以通过如下所述计算容量要求来调整硬件大小。然后，根据您需要的是内存数据库，混合SSD/内存数据库，无持久性的内存缓存还是部署到云的数据库（例如在Amazon EC2上），可以配置SSD和RAM适当。

##### Contents

- [Required Memory](#required-memory)
    - [Memory Required for Primary index](#memory-required-for-primary-index)
    - [Calculating byte size of primary index](#calculating-byte-size-of-primary-index)
        - [Aerospike All Flash](#aerospike-all-flash)
    - [Capacity Planning for Set Indexes](#capacity-planning-for-set-indexes)
    - [Capacity Planning for Secondary Indexes](#capacity-planning-for-secondary-indexes)
    - [Memory Required for Data in Memory](#memory-required-for-data-in-memory)
        - [Memory Required for Data in Single-Bin Namespace](#memory-required-for-data-in-single-bin-namespace)
- [Data Storage Size](#data-storage-size)
    - [All Flash Index Device Space](#all-flash-index-device-space)
    - [Total Storage Required for Cluster](#total-storage-required-for-cluster)
- [Throughput(bytes)](#throughput)
- [Provisioning](#provisioning)

---

### <span id="required-memory"> Required Memory </span>

在Aerospike的 [Hybrid Memory](https://docs.aerospike.com/docs/architecture/storage.html) 体系结构中，索引始终存储在RAM中。 [primary index](https://docs.aerospike.com/docs/architecture/primary-index.html), [secondary indexes](https://docs.aerospike.com/docs/architecture/secondary-index.html) 和 [set indexes](https://docs.aerospike.com/docs/architecture/set-indexes.html) 都必须有足够的 RAM。您应该提供足够的 RAM，以免超出 [high water mark](https://docs.aerospike.com/docs/operations/configure/namespace/retention/index.html) ，默认情况下，高水位线是命名空间 `memory-size` 的 60% 。

从Aerospike Enterprise Edition 4.3开始，命名空间的主索引可以存储在专用设备上。在这种称为Aerospike All Flash的配置中，内存消耗降低到最低限度。

确保命名空间的合并 `memory-size` 不超过计算机上的可用 RAM。应该为操作系统，命名空间开销以及计算机上运行的其他软件保留足够的内存。

#### <span id="memory-required-for-primary-index"> Memory Required for Primary Index </span>

每个名称空间的主索引被划分为4096个分区，并且每个分区的结构都是一组小树枝 (sprigs)（shallow red-black trees, 浅红黑树）。这些小树枝指向记录元数据，每条记录64个字节。

#### <span id="calculating-byte-size-of-primary-index"> Calculating byte size of primary index </span>

在Aerospike的“混合内存”配置中，主索引消耗的内存为：

```
64 bytes × (replication factor) × (number of records)
```

[Replication factor](https://docs.aerospike.com/docs/architecture/data-distribution.html) 是每个记录在命名空间中具有的副本数。命名空间的默认副本因子为2 —— 每个记录的主副本和单个复制副本(replica copy)。

##### <span id="aerospike-all-flash"> Aerospike All Flash </span>

有关更多详细信息，请参阅 [Primary Index Configuration](https://docs.aerospike.com/docs/operations/configure/namespace/index/index.html#flash-index) 。

重要的是要了解全闪存大小的细微差别，因为按比例扩展全闪存命名空间可能需要增加 `paritition-tree-sprigs` ，这将需要滚动 [Cold Restart](https://docs.aerospike.com/docs/operations/manage/aerospike/cold_start/) 。添加节点将增加容量，但是随着小树枝填满并溢出其最初的4KiB磁盘分配，性能将会受到影响。

当命名空间使用全闪存配置时，记录元数据的64个字节不会存储在内存中，而是作为索引设备上4KiB块的一部分。仅主索引的小枝部分消耗RAM，每个小枝13个字节。

为了减少对索引设备的读取操作次数，请考虑索引块的“填充分数”(fill fraction)。您应该确保每个小枝包含少于64条记录（因为64 x 64B是4KiB）。

如果预计命名空间会快速增长，则您将希望使用较低的填充分数，亿流出空间以备将来记录。完整的小树枝将跨域多个单个 4KiB 索引库，并且可能需要读取多个单索引设备。为了缓解这种情况，修改小树枝的数量需要冷启动以重建主索引。因此，最好事先确定填充因子。
```
    sprigs per partition = (total unique records / (64 x fill fraction)) / 4096
```

`partition-tree-sprigs` 必须为2的幂，因此无论以上计算得出的结果如何，均应选择最接近的2的幂。

例如，如果有40亿个唯一对象，并且填充因子为1/2，则每个分区的小枝应为：
```
(4 x 10^9) / (64 x 0.5) / 4096 = ~ 30,517 -> nearest power of 2 = 32,768
```

每个小枝都需要13-bytes的RAM开销，因此
```
13 bytes x 32768 x 4096 x 2 = 3.25GiB
```

因此，如果有40亿个对象且副本因子为2，则与全闪存中的主索引（在整个集群中）相关联的RAM消耗为3.25GiB，而不是在同一示例中将在混合示例中使用的476.8GiB RAM内存配置。

在计算所需的小枝数量时，还必须进行计算，以确保为主索引在磁盘上提供了正确的空间量。然后应该调整 `mounts-size-limit` 。为此，将应用以下公式来获取此配置参数所需的最小大小。
```
mounts-size-limits = (((4096 × replication-factor) ÷ min_cluster_size) × partition-tree-sprigs) × 4KB
```

为解释以上内容，mounts-size-limit应该为4096（主分区数），乘以副本因子即可得到分区总数，再除以您将拥有的最小集群大小，即可得出分区每个节点最大值，乘以分区树小枝的数量即可得到每个节点的最大小枝，然后再乘以4KB（因为每个小枝至少占用4KB）。这应该是主索引可用挂载大小的最小大小。在为All Flash挂载分区磁盘时，还请考虑文件系统开销。

#### <span id="capacity-planning-for-set-indexes"> Capacity Planning for Set Indexes </span>

当索引一个 set 时，整个群集将使用大约4 MiB x 副本因子的RAM开销。

此外，每个记录副本的 set index 成本为16字节RAM。创建 set index 后，将立即为整个集群（跨节点划分）分配 16 MiB x replication factor of RAM，以容纳集合中的前一百万条记录。

因此，如果命名空间中的 1000 个集合具有 set index，则开销为 ~4GiB (在整个群集的各个节点之间分配)。初始阶段会预分配 ~16GiB，并且还会在群集的各个节点之间进行分配，非常适合每组中的前100万条记录。

#### <span id="capacity-planning-for-secondary-indexes"> Capacity Planning for Secondary Indexes </span>

可以选择在数据上建立二级索引。有关更多详细信息，请参见 [Secondary Index Capacity Planning](https://docs.aerospike.com/docs/operations/plan/capacity/secondary_indexes.html) 。

#### <span id="memory-required-for-data-in-memory"> Memory Required for Data in Memory </span>

如果将命名空间配置为将数据存储在内存中，则该存储的 RAM 需求可以计算为以下各项的总和：

- 每个记录的开销：
 ``` 
 2 bytes 
 ```

- 如果 [key](https://docs.aerospike.com/docs/architecture/data-model.html#records) 中保存记录 :
```
+ 12 bytes overhead + 1 byte (key type overhead) + (8 bytes (integer key) OR length of string/blob (string/blob key))
```

- 通常每个 bin 的开销：

任何一个
```
 + 12 bytes (5.4之前的Aerospike数据库版本或设置了XDR bin-policy，从而产生开销)
```
或
```
+ 11 bytes （Aerospike数据库版本5.4或更高版本以及XDR bin-policy不会产生开销）
```

以及
```
+ 6 bytes 如果设置了XDR bin-policy，则使用LUT发生开销。
```

```
+ 1 byte 对于src-id，如果开启XDR bin收敛功能。
```
[bin convergence](https://docs.aerospike.com/docs/operations/configure/cross-datacenter/bin_convergence.html#overhead) 。

- 每个bin的类型相关开销：

```
+ 0 bytes for bin tombstone (see bin-policy) or
+ 0 bytes for integer, double or boolean data or
+ 5 bytes for string, blob, list/map, geojson data
```

- Data : 所有记录的bin中的数据大小（整数，双精度和布尔数据为0字节，通过替换一些常规开销来存储）。Please see [Data Size](https://docs.aerospike.com/docs/operations/plan/capacity/data_sizes.html) 。
```
+ data size
```

例如，对于具有两个包含整数和长度为20个字符的字符串以及5.4之前的数据库版本的bin的记录，我们发现：

```
2 + (2 x 12) + (0 + 0) + (5 + 20) = 51 bytes.
```
或对于相同类型的记录，以及数据库版本5.4或更高版本（并且`bin-policy`不产生开销），我们发现：
```
2 + (2 × 11) + (0 + 0) + (5 + 20) = 49 bytes.
```

该内存实际上分为不同的分配 ———— 记记录开销加上（所有）常规 bin 开销在一个分配中，而类型相关的 bin 加数据开销在每个 bin 中单独分配。请注意，整数数据不需要按单元分配。系统堆将舍入分配大小，因此使用的字节数可能比上述计算所暗示的要多。

#### <span id="memory-required-for-data-in-single-bin-namespace"> Memory Required for Data in Single-Bin Namespaces </span>

如果将命名空间配置为 data-in-memory 且 `single-bin true`，则不需要上述记录开销和常规 bin 开销（首次分配），该开销存储在索引中。唯一需要的分配是与类型相关的开销和数据。因此，数字数据（整数，双精度）和布尔值没有内存存储成本 —— 开销和数据都存储在索引中。如果知道单个bin命名空间中的所有数据都是数字数据类型，则可以将命名空间中的 `data-in-index` 设置为true，以配置该命名空间以表明这一点。尽管已将其配置为将数据存储在内存中，这仍将使此名称空间能够快速重启。

---

### <span id="data-storage-size"> Data Storage Size </span>

一条记录的存储要求是以下各项的总和：

- 每个记录的开销:
```
35 bytes
```

- 如果使用非零的无效时间（TTL）。请注意，tombstone 没有到期时间：
```
+ 4 bytes
```

- 如果使用 set name：
```
+ 1 byte 开销 + set name length in bytes
```

- 如果存储记录的 key。固定key大小是客户端发送的确切不透明字节：
```
+ 1-3 bytes overhead (1 byte for key size <128 bytes, 2 bytes for <16K, 3 bytes for >= 16K) + 1 byte (key type overhead) + flat key size
```

- bin count 开销。single-bin和tombstone记录没有开销：
```
+1 byte for count < 128, +2 bytes for < 16K, or +3 bytes for >= 16K
```

- 每个bin的一般开销。single-bin无开销：
```
+ 1 byte + bin name length in bytes and

+ 6 bytes for LUT depending on XDR bin-policy and

+ 1 byte for src-id if XDR bin convergence is enabled.
```

- 每个 bin 的类型相关开销：
```
+ 1 byte for bin tombstone (see bin-policy) or

+ 2 bytes + (1, 2, 4, 8 bytes) for integer data values 0-255, 256-64K, 64K-4B, bigger or

+ 1 byte + 1 byte for boolean data values or

+ 1 byte + 8 bytes for double data or

+ 5 bytes + data size for all other data types. 
```
See [Data Size](https://docs.aerospike.com/docs/operations/plan/capacity/data_sizes.html) 。

然后，应将由此产生的存储大小四舍五入为16个字节的倍数。例如，一个具有10个字符长的 set name 且没有存储 key 的 tombstone 记录，我们需要：
```
35 + (1 + 10) = 46 -> rounded up = 48 bytes
```
或对于同一 set 中的记录（没有TTL），两个包含整数和20个字符的字符串的 bin（8个字符名称）：
```
  35 + (1 + 10) + 1 + (2 × (1 + 8)) + (2 + 8) + (5 + 20) = 100 -> rounded up = 112 bytes
```

#### <span id="all-flash-index-device-space"> All Flash Index Device Space </span>

有关更多详细信息，请参阅 [Primary Index Configuration](https://docs.aerospike.com/docs/operations/configure/namespace/index/index.html#flash-index) 页面。

在Aerospike All Flash配置中，假设索引块保存的元数据不超过64条记录，则每个分支都需要4KiB的索引设备空间。

```
    4 KiB x total unique sprigs x replication factor
```

在较早的示例中，我们有40亿条唯一记录，填充率为1/2，副本因子为2。我们计算出每分区的唯一小枝总数为32,768。

```
4 KiB x 32768 x 4096 x 2 = 1 TiB index device space needed for the cluster
```

现在，您可以计算每个节点的索引设备空间。

您要容忍群集拆分和节点崩溃。在集群分裂或节点丢失的情况下，请使用要运行的子集群的最小大小。

```
    device size per node = cluster-wide index device space needed / minimal number of nodes
```

对于上面的示例，假设 `min-cluster-size` 为3。每个节点的索引设备空间将需要为
```
 1 TiB / 3 = ~341GiB index device space per node
```

类似于用于命名空间RAM的 `high-water-memory-pct` 和用于SSD数据存储的 `high-water-disk-pct` ，有一个 `mounts-high-water-pct` 配置参数。默认情况下，它是`mounts-size-limit` 的80％，并且在违反此高水位标记时将从索引设备中退出。

在我们的示例中，我们将考虑这个高水位线，并相应地为每个节点设置索引设备：

```
 341GiB / 0.8 = 427GiB index device space per node
```

#### <span id="total-storage-required-for-cluster"> Total Storage Required for Cluster </span>

```
(size per record as calculated above) x (Number of records) x (replication factor)
```

数据可以存储在RAM或闪存（SSD）中。您的SSD容量不应超过50-60％。您可以使用 [our recommended SSDs](https://docs.aerospike.com/docs/operations/plan/ssd/ssd_certification.html) 或使用 [Aerospike Certification Tool](https://github.com/aerospike/act) 认证自己的SSD。

---

### <span id="throughput"> Throughput (bytes) </span>

```
    (number of records to be accessed in a second) × (the size of each record)
```

计算所需的吞吐量，以使即使一个节点出现故障，集群也将继续工作（即，我们要确保每个节点都能处理全部流量负载）。

---

### <span id="provisioning"> Provisioning </span>

See [Provisioning a Cluster](https://docs.aerospike.com/docs/operations/plan/capacity/provisioning.html) for examples.