## Configuring Storage Compression

### Storage Compression

Aerospike的存储压缩功能可对写入持久性存储的记录进行无损压缩。 Aerospike支持三种不同的压缩算法，可提供出色的压缩和解压缩速度：

- *LZ4* - [http://lz4.org/](http://lz4.org/). Aerospike支持广泛部署的LZ4速度优化版本。还有一个HC（“高压缩率”）变体，它变慢并且不受支持。
- *Snappy* - [https://google.github.io/snappy/](https://google.github.io/snappy/).
- *Zstandard* - [https://facebook.github.io/zstd/](https://facebook.github.io/zstd/)

压缩仅适用于写入永久性存储（例如SSD或永久性存储器（pmem））的记录。内存（RAM）中的记录数据未压缩。

压缩的折衷是增加了 transaction 延迟和更高的CPU负载，因为除了访问SSD或pmem文件之外，还必须进行压缩和解压缩。通常，写延迟比读延迟受到的影响更大，因为压缩比解压缩要占用更多的CPU资源。

算法的性能-速度以及压缩率-高度特定于要压缩的数据。选择算法时，最好的做法是使用代表真实数据的记录来评估每种算法的性能。

评估最佳压缩算法的一种经验方法是在多个服务器上配置不同的压缩算法，然后查看服务器之间的压缩百分比。

请注意，压缩不允许单个记录大于配置的写块大小。未压缩的记录大小会影响记录大小检查。这样可以确保记录始终适合写入块，即使以后在不压缩的情况下进行存储也是如此。但是，只要每个单独记录的未压缩大小不大于写入块的大小，就可以将多个压缩记录存储在一个物理写入块中，直到其大小为止。

尽管可以为不同节点上的相同命名空间进行不同（或没有）压缩设置，但这仅应视为过渡到集群的临时方法，在该集群中，该命名空间在每个节点上具有相同的压缩设置。

后来的Aerospike客户端除了支持服务器端压缩外，还支持客户端压缩。配置此功能后，服务器将解压缩bin数据（从不压缩元数据），因为它会离开磁盘，然后再重新压缩以发送给客户端。

以相同的方式，来自客户端的传入数据在重写之前由服务器解压缩。对于入站数据，这允许在写入数据之前对数据执行操作。对于入站和出站数据，它都允许客户端运行与服务器不同的压缩级别或类型。

---

### Configuration

使用两个配置指令（ `compression` 和 `compression-level`）为每个节点的每个命名空间配置存储压缩。

- `compression`选择压缩算法。有效参数为`none`，`lz4`，`snappy`和`zstd`。<br/>
  如果未指定此设置，则默认为 `none` ，即不压缩。
- `compression-level`控制压缩速度和压缩率之间的权衡。<br/>
  有效的参数值`1`到`9`被用作一个刻度。在尺度的一端，`1`选择更快但效率较低的压缩。在另一端，`9`选择更慢、更高效的压缩。<br/>
  对于压缩算法`zstd`，在4.6.x之前的Aerospike Server版本中，如果在使用`compression zstd`时从未指定此设置，则将显示默认标志`0`，并将使用`compression-level 9`。<br/>
  对于压缩算法`zstd`，在Aerospike Server版本4.6.x或更高版本中，如果在使用`compression zstd`时从未指定此设置，则会显示默认标志`9`，并将使用`compression-level 9`。<br/>
  仅 *Zstandard* 接受此设置。它对*LZ4*和*Snappy*没有任何作用。

配置伪指令属于名称空间的 `storage-engine` 部分，例如：
```
namespace test {
    ...
    storage-engine device {
        device /dev/sda1
        ...
        compression zstd
        compression-level 1
    }
    ...
}
```
或
```
namespace test {
    ...
    storage-engine pmem {
        file /dev/pmem1/test.dat
        ...
        compression zstd
        compression-level 1
    }
    ...
}
```

这两个设置也可以动态配置，例如：
```
asinfo -v 'set-config:context=namespace;id=test;compression=zstd'
asinfo -v 'set-config:context=namespace;id=test;compression-level=1'
```

每当新插入的记录或更新的记录在存储过程中被压缩时，都将查询当前配置的压缩设置。然后，压缩设置控制如何压缩此记录。

这意味着更改压缩设置不会影响存储中的现有记录。它仅影响更改压缩设置后新插入或更新的记录。如果将压缩从`lz4`切换到`snappy`，则现有记录将使用*LZ4*进行压缩，直到对其进行更新。然后，它们将根据当前设置（即使用*Snappy*）进行压缩。

这也意味着存储可能包含未压缩记录的混合以及使用三种压缩算法中的任何一种进行压缩的记录。很好，因为解压缩独立于 `compression` 设置而起作用。它在解压缩记录时始终应用正确的算法，即用于压缩记录的算法。

请注意，记录可能不可压缩，因此应用压缩将增加其大小。在这种情况下，记录将以未压缩状态存储，而不是存储较大的压缩版本。

从Aerospike 4.5.3开始，当在节点之间迁移记录时，数据将保持与磁盘上相同的格式。这非常有效，这意味着如果记录被压缩，则在迁移期间它仍保持压缩状态。这也意味着，与以前的版本不同，迁移不能用作更改压缩级别的手段。

要更改现有记录的压缩级别，应用程序必须触发对集群中每个记录的重写。这些记录重写也可以通过 touch 操作触发。这将需要以下步骤：

- 动态更改所有群集节点上的压缩设置。
- 读取集群中的每条记录并将其写回。

另一种方法是 touch 现有记录。可以使用运行 touch 操作的 background scan 或 scan 和 background UDF来完成此操作。这将用于使用新的压缩设置来重写记录。可以根据所选择的方法在不更改记录的TTL的情况下完成此操作。 可以使用 [Predicate filter](https://docs.aerospike.com/docs/guide/predicate.html) 来确保仅 touch 压缩级别更改之前的记录。 如何创建 [How to Create a Scan and Touch UDF](https://discuss.aerospike.com/t/how-to-create-a-scan-and-touch-udf/7656) 知识库文章将更详细地介绍如何整体 touch 记录。

---

### Statistics

Aerospike报告当前的压缩率作为命名空间统计信息的一部分。例如：
```
$ asinfo -l -v 'namespace/test' | grep device_compression_ratio
device_compression_ratio=1.000
```
给定的数字是平均 *compressed size* 与 *uncompressed size* 之比。因此，上述值1.000表示完全没有压缩。相反，0.100表示比例为1∶10，即尺寸减小了90％。

请注意，除非将 `compression` 设置为`none`，否则`device_compression_ratio`将包含在名称空间统计信息中。

压缩比是移动平均值。它是根据最近写入（=最近压缩）的记录计算的。读取记录不计入比率。

如果写入的数据（及其可压缩性）随时间变化，则压缩率也会随之变化。如果数据突然变化，则指示的压缩率可能会落后一点。根据经验，假设压缩率覆盖了最近写入的100,000至1,000,000条记录。

特别是，这意味着压缩率可能无法准确反映存储中所有记录的压缩率。所有记录的实际存储节省量可能高于或低于当前的压缩率，后者仅涵盖最近写入的记录。

在评估实际数据的不同压缩设置时，请按照下列步骤操作：
- 设置 `compression` and `compression-level`
- 写入1,000,000个代表性记录，使比率收敛以反映这些记录的实际压缩率。
- 通过上述info命令查询压缩率。

对要评估的所有压缩设置重复这些步骤。