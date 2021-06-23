## List - get_by_value_rel_rank_range

### [List](https://docs.aerospike.com/docs/guide/cdt-list-ops.html) get_by_value_rel_rank_range(bin, value, rank, returnType[, count, context])


获取从给定 value 的相对 rank (**relative rank of the given value**)的 _rank offset_ 开始的范围内的元素计数。`INVERTED` 标志可用于获取不在此范围内的元素。

该值可以是任何值，不一定是 list 中存在的值。rank 和 count 可以是任何数字，但如果指定的范围不包含任何元素，则结果可能是一个空 list。

根据返回类型返回 zero, one or a list of values。返回的元素可能不按 rank 顺序排列。

与当前列表相比，value 的相对 rank 遵循 [ordering rules](https://docs.aerospike.com/docs/guide/cdt-ordering.html) 。

---

#### Formula

```
origin = relative-rank(_value_) # compared to the current list elements
r = rank(element)

Get all elements in the range where
(r >= (origin + _rank_)) and (r < (origin + _rank_ + _count_))
```

---

#### Example

对于 `[0, 3, 9, 12, 15]` ，相对与当前列表，值 5 将排在 3 和 9 之间。如果rank为-1，则从那里选择3作为范围的起点，然后计数将选择元素。所以计数3将返回范围3,9和12。

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

对于存储在内存中的有序列表，按 relative rank range 获取元素的最坏情况性能是 𝓞(M + log N)。存储在 SSD 上的有序列表为 𝓞(N + M + log N)，其中 M 是范围的元素数，N 是列表的元素数。无序列表为 𝓞(R log N + N)，其中 R 表示 rank 。

有关 List API 的完整最坏情况性能分析，请参阅 [List Performance](https://docs.aerospike.com/docs/guide/cdt-list-performance.html) 。

---

#### Code Sample

以下是 Python 代码示例。

```
world_records = [
    [10.03, "Jim Hines", "Sacramento, USA", "June 20, 1968"],
    [10.02, "Charles Greene", "Mexico City, Mexico", "October 13, 1968"],
    [9.95, "Jim Hines", "Mexico City, Mexico", "October 14, 1968"],
    [9.93, "Calvin Smith", "Colorado Springs, USA", "July 3, 1983"],
    [9.93, "Carl Lewis", "Rome, Italy", "August 30, 1987"],
    [9.92, "Carl Lewis", "Seoul, South Korea", "September 24, 1988"],
]
nil = aerospike.null()  # NIL
# Python client equivalent of get_by_value_rel_rank_range("l", [10.0, NIL], -1, VALUE, 2)
key, metadata, bins = client.operate(
    key,
    [
        operations.write("l", world_records),
        list_operations.list_get_by_value_rank_range_relative(
            "l", [10.0, nil], -1, aerospike.LIST_RETURN_VALUE, 2
        ),
    ],
)
print(bins["l"])
# The two closest world records to 10.0s are
# [
#   [9.95, 'Jim Hines', 'Mexico City, Mexico', 'October 14, 1968'],
#   [10.02, 'Charles Greene', 'Mexico City, Mexico', 'October 13, 1968']]
```

[Python](https://github.com/aerospike-examples/aerospike-operations-examples/blob/master/python/list/get_by_value_rel_rank_range.py) | [Java](https://github.com/aerospike/aerospike-client-java/blob/master/test/src/com/aerospike/test/sync/basic/TestOperateList.java) | C | [C#](https://github.com/aerospike/aerospike-client-csharp/blob/master/Framework/AerospikeTest/Sync/Basic/TestOperateList.cs) | Go | Node.js | PHP | Ruby | REST | Rust |

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

[Java](https://www.aerospike.com/apidocs/java/com/aerospike/client/cdt/ListOperation.html#getByValueRelativeRankRange-java.lang.String-com.aerospike.client.Value-int-int-int-com.aerospike.client.cdt.CTX...-) | [C](https://www.aerospike.com/apidocs/c/df/d6c/group__list__operations.html#gae86fe8c104a47d643c25b9c48f250c61) | [C#](https://www.aerospike.com/apidocs/csharp/html/M_Aerospike_Client_ListOperation_GetByValueRelativeRankRange_1.htm) | [Go](https://godoc.org/github.com/aerospike/aerospike-client-go#ListGetByValueRelativeRankRangeCountOp) | [Python](https://aerospike-python-client.readthedocs.io/en/latest/aerospike_helpers.operations.html#aerospike_helpers.operations.list_operations.list_get_by_value_rank_range_relative) | [Node.js](https://www.aerospike.com/apidocs/nodejs/module-aerospike_lists.html#.getByValueRelRankRange__anchor) | PHP | [Ruby](https://www.rubydoc.info/gems/aerospike/Aerospike/CDT/ListOperation#get_by_value_rel_rank_range-class_method) | REST | [Rust](https://docs.rs/aerospike/0.6.0/aerospike/operations/lists/fn.get_by_value_relative_rank_range_count.html) |