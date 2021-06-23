## List - sort

### [List](https://docs.aerospike.com/docs/guide/cdt-list-ops.html) sort(bin[, sortFlags, context])

对 list 进行排序，但不更改 list 顺序。无需列表在排序后将保持无序。有序列表不需要排序，因为根据定义，只要向其中添加元素，它就会对自身排序。

请参见 [CDT Element Ordering and Comparison](https://docs.aerospike.com/docs/guide/cdt-ordering.html) 。

---

#### Sort Flags

| Flag | Description |
| --- | --- |
| `DEFAULT` | 默认保留重复项 |
| `DROP_DUPLICATES` | 排序时删除重复项 |

---

#### Returns

Returns null

---

#### Write Flags

| Flag | Description |
| --- | --- |
| `MODIFY_DEFAULT` | 默认值: upserts; 不受 list 大小的限制; 接受重复元素; 失败时抛出错误 |
| `ADD_UNIQUE` | 仅添加 list 中尚不存在的元素 |
| `INSERT_BOUNDED` | 禁止插入超过索引 N，其中 N 是元素计数 |
| `NO_FAIL` | 如果发生违反策略的行为，例如 `ADD_UNIQUE` 或 `INSERT_BOUNDED`，则无操作而不是失败 |
| `DO_PARTIAL` | 与 `NO_FAIL` 一起使用时，添加不违反策略的元素 |

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

在已排序的 list 上调用时，排序操作将不起作用。对于无序列表进行排序的最坏情况是 𝓞(N log N)。

有关 List API 的完整最坏情况性能分析，请参阅 [List Performance](https://docs.aerospike.com/docs/guide/cdt-list-performance.html) 。

---

#### Code Sample

以下是 Python 代码示例。

```
# [5, 1, 8, 2, 7, [3, 2, 4, 1], 9, 6, 1, 2]
# sort the inner list (at index 5)
ctx = [
    cdt_ctx.cdt_ctx_list_index(5)
]
client.operate(key, [list_operations.list_sort("l",ctx=ctx)])
# [5, 1, 8, 2, 7, [1, 2, 3, 4], 9, 6, 1, 2]
```

[Python](https://github.com/aerospike-examples/aerospike-operations-examples/blob/master/python/list/sort.py) | [Java](https://github.com/aerospike/aerospike-client-java/blob/master/test/src/com/aerospike/test/sync/basic/TestOperateList.java) | C | [C#](https://github.com/aerospike/aerospike-client-csharp/blob/master/Framework/AerospikeTest/Sync/Basic/TestOperateList.cs) | Go | [Node.js](https://github.com/aerospike/aerospike-client-nodejs/blob/master/test/lists.js) | PHP | Ruby | REST | Rust |

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

[Java](https://www.aerospike.com/apidocs/java/com/aerospike/client/cdt/ListOperation.html#sort-java.lang.String-int-com.aerospike.client.cdt.CTX...-) | [C](https://www.aerospike.com/apidocs/c/df/d6c/group__list__operations.html#gaca4d8217d9df88b164705073063fa117) | [C#](https://www.aerospike.com/apidocs/csharp/html/M_Aerospike_Client_ListOperation_Sort.htm) | [Go](https://godoc.org/github.com/aerospike/aerospike-client-go#ListSortOp) | [Python](https://aerospike-python-client.readthedocs.io/en/latest/aerospike_helpers.operations.html#aerospike_helpers.operations.list_operations.list_sort) | [Node.js](https://www.aerospike.com/apidocs/nodejs/module-aerospike_lists.html#.sort__anchor) | PHP | [Ruby](https://www.rubydoc.info/gems/aerospike/Aerospike/CDT/ListOperation#sort-class_method) | REST | Rust |