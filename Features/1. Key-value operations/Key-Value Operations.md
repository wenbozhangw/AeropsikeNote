## Key-Value Operations

Aerospike为 key-value 存储和文档存储模型(document store models)提供了全面支持。实际上，它的操作比那些教科书定义的模型提供更广泛的支持。Aerospike是面向行(row-oriented)的数据库，其中每个记录（相当于关系型数据库的一行）均由 key 唯一标识。记录的 key 及其他元数据位于主索引中。记录的数据位于其占用的命名空间的预定义存储设备中。有关 Aerospike的 Record, Namespace, Set, Key, Bins, and Data Types 的详细说明，请参阅 [Data Model](https://docs.aerospike.com/docs/architecture/data-model.html) 。

### Supported models

Aerospike 支持 key-value store 和 document store 模型 ：

##### Key-value store model

记录由一个 key 唯一标识，该 key 由 [Set](https://docs.aerospike.com/docs/architecture/data-model.html#sets) 的名称和 user *key* 组成。 user key 是您在 Aerospike 之上构建的应用程序中的记录的主要标识。尽管传统的 K-V store model 中的值是一个不透明且无模式的 Blob 数据，但 Aerospike 记录由一个或多个 [Bins](https://docs.aerospike.com/docs/architecture/data-model.html#bins-and-data-types) 类型或无类型 Blob 数据组成。

##### Document store model

key-value 存储模型的一种变体，其中单个记录是数据的单个文档，以XML，YAML，JSON，BSON编码。Aerospike 客户通常以 [Map](https://docs.aerospike.com/docs/guide/cdt-map.html) 数据类型存储数据文档。 Aerospike Map是 JSON 的超集，可以包含不同的 [data types](https://docs.aerospike.com/docs/guide/data-types.html) ，包括嵌套的 Map 和 [List](https://docs.aerospike.com/docs/guide/cdt-list.html) 结构。 Map 透明地使用 MessagePack 压缩来进行有效存储。传统的文档存储模型每个记录只存储一个数据文档，而 Aerospike 记录可以包含多个 bins，因此每个记录可以包含多个 scalar 和 collection 数据类型。


### Operation types

Aerospike提供两种类型的原子操作，与上述两种模型相比，它们支持更广泛的用例。由于 Aerospike 使用键值对存储数据，因此本文档将其描述为键值操作 (Key-Value Operations) :

##### Record operations
单独的CRUD（创建，读取，更新，删除）记录操作。

#### Transaction operations
使用 operator() 客户端方法在记录上按顺序执行的基于 bin 的操作。


## Implementation considerations

Aerospike 的以下功能为实现数据模型提供了最佳实践。应用程序开发人员应牢记以下几点：

- Aerospike是无模式的。在将数据放入数据库之前，您需要定义命名空间以将数据与硬件存储介质类型相关联。相比之下，应用程序在对 Aerospike 执行写操作时，会实时创建 set（相当于关系型数据库中的表）和 bin （相当于关系型数据库中的列）。
- Aerospike 支持 *record-atomic* transactions。可以将一个或多个记录的 bins 合并为一个 transaction。Transaction 在具有隔离性和持久性的记录锁下执行。
- Aerospike 数据类型每个都有自己的原子操作和服务器端读取操作的 API，例如 Map 数据类型的 `get_by_rank_range()` 操作。
- 从 Aerospike 4.6 版开始， Map 和 List 操作可以 [applied on deeply nested elements](https://docs.aerospike.com/docs/guide/cdt-context.html) 。
- 对于非原子操作，可以使用记录的 generation metadata (data modification count) 来确保自上次读取以来未修改正在写入的数据。这种乐观的并发性允许应用程序开发人员使用 check-and-set (read-modify-write) 模式。有关特定语言的示例，请参见 [Java Read-Modify-Write](https://docs.aerospike.com/docs/client/java/usage/kvs/write.html#read-modify-write) 。