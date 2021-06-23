## Map 

Aerospike [Maps](https://en.wikipedia.org/wiki/Associative_array) 是一种抽象数据类型，具有三个具体的子类型 - Key Ordered Map (K-ordered)、Key-Value Ordered Map (KV-ordered) 和 Unordered Map。所有类型的 Map 都共享相同的 API，并且可以通过 [`set_type()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#setType) 操作在彼此之间进行转换。

Map 可以存储 [scalar data types](https://docs.aerospike.com/docs/guide/data-types.html) 或包含嵌套的 [List](https://docs.aerospike.com/docs/guide/cdt-list.html) 或 Map 元素。Map operations 最适合直接在 Aerospike 数据库上操作 map。

Map 中元素是根据排序类型存储的。无序 map 没有保证的顺序，而 K-ordered map 总是按 key 顺序存储。当 K-ordered Map 在 data-in-memory namespace中时，他还会有一个 key offset 索引加快访问速度。

KV-ordered map 也按照 key 排序存储。当 KV-ordered Map 位于 data-in-memory namespace 是，它还会有个完整索引 —— a key offset index, and a rank offset index。

Map key 可以是 String, Bytes, Integer, or Double。


##### Content

- [Map Terminology](#map-terminology)
- [Element Ordering](#element-ordering)
- [Map API](#map-api)
- [Development Guidelines and Tips](#development-guidelines-and-tips)
- [Performance](#performance)
- [Known Limitations](#known-limitations)
- [Language-Specific Client Operations](#language-specific-client-operations)

---

### <span id="map-terminology"> Map Terminology </span>

- **key** 是 map 中元素的标识符。
- **value** 是由特定 map key 标识的元素的值。
- **index** 是 map 的元素的 key order。
- **rank** 是 map 中元素的 [value order](https://docs.aerospike.com/docs/guide/cdt-ordering.html) 。如果有多个具有相同值的元素，则它们的 rank 基于它们的 index order。

```
{a:1, b:2, c:30, y:30, z:26}
```

map 中的元素有

| | a:1 | b:2 | c:30 | y:30 | z:26 |
| --- | --- | --- | --- | --- | --- |
| key | a | b | c | y | z |
| value | 1 | 2 | 30 | 30 | 26 |
| index | 0 or -5 | 1 or -4 | 2 or -3 | 3 or -2 | 4 or -1 |
| rank | 0 or -5 | 1 or -4 | 4 or -2 | 5 or -1 | 2 or -3 |

---

### <span id="element-ordering"> Element ordering </span>

在有序 Map 中，混合数据类型的元素首先 [order by type, then by value](https://docs.aerospike.com/docs/guide/cdt-ordering.html) 。无论 map 的顺序类型如何，都支持基于 key, rank, index and value 的操作，但它们的运行时性能会有所不同。

---

### <span id="map-api"> Map API </span>

使用 Aerospike 客户端 API，应用程序可以检索整个 map，或对单个元素进行操作。 原子 map 操作 (Atomic map operations) 适用于顶级元素，以及具有附加集合数据类型(CDT) [nested context](https://docs.aerospike.com/docs/guide/cdt-context.html) 的嵌套元素。

您可以单机下面每个操作以查看其详细说明。

- [`set_type()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#setType)
- [`put()`](https://docs.aerospike.com/docs/guide/operations/map/put.html), 
  [`put_items()`](https://docs.aerospike.com/docs/guide/operations/map/put_items.html)
- [`increment()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#incr),
  [`decrement()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#incr)
- [`clear()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#clear)
- [`size()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#size)
- [`get_by_key()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#getByKey),
  [`get_by_index()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#getByIndex),
  [`get_by_rank()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#getByRank)
- [`get_by_key_interval()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#getByKeyInterval), [`get_by_index_range()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#getByIndexRange)
- [`get_by_value_interval()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#getByValueInterval),
  [`get_by_rank_range()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#getByRankRange),
  [`get_all_by_value()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#getAllByValue)
- [`get_by_key_rel_index_range()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#getByKeyRelIndexRange),
  [`get_by_value_rel_rank_range()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#getByValueRelRankRange)
- [`get_all_by_key_list()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#getAllByKeyList),
  [`get_all_by_value_list()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#getAllByValueList)
- [`remove_by_key()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#removeByKey),
  [`remove_by_index()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#RemoveByIndex),
  [`remove_by_rank()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#removeByRank)
- [`remove_by_key_interval()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#removeByKeyInterval),
  [`remove_by_index_range()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#removeByIndexRange)
- [`remove_by_value_interval()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#removeByValueInterval),
  [`remove_by_rank_range()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#removeByRankRange),
  [`remove_all_by_value()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#removeAllByValue)
- [`remove_by_key_rel_index_range()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#removeByKeyRelIndexRange), [`remove_by_value_rel_rank_range()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#removeByValueRelRankRange)
- [`remove_all_by_key_list()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#removeAllByKeyList),
  [`remove_all_by_value_list()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html#removeAllByValueList)
  
Aerospike 数据库 4.6 版中添加了对嵌套 list/map 元素的操作。

Aerospike 4.3 版本中添加了 Relative rank operations。


#### <span id="map-operation-flag"> Map operation flag </span>

`get_*` 或 `remove_*` 操作有一个 `opFlags` 参数来控制它们返回的内容并修改它们的搜索条件。

| OpFlag | Description | 
| --- | --- |
| INVERTED | 反转 op 参数中提供的搜索条件。 |

- 删除 map 中 key values 最大的 10 个元素： `remove_by_index_range(-10, 10, NONE)`
- 删除 map 中除了 key values 最大的 10 个元素之外的所有元素: `remove_by_index_range(-10, 10, INVERTED)`

从 Aerospike 版本 3.16.0。


##### Map result types

`get_*` 和 `remove_*` 操作有一个 `resultType` 参数来控制它们返回的内容。

| ResultType | Description |
| --- | --- |
| None | 没有结果，由于不构建回复，因此对于更快的删除操作很有用。 |
| Index | Key order : 0 = smallest key, (N - 1) = largest key, 其中 N 是 map 中的 key 条目数量。 |
| RevIndex | 反转 key 顺序 : 0 = largest key. |
| Rank | Value order : 0 = smallest value, (N - 1) = largest value. |
| RevRank | 反转 value 顺序 : 0 = largest value. |
| Count | 返回符合条件的项目数量。 |
| Key | key for single item operations and list of keys for multi-ops. |
| Value | Value for single item operations, list of values for multi-ops. |
| KeyValue | Key Value pairs. 确切的格式取决于客户端。 |

- 只保留前 10 个关键元素，返回计数 : `remove_by_index_range(-10, 10, INVERTED | COUNT)`

从 Aerospike 版本 3.16.0 开始，`resultType` 被折叠到 `opFlags` 中。

---

### <span id="development-guidelines-and-tips"> Development guidelines and tips </span>

- 对 List、Map 和 scalar data types 的多个操作可以组合成一个 single-record transaction。
- 当 map value 写入 bin 时，或通过使用 map `put`, `put_items` 或 `increment` 操作将创建 map bin。
- 使用 `set_type()` 将无序 map 转换为 K-ordered or KV-ordered map。或者，使用带有 `put` or `put_items` 的 map policy，如果 map bin 不存在，policy 中的 `map_type` 将用于创建 map bin。
- 所有的 map 操作都适用于任何 `map_type`。唯一的区别是如下性能表中详述的性能。
- 通常，当命名空间是 SSD 上的数据时，K-ordered 以磁盘上 4 个额外字节为代价为所有 map 操作提供最佳性能。

---

### <span id="performance"> Performance </span>

Map operations的操作复杂度分析请参考 [Map Performance](https://docs.aerospike.com/docs/guide/cdt-map-performance.html) 文档。

---

### <span id="known-limitations"> Known Limitations </span>

- Map 受最大记录大小的限制。对于 SSD 上的数据，最大记录大小受 `write-block-size` 配置参数的限制。
- Lua UDF 不支持 map 操作。

---

### <span id="language-specific-client-operations"> Language-specific client operations </span>

Language-specific list operation examples:

- [Java](https://github.com/aerospike/aerospike-client-java/tree/master/client/src/com/aerospike/client/cdt)
- [C](https://github.com/aerospike/aerospike-client-c/blob/master/examples/basic_examples/list/src/main/example.c)
- [C#](https://github.com/aerospike/aerospike-client-csharp/tree/master/Framework/AerospikeDemo)
- [Go](https://github.com/aerospike/aerospike-client-go/tree/master/examples)
- [Python](https://github.com/aerospike/aerospike-client-python/tree/master/examples)
- [Node.js](https://github.com/aerospike/aerospike-client-nodejs/tree/master/examples)
- [Rust](https://github.com/aerospike/aerospike-client-rust/)