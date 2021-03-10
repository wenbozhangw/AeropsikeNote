## Aerospike配置

### 配置Aerospike数据库
Aerospike仅使用单个文件配置数据库节点。配置文件默认位置为`/etc/aerospike/aerospike.conf`。

配置文件划分为不同的 ***contexts（上下文）***，其中有些是可选的。这些上下文可以在配置文件中排列顺序是任意的。上下文可以进一步分为 ***sub-contexts（子上下文）***。

配置文件中一行最大长度为 ***1024***个字符。（在Aerospike Server 5.0 之前，行长度限制为 ***256*** 个字符）。

下面是常见上下文和子上下文的配置文件结构示例。下面显示的大多数上下文都是空的，它们没有实际参数，用大括号表示`{}`。子上下文缩进。

```
service {} 						# 调整参数和进程所有者

network {							# 用于配置集群内和应用程序节点通讯
  service {}					# 工具/应用程序通讯协议
  fabric {}						# 集群内通讯协议
  info {}							# 管理员telnet控制台协议
  heartbeat {}				# 集群形成协议
}

security {						# （可选，仅限企业版）在集群上启用 ACL
  enable-security true
}

logging {}						# 日志配置

xdr {									# （可选，仅限企业版）配置 Cross-Datacenter（跨数据中心） 复制
  dc <name> {} 				# 远程数据中心节点列表
}

namespace <name> {		# 定义命名空间记录策略和存储引擎
  storage {}					# 配置持久化或缺乏持久化
  set {}							# （可选）设置特定的记录策略
}

mod-lua {							# UDF模块的位置
  user-path /opt/aerospike/usr/udf/lua
}
```

### 配置步骤

