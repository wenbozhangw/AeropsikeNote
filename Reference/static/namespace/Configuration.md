# Namespace 配置说明

## 一、配置说明
| name | default | Subcontext | Introduced | comment|
| ---- | ---- | ---- | ----| ----|
| **cold-start-empty**| false | SubContext: storage-engine device | 3.3.21 | 将此设置为 true 将会导致冷启动忽略驱动器上的现有数据，并像空启动一样启动。不影响[快速重启](fast-restart/FastRestart.md)。|
| ~~cold-start-evict-ttl~~ | 4294967295 |   | Removed : 4.5.1 | 设置 TTL，在此 TTL 以下，在冷启动期间将逐出(不加载)记录。当逐出深度较深时，通常用于加速冷启动。默认值表示 -1。 |
| _commit-min-size_ | 0 | SubContext: storage-engine device | `Eneterprise` 4.0 | 启用`commit-to-device`后，磁盘刷新的最小大小（以字节为单位）。必须是 `2` 的次幂。可以设置为 `4K`。默认值为 0 将自动检测设备的最小尺寸。通常建议保留此配置的默认值。 |
| _commit-to-device_ | false | SubContext: storage-engine device or pmem  | `Enterprise` 4.8.0 (pmem) | 在确认客户端之前，等待写入刷新到磁盘或 pmem。仅适用于启用 `strong-consistency` 的命名空间。如果使用 `commit-to-device` 设置为 true 的 `storage-engine device` 文件存储，则将 `read-page-cache` 设置为 true 可能很有用。 如果发生崩溃，则在将 `commit-to-device` 设置为 true 的情况下运行时，在随后的冷启动时将信任所有分区。使用 `shadow devices` 时，此设置将在返回客户端之前同时应用于主服务器和 `shadow device` 服务器，因此可能会进一步减慢处理的等待时间。|
| _compression_ | none | SubContext: storage-engine device or pmem | `Enterprise` 4.5.0 (device) 4.8.0 (pmem) | Options : none, lz4, snappy, zstd. <br/> 使用压缩功能需要启动一项功能`feature-key-file`，并制定用于压缩 SSD 或 pmem 存储文件上的记录的算法。对于 `zstd` 中 `compression-level` 可以指定。 从版本 4.5.3.2 开始，flat storage format 还用于复制、迁移 和 副本解析 wire format，在使用压缩时可提供潜在的显著网络带宽和 CPU 使用率。 <br/>  将命名空间的压缩算法设置为`zstd`:<br/>    `asinfo -v 'set-config:context=namespace;id=namespaceName;compression=zstd'` <br/> 注意: 压缩不允许写入大于配置的写入块大小(对于 pmem 固定为 8 MB) 的记录，即使它们的压缩大小小于写入块的大小也是如此。 压缩发生在存储层和结构层。支持在不同的节点上使用不同的压缩选项进行基准测试。      警告：对于 Aerospike 4.9 之前的版本，请勿动态设置 `storage-engine memory` 的 compression。这可能会损坏内存并导致服务器崩溃。 |
| _compression-level_ | 0 | SubContext: storage-engine device or pmem | `Enterprise` 4.5.0 (device) 4.8.0 (pmem) | 注意：这是 `storage-engine` 的 `compression-level`， 不是 XDR `dc namespace` 的 `compression-level`。向下查看该参数。 <br/>   允许范围: 1-9  <br/>  与 zstd 压缩一起使用的压缩级别。控制压缩速度和压缩比之间的权衡。例如 9，较高的级别值表示效率更高，但压缩速度较慢。例如 1，较低的水平意味着效率较低，但压缩速度更快。请注意，只有在该项使用时应指定`compression zstd`。 <br/>     在 4.6.x 之前的 Aerospike Server 版本中，如果在使用时从未指定此设置 `compression zstd`，则显示默认标志 0，并将使用压缩级别 9。在 Aerospike Server 4.6.x 或更高版本中，如果在使用时从未指定此设置 `compression zstd`，则显示默认标志 9，并将使用压缩级别9 。 <br/>  该压缩配置指令属于一个命名空间的存储引擎部分。 <br/>  将命名空间的压缩级别设置为 1 : <br/>  `asinfo -v 'set-config:context=namespace;id=namespaceName;compression-level=1'` |
| data-in-index | false | unanimous | | 在单个 bin 情况下的优化将仅允许将整数或浮点数存储在索引空间中。仅当`storage-engine` 为 `device` 并且 `single-bin` 为 `true` 时才可以使用。<br/> 注意：允许快速重启 single bin, data in memory, integer or float data only pattern。对于未配置`data-in-index` 的 `single-bin` 命名空间，整数或浮点数据也将存储在索引中，但当 `data-in-memory`设置为 ture 时将不允许快速重启。 |
| data-in-memory | false | Subcontext : storage-engine device | | 始终将所有数据的副本保存在内存中。 |
| defrag-startup-minimum | 10 | Subcontext : storage-engine device or pmem | | 服务在启动时至少需要指定的可用空间量(百分比)。 |
| **device** | | Subcontext : storage-engine device | | 用于存储命名空间的原始设备。  <br/> 持久化到两个设备： <br/>   `device /dev/sdb`    <br/> `device /dev/sdc`   <br/>    持久化到设备和影子设备 ：<br/>`device /dev/nvme0n1 /dev/sdb`    <br/>  你可以为多个命名空间指定多个设备。 <br/> 从 4.3.0.2 版本开始，通过 `info` API请求配置时，特定的设备的 key 为 `storage-engine.device[ix]`，其中 `ix` 是一个索引，用于标识该设备及其关联的统计数据(如统计 `storage-engine.device[ix].age`)。 如果已配置，则该设备的影子设备将显示为 `storage-engine.device[ix].shadow`。  <br/>   警告：每个设备尺寸最大限制为 2 TiB。您不能在同一个命名空间中同时使用使用 `device` 和 `file`。从版本 4.2 开始，每个命名空间限制为 128 个设备(对于 3.12.1 之前的版本，限制为 64，而在先前版本中，限制为 22。) |
| direct-files | false | Subcontext: storage-engine device or pmem | Introduced : 4.3.1 (device) 4.8.0 (pmem) | 仅与文件存储相关。如果使用 `storage-engine pmem`，则仅与影子文件存储相关。如果 `direct-files` 设置为 true，那么将为文件 IO 启用 `odirect` 和 `odsync` 标志。这意味着写缓冲区一直同步写入文件系统下的设备。如果使用 `storage-engine device` 的 `data-in-memory` 设置为 false，则将 `read-page-cache` 设置为 true 可能会有用。有关更多详细信息，请参阅 [Buffering and Caching in Aerospike](https://discuss.aerospike.com/t/buffering-and-caching-in-aerospike/5623) 。 <br/> 警告：可能会影响性能，尤其是在文件由硬盘支持的情况下。 |
| disable-cold-start-eviction | false | | Introduced : 4.3.0.2 | 如果为 `true`，则仅对此命名空间禁用在冷启动时可能发生的逐出。 | 
| ~~disable-odirect~~ | false | Subcontext: storage-engine device | Removed : 4.3.1 | 如果为`true`，则在读取或写入原始设备时禁用 `odirect` 标志。这使操作系统可以利用页缓存，并可以帮助解决某些工作负载类型的延迟。在全面投入生产之前，应该对单节点进行测试或部署。有关更多详细信息，请参阅 [Buffering and Caching in Aerospike](https://discuss.aerospike.com/t/buffering-and-caching-in-aerospike/5623) 。 <br/>  有关类似功能，请参阅[read-page-cache](https://www.aerospike.com/docs/reference/configuration/index.html?show-removed=1#read-page-cache) <br/>  注意：在此内核上运行的高性能存储子系统可能会受到此设置的不利影响，因为在访问存储子系统之前检查页面缓存可能会受到不利影响。可以考虑使用具有较高 `cache_read_pct` 的工作负载，但也应检查 增加 `post-write-queue` 配置参数的影响。性能较低的存储子系统（例如，连接到网络的存储子系统）可能会从禁用 odirect 标志中受益匪浅。 |
| disable-odsync | false | Subcontext : storage-engine device or pmem |  Introduced : 4.5.0.12, 4.5.1.8, 4.5.2.3, 4.5.3.3 | 如果 `disable-odsync` 设置为 `true`，那么 Linux `O_DSYNC` I/O 标志设置为 false（即对 file，direct-files 设置为true）。禁用 `O_DSYNC` 可能会以宽松的耐用性为代价来提高性能。有关更多详细信息，请参阅 [Buffering and Caching in Aerospike](https://discuss.aerospike.com/t/buffering-and-caching-in-aerospike/5623) 。 <br/> 注意：`disbale-odsync` 和 `commit-to-device` 不能都设置为 `true`。将两者都设置为 `true` 将阻止服务器在 持久性/性能 权衡的情况下启动。 <br/> 注意：对于 PMEM 中的数据，此设置仅与影子文件存储有关。有关此设置的影响的更多详细信息：写入或更新数据库记录时，更改后的记录最初位于服务器节点上的内存缓冲区中，其结构称为当前写入块。定期通过 Linux pwrite(2) 系统调用将写入块刷新到 SSD，间隔由 `flush-max-ms` 配置参数限制。在将写入块保存到 SSD 之前（默认情况下，写入 DRAM 之后最多 1 秒），如果系统发生故障（例如断电），其内容将丢失。默认行为（启动`O_DSYNC`）是 pwrite(2) 将阻塞调用线程，直到将数据写入SSD。这种延迟减少了线程可以完成的单位时间的工作，从而可能导致性能下降。禁用`O_DSYNC`后，调用 pwrite(2) 的线程将立即返回，从而使该线程能够执行其他工作。但是，数据可能要到将来某个时候才能传输到设备。如果在调用 pwrite(2) 与将数据完全写入SSD之间的时间间隔内发生系统故障，则将丢失数据（仅在该特定节点上）。在持久性和性能之间进行权衡是否值得，取决于应用程序，Linux I/O 实现（这会影响数据传输的速度）以及记录数据的敏感性。对于某些数据（例如，频繁更新的传感器读数），风险可能是可以接受的，而对于其他数据（例如，金融交易），风险可能是不能接受的。有关Aerospike缓存的完整说明，请参见[Buffering and Caching in Aerospike](https://discuss.aerospike.com/t/buffering-and-caching-in-aerospike/5623) 。<br/> 如果你要为集群使用机架感知功能，我们预期潜在数据丢失的唯一方法是在SAME短时间内发生故障的服务器的副本因子（RF）数量，如上所述，一秒（默认情况下） + 虚拟/云延迟（如果启用了`disbale-odsync`），在存储记录副本的机架的RF数量中，每个机架一秒。因此，例如，在将RF服务器配置为两个机架的 RF = 2 配置中，如果要发生潜在的数据丢失，则单个服务器将必须在非常短的时间内在 EACH 机架中发生故障。 |
| earth-radius-meters | 6571000 | Subcontext : geo2dsphere-within | Introduced : 3.7.0.1 | 地球的半径（以米为单位），因为此处的工作空间是完整的地球。 |
| ~~enable-osync~~ | false | Subcontext : storage-engine device | Introduced : 3.3.21 <br/> Removed : 4.3.1.1 | 仅与原始设备相关（与文件存储无关）。告诉设备在每次写入时刷新。这可能会影响性能。有关Aerospike缓存的完整说明，请参见[Buffering and Caching in Aerospike](https://discuss.aerospike.com/t/buffering-and-caching-in-aerospike/5623) 。 |
| _encryption_ | aes-128 | Subcontext: storage-engine device or pmem | `Enterprise` Introduced : 4.5.0 (device) 4.8.0 (pmem) | 选项： aes-128, aes-256 <br/> 指定静态加密所使用的算法。需要在功能密钥文件中启用功能。 |
| _encryption-key-file_ | N/A | Subcontext: storage-engine device or pmem | `Enterprise` Introduced : 3.15.1 (device) 4.8.0 (pmem) | 通过指定用户提供的随机生成的加密密钥的文件系统路径或存储该用户提供的随机生成的加密密钥的 Vault secret 的名称，来启用静态加密。在版本 5.3+ 中，也可以指定一个保存加密密钥的环境变量。 <br/> 在 5.1+ 版本中，为了与 HashiCorp Vault 进行替代集成，必须在配置参数的值前面加上 literal `vault`：并且必须在 Vault 服务上后面加上机密名称。有关更多信息，请参阅 [Optional security with Vault integration](https://www.aerospike.com/docs/operations/configure/security/vault/index.html) 。 <br/>  在 5.3+ 版本中，可以将配置参数设置为 `env-b64:<variable_name>`，将从指定的环境变量中读取 base64 编码的密钥，并将其解码为二进制形式。 <br/>  有关静态加密的工作方式，警告和其他注意事项的信息，请参阅[Configring Encryption-at-Rest](https://www.aerospike.com/docs/operations/configure/security/encryption-at-rest/index.html#technical-details) 。 <br/> 一个相关的参数是 `encryption`。 <br/> 需要在 `feature-key-file` 中启用功能。 <br/> Example: <br/>     为 test 命名空间开启 encryption-at-rest : <br/>  `namespace test {` <br/>  `...`   <br/>   `   storage-engine device {`  <br/>  `device /dev/sda1`  <br/>  `...` <br/>  `encryption-key-file key.dat`  <br/>  `}`  <br/>  `...`   <br/>  `}`  <br/>  为通过 HashiCorp Vault包括的命名空间 test 启用 encryption-at-rest ：<br/>  `namespace test {` <br/>  `...`   <br/>   `   storage-engine device {`  <br/>  `device /dev/sda1`  <br/>  `...` <br/>  `encryption-key-file vault:encryption-key-file`  <br/>  `}`  <br/>  `...`   <br/>  `}`  <br/> 仅在解析配置文件后，才在启动时加载密钥文件的内容。一旦运行了Aerospike进程，就可以安全地删除密钥文件，但是请记住，您需要安全地存储密钥文件，以便能够重新使用该文件来重新启动Aerospike进程。   <br/>   警告：添加，删除或更改密钥文件需要停止Aerospike守护程序，将存储设备清零并重新启动Aerospike守护程序。迁移应先完成，然后再进行集群中的下一个节点。因此，可以在整个集群中进行诸如滚动方式的更改。 |
| file |  | Subcontext : storage-engine device or pmem | | 硬盘（使用文件系统）或pmem（从4.8版本开始）上的数据文件路径。从 4.3.0.2 版本开始，该文件可以包含一个可选的"影子文件"作为第二个参数。 <br/> Example： <br/> 持久化到两个文件  :  <br/> `file /mnt/disk1/myfile1.dat` <br/> `file /mnt/disk2/myfile2.dat` <br/> 持久化到两个文件(pmem)  :  <br/> `file /mnt/pmem/myfile1.dat` <br/> `file /mnt/pmem/myfile2.dat` <br/> 持久化到带有影子文件的文件： <br/> `file /mnt/pmem1/rw_file.dat /mnt/sdb1/shadow_file.dat` <br/> `file /mnt/nvme0n1/rw_file.dat /mnt/sdb1/shadow_file.dat` <br/> 你可以为每个命名空间指定多个文件。目录路径必须存在，并且正在运行Aerospike进程的用户/组应具有读/写权限。文件本身将由该过程创建。 <br/> <br/> 从 4.3.0.2 版本开始，通过 `info` API请求配置时，特定的设备的 key 为 `storage-engine.device[ix]`，其中 `ix` 是一个索引，用于标识该设备及其关联的统计数据(如统计 `storage-engine.device[ix].age`)。 如果已配置，则该设备的影子设备将显示为 `storage-engine.device[ix].shadow`。 <br/> 警告：每个设备尺寸最大限制为 2 TiB。您不能在同一个命名空间中同时使用使用 `device` 和 `file`。从版本 4.2 开始，每个命名空间限制为 128 个设备(对于 3.12.1 之前的版本，限制为 64，而在先前版本中，限制为 22。) |
| **filesize** | | SubContext: storage-engine device or pmem | required | 在此命名空间中定义的每个文件存储的最大大小。 在 4.3.0.2 之前，默认值为 16GiB。从 4.3.0.2 版本开始，在将命名空间配置为使用文件时，需要显式设置文件大小。 <br/> 支持以下后缀： <br/> `K Kibibyte (KiB)`  <br/> `M Mebibyte (MiB)`  <br/> `G Gibibyte (GiB)`   <br/> `T Tebibyte (TiB)`   <br/> `P Pebibyte (PiB)` <br/> Example : `filesize 500G` <br/>  注意：文件大小最大限制为 2 TiB。 2.x的默认值：17179869184。 |
| **index-stage-size** | 1G | | Introduced : 4.2.0.2 | 用于确定主索引区域大小的配置。 <br/> 配置必须为2的幂。 4.2.0.2 之前的下限为 128MB，上线为 1GB，对于版本 4.2.0.2 及更高版本，上限为 16GB。此设置将更改每个 2048（EE）或 256（CE）可能的竞技场阶段的大小，并且需要冷启动。支持的符号例如 G 表示千兆字节， M 表示兆字节， K 表示千字节。 |
| _index-type_ | shmem |  | `Enterprise` Introduced : 4.3.0.2 (shmem) 4.3.0.2 (flash) 4.5.0.1 (pmem) | 选项 : shmem, flash, pmem  <br/> 如果是 `shmem`，则索引存储在 Linux 共享内存(DRAM)中；即，在重启节点后，需要冷启动来重建节点。 <br/> 如果是 `flash`，则索引存储在块存储设备（通常为 `NVMe SSD`）中;即使在重启后，节点也能够快速重启。有关大小调整的详细信息，请参阅 [Aerospike All Flash](https://www.aerospike.com/docs/operations/plan/capacity/index.html#aerospike-all-flash) 容量计划页面。 <br/> 如果是 `pmem`，则索引存储在持久性内存中（例如，Intel Optane DC持久性内存）；即使在重启后，节点也能够快速重启。 <br/> 要为 Aerospike Server 4.3.0.2 到 4.7 版本设置为 `flash`，需要在`feature-key-file`中启用功能。设置为 pmem 需要在 `feature-key-file` 文件中启用功能。 <br/>  注意：在社区版上，这将显示为 `undefine` 且不可配置。|
| level-mod | 1 | Subcontext : geo2dsphere-within | Introduced : 3.7.0.1 | 如果指定，则仅使用 (`level-min-level`) 是 `level-mod` 数倍的单元格 (默认为1)。这有效地允许增加 S2 Cell Id hierarchy 的 branching factor。当前，唯一允许的参数值为 1、2或3，分别对应于 4、16和64的分支因子。 | 
| _mount_ | | Subcontext : index-type flash,<br/> index-type pmem| `Enterprise` Introduced : 4.3.0.2 (flash) 4.5.0.1 (pmem) | 挂在目录的路径（通常在 NVMe SSD 上），每个命名空间可能有多个安装。尽管不建议这样做，但挂载可能会与其他命名空间共享。有关在使用 `index-type flash`时，调整大小的详细信息，请参阅 [Capacity Planning](https://www.aerospike.com/docs/operations/plan/capacity/index.html#all-flash-index-device-space) 页面。当 `index-type pmem` 和 `auto-pin numa`一起使用时，不在本地 NUMA 节点上的已配置安装将被忽略。因此，在不同 NUMA 节点上运行的 Aerospike 服务的不同势力可以相同的已配置安装，而操作员无需确定哪些安装在哪个 NUMA 节点上。 <br/> 注意： 通过 `info` API请求配置时，特定挂在的密钥将是 `storage-engine.mount[ix]`，其中 `ix` 是用于标识此挂载及其相关统计信息的索引（例如统计 `index-type.mount[ix].age`）。 |
| **namespace** | | | | 注意：这是命名空间上下文中的命名空间，而不是 XDR 上下文中的命名空间。搜索命名空间，然后查看Context标题，以确保使用正确的参数。 <br/> 定义一个命名空间，有关更多信息，请参见 [Namespace Configuration](https://www.aerospike.com/docs/operations/configure/namespace/index.html) 。 <br/> 警告：集群中命名空间的数量是有限制的。请参见 [Upper Sizing Bounds and Naming Constraints](https://www.aerospike.com/docs/guide/limitations.html) |
| **num-partitions** | 32 | Subcontext : si |  | 配置以更改用于查询查找的二级索引树的数量。 <br/> 增加此配置可以减少索引树的深度，并且可以帮助二级索引查找提高执行性能。但是，增加这些也会导致内存开销，因此建议您在调整此配置时监视内存利用率和基准测试。 | 
| ~~partition-tree-locks~~ | 8 | | Introduced : 3.11 <br/> Removed : 4.2 | 每个分区的 lock pairs 数量(tree lock and reduce lock)。从 4.2 版本开始删除（在该版本及更高版本中，硬编码为最大256个）。必须是 2 的精确幂，介于 1 和 256 之间。不得超过`partition-tree-sprigs`。提供更多锁可以减少在搜索不同记录（tree lock）之间以及在创建/删除和归约（reduce lock）之间的潜在争用。<br/> 注意：每个命名空间的内存开销：固定的基本大小为 64K，每 16 个`partition-tree-sprigs`加1M，每个`partition-tree-sprigs`锁为320K。此外，企业版还需要每 16 个`partition-tree-sprigs`额外增加 320K 来支持快速重启。 <br/> 一个好的最小准则是保持默认值 8，直到集群大小超过 15，然后集群大小每次加倍时对其加倍（对于16到31的集群大小为16，对于32到63的集群大小为32，等等）。实际上，集群越大，每个节点将拥有的分区越少，从而在锁上产生更多的潜在争用。 <br/> [What are sprigs?](https://discuss.aerospike.com/t/faq-what-are-sprigs/4936) |
| partition-tree-sprigs | 256 | | Introduced : 3.11 | 每个分区要使用的 tree sprigs 数量。对于 4.2 及更高版本，默认值为 256。 必须是 2 的精确次幂。 4096 或 8192 sprigs 将使常见的工作负载和用例受益。对于可能需要更多负载的工作负载（值大于 32K），Enterprise Edition 被许可方应联系 Aerospike 支持以获取指导。即使内存开销似乎可以接受，配置太多的 sprigs 可能不仅没有好处，而且实际上可能会对集群产生不利影响 ： <br/> - 子集群将必须容纳较大集群中的所有 sprigs（除非已将 `min-clustter-size` 配置为防止此类子集群的形成）。  <br/> - 所需的内存也必须是连续的（碎片内存可能会阻止分配内存）。 <br/>     - 节点上的分支过多会延迟关闭，并在随后的重新启动时导致不必要的冷重启。 <br/>  **更改此配置参数将强制冷启动**。提供更多的 trees (sprigs) 可减少级别并加快搜索速度。它还会导致减少 reduce lock 的阻塞（在每个 sprigs 之间解除 reduce lock，并且与单个分区树相比， sprigs 的遍历时间要少得多）。 <br/> Example : <br/> A 4-node cluster, `replication-factor 2, 2048 partition-tree-sprigs`。对于小于 4.2 的版本, using `8 partition-tree-locks`。 对于大于等于 4.2 的版本, 每个分区硬编码为 `256` 个 `partition-tree-locks`。 <br/> 对于 4.2 及更高版本，sprigs 的每个节点命名空间内存开销为 : `Community Edition:  64K + (8M x 2 + 8B x 2048 x 4096 x 2) / 4  = 64K + 4M + 32M = 36.06M` <br/> `Enterprise Edition: 64K + (8M x 2 + (8B + 5B) x 2048 x 4096 x 2) / 4 = 64K + 4M + 32M + 20M = 56.06M` <br/> 对于 4.2 之上的版本， sprigs 的每个节点命名空间内存开销为 : <br/> `Community Edition:  64K + 2.5M + 128M = 130.56M` <br/> `Enterprise Edition: 64K + 2.5M + 128M + 40M = 170.56M` <br/> 注意：  <br/>  对于4.2及更高版本： <br/> - sprigs 的默认值和最小值为 256，最高为 256M。 <br/> - 现在，将`partition-tree-locks`的值硬编码为每个分区 256 个。每个锁对应 8 个字节。 <br/> - 每个 sprigs 为 8 bytes。此外，企业版还要求每个 sprigs 有 5 bytes。<br/> - sprigs 和 lock 仅分配给该节点拥有的分区。因此，随着集群变大，每个节点的开销会减少。 <br/> 对于 4.2 之前的版本： <br/> - sprigs 可以设置为最小 16，最大 4096，但必须大于 `partition-tree-locks`。 <br/>    - 每个节点的命名空间内存开销：固定的基本大小为 64K，外加每 16 个`partition-tree-sprigs` 1M，每个 `partition-tree-locks` 320K。此外，企业版还需要每 16 个 `partition-tree-sprigs` 额外增加 320K 来支持快速启动。 <br/> - 负担得起额外内存开销的用户应将其更改为至少 256 （企业版为 21M 的开销），甚至可能希望一直打到最大值 4096（企业版为 336M 的开销），预测未来的增长（因为更改此参数将强制冷启动）。 | 
| **replication-factor** | 2 | | `unanimous` | 整个集群中维护的记录的副本数（包括主副本）。 <br/> 注意：对于 3.15.1.3 之前的版本，有效的副本因子在 `repl-facor` 名称下返回。对于 3.15.1.3 及更高版本，有效副本因子将在`effective_replication_factor`下返回。 <br/> 警告：对副本因子的更改要求重新启动整个集群。 |
| scheduler-mode | (set by system) | Subcontext : storage-engine device | | 非 NVMe 驱动器 (SSD或HDD) 的可选 I/O 调度程序。 <br/> See FAQ - [What is the purpose of setting the disk scheduler?](https://discuss.aerospike.com/t/faq-what-is-the-purpose-of-setting-the-disk-scheduler/3680). |
| serialize-tomb-raider | false | SubContext: storage-engine device or pmem | `Enterprise` Introduced : 4.3.0 (device) 4.8.0 (pmem) | 防止不同命名空间的 tombstone raids 同时运行。 |
| **Set** | | | | 开始一个集合上下文，集合后必须跟集合名称。 | 
| si | | | | 以 si(二级索引)上下文开头，si后面必须是二级索引名称。 |
| sindex-startup-device-scan | false | Subcontext : storage-engine device | Introduced : 5.3.0 | 启动时，通过扫描设备来建立二级索引。 <br/> 如果命名空间中的大多数记录都在带有二级索引的集合中，则将此配置设置为 true 很可能会加快二级索引的重建速度。是否更快，还取决于其他因素，例如平均记录大小和已配置设备的数量。最终，实验是确定是否设置此配置的最佳方法。 <br/> 警告: `sindex-startup-device-scan` 和 `data-in-memory` 都必须设置为 `true`。 |
| single-bin | false | | `unanimous` | 将其设置为 true 将不允许有多个 bin (列)用于一条记录。 <br/> 注意：用于节省存储空间并在不需要事先读取和更新操作上提供性能增强。例如 List 追加，Map key-value 更新或增量操作之类的操作仍需要读取。需要重新初始化存储。具有`data-in-memory`的`single-bin`不允许存储用户密钥(sendKey true)。要将`user-key`和`single-bin`一起存储，storage不得将`data-in-memory`设置为true。对于针对 `single-bin` 命名空间的UDF操作，要求 bin 名称必须为空字符串才能读取或写入 bin（仅对于 3.5 及更高版本，对于以前的版本，不支持对于 `single-bin` 命名空间的UDF）。有关此参数的进一步建议，请联系Aerospike。 | 
| **storage-engine** | | | | 确定写入是否持久化，可接受的值为： <br/> - device 写入该节点的数据将会被保存到原始设备或文件中。 <br/> - memory 写入该节点的数据将仅写入DRAM。 <br/> - pmem 写入该节点的数据将被写入持久性内存（仅企业版，并且需要 `feature-key-file` 中启用功能）。 <br/> Example : <br/> 定义一个 In-Memory Only Namespace : `storage-engine memory` <br/> 定义一个持久化命名空间 ： `storage-engine device { ... }` <br/> 定义一个持久化内存命名空间 ： `storage-engine pmem { ... }` | 
| strict | true | Subcontext : geo2dsphere-within | Introduced : 3.7.0.1 | Aerospike进行了额外的完整性检查，以验证 S2 返回的点是否在用户的查询区域内。设置为false时，Aerospike不会进行此附加项检查，并按原样发送结果。 |
| _strong-consistency_ | false | | `Enterprise` Introduced : 4.0 | 将命名空间设置为 [Strong Consistency](https://www.aerospike.com/docs/guide/consistency.html) 模式，以使一致性优于可用性。允许启用线性化读取。有关更多详细信息，请参考 [Configuring Strong Consistency](https://www.aerospike.com/docs/operations/configure/consistency/index.html) 和 [Consistency Management](https://www.aerospike.com/docs/operations/manage/consistency/index.html) 。 <br/> 需要在 `feature-key-file` 中启用功能。 <br/> 注意：不支持通过简单的在配置中打开功能将 [`Available mode (AP)`](https://www.aerospike.com/docs/architecture/consistency.html#available-mode) 命名空间更改为 `Strong Consistency mode (SC)` 命名空间。为了创建一个高度一致的命名空间，需要情况存储空间。可以通过在 AP 命名空间上执行备份并还原到 SC 命名空间来迁移到 SC 命名空间。 |
| write-block-size | 1M | Subcontext : storage-engine device | | 写入磁盘的每个 I/O 块的大小（以字节为单位），这有效地设置了最大的对象大小。对于 4.2 以及更高版本，最大允许大小为 8388608（或8M）。 对于4.2之前的版本，最大允许大小为1048576（或1M）。较大的 `write-block-size` 可能会对性能产生不利影响，请参阅 [ FAQ - Write Block Size ](https://discuss.aerospike.com/t/faq-write-block-size/681) 。 <br/> 支持以下后缀： <br/> - K Kibibyte (KiB) <br/> - M Mebibyte (MiB) <br/> Example : `write-block-size 128K` <br/> 建议： <br/> `SSD: 131072 (128K)` <br/> `HDD: 1048576 (1M)` <br/> 调整块大小以使其对 I/O 有效。对于 pmem，此配置不可用，因为 `write-block-size` 已硬编码为 8MiB。 |





















### interface
 - [compression] https://www.aerospike.com/docs/operations/configure/namespace/storage/compression.html#configuration
- [shadow-device] https://www.aerospike.com/docs/cloud/Shadow-device.html

- partition-tree-sprigs
- strong-consistency