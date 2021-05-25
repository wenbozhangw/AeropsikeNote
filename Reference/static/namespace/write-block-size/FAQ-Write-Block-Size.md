## FAQ - Write Block Size

### Details

 配置 `write-block-size` 定义了写入磁盘的每个 I/O 块的大小 (以字节为单位)。您可以根据记录的大小来增加或减少它。默认值为 1MB, 并且此参数的配置值必须为 2 的幂，因此，不同的选项为：128K，256K，512K，1M（以及从4.2版开始的2M，4M或8M）。为了确定最佳配置，我们建议运行基准测试工具 ( [ACT](https://github.com/aerospike/act) ) 或联系 Aerospike 支持以获取指导 (Enterprise Licenses)。

----

#### 建议的 `write-block-size` 是多少 ?

经过测试结果表明，一般而言， Flash devices 的 `write-block-size` 为 128K， Hard Disk Drives 的 `write-block-size` 为 1M（默认值）可提供最佳性能，但这会因设备品牌，大小和工作负载而异。

---

#### Read transactions 是否受到 `write-block-size` 的影响?

Read transactions 不受 `write-block-size` 的直接影响。从版本 4.2 开始，记录以 16 字节的增量 (`RBLOCK`) 进行存储，在此之前以 128 字节的形式进行存储。磁盘上的 I/O 大小取决于磁盘本身，Aerospike 将检测可能的最小读 I/O 大小。话虽如此，`write-block-size` 将对性能产生总体影响。这将取决于工作负载的性质，并且最好使用不同的值最为基准测试。

---

#### 我可以将 `write-block-size` 设置为小于 128KB 吗？

Flash 存储的典型 `write-block-size` 为 128K，但最佳值取决于所使用的 Flash 设备。具有较小的 `write-block-size` 会导致对 SSD 的更多命中，进而产生更多的 I/O 操作并增加碎片整理过程。这也将导致 SSD 上的 写入放大（WA : Write Amplification）。

WA（Write Amplification）写入放大：WA是闪存及SSD相关的一个极为重要的属性。由于闪存必须先擦除才能再写入的特性，在执行这些操作时，数据都会被移动超过1次。这些重复的操作不单会增加写入的数据量，还会减少闪存的寿命，更吃光闪存的可用带宽而间接影响随机写入性能。

---

#### 我的记录大于 1MB - 我有什么选择 ?

对于 4.2 之前的服务器版本，唯一的选择是考虑拆分记录并在客户端上处理合并。对于服务器版本 4.2 及更高版本，可以增加 `write-block-size` 。但是，这可能会对系统的整体性能产生不利影响。较大块的碎片整理涉及更长的大块读取，其中读取整个块，为其他操作带来延迟。应该使用基准工具，例如 ([ACT](https://github.com/aerospike/act)) ，以量化较大块对延迟的影响。

默认使用 64MB 的 `max-write-cache` 和 8MB 的 write block size，您可以更快地达到 `queue too deep` :

```log
WARNING (drv_ssd): (drv_ssd.c:3691) {test} write fail: queue too deep: exceeds max 8
```

---

#### 如何更改 `write-block-size` 配置?

要更新 `write-block-size` 设置，请执行以下操作:

1. 打开 /etc/aerospike/aerospike.conf.
2. 将 namespace 的 `write-block-size` 配置为新的所需大小。请注意，此配置防止在 `storage-engine device` 节中。这不适用于 `storage-engine memory` .
3. 重新启动服务器 : `/etc/init.d/aerospike restart` .
4. 继续集群中的其他节点。

---

#### 我可以在集群上滚动更改 `write-block-size` 配置吗?

`write-block-size` 配置是静态的，但可以在集群中的所有节点之间滚动地进行更改。不过，请注意以下几点：

- 当增加 `write-block-size` 时，使用新的"增加的"大小的记录不应该被写入，只有在集群中的所有节点都重新配置了增加的 `write-block-size`。
- 如果减少`write-block-size`，则必须首先删除大于 新的较小的 `write-block-size`的所有记录，并且还需要对磁盘进行清零，浙江需要等待每个节点之间的迁移完成。
- 如果运行较旧的集群协议（版本 3.13 及更早版本），则可能还需要等待每个节点之间的迁移完成（取决于写入工作负载的性质）。

---

#### 是否可以仅通过集群上滚动 service-restarts 来增加和减少该值 ?

一旦增加了配置，就无法在不将磁盘归零的情况下减少配置。

---

#### 当我写入大于配置的 `write-block-size` 的记录时，会发生什么 ?

此配置限制了可以在集群上的记录的最大大小。任何大小大于`write-block-size`的记录都会触发客户端错误和写入失败。

##### 服务器端日志/统计
 - 3.16之前的版本：

```log
Jan 22 2017 16:39:55 GMT: WARNING (rw): (thr_rw.c:write_local_ssd:4658) {namespace1} write_local: failed as_storage_record_write() <Digest>:0x88bb698a04a26517e5528hje57ed188e12ab29f4f
Jan 22 2017 16:39:59 GMT: WARNING (drv_ssd): (drv_ssd.c::1568) write: rejecting 1765a2048a69bb88 write size: 131328
```

 - 版本3.16及更高版本：

```log
Aug 09 2019 00:06:21 GMT: DETAIL (drv_ssd): (drv_ssd.c:1516) write: size 9437246 - rejecting <Digest>:0xd751c6d7eea87c82b3d6332467e8bc9a3c630e13
Aug 09 2019 00:06:21 GMT: WARNING (rw): (write.c:1265) {bar} write_master: failed as_storage_record_write() <Digest>:0xd751c6d7eea87c82b3d6332467e8bc9a3c630e13
Aug 09 2019 00:06:21 GMT: DETAIL (rw): (write.c:822) {bar} write_master: record too big <Digest>:0xd751c6d7eea87c82b3d6332467e8bc9a3c630e13
```
```log
Aug 09 2019 00:06:22 GMT: INFO (info): (ticker.c:884) {bar} special-errors: key-busy 0 record-too-big 217
```

- 仅当将适当的日志上下文 (rw and drv_ssd) 设置为 detail 时，DETAIL 行才会出现。
- 与 stat，special-error log ticker and 'rw' context line 不同，'drv_ssd' context line 出现在所有超大记录尝试上，包括副本写入，迁移和 duplicate resolution winners。
- 每次出现时，[fail_record_too_big](https://www.aerospike.com/docs/reference/metrics/#fail_record_too_big) 统计信息都会增加。 

##### 在客户端看到的错误

```log
AS_PROTO_RESULT_FAIL_RECORD_TOO_BIG - Error code 13
```

---

#### 这些服务器日志消息出现时，我可以确定要写入什么 set 吗 ?

请参阅, [How to return the set name of a record using its digest](https://discuss.aerospike.com/t/how-to-return-the-set-name-of-a-record-using-its-digest/5214) .

##### 重要注意事项

关于 `write-block-size` 参数的一些注意事项 :
1. 此配置基于每个命名空间，并且仅当 `storage-engine device` 时才可配置。
2. 此参数的值必须是2的幂，因此要减小的选项是：128K，256K，512K，1M等。
3. 集群的性能特征可能会更改，因此需要仔细监控。

---

#### References

链接到配置参考: [http://www.aerospike.com/docs/reference/configuration/#write-block-size](http://www.aerospike.com/docs/reference/configuration/#write-block-size)

为了确定适合您的设置的最佳配置，我们建议您使用我们的认证工具（ACT）测试您的SSD。[ACT](https://github.com/aerospike/act)

Server log reference for “[write_master: failed as_storage_record_write](https://www.aerospike.com/docs/reference/serverlogmessages/index.html?show-removed=1#59OmgCmTR9bG7xXuDMDCnM4-nj8_)”.

Server log reference for “[write_master: record too big](https://www.aerospike.com/docs/reference/serverlogmessages/index.html?show-removed=1#XDvpiZocq1LqQcgbfEz75kAOxMc_)” 。


