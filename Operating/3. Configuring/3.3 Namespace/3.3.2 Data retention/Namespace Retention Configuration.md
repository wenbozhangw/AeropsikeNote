## Namespace Retention Configuration

为了确保有足够的内存用于您的操作并允许集群处理新的写入，可以将 Aerospike 命名空间配置为根据您定义的条件使记录 *expire* or *evict* 。

**Content**

- [Definition of eviction and expiration](#definition-of-eviction-and-expiration)
- [Configuration parameters](#configuration-parameters)
    - [Choosing appropriate parameter values](#choosing-appropriate-parameter-values)
        - [High-water marks](#high-water-marks)
        - [Be careful changing default Time-To-Live](#be-careful-changing-default-time-to-live)
        - [Be careful with records that have a TTL when nsup is not running](#be-careful-with-records-that-have-a-ttl-when-nsup-is-not-running)
- [Examples of configuration stanzas](#example-of-configuration-stanzas)
    - [Turning Parameters](#turning-parameters)
    - [Set Disabled Eviction](#set-disable-eviction)
    - [Set Stop Writes Count](#set-stop-writes-count)
- [Historical behavior by Aerospike Version](#historical-behavior-by-aerospike-version)
    - [Aerospike Version 4.9](#aerospike-version-4.9)
    - [Aerospike Version 4.5.1](#aerospike-version-4.5.1)
    - [Aerospike Version 3.8.1](#aerospike-version-3.8.1)
- [Related Knowledge Base Articles](#related-knowledge-base-articles)
- [Where to Next ?](#where-to-next)

---

### <span id="definition-of-eviction-and-expiration">Definition of eviction and expiration </span>

为了确定记录的删除资格，过期算法使用记录的 TTL (Time To Live)。
与 expiration 是独立地，如果正在使用的存储超过可配置的 high-water mark，则要回收内存，收回过程将删除即将到期的尚未到期的记录（哪些最接近到期时间的记录）。Eviction 将持续进行，直到收回足够的空间为止。驱逐可以被认为是 `early expiration`。
默认情况下， expiration 和 eviction 是禁用的。您必须使用某些配置参数对其进行配置。

---

### <span id="configuration-parameters">Configuration parameters</span>

这些参数可按照 namespace 配置。有关参数的确切定义和使用，请参阅 [configuration reference](https://www.aerospike.com/docs/reference/configuration/) 。
其他相关参数在 [配置节的示例](#example-of-configuration-stanzas) 中显示。

| Configuration parameter | Description | Default value | Notes |
|--- |--- |--- |--- |
| nsup-period | expiration and eviction 过程的频率(以秒为单位)。 | 0 | 默认情况下， expiration 和 eviction 不会发生。 <br/>  当 `nsup-period` 为0时，要允许具有非零 TTL 的写入，必须将 `allow-ttl-without-nsup` 设置为 true。请参阅下面的`allow-ttl-without-nsup` 。 |
| high-water-disk-pct | eviction 过程开始的磁盘阈值。 | 0 | |
| high-water-memory-pct | eviction 过程开始的内存阈值。 | 0 | |
| mounts-high-water-pct | 索引配置为 flash 或 pmem 的 eviction 过程开始的阈值。 | 0 | |
| default-ttl | 记录的默认生存时间(以秒为单位)。 | 0 | 如果 `default-ttl` 的值不为零，并且 `nsup-period` 的值为零（默认值），则 Aerospike 服务器将不会启动。 <br/> Aerospike 永远不会过期或逐出 TTL 为 0 存储的记录。  <br/> 客户端应用程序创建记录时，它们可以覆盖此值。 |
| allow-ttl-without-nsup | 即使 `nsup-period` 为 0，也允许使用非零 TTL 写入记录。除特殊用例（应由应用程序处理到期时间）外，不建议更改。请参见，[nsup未运行时, 请谨慎处理具有 TTL 的记录](#be-careful-with-records-that-have-a-ttl-when-nsup-is-not-running) 。 | false | 当 `nsup-period` 为0时，如果要写入具有非零TTL的记录，则必须将 `allow-ttl-without-nsup` 设置为true。 |
| stop-writes-pct | 使用的内存限制，不允许进一步写入。 | 90 | 仍然允许删除，复制写和迁移写。 |
| min-avail-pct | 允许进一步写操作的可用连续存储空间限制。 | 5 | 仍然允许删除，复制写和迁移写。 |

<br/>

#### <span id="choosing-appropriate-parameter-values">Choosing appropriate parameter values</span>

请注意您为这些参数设置的值的后果。

##### <span id="high-water-marks">High-water marks</span>

例如，将 `high-water-disk-pct` 设置为20意味着您要确保存储设备的满容量不会超过20％，这可能是不希望的。

##### <span id="be-careful-changing-default-time-to-live">Be careful changing default Time-To-Live</span>

根据您的用例，可能不设置 `default-ttl`（将其保留为默认值0）并管理来自客户端应用程序的数据（可以为每个记录设置TTL）可能是可取的。

##### <span id="be-careful-with-records-that-have-a-ttl-when-nsup-is-not-running">Be careful with records that have a TTL when nsup is not running</span>

当nsup未运行时（即`nsup-period`为0，这是默认值），请注意对TTL记录的处理。读取记录的请求使用TTL，在记录已超过其到期时间时返回`AEROSPIKE_ERR_RECORD_NOT_FOUND`并删除该记录。但是，如果nsup没有运行，则过期的记录永远不会从索引中删除，也不会从存储中物理删除（除非通过 transaction 显式访问它们）。请注意，当`nsup-period`设置为0时，默认情况下将不允许使用非0 TTL的记录写入，并会由于`AEROSPIKE_ERR_FAIL_FORBIDDEN`错误而被拒绝。
使用客户端API，将记录的TTL减小到小于其当前剩余时间可能会在冷重启时产生不良的副作用。更多详情请参见 [冷启动恢复已删除记录的问题](https://discuss.aerospike.com/t/issues-with-cold-start-resurrecting-deleted-records/3930) 。

---

### <span id="example-of-configuration-stanzas">Examples of configuration stanzas</span>

以下示例显示了可用的命名空间数据保留参数，以及描述如何使用它们的简短注释。

```
namespace namespaceName {
    default-ttl <VALUE>             # 被写入（如果未被客户端覆盖）之后要保留数据多长时间（以秒为单位） 

    high-water-disk-pct <PERCENT>   # 在服务器开始逐出之前，磁盘可能已满了多少

    high-water-memory-pct <PERCENT> # 在服务器开始逐出之前，内存可能已满了多少

    stop-writes-pct <PERCENT>       # 在我们禁止新写操作之前，内存有多大容量

    index-type flash/pmem {             
        mounts-high-water-pct <PERCENT>  # 如果适用，服务器开始逐出之前，索引存储的 flash 使用了多少
        ...
    }

    ...
}
```

#### <span id="turning-parameters"> Turning parameters </span>

以下示例显示了用于调整数据过期和逐出的其他命名空间数据保留参数，并带有说明用法的注释。

```
namespace <namespace-name> {
    nsup-period <SECONDS>           # 从Aerospike 4.5.1开始（在 service level 之前），在两次连续的 expiration or eviction 之间开始的最长时间-零值将禁用expiration and eviction

    nsup-threads <NUMBER>           # 自Aerospike 4.5.1起，每轮 expiration or eviction 有多少个线程

    evict-tenths-pct <NUMBER>       # 可驱逐记录的分数，每轮驱逐都将其删除。例如，5表示删除0.5％的可收回记录

    ...
}
```

#### <span id="set-disable-eviction"> Set Disable Eviction </span>

从 Aerospike 3.6.1 开始，可以使用参数 `set-disable-eviction  true` 来保护集合免受逐出，如以下实例所示。可以在配置文件中[动态](https://docs.aerospike.com/docs/reference/configuration/index.html#set-disable-eviction) 设置此参数。

```
namespace <namespace-name> {
    ...
    set <set-name> {
        set-disable-eviction true     # 保护此 set 免受逐出
    }
}
```

#### <span id="set-stop-writes-count"> Set Stop Writes Count </span>

从Aerospike 3.7.0.1开始，一个 set 可以具有 `set-stop-writes-count` 来限制可以写入该记录的数量。可以在配置文件中[动态](https://docs.aerospike.com/docs/reference/configuration/index.html#set-disable-eviction) 设置此参数。

```
namespace <namespace-name> {
    ...
    set <set-name> {
        set-stop-writes-count 5000     # 将可以写入该 set 的记录数限制为5000。

    }
}
```

---

### <span id="historical-behavior-by-aerospike-version"> Historical behavior by Aerospike Version </span>

#### <span id="aerospike-version-4.9"> Aerospike Version 4.9 </span>

数据保留的设置和行为如上所述。默认情况下，到期和驱逐是禁用的。

#### <span id="aerospike-version-4.5.1"> Aerospike Version 4.5.1 </span>

从Aerospike Server 4.5.1开始，到期和逐出是通过在每个节点上本地删除记录来完成的，而不会产生任何 transaction 负载。这使得用例具有大量的expiration and eviction 负载，这是以前不可能的。

这种新方法对集群中各节点之间的时钟同步有更强的依赖性。从Aerospike Server 4.5.1开始，对于每个启用nsup（即 `nsup-period` 不为零）的命名空间，如果集群时钟偏斜超过40秒，写入将被挂起。

同样，从Aerospike Server 4.5.1开始，如果在集群中的所有节点上的逐出阈值配置不相同，那么具有最低阈值的节点将驱动整个集群的逐出。当一个节点超过了它的逐出阈值时，它会向集群的其余部分广播一个截止无效时间。在4.5.1之前，具有较低阈值的节点会从其主分区中大量退出。

#### <span id="aerospike-version-3.8.1"> Aerospike Version 3.8.1 </span>

Evictions 生效时，Aerospike会根据桶的 到期时间和逐出 将其分组，在每个桶中随机记录，先清空一个桶，然后再移至下一个桶。在Aerospike Server 3.8.1及更高版本中，引入了更细化的逐出直方图以及配置参数： `evict-hist-buckets` 。在Aerospike Server 3.7.5.1或更早版本中，始终有100个存储桶。存储桶边界是根据命名空间中具有最长到期时间的记录确定的。该记录的到期时间越长，逐出的精确度就越低，因为桶的范围会更广。

在Aerospike Server 4.5.1之前，通过内部生成删除 transaction （包括通过 fabric 删除副本）来完成 expiration and eviction 。对于大量的 expiration and eviction 负载，这些 transaction 会显着影响整体性能。

---

### <span id="related-knowledge-base-articles"> Related Knowledge Base Articles </span>

See [FAQ What are Expiration, Eviction and Stop-Writes ?](https://discuss.aerospike.com/t/faq-what-are-expiration-eviction-and-stop-writes/2311/1) 

---

### <span id="where-to-next"> Where to Next ? </span>

- 配置 [Storage Engine](https://docs.aerospike.com/docs/operations/configure/namespace/storage) ，该存储引擎确定是否以及在何处保留记录。
- 配置 [Data Durability Policy](https://docs.aerospike.com/docs/operations/configure/namespace/durability) ，该策略确定在集群中保留多少条记录的复制副本 (replica copies) 。
- 或者返回 [Configure Page](https://docs.aerospike.com/docs/operations/configure) 。