## List - remove_by_rank

### [List](https://docs.aerospike.com/docs/guide/cdt-list-ops.html) remove_by_rank(bin, rank[, returnType, context])

删除具有指定 rank 顺序的元素。rank 必须是介于 0（最低等级）和 N-1（最高等级）之间的数字，其中 N 是列表的长度。等级也可以是负数，-1 是 rank 最高的元素。

根据返回类型返回 zero, one or a list of values。`NONE` 返回类型将执行得更快，因为没有为操作构造返回。

尝试删除无法访问的 rank 时，抛出 Operation Not Applicable 错误（code 26）。

无论列表是无序还是有序，`remove_by_rank` 都会根据 [ordering rules](https://docs.aerospike.com/docs/guide/cdt-ordering.html) 返回相同的元素。 “by rank”操作在有序列表上的性能优于无序列表。

---

#### Example

`[1, 3, [3, 7], 0, [3, 2]]` 用 `set_order` 修改为 `ORDERED` 会变成 `[0, 1, 3, [3, 2], [3, 7]]` ，因为 list 具有比 integer 更高的排序顺序，并且列表元素在排序时按索引进行比较。

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

`RANK` 和 `COUNT` 对于此操作是多余的。 `INVERTED` 对这个操作没有意义。

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

对于存储在内存中的有序列表，by rank 删除元素的最坏情况性能是 𝓞(1) ，对于存储在 SSD 上的有序列表为 𝓞(N)，对于无序列表为 𝓞(R log N + N) 。

有关 List API 的完整最坏情况性能分析，请参阅 [List Performance](https://docs.aerospike.com/docs/guide/cdt-list-performance.html) 。

---

#### Code Sample

以下是 Python 代码示例。

```
# [1, 4, 7, 3, 9, 26, 11]

# get the value of the element with the second highest rank
client.operate(key, [list_operations.list_remove_by_rank("l", -2, aerospike.LIST_RETURN_VALUE)])
# 11
client.operate(key, [list_operations.list_remove_by_rank("l", 2, aerospike.LIST_RETURN_VALUE)])
# 4
```

[Python](https://github.com/aerospike-examples/aerospike-operations-examples/blob/master/python/list/remove_by_rank.py) | [Java](https://github.com/aerospike/aerospike-client-java/blob/master/test/src/com/aerospike/test/sync/basic/TestOperateList.java) | C | [C#](https://github.com/aerospike/aerospike-client-csharp/blob/master/Framework/AerospikeTest/Sync/Basic/TestOperateList.cs) | Go | Node.js | PHP | Ruby | REST | Rust |

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

[Java](https://www.aerospike.com/apidocs/java/com/aerospike/client/cdt/ListOperation.html#removeByRank-java.lang.String-int-int-com.aerospike.client.cdt.CTX...-) | [C](https://www.aerospike.com/apidocs/c/df/d6c/group__list__operations.html#ga3176604f53e21a6e655d6a83b5da8a4d) | [C#](https://www.aerospike.com/apidocs/csharp/html/M_Aerospike_Client_ListOperation_RemoveByRank.htm) | [Go](https://godoc.org/github.com/aerospike/aerospike-client-go#ListRemoveByRankOp) | [Python](https://aerospike-python-client.readthedocs.io/en/latest/aerospike_helpers.operations.html#aerospike_helpers.operations.list_operations.list_remove_by_rank) | [Node.js](https://www.aerospike.com/apidocs/nodejs/module-aerospike_lists.html#.removeByRank__anchor) | PHP | [Ruby](https://www.rubydoc.info/gems/aerospike/Aerospike/CDT/ListOperation#remove_by_rank-class_method) | REST | [Rust](https://docs.rs/aerospike/latest/aerospike/operations/lists/fn.remove_by_rank.html) |