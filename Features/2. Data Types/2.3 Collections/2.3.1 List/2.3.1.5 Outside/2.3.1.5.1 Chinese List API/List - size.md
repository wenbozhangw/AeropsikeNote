## List - size

### [List](https://docs.aerospike.com/docs/guide/cdt-list-ops.html) size(bin[, context])

获取 list 的大小，可选择在给定的子上下文中。

---

#### Returns

返回 list 中指定 level 的元素计数。

---

#### Return Types

| Return Type | Description |
| --- | --- |
| `VALUE` | element 的 value（在单结果操作中）或 elements（在多结果操作中） |
| `NONE` | 无返回。通过不构建回复来加速删除操作很有用。 |
| `COUNT` | 返回的 elements 数量。 |
| `INDEX` | 元素的索引位置，从 0（第一个）到 N-1（最后一个） |
| `REVERSE_INDEX` | 元素的反向索引位置，从 0（最后一个）到 N-1（第一个） |
| `RANK` | 元素的取值顺序，0为最小 |
| `REVERSE_RANK` | 反转元素的排序顺序，0 为最大值 |
| `INVERTED` | 反转搜索条件。可以与范围操作中的另一种返回类型结合使用。 |

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

获取列表大小的最坏情况性能是𝓞(1)。

有关 List API 的完整最坏情况性能分析，请参阅 [List Performance](https://docs.aerospike.com/docs/guide/cdt-list-performance.html) 。

---

#### Code Sample

以下是 Python 代码示例。

```
# [1, 2, [3, 4]]
k, m, b = client.operate(key, [list_operations.list_size("l")])
print("\nThe size of the list is {}".format(b["l"]))
# The size of the list is 3

# get the size of the inner list (at index 2)
ctx = [
    cdt_ctx.cdt_ctx_list_index(2)
]
k, m, b = client.operate(key, [list_operations.list_size("l",ctx=ctx)])
print("\nThe size of the sub-list is {}".format(b["l"]))
# The size of the sub-list is 2
```

[Python](https://github.com/aerospike-examples/aerospike-operations-examples/blob/master/python/list/size.py) | [Java](https://github.com/aerospike/aerospike-client-java/blob/master/test/src/com/aerospike/test/sync/basic/TestOperateList.java) | C | [C#](https://github.com/aerospike/aerospike-client-csharp/blob/master/Framework/AerospikeTest/Sync/Basic/TestOperateList.cs) | Go | [Node.js](https://github.com/aerospike/aerospike-client-nodejs/blob/master/test/lists.js) | PHP | Ruby | REST | Rust |

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

[Java](https://www.aerospike.com/apidocs/java/com/aerospike/client/cdt/ListOperation.html#size-java.lang.String-com.aerospike.client.cdt.CTX...-) | [C](https://www.aerospike.com/apidocs/c/df/d6c/group__list__operations.html#ga59aeef1402647537435019a1e3fbd532) | [C#](https://www.aerospike.com/apidocs/csharp/html/M_Aerospike_Client_ListOperation_Size.htm) | [Go](https://godoc.org/github.com/aerospike/aerospike-client-go#ListSizeOp) | [Python](https://aerospike-python-client.readthedocs.io/en/latest/aerospike_helpers.operations.html#aerospike_helpers.operations.list_operations.list_size) | [Node.js](https://www.aerospike.com/apidocs/nodejs/module-aerospike_lists.html#.size__anchor) | PHP | [Ruby](https://www.rubydoc.info/gems/aerospike/Aerospike/CDT/ListOperation#size-class_method) | REST | [Rust](https://docs.rs/aerospike/latest/aerospike/operations/lists/fn.size.html) |