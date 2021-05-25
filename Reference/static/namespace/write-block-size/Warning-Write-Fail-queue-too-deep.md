## Why do I see a warning - “write fail - queue too deep” on server and “Error code 18 Device Overload” on the client

### Warning : write fail : queue too deep

#### Problem Description

当 Aerospike 数据库使用 `storage-engine device` 并且具有一个或多个 namespace 时，在 aerospike.log 中可能会看到以下消息。

```log
WARNING (drv_ssd): (drv_ssd.c:4153) {test} write fail: queue too deep: exceeds max 256
WARNING (drv_ssd): (drv_ssd.c:4153) (repeated:1096) {test} write fail: queue too deep: exceeds max 256
```

客户端以以下形式报告这些错误：

```log
com.aerospike.client.AerospikeException: Error Code 18: Device overload.
```

对于 AWS 中的云部署，如果您使用的是 [shadow device](), 则可能违反了影子设备而不是主要设备的队列。在这种情况下，警告将如下所示:

```log
WARNING (drv_ssd): (drv_ssd.c:4153) (repeated:1096) {test} write fail: shadow queue too deep: exceeds max 256
```

注意：在3.13之前的版本中，您可能会看到这些警告更加冗长。在较新的版本中，重复的消息被折叠为带有“repeated”标志的一行。

---

#### Explanation

此错误表明，尽管磁盘本身不一定有故障或寿命将尽，但它们无法跟上施加在磁盘上的负载。当 Aerospike 写入时，当记录已写入到 *storage write blocks* 时，它将成功返回给客户端，这些 *storage write
blocks* 被异步刷新到磁盘。如果存储设备无法满足写负载要求，则将 *storage write blocks* 缓存在*write cache*
中。当此缓存已满时，将报告上述错误。在这种情况下，在读取的直方图中可能会出现延迟，因为在大多数情况下读取将涉及磁盘访问（不包括内存中的数据和 *post write queue* 中读取的数据）。

注意：如果 `commit-to-device` 配置参数设置为 true, 则Aerospike服务器将每个写入 transaction 刷新到磁盘，然后再返回到客户端。

---

#### Cause and Solution

##### 1 - 检查写入负载/客户端流量是否增加

