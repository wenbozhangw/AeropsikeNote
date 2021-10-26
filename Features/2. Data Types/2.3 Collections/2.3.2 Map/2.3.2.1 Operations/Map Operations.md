## Map Operations

以下是 server name ，可能与您的客户端语言中实现的名称不同。有关等效操作，请参阅您的客户端文档 。

| Operation | Op Type | Supported Result Types | Supported Result Types | Supported Result Types | Supported Result Types |
| --- | --- | --- |  --- | --- | --- |
| **Set Type Op** | |  **Index** | **Rank** | **IndexRange** |  **RankRage** |
| `set_type` | bin | N/A | N/A | N/A | N/A |
| **Modify Ops** | |  **Index** | **Rank** | **IndexRange** |  **RankRage** |
| `put` | single | N/A | N/A | N/A | N/A |
| `put_items` | multi | N/A | N/A | N/A | N/A |
| `increment` | single | N/A | N/A | N/A | N/A |
| `decrement` | single | N/A | N/A | N/A | N/A |
| `clear` | bin | N/A | N/A | N/A | N/A |
| `remove_by_key` | single | ✓ | ✓ | ✓ | ✗ |
| `remove_by_index` | single | ✓ | ✓ | ✓ | ✗ |
| `remove_by_rank` | single | ✓ | ✓ | ✗ | ✓ |
| `remove_by_key_interval` | multi | ✓ | ✓ | ✗ | ✓ |
| `remove_by_index_range` | multi | ✓ | ✓ | ✓ | ✗ |
| `remove_by_value_interval` | multi | ✓ | ✓ | ✗ | ✓ |
| `remove_all_by_value` | multi | ✓ | ✓ | ✗ | ✓ |
| `remove_by_rank_range` | multi | ✓ | ✓ | ✗ | ✓ |
| `remove_by_key_rel_index_range` | multi | ✓ | ✓ | ✓ | ✗ |
| `remove_by_value_rel_index_range` | multi | ✓ | ✓ | ✗ | ✓ |
| `remove_all_by_key_list` | multi | ✓ | ✗ | ✗ | ✗ |
| `remove_all_by_value_list` | multi | ✓ | ✓ | ✗ | ✗ |
| **Read Ops** | |  **Index** | **Rank** | **IndexRange** |  **RankRage** |
| `size` | bin | ✓ | ✓ | ✓ | ✗ |
| `get_by_key` | single | ✓ | ✓ | ✓ | ✗ |
| `get_by_index` | single | ✓ | ✓ | ✓ | ✗ |
| `get_by_rank` | single | ✓ | ✓ | ✗ | ✓ |
| `get_by_key_interval` | multi | ✓ | ✓ | ✓ | ✗ |
| `get_by_index_range` | multi | ✓ | ✓ | ✓ | ✗ |
| `get_by_value_interval` | multi | ✓ | ✓ | ✗ | ✓ |
| `get_all_by_value` | multi | ✓ | ✓ | ✗ | ✓ |
| `get_by_rank_range` | multi | ✓ | ✓ | ✗ | ✓ |
| `get_by_key_rel_index_range` | multi | ✓ | ✓ | ✓ | ✗ |
| `get_all_by_key_list` | multi | ✓ | ✗ | ✗ | ✗ |
| `get_all_by_value_list` | multi | ✓ | ✓ | ✗ | ✗ |

`None`, `Count`, `Key`, `Value` and `KeyValue` 结果类型始终可用于带有 `resultType` 参数的操作。

