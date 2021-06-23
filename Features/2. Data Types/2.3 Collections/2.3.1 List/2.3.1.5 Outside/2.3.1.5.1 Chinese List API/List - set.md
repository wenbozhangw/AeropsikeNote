## List - set

### [List](https://docs.aerospike.com/docs/guide/cdt-list-ops.html) set(bin, index, value[, writeFlags, context])

如果 bin 不存在，则创建一个无序 list bin。设置或覆盖指定 list 索引处的值。（set函数与记录的 sets 的 Aerospike 概念无关）。

默认情况下，索引可以是任何数字，并且 `NIL` 值将根据需要添加为新元素左侧的填充。使用 `INSERT_BOUNDED` 策略，开发人员可以限制这种行为，并且只允许在 0 和 list 大小之间的索引值处插入元素。

**Note :** 您不能在有序列表设置值，只能将 `append` 或 `append_items`之一设置。

---

#### New List Always Unordered

使用 `set()` 新建的 list 始终是无序的。

---

#### Returns

返回由于操作而产生的元素计数，位于 bin 名称下记录的 'bins' 部分。

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

`DO_PARTIAL` 不适用于此操作，因为我们一次只插入一个元素。

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

将 item 设置在 list 的头部（在其第 0 个索引处）的最坏性能是 𝓞(1)。在任何其他索引处，它是 𝓞(N)。

有关 List API 的完整最坏情况性能分析，请参阅 [List Performance](https://docs.aerospike.com/docs/guide/cdt-list-performance.html) 。

---

#### Code Sample

以下是 Python 代码示例。

```
# [None, 'b']
# set an element at right the boundary of the current list (index == count)
# with a INSERT_BOUNDED write flags
policy = {
    "write_flags": aerospike.LIST_WRITE_INSERT_BOUNDED
    | aerospike.LIST_WRITE_NO_FAIL
}
client.operate(key, [list_operations.list_set("l", 2, "c", policy)])
# [None, 'b', 'c']
```

[Python](https://github.com/aerospike-examples/aerospike-operations-examples/blob/master/python/list/set.py) | Java | C | C# | Go | Node.js | PHP | Ruby | REST | Rust |


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

[Java](https://www.aerospike.com/apidocs/java/com/aerospike/client/cdt/ListOperation.html#set-com.aerospike.client.cdt.ListPolicy-java.lang.String-int-com.aerospike.client.Value-com.aerospike.client.cdt.CTX...-) | [C](https://www.aerospike.com/apidocs/c/df/d6c/group__list__operations.html#ga1f282ceeded9c54dfb76247df2cfa138) | [C#](https://www.aerospike.com/apidocs/csharp/html/M_Aerospike_Client_ListOperation_Set.htm) | [Go](https://godoc.org/github.com/aerospike/aerospike-client-go#ListSetOp) | [Python](https://aerospike-python-client.readthedocs.io/en/latest/aerospike_helpers.operations.html#aerospike_helpers.operations.list_operations.list_set) | [Node.js](https://www.aerospike.com/apidocs/nodejs/module-aerospike_lists.html#.set) | PHP | [Ruby](https://www.rubydoc.info/gems/aerospike/Aerospike/CDT/ListOperation#set-class_method) | REST | [Rust](https://docs.rs/aerospike/latest/aerospike/operations/lists/fn.set.html) |