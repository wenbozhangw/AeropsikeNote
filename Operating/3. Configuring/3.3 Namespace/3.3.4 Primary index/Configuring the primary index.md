## Configuring the primary index

Aerospike 的主索引可以用三种不同的方式存储：RAM, persistent memory (PM), and flash (normally NVMe SSDs)。同一集群的不同命名空间可以使用不同的索引存储方法。

##### Contents

- [index-type](#index-type)
    - [Cautions for Systemd](#cautions-for-systemd)
- [Persistent memory index](#persistent-memory-index)
    - [Filesystem configuration](#filesystem-configuration)
    - [index-type for PM](#index-type-for-pm)
- [Flash index](#flash-index)
    - [Cautions](#cautions)
    - [Enable Flash index for a namespace](#enable-flash-index-for-a-namespace)
    - [Recommended filesystem type - XFS](#recommended-filesystem-type-xfs)
    - [Sample configuration snippet](#sample-configuration-snippet)
- [Flash index calculations summary](#flash-index-calculations-summary)
    - [Number of sprigs required](#number-of-sprigs-required)
    - [Disk space required](#disk-space-required)
    - [RAM required](#ram-required)

---

### <span id="index-type"> index-type </span>

要指定索引存储方法，请使用命名空间上下文配置项 `index-type` 。

默认的 `index-type` 为 `shmem`，索引存储在 RAM （Linux共享内存）中。要指定持久性内存索引 (persistent memory index)，请使用 `index-type` `pmem` ，如果指定索引在
flash，使用 `index-type` `flash` 。

#### <span id="cautions-for-systemd"> Cautions for Systemd </span>

在 [Systemd environment](https://discuss.aerospike.com/t/the-asd-process-cold-starts-after-a-dirty-exit-which-appears-to-be-clean/6472)
中，您可能需要将 **TimeoutSec** 从默认值 15s 增加。此设置位于 `/usr/lib/systemd/system/aerospike.service` 中。这可以防止 systemd 在关闭服务时过早地终止 asd
进程。关闭集群的主索引清理过程可能会比默认的 15s 花费更长的时间。从 [4.6.0.2](https://docs.aerospike.com/enterprise/download/server/notes.html#4.6.0.2)
版本开始，在 Aerospike 中，此默认值已增加到 10 分钟。

---

### <span id="persistent-memory-index"> Persistent memory index </span>

Aerospike的持久性内存 (Persistent memory, PM)索引功能允许将主索引存储在持久性内存（例如 Intel Optane DC NVDIMM）中，而不是基于 RAM 的共享内存段中。

与基于 RAM
的索引不同，PM索引在集群节点OS的重新启动过程中得以保留，这允许在重新启动后 [fast restarts](https://docs.aerospike.com/docs/operations/manage/aerospike/fast_start/index.html)
Aerospike。

Aerospike 要求可通过 `fsdax`，即通过 /dev/pmem0 之类的块设备来访问持久性内存：

- 必须将 NVDIMM 区域配置为 `AppDirect` 区域，如以下示例中从具有 750-GiB `AppDirect` 区域的计算机中配置：

```
$ sudo ipmctl show -region
SocketID             ISetID    PersistentMemoryType  Capacity FreeCapacity HealthState
0 0x59727f4821b32ccc               AppDirect        750.0 GiB      0.0 GiB     Healthy
```

- 必须将 NVDIMM 区域转换为 `fsdax` 命名空间，如以下示例在同一台计算机上所示：

```
$ sudo ndctl list
[
  {
    "dev":"namespace0.0",
    "mode":"fsdax",
    "blockdev":"pmem0",
    ...
  }
]
```

#### <span id="filesystem-configuration"> Filesystem configuration </span>

PM块设备必须包含具有DAX（直接访问）功能的文件系统，例如EXT4或XFS。在上面示例中的机器上，这可以通过通常的方式来完成：

EXT4 filesystem:

```
$ sudo mkfs.ext4 /dev/pmem0
```

XFS filesystem:

```
$ sudo mkfs.xfs -f -d su=2m,sw=1 /dev/pmem0
```

最后，需要使用 `dax` 挂载选项挂载文件系统。`dax` 挂载选项很重要。如果没有此选项，则在持久内存之间的所有 I/O 中都将包含 Linux 页缓存，这将大大降低性能。

在下面的示例中，我们使用 `/mnt/pmem0` 作为安装点：

```
$ sudo mount -o dax /dev/pmem0 /mnt/pmem0
```

请记住，通过将挂载添加到 `/etc/fstab` 中，使其保持持久状态，以使其在系统重新引导后仍然有效。可以将安装点配置行从 `/etc/mtab` 复制到 `/etc/fstab` 。

#### <span id="index-type-for-pm"> index-type for PM </span>

主索引类型(primary index type)是按命名空间配置的。要为命名空间启用 PM 索引，请将索引类型为 `pmem` 的 `index-type` 子部分添加到其命名空间部分。添加的 `index-type` 子节必须包含：

- 一个或多个 `mount` 指令，用于指示要用于 PM 索引的持久性存储器的安装点。 <br/>
  单个命名空间可以在多个安装点之间使用持久性内存，并将在所有安装点之间平均分配。<br/>
  相反，可以在多个命名空间之间共享安装点。命名空间的持久性内存分配所给予的文件名是特定于命名空间的，这避免的命名空间共享安装点时文件名之间的冲突。
- 一个 `mounts-size-limit` 指令，用于指示此命名空间在给定安装点之间可用空间的份额。<br/>
  当多个命名空间共享安装点时，此配置指令会高速 Aerospike 预期每个命名空间跨安装点使用的总可用内存量是多少。<br/>
  例如，指定的值与配置项 `mounts-high-water-pct`（默认值为80）一起构成计算逐出的高水位线的基础。<br/>
  如果未在命名空间之间共享安装点，则只需指定总可用空间。<br/>
  确保 `mounts-size-limit` 小于或等于文件系统的大小。

一下配置代码片段扩展了上面的示例并使用所有 `/mnt/pmem0` 内存（即750GiB）可用于命名空间：

```
namespace test {
    ...
    index-type pmem {
        mount /mnt/pmem0
        mounts-size-limit 750G
    }
    ...
}
```

---

### <span id="flash-index"> Flash Index </span>

Aerospike全闪存功能允许将主索引存储在闪存设备（通常为 NVMe SSD）上。

此索引存储方法通常用于具有相对较小记录的超大型主索引。准确性对于容量规划和配置的某些方面至关重要。

#### <span id="cautions"> Cautions </span>

重要的是要了解 [All Flash Sizing](https://docs.aerospike.com/docs/operations/plan/capacity/index.html#aerospike-all-flash)
的微妙之处，因为按比例扩展全闪存命名空间可能需要增加 `partition-tree-sprigs`
，这将需要滚动的 [Cold Restart](https://docs.aerospike.com/docs/operations/manage/aerospike/cold_start/) 。

虽然建议在任何配置中都调整内核的 `min_free_kbytes`
参数，但在使用全闪存时尤其重要。linux内核将通过缓存磁盘写入来尝试利用所有可用空间。对于全闪存配置，如果没有足够的可用RAM用于正常的系统操作，则可能导致OOM终止。为此，Aerospike建议设置 `min_free_kbytes=1153434`(
1.1GiB)。有关更多信息，请参考 [here](https://discuss.aerospike.com/t/how-to-tune-the-linux-kernel-for-memory-performance/4195) 。

#### <span id="enable-flash-index-for-a-namespace"> Enable Flash index for a namespace </span>

要为命名空间启用 Flash 索引，请在配置文件中，将具有 Flash 索引类型的 `index-type` 子部分添加到其命名空间部分。添加的 `index-type` 子节必须包含：

- 一个或多个 `mount` 指令，用于指示要用于 PM 索引的持久性存储器的安装点。 <br/>
  单个命名空间可以在多个安装点之间使用持久性内存，并将在所有安装点之间平均分配。<br/>
  相反，可以在多个命名空间之间共享安装点。命名空间的持久性内存分配所给予的文件名是特定于命名空间的，这避免的命名空间共享安装点时文件名之间的冲突。
- 一个 `mounts-size-limit` 指令，用于指示此命名空间在给定安装点之间可用空间的份额。<br/>
  当多个命名空间共享安装点时，此配置指令会高速 Aerospike 预期每个命名空间跨安装点使用的总可用内存量是多少。<br/>
  例如，指定的值与配置项 `mounts-high-water-pct`（默认值为80）一起构成计算逐出的高水位线的基础。<br/>
  如果未在命名空间之间共享安装点，则只需指定总可用空间。

#### <span id="recommended-filesystem-type-xfs"> Recommended filesystem type - XFS </span>

推荐使用XFS文件系统，因为与EXT4相比，它已显示出对文件的更好的并发访问。

#### <span id="Recommmendation-for-multiple-physical-devices"> Recommendation for multiple physical devices </span>

拥有更多的物理设备可以通过增加跨设备的并行度来提高性能。每个物理设备更多的分区并不一定会提高性能。Aerospike实例化至少4个不同的空间分配(文件)，如果有更多的设备(逻辑分区或物理设备)
，将分配更多。一次实例化1个以上的空间有助于与同一个空间竞争，这在沉重的插入负载中很重要。

#### <span id="sample-configuration-snippet"> Sample configuration snippet </span>

```
namespace test {
    ...
    partition-tree-sprigs 1M # Typically very large for flash index - see sizing guide.
    ...
    index-type flash {
        mount /mnt/nvme0
        mount /mnt/nvme1
        mount /mnt/nvme2
        mount /mnt/nvme3
        mounts-size-limit 1T
    }
    ...
}
```

---

### <span id="flash-index-calculations-summary"> Flash index calculations summary </span>

有关更多信息，请参阅 [Linux capacity planning](https://docs.aerospike.com/docs/operations/plan/capacity/index.html#aerospike-all-flash)
。

这是计算40亿条记录的命名空间（复制因子为2）所需的磁盘空间和内存的摘要。

#### <span id="number-of-sprigs-required"> Number of sprigs required </span>

- 40亿条记录÷4096个分区÷每个 sprigs 32条记录，保留 half-fill-factor =〜30,517
- 舍入为2的幂：每个分区32,768个sprigs

#### <span id="disk-space-required"> Disk space required </span>

- 每个分区32,768个 sprigs ×4096个分区×2副本因子×每个块的4KiB大小=整个集群1TiB
- 整个集群所需的1 TiB÷3个最少的节点数÷0.8的 `mounts-high-water-pct` 为80％ = 每个节点427 GiB

由于全闪存使用包含多个文件的文件系统，因此挂载点大小应略大于427 GiB，以容纳文件系统的开销。这是与文件系统有关的。 427 GiB用于文件内的实际可用空间存储。

#### <span id="ram-required"> RAM required </span>

- 每个分区 32,768 个sprigs × 4096个分区 × 2副本因子×每个sprigs所需的13字节内存=整个群集3,328 MiB
- 整个群集所需的3,328 MiB÷3的最小节点数÷0.8的`mounts-high-water-pct`为80％=每个节点1,387 MiB


