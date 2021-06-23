## Transaction Operations

Aerospike 记录包含一个或多个可能具有不同 [data types](https://docs.aerospike.com/docs/guide/data-types.html) 的 Bins。嵌套 [Lists](https://docs.aerospike.com/docs/guide/cdt-list.html) and [Maps](https://docs.aerospike.com/docs/guide/cdt-map.html) ， 也称为集合数据类型(collection data type, CDT)，可以组合起来实现复杂的数据结构作为文档(document)。为了处理这些可能复杂的数据文档，每个 Aerospike 客户端都提供以下方法：

`operate` : 将单个记录上的多个操作合并到一个 transaction 中。操作包括 bin 的读写操作(bin read and write operations)，bin中的数据类型的任何操作(any operation on a data type)。示例包括在整数 bin 上 add ，在 HyperLogLog  bin 上进行 HLL 操作，在 Blob(Bytes) bins 上进行按位操作，或在 list and map bin 上进行 large API 操作。

`operate`方法的参数之一是要执行的 bin 操作的有序列表。Aerospike客户端使用语法因编程语言而异。与返回 Aerospike 记录的 `get` 操作不同，operation 返回每个 bin 的每个 sub-operation 的输出。这对于确认 transaction processing 以及排除导致意外 transaction 成功或失败的逻辑非常有用。

每个 `operate` 调用都会产生一个符合 ACID的 transaction，该 transaction 由记录上的一个或多个有序 bin 操作组成。该操作是由Aerospike在具有隔离性和持久性的记录锁定下处理的单个原子 transaction 。相比之下，每个 record operation 都会导致单个操作组成的原子 transaction。

以下是 bin 操作的完整列表，这些 operations 可以是使用 operate 的 transaction 的一部分。

| Date Type | Operation | Description |
| --- | --- | --- |
| All | `op_put` | 更新插入(create or update)一个bin。 也叫 `write` 。 |
| All | `op_get` | 读取一个bin。也叫 `read`。 |
| All | `op_touch` | 增加一个记录的 generation counter。 |
| All | `op_delete` | 从数据库中删除一条记录。使用 [durable delete](https://docs.aerospike.com/docs/guide/durable_deletes.html) 策略时生成一个 tombstone 。 |
| Integer Float | `op_add` | 加(或减)一个值。用于实现计数器。也称为 `increment` 。| 
| String | `op_append` `op_prepend` | 修改一个字符串。 |
| List |  | 有关 list 的全部 List Operations 列表，请参见 [Collection Data Types: Lists](https://docs.aerospike.com/docs/guide/cdt-list.html) 。 |
| Map |  | 有关全部 Map Operations 列表，请参见 [Collection Data Types: Map](https://docs.aerospike.com/docs/guide/cdt-map.html) 。 |
| Blob/Bytes |  | 有关全部 Blob/Byte Operations 列表，请参见 [Data Types: Blob/Bytes](https://docs.aerospike.com/docs/guide/blob/index.html) 。 |
| HLL | | 有关全部 HLL Operations 列表，请参见 [Data Types: HyperLogLog](https://docs.aerospike.com/docs/guide/hyperloglog/index.html) 。 |
| Geo |  | 有关全部 Geospatial Operations 列表，请参见 [Data Types: Geospatial](https://docs.aerospike.com/docs/guide/geospatial.html) 。 |

从 Aerospike Enterprise Edition 5.6 版开始，[Aerospike operation expressions](https://docs.aerospike.com/docs/guide/expressions/index.html) 可以应用于上述任何 transaction 操作。

---

### Code example of a transaction operation

以下 Python 示例对记录执行读取和更新 bin 操作。每个 Aerospike 客户端的语法不同，具体取决于操作系统的需求。

```
# The following is the primary key of a record
#
# key = (namespace_name, set_name, user_key)
#
# The following is the Aerospike record associated with that key. 
# 
# bins = {
#         'l': [ "qwertyuiop", 1, bytearray("asd;as[d'as;d", "utf-8") ],
#         'm': { "key": "asd';q;'1';" },
#         'i': 1234,
#         'f': 3.14159265359,
#         's': '!@#@#$QSDAsd;as'
#     }


ops = [ 
   operations.increment('i', 1)
   operations.read('i')
   operations.append('s', 'foobarbaz')
   operations.read('s')
   ]

client.operate(key, ops) 

# Expected result:
#   i is now 1235
#   s is now !@#@#$QSDAsd;asfoobarbaz
```

---

### Usage notes 

记录上的所有读取和写入操作均接受 [policy](https://docs.aerospike.com/docs/guide/policies.html) 参数：Max Retries and Total Timeout。

写操作采用一个可选的 time-to-live(TTL) 参数指定保护记录不受系统自动驱逐的时间。将 TTL 设置为 -1，以告诉 Aerospike 永远不要逐出该数据。

transaction 的组成 operations 可能会接受或需要其他策略参数。有关这些参数的详细信息，请参阅所有客户端和操作的 API reference ...

---

### Reference

有关特定于语言的示例，请参阅以下主题：

- [Java](https://docs.aerospike.com/docs/client/java/usage/kvs/multiops.html)
- [C# .NET](https://docs.aerospike.com/docs/client/csharp/usage/kvs/multiops.html)
- [Node.js](https://docs.aerospike.com/docs/client/nodejs/usage/kvs/multiops.html)
- [Go](https://docs.aerospike.com/docs/client/go/usage/kvs/multiops.html)
- [Python](https://docs.aerospike.com/docs/client/python/usage/kvs/multiops.html)
- [C](https://docs.aerospike.com/docs/client/c/usage/kvs/multiops.html)
- [Ruby](https://docs.aerospike.com/docs/client/ruby/usage/kvs/multiops.html)
- [Rust](https://docs.aerospike.com/docs/client/rust/usage/kvs/multiops.html)