可以在开始发生错误的时间段内，使用服务器日志上的 asloglatency [tool](https://www.aerospike.com/docs/tools/asloglatency) 进行检查。

**Solution :** 如果增加的负载主要来自客户端流量，则可能有必要与 Aerospike
解决方案架构师进行规模确定对话，以检查系统是否与当前用例匹配。对于企业版用户，可以通过与支持部门联系来启动。或作为临时缓解措施，减少流量负载以减轻错误。

##### 2 - 找出受影响的设备和命名空间

下面的 [log](https://www.aerospike.com/docs/reference/serverlogmessages/#nf-wHvbiIo7Hlif.B7yLetsyeGA_) 行将指示当前正在使用多少 **
storage-write_blocks** 以及 (**write-q**)，(**shadow-q**)的趋势。如果问题仅在一台设备上发生，则可能会推断出现有的 hot-key 情况。

```log
{namespace} /dev/sda: used-bytes 296160983424 free-wblocks 885103 write-q 0 write (12659541,43.3) defrag-q 0 defrag-read (11936852,39.1) defrag-write (3586533,10.2) shadow-write-q 0 tomb-raider-read (13758,598.0)
```

Server Log Reference: https://www.aerospike.com/docs/reference/serverlogmessages/

**Solution :** 如果是 hot-key 情况，请确认客户端是否也收到 "Error code 14" 错误。确认吞吐量在同一命名空间的集群中不同节点之间是否有所不同。要环节 hot-key
问题，请参阅有关 [Hot Key Error Code 14](https://discuss.aerospike.com/t/hot-key-error-code-14/986) 。

##### 3 - 检查是否对迁移进行了过于积极的调整

检查以下配置参数以及它们是否已从其默认值增加：

- migrate-sleep
- migrate-threads
- migrate-max-num-incoming

请注意，传入的迁移数据也将通过写队列，并且可能将write-q设置为很高的值。 请注意，这仅限于按设备跟踪 write-queue ，我们没有直接的集群统计信息。

##### 4 - 检查是否对碎片整理进行了过于积极的调整

检查以下配置参数以及它们是否已从其默认值增加：

- defrag-sleep
- defrag-lwm-pct

这可能会导致超大的块读取和写入，这会损害磁盘性能（额外的碎片整理读取/碎片整理写入），还会由于碎片整理写入而进一步增加写入队列（日志行中的defrag-write）。

**Solution for 3 and 4:** 从长远来看，应该考虑集群上的负载，以查看磁盘是否能够跟上吞吐量。应该考虑诸如碎片整理级别以及迁移是否正在进行之类的因素，因为这些因素可能会增加磁盘的压力。考虑保留默认配置以降低磁盘上的负载。如果您的集群有能力处理可用百分比的下降，则可以临时考虑降低 `defrag-lwm-pct` 或增加 `defrag-sleep` 。有关碎片整理的文字，请参考 [Defragmentation](https://discuss.aerospike.com/t/defragmentation/718) 。

减少 `write-block-size` 可以减少碎片整理，因为在读取每个大块之后都会执行 `defrag-sleep` 的延时。因此，对于较小的`write-block-size`，通过碎片整理活动填充写入块可能会更平滑。但是，始终建议您对相关工作负载进行测试。更多详情， [write-block-size FAQ Article](https://discuss.aerospike.com/t/faq-write-block-size/681) . 

##### 5 - 确认设备的健康状况

**Solution :** 即使错误本身并不表示设备有故障，也最好确认磁盘是否还没有达到使用寿命，并且在运行磁盘是通过 S.M.A.R.T 测试。

---

#### Workaround

##### Option 1 - Tune Max-write-cache

如果发生短暂的写入突发，则增加 write cache 将为 write buffers (w-q) 提供更多空间，并有可能允许客户端进行写入。如果磁盘无法处理写负载，则这将不是永久解决方案。可以使用以下命令来增加 write cache ：
```
asinfo -v 'set-config:context=namespace;id=test;max-write-cache=128M'
```

 *write cache* 缓存的默认大小为 64MB (如果 `write-block-size` 为128K，则 64MB 缓存将包含 512 个 `storage-write-blocks`，因此，hence the q 513, max 512 portion)。 增加 cache 时，应始终将其增加到 `write-block-size` 的倍数。
 
**Note :** 在将 `max-write-cache` 增加到较高的值之前，请确保您具有足够的内存容量，以避免内存不足。上面的设置适用于为命名空间配置的每个设备。

更多详细信息以准确了解可能是引起问题的设备，以下类似的日志消息可能会有所帮助。
```log
Oct 08 2019 06:46:35 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70507569408 free-wblocks 3269912 write-q 0 write (80395102,140.9) defrag-q 0 defrag-read (79565128,178.9) defrag-write (36171273,82.2) shadow-write-q 0 tomb-raider-read (13758,598.0)
Oct 08 2019 06:46:55 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70515256192 free-wblocks 3269869 write-q 0 write (80397215,105.7) defrag-q 0 defrag-read (79567198,103.5) defrag-write (36172224,47.5) shadow-write-q 1 tomb-raider-read (13758,598.0)
Oct 08 2019 06:47:15 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70517832704 free-wblocks 3269777 write-q 35 write (80397876,33.0) defrag-q 29 defrag-read (79567797,30.0) defrag-write (36172491,13.4) shadow-write-q 0 tomb-raider-read (13758,598.0)
Oct 08 2019 06:47:35 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70516947328 free-wblocks 3269817 write-q 56 write (80398042,8.3) defrag-q 63 defrag-read (79568037,12.0) defrag-write (36172589,4.9) shadow-write-q 0 tomb-raider-read (13758,598.0)
Oct 08 2019 06:47:55 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70515790720 free-wblocks 3269841 write-q 106 write (80398246,10.2) defrag-q 177 defrag-read (79568379,17.1) defrag-write (36172696,5.3) shadow-write-q 0 tomb-raider-read (13758,598.0)
Oct 08 2019 06:48:15 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70516167424 free-wblocks 3269857 write-q 161 write (80398456,10.5) defrag-q 240 defrag-read (79568668,14.4) defrag-write (36172801,5.2) shadow-write-q 0 tomb-raider-read (13758,598.0)
...
Oct 08 2019 06:56:35 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70499140480 free-wblocks 3270541 write-q 1086 write (80403407,9.7) defrag-q 1125 defrag-read (79575188,12.1) defrag-write (36175452,5.2) shadow-write-q 0 tomb-raider-read (13758,598.0)
Oct 08 2019 06:56:55 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70499059968 free-wblocks 3270559 write-q 1134 write (80403602,9.8) defrag-q 1186 defrag-read (79575462,13.7) defrag-write (36175551,4.9) shadow-write-q 0 tomb-raider-read (13758,598.0)
Oct 08 2019 06:57:15 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70497662208 free-wblocks 3270572 write-q 1184 write (80403802,10.0) defrag-q 1263 defrag-read (79575752,14.5) defrag-write (36175652,5.1) shadow-write-q 0 tomb-raider-read (13758,598.0)
Oct 08 2019 06:57:35 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70496528768 free-wblocks 3270604 write-q 1212 write (80404000,9.9) defrag-q 1325 defrag-read (79576044,14.6) defrag-write (36175761,5.4) shadow-write-q 0 tomb-raider-read (13758,598.0)
Oct 08 2019 06:57:55 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70494422912 free-wblocks 3270633 write-q 1255 write (80404192,9.6) defrag-q 1386 defrag-read (79576326,14.1) defrag-write (36175865,5.2) shadow-write-q 0 tomb-raider-read (13758,598.0)
Oct 08 2019 06:58:15 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70493322368 free-wblocks 3270675 write-q 1277 write (80404359,8.4) defrag-q 1370 defrag-read (79576519,9.6) defrag-write (36175964,4.9) shadow-write-q 0 tomb-raider-read (13758,598.0)
Oct 08 2019 06:58:35 GMT: INFO (drv_ssd): (drv_ssd.c:2115) {ns1} /dev/sda: used-bytes 70491328128 free-wblocks 3270709 write-q 1310 write (80404539,9.0) defrag-q 1437 defrag-read (79576800,14.1) defrag-write (36176066,5.1) shadow-write-q 0 tomb-raider-read (13758,598.0)
```

在这里，我们可以看到，最初，我们每秒写入 100 个以上的块 (write (, xxx) 之后的第二个数字)，每个块都是配置的 `write-block-size` 的内容，我们可以参考 [log reference](https://www.aerospike.com/docs/reference/serverlogmessages/#nf-wHvbiIo7Hlif.B7yLetsyeGA_) 详细信息。

在这段时间内，write-q 为 0，这就是我们期望的... write queue 应始终为0，或者非常接近。只要它高于此值，就意味着我们无法像填充块一样快速地冲洗块。

现在，随着write-q开始增加，每秒写入数减少到10个块左右（减少10倍）。因此，即使写负载减少了一个数量级，disk-io子系统仍然无法处理它。

这可能会导致CPU负载过高，从而可能导致整个系统变慢，例如，这还会使nsup周期非常长。

##### Option 2 - 考虑启用 read page cache 以使读取是从缓存而不是磁盘进行的

此选项仅适用于 read heavy use-cases 的情况，在这些情况下可以调整由于 write device overload 而导致的读取性能影响。

检查日志行，该行显示命中缓存与磁盘的读取。
```log
{ns_name} device-usage: used-bytes 2054187648 avail-pct 92 cache-read-pct 12.35
```

有关日志行的更多详细信息，请参见 [Log Reference](https://www.aerospike.com/docs/reference/serverlogmessages/#DwcGnseOo62HVCS6TZxUxmf1ehs_) 。

可以增加值 `cache-read-pct` 作为临时的解决方法，以减轻设备的负担。为了对此进行调整，可以增加 `post-write-queue`。请注意，这可能会影响内存使用率，因此会逐步增加并进行监视。

为了使副本读取进入 `post-write-queue` ，可以启用[cache-replica-writes]。

Aerospike提供了另一种配置，允许操作系统利用 page cache 并可以帮助某些工作负载类型延迟。有关此问题的更多详细信息，请参见我们有关 [ Buffering and Caching in Aerospike](https://discuss.aerospike.com/t/buffering-and-caching-in-aerospike/5623) 的知识库文章。

---

#### Note

 - 如本文所述，即使已达到 `max-write-cache` 限制，也将继续处理迁移写入。Prole writes 和 defragmentation writes 也将继续。这可能会导致很高的内存使用率及其潜在后果。
 - 请注意，在配置参考手册的详细信息中，在5.1版及更高版本中解释了 `max-write-cache` 参数的更改。
 - SAR工具可用于检查各种设备上的负载。这将在一个月的滚动期内（取决于当月的一天）收集并保留与网络和设备使用情况有关的数据。收集数据的默认SAR间隔为10分钟。将其减少到5或2分钟可能是有用的，具体取决于队列持续时间过长而导致的错误。有关此问题的更多详细信息，请参见我们有关[Configuring SAR](https://discuss.aerospike.com/t/how-to-configure-sysstat/3929) 的知识库文章。
 - iostat也可以用于检查磁盘性能，但是在发生问题时必须收集此信息。有关此问题的更多详细信息，请参见我们的 [Interpreting iostat](https://discuss.aerospike.com/t/how-to-interpret-iostat-output-related-to-aerospike/7113) 的知识库文章。
 - 在这种情况下，除非有大于或等于已配置的 `replication-factor` 的节点与这些正在进行的警告并行发生故障，否则不应有任何数据丢失。write-q 将继续增长(超过`max-write-cache`限制)，但最终将刷新到存储子系统。在这种情况下，可以对客户端应用程序进行编码以重试和/或限制。
 - 如果仅在影子设备上看到问题，则问题可能特定于大小调整（如果影子设备的大小与主设备的大小不同）。