`RevIndex` 和 `RevRank` 分别在 [Index and Rank](https://docs.aerospike.com/docs/guide/cdt-map.html#map-terminology) 可用时可用。

#### Op Type

| Op Type | Description |
| --- | --- |
| bin | 影响整个 bin。 |
| single | 影响 1 个 map entry，返回单个结果。 |
| multi | 影响多个 map entry，返回结果 list or map。 |

#### Performance

Map operations 的操作复杂度分析请参考 [Map Performance](https://docs.aerospike.com/docs/guide/cdt-map-performance.html) 文档。

---

### Details

#### Modify Operations

- `set_type(type)`
  
  Return : Nothing

  | Type |
  | --- |
  | UNORDERED |
  | K_ORDERED |
  | KV_ORDERED |

---

##### Modify Flags

Since version 4.3

| Flag | Description |
| --- | --- |
| MODIFY_DEFAULT | 默认行为。 |
| NO_OVERWRITE | 不允许替换现有元素。 |
| NO_CREATE | 不允许创建新元素。 |
| NO_FAIL | 如果发生策略违规，则无操作而不是失败，例如 NO_OVERWRITE 或 NO_CREATE |
| DO_PARTIAL | 与 NO_FAIL 一起使用时，添加不违反策略的元素。 |

---

- `increment(createType, key, delta-value)`, `decrement(createType, key, delta-value)`
    
  Return : 递增/递减后的新值
  
  仅适用于 `{key: value}` pair 中的 integer 或 float 类型。如果 bin 不存在，则使用 `type = createType` 创建 map bin。

  ##### Type Interaction between `value` and `delta-value`
  
  | Value-type | Delta: integer | Delta: float |
  | --- | --- | --- |
  | Map Entry: `integer` | Add normally. | Truncate to nearest `integer` and add. |
  | Map Entry: `float` | Convert `integer` to `float` and add. | Add normally. |

  使用减法代替加法进行 `decrement` operation 。

---

- `clear()`

  Return : Nothing.
  
  清除 map。map 类型保持不变。

---

- `remove_by_key(resultType, key)`

  Return : Single result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table
 
  删除 entry `{key: value}`，其中 `map.key = key`。

---

- `remove_by_index(resultType, index)`

  Return : Single result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  删除 entry `{key: value}`，其中 `map.key` 是 `i == index` 的第 i 个最小 key。

---

- `remove_by_rank(resultType, rank)`

  Return : Single result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  删除 entry `{key: value}`，其中 `map.key` 是 `i == rank` 的第 i 个最小 key。

---

- `remove_by_key_interval(resultType, keyStart[, keyStop])`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  删除所有 `{key: value}` pair，其中 `map.key >= keyStart` 且 `map.key < keyStop`。省略 `keyStop` 选择的元素，其中 `map.key >= keyStart` 。

请参阅 [interval comparison](https://docs.aerospike.com/docs/guide/cdt-ordering.html#intervals) 。

---

- `remove_by_index_range(resultType, index[, count])`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  删除所有 `{key: value}` pair，其中 `key = indexof(map.key)` 且 `key >= index` 且 `k < index + count`。省略 `count` 选择的元素，其中 `key >= origin + index` 。

---

- `remove_by_value_interval(resultType, valueStart[, valueStop])`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  删除所有 `{key: value}` pair，其中 `map.value >= valueStart` 且 `map.value < valueStop`。省略 `valueStop` 选择的元素，其中 `map.value >= valueStart` 。

请参阅 [interval comparison](https://docs.aerospike.com/docs/guide/cdt-ordering.html#intervals) 。

---

- `remove_all_by_value(resultType, value)`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  删除所有 `{key: value}` pair，其中 `map.value == value` 。

---

- `remove_by_rank_range(resultType, rank[, count])`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  删除所有 `{key: value}` pairs，其中 `r = rankof(map.value)` 且 `r > rank` 且 `r < rank + count`。省略 `count` 选择的元素，其中 `r >= rank`。

---

- `remove_by_key_rel_index_range(resultType, key, index[, count])`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  删除所有 `{key: value}` pairs，其中 `origin = index(key), k = indexof(map.key)` 且 `k >= origin + index` 且 `k < origin + index + count`。省略 `count` 选择的元素，其中 `k >= origin + index`。

--- 

- `remove_by_value_rel_rank_range(resultType, value, rank[, count])`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  删除所有 `{key: value}` pairs，其中 `origin = index(key), r = rankof(map.value)` 且 `r >= origin + range` 且 `r < origin + range + count`。省略 `count` 选择的元素，其中 `r >= origin + rank`。

  Relative operations added since version 4.3

---

- `remove_all_by_key_list(resultType, keyList)`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  删除所有 `{key: value}` pairs，其中 `map.key ∈ keyList` 。

---

- `remove_all_by_value_list(resultType, valueList)`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  删除所有 `{key: value}` pairs，其中 `map.value ∈ valueList` 。

  INDEX result supported since version 3.16.0

---

#### Read Operations

- `size()`

  Return: Element count

---

- `get_by_key(resultType, key)`

  Return: Single result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  获取 entry `{key: value}` 其中 `map.key == key`。

---

- `get_by_index(resultType, index)`

  Return: Single result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  获取 entry `{key: value}`，其中 `map.key` 是 `i == index` 的第 i 个最小 key。

---

- `get_by_rank(resultType, rank)`

  Return: Single result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  获取 entry `{key: value}`，其中 `map.value` 是 `i == rank` 的第 i 个最小 key。

---

- `get_by_key_interval(resultType, keyStart[, keyStop])`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  获取所有 `{key: value}` pair，其中 `map.key >= keyStart` 且 `map.key < keyStop`。省略 `keyStop` 选择的元素，其中 `map.key >= keyStart` 。

  请参阅 [interval comparison](https://docs.aerospike.com/docs/guide/cdt-ordering.html#intervals) 。

---

- `get_by_index_range(resultType, index[, count])`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  获取所有 `{key: value}` pair，其中 `key = indexof(map.key)` 且 `key >= index` 且 `k < index + count`。省略 `count` 选择的元素，其中 `key >= origin + index` 。

---

- `get_by_value_interval(resultType, valueStart[, valueStop])`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  获取所有 `{key: value}` pair，其中 `map.value >= valueStart` 且 `map.value < valueStop`。省略 `valueStop` 选择的元素，其中 `map.value >= valueStart` 。

  请参阅 [interval comparison](https://docs.aerospike.com/docs/guide/cdt-ordering.html#intervals) 。

---

- `get_all_by_value(resultType, value)`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  获取所有 `{key: value}` pair，其中 `map.value == value` 。

---

- `get_by_rank_range(resultType, rank[, count])`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  获取所有 `{key: value}` pairs，其中 `r = rankof(map.value)` 且 `r > rank` 且 `r < rank + count`。省略 `count` 选择的元素，其中 `r >= rank`。

---

- `get_by_key_rel_index_range(resultType, key, index[, count])`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  获取所有 `{key: value}` pairs，其中 `origin = index(key), k = indexof(map.key)` 且 `k >= origin + index` 且 `k < origin + index + count`。省略 `count` 选择的元素，其中 `k >= origin + index`。

--- 

- `get_by_value_rel_rank_range(resultType, value, rank[, count])`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  获取所有 `{key: value}` pairs，其中 `origin = index(key), r = rankof(map.value)` 且 `r >= origin + range` 且 `r < origin + range + count`。省略 `count` 选择的元素，其中 `r >= origin + rank`。

---

- `get_all_by_key_list(resultType, keyList)`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  获取所有 `{key: value}` pairs，其中 `map.key ∈ keyList` 。

---

- `get_all_by_value_list(resultType, valueList)`

  Return : Multi result, see [resultType](https://docs.aerospike.com/docs/guide/cdt-map.html#resultType) table

  获取所有 `{key: value}` pairs，其中 `map.value ∈ valueList` 。
