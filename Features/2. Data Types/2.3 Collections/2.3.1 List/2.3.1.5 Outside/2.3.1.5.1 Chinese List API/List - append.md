## List - append

### [List](https://docs.aerospike.com/docs/guide/cdt-list-ops.html) append(bin, value[, writeFlags, context])

如果 bin 不存在，则创建具有指定顺序的 list bin。向 list 中添加一个值。在 `UNORDERED` list 中，该值被追加到 list 的末尾。在 `ORDERED` list 中，该值按 [value order](https://docs.aerospike.com/docs/guide/cdt-ordering.html) 插入。

---

#### Returns

返回由于操作而产生的元素计数，位于 bin 名称下记录的 'bins' 部分。

---

#### Order

使用 append 新创建的 list 可以使用 list policy 的 list order 属性声明为 `UNORDERED`（默认）或 `ORDERED`。创建 list 后，只能使用 `set_order` 修改其顺序。

---

#### Write Flags

| Flag | Description |
| --- | --- |
| `MODIFY_DEFAULT` | 默认值: upserts; 不受 list 大小的限制; 接受重复元素; 失败时抛出错误 |
| `ADD_UNIQUE` | 仅添加 list 中尚不存在的元素 |
| `INSERT_BOUNDED` | 禁止插入超过索引 N，其中 N 是元素计数 |
| `NO_FAIL` | 如果发生违反策略的行为，例如 `ADD_UNIQUE` 或 `INSERT_BOUNDED`，则无操作而不是失败 |
| `DO_PARTIAL` | 与 `NO_FAIL` 一起使用时，添加不违反策略的元素 |

可以组合写标志来改变操作的行为。例如， `ADD_UNIQUE | NO_FAIL` 如果元素已存在，将优雅地失败而不抛出异常。

`INSERT_BOUNDED` 和 `DO_PARTIAL` 不适用于此操作。

---

#### Context

| Context Type | Description |
| --- | --- |
| `BY_LIST_INDEX` | 按索引查找 List 中的元素。负索引是从 list 末尾反向执行的查找。如果超出范围，将返回参数错误。 |
| `BY_LIST_RANK` | 按排名查找 List 中的元素。负排名是从最高排名反向执行的查找。 | 
| `BY_LIST_VALUE` | 按值查找 List 中的元素。 |
| `LIST_INDEX_CREATE` | 按索引查找 List 中的元素。如果元素不存在，则创建该元素。 |
| `BY_MAP_INDEX` | 按索引查找 Map 中的元素。负索引是从 map 末尾反向执行的查找。如果超出范围，将返回参数错误。 |
| `BY_MAP_KEY` | 通过 key 查找 Map 中的元素。 |
| `BY_MAP_RANK` | 按排名查找 Map 中的元素。负排名是从最高排名反向执行的查找。 |
| `BY_MAP_VALUE` | 按值查找 Map 中的元素。 |
| `MAP_KEY_CREATE` | 通过 key 查找 Map 中的元素。如果元素不存在，则创建该元素。 |

上下文参数 (context parameter) 是一个上下文类型列表(list of context type)，定义了操作应该应用到的嵌套 list or map 元素的路径。如果没有上下文，则讲定操作发生在 list or map 的顶层。

---

#### Performance

append() 在无序列表上的最差性能是 𝓞(N)。

有关 List API 的完整最坏情况性能分析，请参阅 [List Performance](https://docs.aerospike.com/docs/guide/cdt-list-performance.html) 。

---

#### Code Sample

以下是 Python 代码示例。

```
# [1, [2]]

# append to the list element at index 1
ctx = [cdt_ctx.cdt_ctx_list_index(1)]
client.operate(key, [list_operations.list_append("l", 3, ctx=ctx)])
# [1, [2, 3]]
```

[Python](https://github.com/aerospike-examples/aerospike-operations-examples/blob/master/python/list/append.py) | [Java](https://github.com/aerospike/aerospike-client-java/blob/master/examples/src/com/aerospike/examples/OperateList.java) | [C](https://github.com/aerospike/aerospike-client-c/blob/master/examples/basic_examples/list/src/main/example.c#L64-L88) | [C#](https://github.com/aerospike/aerospike-client-csharp/blob/master/Framework/AerospikeDemo/OperateList.cs) | Go | Node.js | PHP | Ruby | REST | Rust |

---

#### Availability

可用时间 : [3.7.0](https://www.aerospike.com/enterprise/download/server/notes.html#3.7.0.1)

List order 可用时间 : [3.16.0](https://www.aerospike.com/enterprise/download/server/notes.html#3.16.0.1)

`INVERTED` 标志可用时间 : [3.16.0](https://www.aerospike.com/enterprise/download/server/notes.html#3.16.0.1)

`NO_FAIL` 和 `DO_PARTIAL` 写入标志可用时间 : [4.3.0](https://www.aerospike.com/enterprise/download/server/notes.html#4.3.0.2)

相对排名(Relative rank) 操作可用时间 : [4.3.0](https://www.aerospike.com/enterprise/download/server/notes.html#4.3.0.2)

Context 可用时间: [4.6.0](https://www.aerospike.com/enterprise/download/server/notes.html#4.6.0.2)

---

#### Client API

[Java](https://www.aerospike.com/apidocs/java/com/aerospike/client/cdt/ListOperation.html#append-com.aerospike.client.cdt.ListPolicy-java.lang.String-com.aerospike.client.Value-com.aerospike.client.cdt.CTX...-) | [C](https://www.aerospike.com/apidocs/c/df/d6c/group__list__operations.html#gafb9ff0d50a167a2c745f645439bb3b95) | [C#](https://www.aerospike.com/apidocs/csharp/html/M_Aerospike_Client_ListOperation_Append.htm) | [Go](https://godoc.org/github.com/aerospike/aerospike-client-go#ListAppendOp) | [Python](https://aerospike-python-client.readthedocs.io/en/latest/aerospike_helpers.operations.html#aerospike_helpers.operations.list_operations.list_append) | [Node.js](https://www.aerospike.com/apidocs/nodejs/module-aerospike_lists.html#.append) | PHP | [Ruby](https://www.rubydoc.info/gems/aerospike/Aerospike/CDT/ListOperation#append-class_method) | REST | [Rust](https://docs.rs/aerospike/latest/aerospike/operations/lists/fn.append.html) |