1. [配置 Network service 和 heartbeat sub-contexts](https://www.aerospike.com/docs/operations/configure/network/)
2. [配置 Namespaces](https://www.aerospike.com/docs/operations/configure/namespace/)
3. 配置 [Logging](https://www.aerospike.com/docs/operations/configure/log/) 和 [Log Rotate](https://www.aerospike.com/docs/operations/configure/log/index.html#log-rotate)
4. （可选）配置 [Security](https://www.aerospike.com/docs/operations/configure/security/)
5. （可选）配置 [Rack-Aware](https://www.aerospike.com/docs/operations/configure/network/rack-aware/)
6. （可选）配置 [Cross-Datacenter Replication](https://www.aerospike.com/docs/operations/configure/cross-datacenter/)

### 静态配置参数详解

#### 1. logging配置参数

| name | Default                          | tag  | Comment                                                      |
| ---- | -------------------------------- | ---- | ------------------------------------------------------------ |
| file | /var/log/aerospike/aerospike.log |      | 指定用于记录的服务器日志文件的路径。不要与命名空间上下文中的文件混淆。你可以针对不同的上下文使用多个文件。 |

example：

```
logging {
  file /var/log/aerospike/aerospike.log {
    context any info
  }
  
  file /var/log/aerospike/aerospike_debug.log {
    context any debug
  }
}
```

上下文指定要记录的日志的上下文和级别。你可以结合使用上下文和日志记录级别。此配置可以与 `file` 或 `console`类型的的日志记录一起使用。可以从命令中获得不同的上下文及其日志记录级别。

```
asinfo -v log/0
```

简要输出：

```
misc:INFO;alloc:INFO;arenax:INFO;hardware:INFO;jem:INFO;msg:INFO;...
```

支持以下日志记录级别：

- context any info

- context any debug

- context any warning

- context any critical

- context any detail



### 动态配置参数详解

#### 一、service配置参数

| name | Default | tag               | Comment                        |
| ----------------------------- | ------- | ----------------- | ------------------------------ |
| advertise-ipv6                | false   | enterprise/static | 需要心跳v3。设置为true启动IPv6 |
| ~~allow-inline-transactions~~ | true    | removed:3.11 | 默认情况下，内存中的读操作由服务线程直接处理，而不是卸载转移到操作队列，然后由操作线程拾取。将 allow-inline-transactions设置为false将强制所有操作分配到一个操作队列。这在有限的网络队列和/或服务线程的环境中很有帮助。要在整个集群中禁用内联操作：`asadm -e "asinfo -v 'set-config:context=service;allow-inline-transactions=false'"` |
| auto-pin | none | static | 这个配置控制CPU固定的不同选项。在 *4.7* 之前的Aerospike版本中使用此配置时，服务线程和处理队列都不能在配置文件中配置；两者都默认为cpu的数量。在Aerospike 4.7+中，可以配置服务线程，但如果该配置生效，则必须是cpu数量的倍数。可能的值是：<br />- `none`        依赖于Linux的 [irqbalance](#irqbalance)。<br />- `cpu`          CPU固定 - Aerospike控制所有NIC队列中断的亲缘性。<br />- `numa`        CPU和NUMA固定 - 将 `asd` 的内存和CPU使用率限制为单个NUMA节点。<br />- `adq`          应用程序设备队列固定 - Aerospike根据与相应客户端连接关联的NIC队列将客户端请求分派给CPU。需要启用了 ADQ 的NIC 和 NIC 的手动配置。<br />`cpu`和`numa`需要Linux内核 3.19+。默认来说需要 Ubuntu 15.04+， Debian 8 (3.16)，CentOS 7 (3.10)。如果需要，Linux内核必须升级。`adq`需要Linux内核 4.12+。取消任何自动固定后，需要重新启动以恢复中断的系统默认值。将 `auto-pin` 设置为 `cpu`时，Aerospike 4.7之前的版本不允许在配置文件中设置事务队列和服务现场。两者将被强制为CPU的数量 - 这也是 Aerospike版本3.12+中的默认值。 Aerospike 4.7+版本允许设置服务线程，但要求配置的数量必须是CPU数量的倍数。使用这些配置之前，请联系Aerospike支持以获取建议和基准详细信息。<br />**注意**：网络接口硬件应支持MSI。MSI通过PCI总线将终端从外围设备（例如NIC）发送到CPU。较老的硬件有专门的线路，因此CPU和设备之间的任何数据交换通过PCI总线，中断通过一个单独的路径处理。但最近几天，一切都通过PCI总线。不支持MSI的网络接口将assert如下:<br />`FAILED ASSERTION (hardware): (hardware.c:1087) interface eth0 does not support MSIs`<br />NIC队列与CPU核的比例也必须大于1/4。以下消息将被记录到控制台，服务器将不会启动:<br />`WARNING (hardware): (hardware.c:1605) eth0 has very few NIC queues; only 8 out of 32 CPUs handle(s) NIC interrupts` |
| batch-index-threads | #cpu | dynamic | 批处理索引响应工作线程数。在版本3.12+中，默认情况下将其设置为可用的CPU内核数。在以前的版本中，默认值为4。每个线程都有自己的队列。这些线程只处理通过套接字将批处理响应缓冲区发送回客户端。值范围：1-256。<br />动态设置 batch-index-threads 为16：<br />`asinfo -v "set-config:context=service;batch-index-threads=16"`<br />Tip: 3.12版本之前允许最大值为64。 |
| batch-max-buffers-per-queue | 255 | dynamic | 每个批处理索引队列中被标记为已满之前允许的128KB响应缓冲区的数量。批处理索引队列（每个批处理索引线程一个）可以具有多于`batch-max-buffers-per-queue`缓冲区，但是直到它低于该数量，它才会接收任何新的批处理。当所有的队列都大于`batch-max-buffers-per-queue`时，新的批处理请求将被拒绝，并且将在服务器上记录错误："*Failed to find active batch queue that is not full*."。<br />动态设置batch-max-buffers-per-queue为512：<br />`asinfo -v "set-config:context=service;batch-max-buffers-per-queue=512"` |
| batch-max-requests | 5000 | dynamic | 每个节点允许的最大key个数。<br />动态设置batch-max-requests为6000：<br />`asinfo -v "set-config:context=service;batch-max-requests=6000"` |
| batch-max-unused-buffers | 256 | dynamic | 缓冲池中允许的最大128KB响应缓冲区的个数。如果达到限制，则已完成的缓冲区将在批处理请求结束时销毁。对于大量的批量工作负载，建议增加这个配置参数，以避免不必要的破坏和重新创建缓冲区，这会影响CPU负载。<br />动态设置 batch-max-unused-buffers为512：<br />`asinfo -v "set-config:context=service;batch-max-unused-buffers=512"` |
| ~~batch-priority~~ | 200 | removed:4.4/dynamic | 生成前的顺序读取数。数字越大，优先级越高。仅适用于旧的batch direct协议，该协议已在版本4.4中删除。<br />动态设置 batch-priority为300：<br />`asinfo -v "set-config:context=service;batch-priority=300"` |
| batch-threads | 4 | removed:4.4/dynamic | batch direct 工作线程的数量。batch direct是旧版的批处理协议。该线程处理完整的批处理请求。所有批处理线程只有一个批处理队列。值范围：0-256。注意旧的batch direct协议已经在4.4版本移除。<br />动态设置 batch-threads 为8：<br />`asinfo -v "set-config:context=service;batch-threads=8"`<br />tips: 3.12版本之前最大值为 64。 |

### 补充内容

#### <span id="irqbalance">irqbalance</span>

irqbalance用于优化中断分配，它会自动收集系统数据以分析使用模式，并依据系统负载状况将工作状态置于 Performance mode 或 Power-save mode。

- 处于Performance mode 时，irqbalance 会将中断尽可能均匀地分发给各个 CPU core，以充分利用 CPU 多核，提升性能。
- 处于Power-save mode 时，irqbalance 会将中断集中分配给第一个 CPU，以保证其它空闲 CPU 的睡眠时间，降低能耗。（暂不讨论这种模式）。

简单来说，Irqbalance的主要功能是优化中断分配，收集系统数据并分析，通过修改中断对于cpu的亲和性来尽量让中断合理的分配到各个cpu，以充分利用多核cpu，提升性能。

分析代码之前，首先来了解两个概念，numa架构和smp_affinity。简单画个图来解释下numa：



​	NUMA模式时一种分布式存储器访问方式，处理器可以同时访问不同的存储器地址，大幅度提高并行性。NUMA模式下，处理器被划分成多个”节点“ （node），每个节点被分配的有本地存储器空间。所有节点中的处理器都可以访问全部的系统物理存储器，但是访问本节点内的存储器所需要的时间，比访问某些远程节点内的存储器所花的时间要少的多。 irqbalance就是根据这种架构来分配中断的。主要的原因是避免中断在节点中迁移产生过多的代价。

smp_affinity是用来设置中断亲缘的CPU的mask码，简单来说就是在CPU上分配中断。SMP affinity 是通过操作 /proc/irq/ 目录中的文件来控制的。在 /proc/irq 中是与你系统中存在的 irq 相对应的目录（不是所有的 irq 都可用）。在每个目录中都有 ”smp_affinity“文件。

首先看一下 irqbalance用到的数据结构是什么样的：



简单来说就是根据cpu的结构由上到下建立了一个树形结构，当然，为了平衡中断，每个节点还会挂接本节点分配的中断。

树形结构建立好之后，自然是开始分配中断。irqbalance中把中断分成了八种类型：

```shell
#define IRQ_OTHER   		0
#define IRQ_LEGACY			1
#define IRQ_SCSI				2
#define IRQ_TIMER				3
#define IRQ_ETH					4
#define IRQ_GBETH				5
#define IRQ_10GBETH			6
#define IRQ_VIRT_EVENT	7	
```

依据就是pci设备初始化时注册的类型：/sys/bus/pci/devices/0000:00:01.0/class

每种中断类型又分别对应一种分配方式，分配方式一共有四种，代表中断的分配范围：

```
BALANCE_PACKAGE
BALANCE_CACHE
BALANCE_NONE
BALANCE_CORE
```

首先，中断在numa_node中分配，有两种情况：

/sys/bus/pci/devices/0000:00:01.0/numa_node中指定了非 -1 的numa_code，则把中断分配到对应的numa；如果是 -1 的话，则根据中断数平均的分到两个numa。分配好 numa_node 之后开始在整个树中进行分配，分配哪一个层次的原则是

```
BALANCE_NONE 分配在 numa_node 层
BALANCE_PACKAGE 分配在 package 层
BALANCE_CACHE 分配在 cache 层
BALANCE_CORE 分配在 core 层
```

决定出那一层后，最后就是在每个层次中分配节点，原则是分配在负载最小的子节点，如果负责相同则分配在中断种类最少的节点。

每个节点有各自的负载，自下而上进行计算。处于最底层的每个逻辑CPU的负载的计算方法是：在 /proc/stat 获取每个cpu的信息（cpu0 2383 0 298701 468097 158010 572 121175 0 0 0）取第6、7项，分别代表从系统启动开始累积到当前时刻，硬中断、软中断时间（单位是 jiffies），然后将累加的值转换成纳秒单位，转换方法是：和 * 1 * 10^9/HZ。

了解了逻辑cpu的负载的计算方法不难得到负载所表示的意义：单位时间（10s）内，cpu处理软中断加上硬中断的时间的和。

逻辑cpu这一层的负载计算完成之后，要开始计算上层节点的负载情况，计算方法是父节点负载等于各孩子节点负载的和的平均值，自下向上进行运算，如下图所示



irqbalance的最终目的在于平衡中断，现在环境已经搭建好了，就差平衡中断了。但是，平衡之前还有一件事情要做，就是计算每个中断的负载。中断的负载不同于前面说的负载，运算比较复杂，等于本层次单位中断的负载情况再乘以每个中断新增个数，中断的负载也是自下向上进行运算。

首先是各节点的平均中断数的计算，每个节点的中断数等于父节点的中断数除以该节点的个数再加上该节点的中断数，注意：这里说的中断数不是中断的种数，是所有中断的新增的个数的和然后用每个节点的负载除以该节点的平均处理的中断数，得到该节点单位中断所占用的负载，最后针对每一个中断，用该中断在单位时间内（10s）新增的个数乘以单位中断所占用的负载，得到每个中断自己的负载情况。附上图示：



平衡算法如下：得到每个节点的负载以及每个中断的负载之后，就需要找到负载较高的节点，把该节点的中断从节点中移动到其他的节点来平衡每个cpu的中断。简单来说，是统计每一个层次所有节点的负载的离散状态，找出偏差比较高的节点，把一个或多个中断从本节点剔除，重新分配到该层次负载较小的节点，来达到平衡的目的。

取cpu层次的来解释一下，其他层次类似：经过前面的计算已经得到了每个cpu的负载，也就是得到了一些样本数据，接下来计算负载的平均值和标准差（用于描述数据的离散情况）接下来是找出负载异常的样本数据，方法找到负载数据与平均值的差大于标准差的样本，有一个前提是该样本所包含的中断种数需要多于1种，然后把该样本中的中断按照中断的负载情况由大到小进行排序，依次从该节点移除，直到该节点的负载情况小于等于平均值为止最后就是把剔除的中断重新进行分配，分配的时候是选取负载最小的节点进行分配。

先整理一下irqbalance的流程：初始化的过程只是建立链表的过程，暂不描述，只考虑正常运行状态时的流程

- 处理间隔是10s
- 清除所有中断的负载值
- /proc/interrupts读取中断，并记录中断数
- /proc/stat读取每个cpu的负载，并依次计算每个层次每个节点的负载以及每个中断的负载
- 通过平衡算法找出需要重新分配的中断
- 把需要重新分配的中断加入到新的节点中
- 配置smp_affinity使处理生效

irqbalance支持用户配置每个中断的分配情况，设置在/proc/irq/#irq/affinity_hint中，irqbalance有三种模式处理这个配置

- EXACT模式下用户设置的cpu掩码强制生效
- SUBSET模式下，会尽量把中断分配到用户指定的cpu上，最终生效的是用户设置的掩码和中断所属节点的掩码的交集
- IGNORE模式下，不考虑用户的配置

irqbalance比较适合中断种类非常多，单一中断数量并不是很多的情况，可以很均衡的分配中断。如果遇到中断种类过少或者是某一个中断数量过大，会导致中断不停的在cpu之间迁移，每10s迁移一次，会降低系统性能，并且会导致过多的中断偶尔同时集中同一个cpu上（原因有二，一是平衡中断时优先转移的是负载较大的中断；二是没有计算平衡之后的负载情况）。irqbalance的计算是建立在假设每种中断的处理时间大概相等的情况下，实际的真实状态可能并非如此。irqbalance对于中断的迁移只能在规定的作用域之内进行迁移，特别的，对于numa来说，一旦大部分中断被分配到了同一个numa上，则不论如何平衡，都不会使中断迁移到另一个numa的cpu上。

irqbalance根据系统中断负载的情况，自动迁移中断保持中断的平衡，同时会考虑到省电因素等等。 但是在实时系统中会导致中断自动漂移，对性能造成不稳定因素，在高性能的场合建议关闭。

#### <span id="batch-operation">batch operation</span>

使用 Aerospike 批处理请求快速有效的检索大量记录。

批处理请求与查询不同，因为它们基于主键。该应用程序在批处理请求中传递主键列表。返回一系列的结果。

使用批处理请求可以：

- 有效的实现客户端联接。
- 存储并在计算中检索多个数据点。
- 确定 key 到期或 key 是否仍然存在。

Aerospike支持一下批处理请求：

- `batchGet`
- `batchExists`
- `batchGetHeaders`：仅读取记录元数据（expiration/TTL and generation）。不读取bins。
- `batchGetComplex`：批量读取具有不同命名空间，bin名称和读取类型的记录。（`batchGetComplex` requires Aerospike Server version 3.6.0+）

##### Example

以下Java代码示例在一个批处理调用中获取多个记录。

```
    void batchRead (
        AerospikeClient client,
        BatchPolicy policy,
        String namespace,
        String set,
        String binName,
        int size
    ) throws Exception {
        Key[] keys = new Key[size];
        for (int i = 0; i < size; i++) {
            keys[i] = new Key(namespace, set, i + 1);
        }

        Record[] records = client.get(policy, keys, binName);

        for (int i = 0; i < records.length; i++) {
            Key key = keys[i];
            Record record = records[i];

            if (record != null) {
                Object value = record.getValue(binName);
                // Process value here.
            }
        }
    }
```

##### Implementation

客户端确定哪些记录在哪里，并为服务器创建批处理请求。批处理请求列出了要检索的主键和可选的bin名称。批处理请求是串行还是并行发生，具体取决于批处理策略。同步模式下的并行请求需要额外的线程，这些线程是从线程池创建或获取的。

批处理请求对每个服务器使用单个网络处理，从而使开发人员不必并行处理请求。多个 keys 使用一个网络请求，这对于大量的小记录非常有好处，但是在每台服务器的记录数很小或每条记录返回的数据很大时却没有那么大的好处。请注意，批处理请求可能会增加某些请求的等待时间，但这仅是因为客户端通常会等到从服务器节点检索到所有 key 后才将控制器返回给调用方。

一些客户端（例如 c 客户端）支持在每条记录到达时立即将其交付，从而允许客户端应用程序首先处理最快的服务器中的数据。

##### Batch Protocols

从 3.6.0 版本开始，Aerospike服务器支持两种批处理协议：Batch Direct 和 Batch Index。

###### Batch Index

Batch Index 允许单个请求使用 keys，namespaces， bin name filters，和 read types （read bins，read header，exists）的任意组合来检索记录。服务器通过在批处理中将 keys 作为单记录处理分发来处理此协议。 由于 batch index 使用与单个记录读取相同的处理方式，因此在集群建议期间支持其他服务器的代理。批处理请求的运行优先级也与单记录事务相同。

从 Aerospike 数据库 4.7 版本开始，[predicate expressions](https://www.aerospike.com/docs/guide/predicate.html)可以应用于任何批处理操作。

Batch Index的过程为：

1. 接收来自客户端的批处理请求。
2. 将请求拆分为多个单个记录请求。
3. 在单个记录线程队列中分配这些单个记录请求。
4. 将单个响应在响应缓冲区中合并到128KB。
5. 将响应缓冲区放在批处理响应线程队列上，以使处理记录的线程不会阻塞。
6. 批处理响应线程将这些响应缓冲区返回给客户端。

如果连接的服务器不支持Batch Index，则直接使用Batch Direct协议。

| name                        | default                  | max  | dynamic | description                                                  |
| --------------------------- | ------------------------ | ---- | ------- | ------------------------------------------------------------ |
| batch-max-request           | 5000                     |      | true    | 每个节点允许的最大key数量。用于防止意外的大批量请求由于过多的内存消耗而导致服务器不稳定。 |
| batch-max-buffers-per-queue | 255                      |      | true    | 每个batch index队列中允许的最大128KB响应缓冲区数。如果所有批处理索引队列都已满，则拒绝新的批处理请求。 |
| batch-max-unused-buffers    | 256                      |      | true    | 缓冲池中允许的最大128KB响应缓冲区数。如果达到限制，则将创建新缓冲区（如果在每个队列的 batch-max-buffers-queue 中），并在批处理请求结束时完成，则将其销毁。这基本上是缓冲池的大小。 |
| batch-index-threads         | #cpu or 4 (3.12版本之前) | 64   | true    | 批处理响应工作程序线程数。每个线程都有自己的队列。这些线程只处理通过套接字将批处理响应缓冲区发送回客户端。可以使用的最大内存计算为：batch-index-threads * batch-max-buffer-per-queue * 128KB。 |

客户端可以在服务器运行时更改动态变量。

```
int
as_batch_init()
{
  // 加锁
	cf_mutex_init(&batch_resize_lock);

  // 如果没有指定，设置全局配置的 batch-index-threads 为 cpu的总个数。
	// Default 'batch-index-threads' can't be set before call to cf_topo_init().
	if (g_config.n_batch_index_threads == 0) {
		g_config.n_batch_index_threads = cf_topo_count_cpus();
	}

  // 日志
	cf_info(AS_BATCH, "starting %u batch-index-threads", g_config.n_batch_index_threads);

  // 初始as的批处理线程池
	int rc = as_thread_pool_init_fixed(&batch_thread_pool, g_config.n_batch_index_threads, as_batch_worker,
			sizeof(as_batch_work), offsetof(as_batch_work,complete));

	if (rc) {
		cf_warning(AS_BATCH, "Failed to initialize batch-index-threads to %u: %d", g_config.n_batch_index_threads, rc);
		return rc;
	}

  // 初始化批处理缓冲区池 ， BATCH_BLOCK_SIZE = 128K
	rc = as_buffer_pool_init(&batch_buffer_pool, sizeof(as_batch_buffer), BATCH_BLOCK_SIZE);

	if (rc) {
		cf_warning(AS_BATCH, "Failed to initialize batch buffer pool: %d", rc);
		return rc;
	}
	
  // 初始化批处理线程队列
	rc = as_batch_create_thread_queues(0, g_config.n_batch_index_threads);

	if (rc) {
		return rc;
	}

	return 0;
}

//-------------------------------------------------------------------------------------

int
as_thread_pool_init_fixed(as_thread_pool* pool, uint32_t thread_size, as_task_fn task_fn,
						  uint32_t task_size, uint32_t task_complete_offset)
{
  // 初始化互斥锁
	if (pthread_mutex_init(&pool->lock, NULL)) {
		return -1;
	}
	
  // 加锁
  if (pthread_mutex_lock(&pool->lock)) {
    return -2;
  }
	
	// Initialize queues.
  // 初始化调度队列，即任务队列
	pool->dispatch_queue = cf_queue_create(task_size, true);
  // 初始化完成队列
	pool->complete_queue = cf_queue_create(sizeof(uint32_t), true);
  // task function
	pool->task_fn = task_fn;
  // 完成回调
	pool->fini_fn = NULL;
  // task大小
	pool->task_size = task_size;
  // 偏移量
	pool->task_complete_offset = task_complete_offset;
  // 线程个数
	pool->thread_size = thread_size;
  // 已初始化
	pool->initialized = 1;
	
	// Start detached threads.
  // 启动线程
	pool->thread_size = as_thread_pool_create_threads(pool, thread_size);
	int rc = (pool->thread_size == thread_size)? 0 : -3;
	
  // 释放锁
	pthread_mutex_unlock(&pool->lock);
	return rc;
}

//-------------------------------------------------------------------------------------

static int
as_batch_create_thread_queues(uint32_t begin, uint32_t end)
{
	// Allocate one queue per batch response worker thread.
  // 每个批处理响应工作线程分配一个队列。
	int status = 0;

	as_batch_work work;
	work.complete = false;

	for (uint32_t i = begin; i < end; i++) {
		work.batch_queue = &batch_queues[i];
		work.batch_queue->response_queue = cf_queue_create(sizeof(as_batch_response), true);
		work.batch_queue->complete_queue = cf_queue_create(sizeof(uint32_t), true);
		work.batch_queue->tran_count = 0;
		work.batch_queue->delay_count = 0;
		work.batch_queue->active = true;
		
    // 初始化任务
		int rc = as_thread_pool_queue_task_fixed(&batch_thread_pool, &work);

		if (rc) {
			cf_warning(AS_BATCH, "Failed to create batch thread %u: %d", i, rc);
			status = rc;
		}
	}
	return status;
}
```

服务器还提供以下批处理索引统计信息变量（请参阅完整的度量标准参考手册）：

| Name                          | Description                                                  |
| ----------------------------- | ------------------------------------------------------------ |
| batch_index_initiate          | 接受的批处理索引请求个数                                     |
| batch_index_queue             | 每个批处理队列上剩余的批处理索引请求和响应缓冲区的数量。<br />格式: `<q1 requests> :<q1 buffers> ,<q2 requests> :<q2 buffers>,...` |
| batch_index_complete          | 已完成的批处理索引数量                                       |
| batch_index_timeout           | 批处理索引请求的超时时间                                     |
| batch_index_error             | 由于错误而被拒绝的批处理索引请求数                           |
| batch_index_unused_buffers    | 缓冲池中可用的 128KB 响应缓冲区的数量                        |
| batch_index_huge_buffers      | 创建的临时响应缓冲区数量超过 128KB。当检索一个大于 128KB的记录时，将创建巨大的缓冲区。大量的记录不会从批处理中获益，而且会导致服务器上过多的内存混乱。 |
| batch_index_created_buffers   | 创建的128KB响应缓冲区的数量。当池中没有缓冲区时，将创建响应缓冲区。如果此数目持续增加并且有可用的内存，那么将会增加 `batch-max-unused-buffers`。 |
| batch_index_destoryed_buffers | 销毁的128KB响应缓冲区数。当没有剩余的插槽可将响应缓冲区放回池时，响应缓冲区将被销毁。最大响应缓冲池大小是 `batch-max-unused-buffers`。 |
| batch-index                   | 批处理索引性能直方图。                                       |

对于 batch index 子处理：

| Name                                    | Description                                                  |
| --------------------------------------- | ------------------------------------------------------------ |
| <u>batch_sub_proxy_complete</u>         | <u>已完成的代理 batch-index 子任务的数量。</u>               |
| <u>batch_sub_proxy_error</u>            | <u>失败并发生错误的代理 batch-index 子任务的数量。</u>       |
| <u>batch_sub_proxy_timeout</u>          | <u>代理的 batch-index 子任务超时的数量。</u>                 |
| <u>batch_sub_read_error</u>             | <u>因错误而失败的 batch-index 读取子任务数量。</u>           |
| <u>batch_sub_read_not_found</u>         | <u>找不到结果的 batch-index 读取子任务的数量。</u>           |
| <u>batch_sub_read_success</u>           | <u>成功的 batch-index 读取子任务的数量。</u>                 |
| <u>batch_sub_read_timeout</u>           | <u>超时的 batch-index 读取子任务的数量。</u>                 |
| <u>batch_sub_tsvc_error</u>             | <u>在尝试处理任务之前，由于任务服务中的错误而失败的 batch-index 读取子任务的数量。例如协议错误或安全权限不匹配。</u> |
| <u>batch_sub_tsvc_timeout</u>           | <u>在尝试处理任务之前，在任务服务中超时的 batch-index 读取子任务的数量。例如协议错误或安全权限不匹配。</u> |
| <u>retransmit_all_batch_sub_dup_res</u> | <u>正在重复解析的批处理子事务期间发生的重传次数。注意，这包括在客户端和代理节点上发起的重传。</u> |
| <u>retransmit_batch_sub_dup_res</u>     | <u>正在重复解析的批处理子事务期间发生的重传次数。在 4.5.1.5 版本被替换为`retransmit_all_batch_sub_dup_res`</u> |
| <u>early_tsvc_batch_sub_error</u>       | <u>在任务早期批处理子任务的错误数。例如，坏的/未知的 命名空间或身份验证错误。</u> |

###### Batch Direct

4.4之前的所有服务器版本均支持旧版批处理协议。

bitch direct协议允许单个请求使用单个命名空间和bin名称过滤器从多个key检索记录。服务器使用单独的批处理队列/线程和单记录任务队列/线程来处理此协议，这使批处理请求的优先级低于单记录任务。

请注意，服务器版本4.4或更高版本不支持批 batch direct 协议。

batch direct 过程为：

1. 从客户端接收批处理请求。
2. 将整个批处理请求放入批处理线程队列中。
3. 批处理线程处理整个批处理请求，包括底层读取并将结果发送到客户端。

Aerospike使用直接的底层数据库读取方式实现 batch direct 协议。当所有的 key 都在同一个命名空间中时，batch direct会更快；但是，有一个重要的缺点：Batch direct 不能把记录中 not found 错误代理到其他服务器节点。在将节点添加到集群或从集群中删除节点之后，以及正在迁移的记录和客户端分区映射更新（每秒一次）之间存在滞后时，可能会发生这种情况。

旧版客户端继续使用 batch direct 协议。较新的客户端默认为 batch index 协议（如果可用）。要在客户端中选择 batch direct，请改变批处理策略变量。

服务器提供以下配置 batch direct 性能调整变量。

| Name               | default | max  | dynamic | description                                                  |
| ------------------ | ------- | ---- | ------- | ------------------------------------------------------------ |
| batch-max-requests | 5000    | -    | true    | 每个节点允许的最大 key 数量。防止由于大的批处理请求导致大量内存消耗而造成意外的服务器不稳定。 |
| batch-policy       | 200     | -    | true    | 生成前的顺序读取数。数字越大，优先级越高。                   |
| batch-threads      | 4       | 64   | true    | batch direct 工作线程的数量，该线程处理完整的批处理请求。所有批处理线程只有一个批处理队列。 |

客户端可以在服务器运行时更改动态变量。

服务器还提供以下 batch direct 统计变量：

| Name                    | Description                                     |
| ----------------------- | ----------------------------------------------- |
| batch_initiate          | 收到的 batch direct 请求数量。                  |
| batch_queue             | 队列中剩余的等待处理的 batch direct请求数量。   |
| <u>batch_tree_count</u> | <u>所有的 batch direct 请求的tree查找数量。</u> |
| batch_timeout           | batch direct请求超时的数量。                    |
| batch_errors            | 因错误而被拒绝的 batch direct 请求的数量。      |
| <u>batch_q_process</u>  | <u>batch direct 性能直方图。</u>                |

##### known limitations

批量写入不支持。

##### References

有关特定语言的实例，请参阅以下主题：

- [Java](https://www.aerospike.com/docs/client/java/examples/application/batch.html)
- [C# .NET](https://www.aerospike.com/docs/client/csharp/examples/application/batch.html)
- [C](https://www.aerospike.com/docs/client/c/usage/kvs/batch.html)
- [Node.js](https://www.aerospike.com/docs/client/nodejs/usage/kvs/read.html)
- [Go](https://www.aerospike.com/docs/client/go/usage/kvs/batch.html)
- [Python](https://www.aerospike.com/docs/client/python/usage/kvs/read.html#running-batch-operations)
- [PHP](https://www.aerospike.com/docs/client/php/usage/kvs/read.html#batch-operations)
- [Ruby](https://www.aerospike.com/docs/client/ruby/usage/kvs/batch.html)