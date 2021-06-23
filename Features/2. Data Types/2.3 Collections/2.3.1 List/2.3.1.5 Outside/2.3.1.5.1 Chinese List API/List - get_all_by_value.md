## List - get_all_by_value

### [List](https://docs.aerospike.com/docs/guide/cdt-list-ops.html) get_all_by_value(bin, value[, returnType, context])

获取 list 中匹配 _value_ 的所有元素。`INVERTED` 标志可用于获取与指定 value 不匹配的元素。

根据 return type 返回 zero, one or a list of values。返回的元素可能不按索引顺序排列。

无论 list 是无序还是有序，`get_all_by_value` 都会根据 [ordering rules](https://docs.aerospike.com/docs/guide/cdt-ordering.html) 返回相同的元素。"by value"操作在有序列表上的性能优于无序列表。

---

#### Trivial Example

用指定的标量值计算 list 中元素的数量。

```
# [1, 2, 1, 2]

# count all the elements whose value is 2
client.operate(key, [list_operations.list_get_by_value("l", 2, aerospike.LIST_RETURN_COUNT)])
# 2
```

---

#### Matching Tuples with Wildcard

通常将 Aerospike 中的复杂数据建模为元组列表，每个元素都是一个列表，其中索引顺序具有含义。有关示例，请参阅 [Aerospike Modeling: IoT Sensors](https://dev.to/aerospike/aerospike-modeling-iot-sensors-453a) 。

在大多数有序 pairs 的 list 中，我们可以通过使用通配符（通常在文档中风格化为 * ）来搜索特定类型的元素，例如第 0 个位置为 "v2" 的所有元组。

```
# [["v1", 1], ["v2", 2], ["v1", 3], ["v2", 4, {"a": 1}]]
key, metadata, bins = client.operate(
    key,
    [
        list_operations.list_get_by_value(
            "l", ["v2", aerospike.CDTWildcard()], aerospike.LIST_RETURN_VALUE
        )
    ],
)
print(bins["l"])
# [["v2", 2], ["v2", 4, {"a": 1}]]
```

通配符匹配元组末尾的任何元素序列。在此示例中，这意味着匹配第一个位置为 "v2" 且大小为两个或更多元素的任何元组。

通配符具有不同的特定于语言的构造，例如 Python 客户端中的 `aerospike.CDTWildcard()` 和 Java客户端中的 `Value.WildcardValue` 。

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

对于存储在内存中的有序列表，按值获取元素的最坏情况性能是 𝓞(log N + M)，对于存储在 SSD 上的有序列表 𝓞(log N + N) ，对于无序列表𝓞(N + M)，其中 M 是参数列表中的项目数，N 是列表中已有的项目数。

有关 List API 的完整最坏情况性能分析，请参阅 [List Performance](https://docs.aerospike.com/docs/guide/cdt-list-performance.html) 。

---

#### Code Sample

请参阅上面的实例，或按照下面特定于语言的链接中的任何一个获取操作的详细代码示例。

[Python](https://github.com/aerospike-examples/aerospike-operations-examples/blob/master/python/list/get_all_by_value.py) | [Java](https://github.com/aerospike/aerospike-client-java/blob/master/test/src/com/aerospike/test/sync/basic/TestOperateList.java) | C | [C#](https://github.com/aerospike/aerospike-client-csharp/blob/master/Framework/AerospikeTest/Sync/Basic/TestOperateList.cs) | Go | Node.js | PHP | Ruby | REST | Rust |

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

[Java](https://www.aerospike.com/apidocs/java/com/aerospike/client/cdt/ListOperation.html#getByValue-java.lang.String-com.aerospike.client.Value-int-com.aerospike.client.cdt.CTX...-) | [C](https://www.aerospike.com/apidocs/c/df/d6c/group__list__operations.html#ga9a3ed24a30d0592fa17370d481e9d45f) | [C#](https://www.aerospike.com/apidocs/csharp/html/M_Aerospike_Client_ListOperation_GetByValue.htm) | [Go](https://godoc.org/github.com/aerospike/aerospike-client-go#ListGetByValueOp) | [Python](https://aerospike-python-client.readthedocs.io/en/latest/aerospike_helpers.operations.html#aerospike_helpers.operations.list_operations.list_get_by_value) | [Node.js](https://www.aerospike.com/apidocs/nodejs/module-aerospike_lists.html#.getByValue__anchor) | PHP | [Ruby](https://www.rubydoc.info/gems/aerospike/Aerospike/CDT/ListOperation#get_by_value-class_method) | REST | [Rust](https://docs.rs/aerospike/latest/aerospike/operations/lists/fn.get_by_value.html) |