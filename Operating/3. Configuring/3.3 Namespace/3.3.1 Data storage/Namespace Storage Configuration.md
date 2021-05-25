## Namespace Storage Configuration

Aerospike根据为 namespace 配置的 `storage engine` 来确定将数据存储在何处。这些引擎确定数据将被持久到磁盘，驻留在内存中还是同时都存储。这些决定将影响集群的耐用性，成本和性能。

##### Contents

- [Comparing Storage Engines](#comparing-storage-engines)
- [Storage Engine Configuration Recipes](#storage-engine-configuration-recipes)
    - [Recipe for an SSD Storage Engine](#recipe-for-an-ssd-storage-engine)
    - [Recipe for a Persistent Memory Storage Engine](#recipe-for-a-persistent-memroy-storage-engine)
    - [Recipe for an HDD Storage Engine with Data in Memory](#recipe-for-an-hdd-storage-engine-with-data-in-memory)
    - [Recipe for a HDD Storage Engine with Data in Index Engine](#recipe-for-a-hdd-storage-engine-with-data-in-index-engine)
    - [Recipe for Data in Memory Without Persistence](#recipe-for-data-in-memory-without-persistence)
    - [Recipe for Shadow Device](#recipe-for-shadow-device)
- [Where to Next?](#where-to-next)

---

### <span id="comparing-storage-engines"> Comparing Storage Engines </span> 

| Storage engine | Device (data not in memory) | All Flash | Device (data in memory) | Memory only (no persistence) | Persistent Memory*** |
| --- | --- | --- | --- | --- | --- |
| Index In Memory | ✓ | ✗ | ✓ | ✓ | ✓ | ✓ |
| [Fast Restarts](https://docs.aerospike.com/docs/operations/manage/aerospike/fast_start)* | ✓ | ✓ | ✓** | ✗ | ✓ |
| Survive a Power Outage | ✓ | ✓ | ✓ | ✗ | ✓ |

*在Aerospike Enterprise Edition上可用。 <br/>
**从Enterprise Edition版本3.15.1.3开始，主索引在 `data-in-memory` 重新启动后仍具有持久性。<br/>
** [Fast Restarts](https://docs.aerospike.com/docs/operations/manage/aerospike/fast_start) 还支持 `data-in-index` 配置。<br/>
*** 从Enterprise Edition 4.8开始，并且在 `feature-key-file` 中启用了功能。

---

### <span id="storage-engine-configuration-recipes"> Storage Engine Configuration Recipes </span>

以下配方需要修改位于 `/etc/aerospike/aerospike.conf` 的 Aerospike Server 配置文件。每个配方均描述了启用特定存储引擎所需的最小配置以及该引擎使用的存储大小参数。首先，请在首选编辑器中打开配置文件并进行适当修改。
```
sudo $EDITOR /etc/aerospike/aerospike.conf
```

#### <span id="recipe-for-an-ssd-storage-engine"> Recipe for an SSD Storage Engine </span>

SSD 命名空间的最小配置要求将 [storage-engine](https://docs.aerospike.com/docs/reference/configuration/index.html#storage-engine) 设置为 `device` ，并为该命名空间要使用的每个SSD添加 [device](https://docs.aerospike.com/docs/reference/configuration/index.html#device) 参数。 `device`的最大大小为 2TiB，对于较大的设备，划分为多个大小均小于 2TiB 的分区。此外，可能需要将 [memory-size](https://docs.aerospike.com/docs/reference/configuration/index.html#memory-size) 从默认的 4GiB 更改为适合预期的主索引大小的大小。有关调整主索引的帮助，请参阅 [Sizing Guide](https://docs.aerospike.com/docs/operations/plan/capacity) 。为了提高性能，通常建议将 SSD 支持的命名空间上的 [write-block-size](https://docs.aerospike.com/docs/reference/configuration/index.html#write-block-size) 从默认的 1MiB 减少到 128KiB 。这可能会因具体的工作量和记录的平均大小而有所不同，而找到正确的设置的最佳方法是运行一些具有不同值的基准测试。

```
namespace <namespace-name> {
    memory-size <SIZE>G         #主索引和二级索引的最大内存分配
    storage-engine device {     # 配置storage-engine用于持久性
        device /dev/<device>    # 原始设备。最大大小为 is 2 TiB
        # device /dev/<device>  # (可选) 其他的原始设备
        write-block-size 128K   # 调整块大小，以使其对SSD高效
    }
}
```

设备分区在任何时候都只能与一个命名空间关联。 [This article](https://discuss.aerospike.com/t/how-to-add-replace-remove-disks/746) 讨论了命名空间中添加和删除设备分区。

#### <span id="recipe-for-a-persistent-memroy-storage-engine"> Recipe for a Persistent Memory Storage Engine </span>

持久化命名空间的最小配置要求在配置文件中为该命名空间要使用的每个 pmem 存储文件设置两个参数 ：
- [storage-engine](https://docs.aerospike.com/docs/reference/configuration/index.html#storage-engine)
- [file](https://docs.aerospike.com/docs/reference/configuration/index.html#file). (要在配置中查看该条目的描述，请向下滚动返回结果，以查找具有 namespace context and subcontext `storage-engine device or pmem` 的 `file` 的条目 )。

此外，[filesize](https://docs.aerospike.com/docs/reference/configuration/index.html#filesize) 必须足够大以支持数据大小（最大允许值为 2TiB）。此外，可能需要将 [memory-size](https://docs.aerospike.com/docs/reference/configuration/index.html#memory-size) 从默认的 4GiB 更改为适合预期的主索引大小的大小。有关调整主索引的帮助，请参见 [Sizing Guide](https://docs.aerospike.com/docs/operations/plan/capacity) 。

从 5.1 版开始，永久内存(persistent memory) 命名空间与 Data-In-Memory 命名空间等效，以计算默认的 `service-threads`，并将默认为 CPU 个数（除非至少有一个 SSD 命名空间）。

在具有超线程的系统上，仅对物理核心进行计数，而在多插槽系统中，如果启用了 Non-Uniform Memory Access (NUMA) pining ，则每个 Aerospike 实例仅会对其所服务的套接字上的 CPU 核心进行计数。

```
namespace <namespace-name> {
    memory-size <SIZE>G             # 二级索引的最大内存分配(如果有的话)。
    storage-engine pmem {           # 将存储引擎配置为使用持久性。最大大小是2 TiB。
        file /mnt/pmem/<filename>   # pmem数据文件在服务器上的位置，
                                    # 其中 /mnt/pmem 是驻留在 pmem 中的 EXT4 或 XFS 文件系统的挂载点， 
                                    # 该文件系统已经用 DAX 选项挂载。
        # file /mnt/pmem/<another>  # (可选) pmem数据文件在服务器上的位置。
        filesize <SIZE>G            # 每个文件的最大大小 GiB.
    }
}
```

#### <span id="recipe-for-an-hdd-storage-engine-with-data-in-memory"> Recipe for an HDD Storage Engine with Data in Memory </span>

具有 Data-in-Memory 命名空间的 HDD 的最小配置涉及将 [storage-engine](https://docs.aerospike.com/docs/reference/configuration/index.html#storage-engine) 设置为 `device`，将 [data-in-memory](https://docs.aerospike.com/docs/reference/configuration/index.html#data-in-memory) 设置为 true，最后提供 [file](https://docs.aerospike.com/docs/reference/configuration/index.html#file) 参数列表以指示将数据保留在何处。此外，[filesize](https://docs.aerospike.com/docs/reference/configuration/index.html#filesize) 必须足够大以支持磁盘上的数据大小（最大允许值为 2 TiB）。对于常见的用例，这应该是大约是 `memory-size` 的 4 倍。最后，可能需要将 [memory-size](https://docs.aerospike.com/docs/reference/configuration/index.html#memory-size) 从默认的 4GiB 更改为适合预期的主索引大小和内存中数据的预期大小的大小。 有关调整 `filesize` 和 `memory-size` ，请参阅 [Sizing Guide](https://docs.aerospike.com/docs/operations/plan/capacity) 。

```
namespace <namespace-name> {
    memory-size <SIZE>G             # 主索引，数据和二级索引(如果有的话)的最大内存分配。
    storage-engine device {         # 将存储引擎配置为使用持久性。 Maximum size is 2 TiB
        file /opt/aerospike/<filename>  # 数据文件在服务器上的位置。
        # file /opt/aerospike/<another> # (可选) 数据文件在服务器上的位置。
        filesize <SIZE>G                # 每个文件的最大大小 GiB.
        data-in-memory true             # 指示所有数据也应在内存中。
    }
}
```

#### <span id="recipe-for-a-hdd-storage-engine-with-data-in-index-engine"> Recipe for a HDD Storage Engine with Data in Index Engine </span>

`data-in-index` 配置是针对非常特殊的用例的高度专业化的命名空间。如果您的数据是单个 bin 且可容纳 8 个字节，并且您需要执行 in-memory 命名空间的性能，但又不想失去Aerospike Enterprise Edition中提供的快速重新启动功能，那么 `data-in-index` 就是它。

`data-in-index`命名空间的最小配置涉及将 `single-bin` 设置为 true，将 [data-in-index](https://docs.aerospike.com/docs/reference/configuration/index.html#data-in-index) 设置为 true，将 [data-in-memory](https://docs.aerospike.com/docs/reference/configuration/index.html#data-in-memory) 设置为 true。此外，[storage-engine](https://docs.aerospike.com/docs/reference/configuration/index.html#storage-engine) 必须是 `device`，并且必须配置 [file](https://docs.aerospike.com/docs/reference/configuration/index.html#file) 或 [device](https://docs.aerospike.com/docs/reference/configuration/index.html#device) 参数以映射到此命名空间要使用的持久存储设备。最后，需要将内存大小从默认的 4GiB 调整为可以容纳主索引的大小，并且文件大小从其默认的 16GiB 调整为磁盘上数据的大小（最大允许值为 2 TiB）。有关调整 `filesize` 和 `memory-size` ，请参阅 [Sizing Guide](https://docs.aerospike.com/docs/operations/plan/capacity) 。

```
namespace <namespace-name> {
    memory-size <N>G                # 主索引，数据和二级索引(如果有的话)的最大内存分配。
    single-bin true                 # data-in-index要求为 true
    data-in-index true              # 启用索引整数存储。
    storage-engine device {         # 将存储引擎配置为使用持久性。
    file /opt/aerospike/<filename>  # 数据文件在服务器上的位置。
    # file /opt/aerospike/<another> # (可选) 数据文件在服务器上的位置。
    # device /dev/<device>          # 使用文件的可选选择

    filesize <SIZE>G               # 每个文件的最大大小 GiB. 最大大小 2TiB
    data-in-memory true            # 需要 data-in-index 为 true 。
    }
}
```

#### <span id="recipe-for-data-in-memory-without-persistence"> Recipe for Data in Memory Without Persistence </span>

没有持久化的命名空间的最小配置是将 [storage-engine](https://docs.aerospike.com/docs/reference/configuration/index.html#storage-engine) 设置为 `memory` 。如果您的命名空间需要的不仅仅是主索引和 data-in-memory 的默认 4GiB [memory-size](https://docs.aerospike.com/docs/reference/configuration/index.html#memory-size) 的分配，那么还需要响应地调整 `memory-size` 。有关调整 `memory-size`，请参阅 [Sizing Guide](https://docs.aerospike.com/docs/operations/plan/capacity) 。

```
namespace <namespace-name> {
    memory-size <SIZE>G   # 为数据、主索引和二级索引分配的最大内存。
    storage-engine memory # 将存储引擎配置为不使用持久性。
}
```

#### <span id="recipe-for-shadow-device"> Recipe for Shadow Device </span>

3.5.12版本中引入的影子设备(shadow device)存储模型是为云环境量身定制的，在云环境中，您可能具有临时性（非持久性）的超高性能 SSD。而持久性设备的性能却不如人们希望的那样。

所有写入均复制到另一个（影子）设备。该影子设备将充当持久存储。主设备仍照常接收所有操作。这导致持久的数据量具有较低的IOPS要求，同时仍然获得了非持久卷的IOPS优势。所有这些都无需使用大量的RAM。影子设备仅需要满足工作负载的写入IOPS要求，而不需要读取。

这是SSD存储引擎的扩展。

当利用网络连接的设备（例如AWS上的EBS）或将影子设备重新分配给其他实例时，在运行版本3.16.0.1或更高版本时，建议首先在集群中的各个节点上配置 `node-id` 为了将其保留在可能会重新连接到实例的影子设备的潜在新实例上，并避免在群集中重新分配分区。

要使用Shadow Devices，只需在同一行上声明非持久卷之后添加持久卷即可。

影子设备必须大于或等于主设备的大小。

```
namespace <namespace-name> {
    ...
    storage-engine device {
        device /dev/sdb /dev/sdf  # SDB是快速的临时卷，而SDF是较慢的持久卷
        ...
    }
}
```

在上面的示例中，/dev/sdb是快速的非持久性设备。 /dev/sdf是持久性设备。顺序很重要。设备必须在同一行上列出以进行影子设备配置。

影子设备配置可以与多个设备结合使用。请注意一对一映射：

```
    storage-engine device{
        device /dev/sdb /dev/sdf
        device /dev/sdc /dev/sdg
        ...
    }
```

在配置名称空间以使用任何形式的持久性时，应注意将给定的文件或设备分区仅与单个命名空间相关联。两个命名空间不能共享相同的文件或分区。为多个命名空间配置相同的文件或分区可能会导致节点启动问题和/或损坏该文件或分区中的现有数据。

---

### <span id="where-to-next"> Where to next ?</span>

- 配置 [Data Retention Policy](https://docs.aerospike.com/docs/operations/configure/namespace/retention) ，该策略确定在最初写入记录后将其保留多长时间。
- 配置 [Data Durability Policy](https://docs.aerospike.com/docs/operations/configure/namespace/durability) ，该策略确定在集群中保留多少条记录的复制副本 (replica copies) 。
- 或者返回 [Configure Page](https://docs.aerospike.com/docs/operations/configure) 。