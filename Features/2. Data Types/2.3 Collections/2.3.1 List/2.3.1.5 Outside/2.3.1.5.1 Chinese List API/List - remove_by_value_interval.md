## List - remove_by_value_interval

### [List](https://docs.aerospike.com/docs/guide/cdt-list-ops.html) remove_by_value_interval(bin, valueStart, valueStop[, returnType, context])

删除排序的从 _valueStart_ (inclusive) 和 _valueStop_ (exclusive) 之间的间隔的 list 元素。 `INVERTED` 标志可用于删除此间隔之外的元素。

根据返回类型返回 zero, one or a list of values。返回的元素可能不按索引顺序排列。

无论 list 是无序还是有序，`remove_by_value_interval` 都会根据 [ordering rules](https://docs.aerospike.com/docs/guide/cdt-ordering.html) 返回相同的元素。"by value"操作在有序列表上的性能优于无序列表。

---

#### Trivial Example

移除两个标量值之间的区间内的列表元素。

```
# [1, 2, 3, 4, 5, 4, 3, 2, 1]

# remove all the elements in the interval [2, 4), then read the bin
client.operate(
    key,
    [
        list_operations.list_remove_by_value("l", aerospike.LIST_RETURN_NONE, 2, 4),
        operations.read("l"),
    ],
)
# [1, 4, 5, 4, 1]
```

---

#### Interval Comparison of Tuples

常将 Aerospike 中的复杂数据建模为元组列表，每个元素都是一个列表，其中索引顺序具有含义。有关示例，请参阅 [Aerospike Modeling: IoT Sensors](https://dev.to/aerospike/aerospike-modeling-iot-sensors-453a) 。

这种建模依赖于适用于列表元素的排序规则。有序对(ordered pairs)排序为：
```
[1, NIL] < [1, 1] < [1, 1, 2] < [1, 2] < [1, '1'] < [1, 1.0] < [1, INF]
```

取有序对的列表，`[[100, 1], [101, 2], [102, 3], [103, 4], [104, 5]]` 和以下间隔：

**valueStart:** `[101, NIL]` , **valueStop:** `[103, NIL]`

遍历元素，我们检查是否 `[101, NIL] <= value < [103, NIL]`。对于元素 `[101, 2] 和 [102, 3]` 也是如此。元素`[103, 4]`不在这个区间，因为它的排序高于`[103, NIL]`。比较从第 0 个索引值开始，在这种情况下都是 103。接下来比较第 1 个索引位置值，NIL 小于 3（实际上它的顺序低于任何值）。

**valueStart:** `[101, NIL]`, **valueStop:** `[103, NIL]`

元素`[101, 2]` 和 `[102, 3]` 显然在区间内。元素 `[103, 4]` 也在这个区间，因为它的排序低于 `[103, INF] 。比较从第 0 个索引值开始，在这种情况下都是 103。接下来比较第 1 个索引位置值，并且 INF 高于 3（实际上它的顺序高于任何值）。

**valueStart:** `[101, NIL]`, **valueStop:** `[103, NIL]`

元素 `[102, 2]` 和 `[103, 4]` 在区间内。元素 `[101, 2]` 不在这个区间，因为它排序低于 `[101, INF]` 。

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

对于存储在内存中的有序列表，删除值区间中元素的最坏情况性能是 𝓞(log N + M)，存储在SSD的有序列表为 𝓞(log N + N)，无序列表为 𝓞(N + M)，其中 M 是参数列表中的项目数，N 是列表中已有的项目数。

有关 List API 的完整最坏情况性能分析，请参阅 [List Performance](https://docs.aerospike.com/docs/guide/cdt-list-performance.html) 。

---

#### Code Sample

请参阅上面的示例，或按照这些特定语言的链接中的任何一个获取操作的详细代码示例。

```
# [[100, 1], [101, 2], [102, 3], [103, 4], [104, 5]]
# remove_by_value_interval(VALUE, [103, NIL], [103, INF])

key, metadata, bins = client.operate(
    key,
    [
        list_operations.list_remove_by_value(
            "l", aerospike.LIST_RETURN_VALUE,
            [101, aerospike.null()], [103, aerospike.CDTInfinite()]
        ),
    ],
)
print(bins["l"])
# [[101, 2], [102, 3], [103, 4]]
```

[Python](https://github.com/aerospike-examples/aerospike-operations-examples/blob/master/python/list/remove_by_value_interval.py) | [Java](https://github.com/aerospike/aerospike-client-java/blob/master/test/src/com/aerospike/test/sync/basic/TestOperateList.java) | C | [C#](https://github.com/aerospike/aerospike-client-csharp/blob/master/Framework/AerospikeTest/Sync/Basic/TestOperateList.cs) | Go | Node.js | PHP | Ruby | REST | Rust |

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

[Java](https://www.aerospike.com/apidocs/java/com/aerospike/client/cdt/ListOperation.html#removeByValueRange-java.lang.String-com.aerospike.client.Value-com.aerospike.client.Value-int-com.aerospike.client.cdt.CTX...-) 
| [C](https://www.aerospike.com/apidocs/c/df/d6c/group__list__operations.html#ga9ad5acd8599e4a17de3cd83e7cef7204) 
| [C#](https://www.aerospike.com/apidocs/csharp/html/M_Aerospike_Client_ListOperation_RemoveByValueRange.htm) 
| [Go](https://godoc.org/github.com/aerospike/aerospike-client-go#ListRemoveByValueRangeOp) 
| [Python](https://aerospike-python-client.readthedocs.io/en/latest/aerospike_helpers.operations.html#aerospike_helpers.operations.list_operations.list_remove_by_value_range) 
| [Node.js](https://www.aerospike.com/apidocs/nodejs/module-aerospike_lists.html#.removeByValueRange__anchor) 
| PHP 
| [Ruby](https://www.rubydoc.info/gems/aerospike/Aerospike/CDT/ListOperation#remove_by_value_range-class_method) 
| REST 
| [Rust](https://docs.rs/aerospike/latest/aerospike/operations/lists/fn.remove_by_value_range.html) |