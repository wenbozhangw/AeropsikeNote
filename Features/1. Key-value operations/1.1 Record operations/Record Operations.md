## Record Operations

Aerospike 提供了用于从 Aerospike 集群读取数据或向其中写入数据的简单接口。它们涵盖了标准的 CRUD (create, read, update, delete) 用例。

每个 Aerospike 客户端都可以执行以下完整记录的操作，无论使用哪种语言编写：

| Operation | Description |
|--- |--- |
| `put` | 写入一条记录。执行等同于 `create`, `update` or `replcae` 的操作，具体取决于赋予 `put` 操作的写入策略。 |
| `delete` | 从数据库中删除一条记录。使用 [durable-delete](https://docs.aerospike.com/docs/guide/durable_deletes.html) 策略时生成一个 tombstone 。 |
| `get` | 读取记录。可以选择特定的 bins。 |
| `add` | 在整数 bin中增加 (或减去)一个值。用于实现计数器。也称为 `increment` 。 |
| `append` / `prepend` | 修改字符串或字节数组 bin。 |
| `touch` | 递增 [generation counter](https://docs.aerospike.com/docs/architecture/data-model.html#metadata) 。 |
| `operate` | 将多个 bin 操作应用为原子 transaction。 |

最佳实践是将 `add` 和 `append`/`prepend` 操作作为 [transaction](https://docs.aerospike.com/docs/guide/transactions.html) 的一部分来应用，而不要使用上述的 full-record operation变体。

从 Aerospike 数据库 5.2 版开始， [filter expression](https://docs.aerospike.com/docs/guide/expressions/index.html#filter-expressions) 可以应用于上述任何单个记录操作。

---

### Code examples of record operations

以下示例使用 Python 在记录上执行创建，读取，更新和删除（CRUD）操作。每个 Aerospike 客户端的语法不同，具体取决于操作系统的需求。这些示例为常用用法建模：

#### Create a record

```
key = (namespace_name, set_name, user_key)
bins = {
        'l': [ "qwertyuiop", 1, bytearray("asd;as[d'as;d", "utf-8") ],
        'm': { "key": "asd';q;'1';" },
        'i': 1234,
        'f': 3.14159265359,
        's': '!@#@#$QSDAsd;as'
    }

client.put(key, bins)
```

Arguments :
- `key` : 要创建的记录的主键。
- `bins` : 创建时要插入到 bins 中的 bin name 和 value pairs。

#### Read a record

```
client.get(key)
```

Arguments :
- `key` : 要读取的记录的主键。

#### Update a record

```
update_bins = {
            'b'=: u"\ud83d\ude04"
            'i': aerospike.null()
           }


client.put(key, update_bins)
```

Arguments : 
- `key` : 要更新的记录的主键。
- `update_bins` : 要更新插入到bin中的 bin name 和 value pairs。

#### Delete a record

```
client.remove(key)
```

Arguments : 
- `key` : 要删除的记录的主键。

#### Usage note for all examples

所有 full-record 读取和写入操作均接受 policy 参数："最大重试次数"(Max Retries) 和 "总超时"(Total Timeout) 。

写操作采用一个可选的 time-to-live(TTL) 参数指定保护记录不受系统自动驱逐的时间。将 TTL 设置为 -1，以告诉 Aerospike 永远不要逐出该数据。

---

### Reference

有关特定于语言的示例，请参阅以下主题：

- [Java](https://docs.aerospike.com/docs/client/java/usage/kvs/write.html)
- [C# .NET](https://docs.aerospike.com/docs/client/csharp/usage/kvs/write.html)
- [Node.js](https://docs.aerospike.com/docs/client/nodejs/usage/kvs/write.html)
- [Go](https://docs.aerospike.com/docs/client/go/usage/kvs/write.html)
- [Python](https://docs.aerospike.com/docs/client/python/usage/kvs/write.html)
- [PHP](https://docs.aerospike.com/docs/client/php/usage/kvs/write.html)
- [C](https://docs.aerospike.com/docs/client/c/usage/kvs/write.html)
- [Ruby](https://docs.aerospike.com/docs/client/ruby/usage/kvs/write.html)
- [Rust](https://docs.aerospike.com/docs/client/rust/usage/kvs/write.html)