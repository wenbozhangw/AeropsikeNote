## List

Aerospike [List](https://en.wikipedia.org/wiki/List_%28abstract_data_type%29) 是一种抽象数据类型，具有两个具体的子类型 —— 无序列表 (Unordered List) 和 有序列表 (Ordered List)。两种类型的 List 共享相同的 API，并且可以通过 `set_order()` 操作在彼此之间进行转换。

List 可以存储 [scalar data types](https://docs.aerospike.com/docs/guide/data-types.html) 或包含嵌套的 List 或 [Map](https://docs.aerospike.com/docs/guide/cdt-map.html) 元素。 List操作最适合直接在 Aerospike 数据库上操作列表。


##### Contents

- [List Terminology](#list-terminology)
- [Element Ordering](#element-ordering)
  - [Unordered Lists](#unordered-lists)
  - [Ordered Lists](#ordered-lists)
- [List API](#list-api)
- [Development Guidelines and Tips](#development-guidelines-and-tips)
- [Performance](#performance)
- [Known Limitations](#known-limitations)
- [Language-Specific Client Operations](#language-specific-client-operations)

---

### <span id="list-terminology"> List Terminology </span>

对于以下 List，

```
[ 1, 4, 7, 3, 9, 26, 11 ]
```

- **index** 是list中元素的 (0 起点)位置。 
- **value** 在索引 2 处为 7。索引 -2 处的值为 26。
- **rank** 是列表中元素的值顺序。最低值元素的rank为 0。
- rank 为 2 的元素的值为 4。 rank为 -2 的元素的值为 11。

---

### <span id="element-ordering"> Element Ordering </span>

元素访问可以通过 index, rank (value ordering) 或 element value。

```
[ 1, 4, 7, 3, 9, 26, 11 ] # Unordered
[ 1, 3, 4, 7, 9, 11, 26 ] # Ordered
```

创建后，list的顺序不会更改，除非使用 `set_order()` API 调用。

#### <span id="unordered-lists"> Unordered Lists </span>

- 只要追加了新元素，无序列表(Unordered List)就会保持插入顺序。
- 在无序列表中，按 rank 获取元素将遵循 [the same](https://docs.aerospike.com/docs/guide/cdt-ordering.html) 的 (即时) 排序。

#### <span id="ordered-lists"> Ordered Lists </span>

- 有序列表(Ordered List)维护排名顺序，并在每次添加新元素时对其进行排序。
- 在有序列表中，混合数据类型的元素首先 [ordered by type, then by value](https://docs.aerospike.com/docs/guide/cdt-ordering.html) 。
- 默认情况下，新列表是无序的，除非创建策略另有指示。

---

### <span id="list-api"> List API </span>

使用 Aerospike 客户端 API，应用程序可以检索完整列表，或对单个元素进行操作。原子列表 (atomic list)操作适用于顶级元素(top level elements)，以及具有附加集合数据类型(additional collection data type)(CDT) [nested context](https://docs.aerospike.com/docs/guide/cdt-context.html) 的嵌套元素。

您可以单击以下每个操作以查看其详细说明，翻译文档参见 [Chinese List API](#chinese-list-api)。

- [`set_order()`](https://docs.aerospike.com/docs/guide/operations/list/set_order.html)
- [`sort()`](https://docs.aerospike.com/docs/guide/operations/list/sort.html), [`clear()`](https://docs.aerospike.com/docs/guide/operations/list/clear.html) 
- [`append()`](https://docs.aerospike.com/docs/guide/operations/list/append.html), [`append_items()`](https://docs.aerospike.com/docs/guide/operations/list/append_items.html)
- [`insert()`](https://docs.aerospike.com/docs/guide/operations/list/insert.html), [`insert_items()`](https://docs.aerospike.com/docs/guide/operations/list/insert_items.html)
- [`set()`](https://docs.aerospike.com/docs/guide/operations/list/set.html)
- [`increment()`](https://docs.aerospike.com/docs/guide/operations/list/increment.html)
- [`size()`](https://docs.aerospike.com/docs/guide/operations/list/size.html)
- [`get_by_index()`](https://docs.aerospike.com/docs/guide/operations/list/get_by_index.html), [`get_by_index_range()`](https://docs.aerospike.com/docs/guide/operations/list/get_by_index_range.html)
- [`get_by_rank()`](https://docs.aerospike.com/docs/guide/operations/list/get_by_rank.html), [`get_by_rank_range()`](https://docs.aerospike.com/docs/guide/operations/list/get_by_rank_range.html)
- [`get_all_by_value()`](https://docs.aerospike.com/docs/guide/operations/list/get_all_by_value.html), [`get_all_by_value_list()`](https://docs.aerospike.com/docs/guide/operations/list/get_all_by_value_list.html), [`get_by_value_interval()`](https://docs.aerospike.com/docs/guide/operations/list/get_by_value_interval.html), [`get_by_value_rel_rank_range()`](https://docs.aerospike.com/docs/guide/operations/list/get_by_value_rel_rank_range.html)
- [`remove_by_index()`](https://docs.aerospike.com/docs/guide/operations/list/remove_by_index.html), [`remove_by_index_range()`](https://docs.aerospike.com/docs/guide/operations/list/remove_by_index_range.html)
- [`remove_by_rank()`](https://docs.aerospike.com/docs/guide/operations/list/remove_by_rank.html), [`remove_by_rank_range()`](https://docs.aerospike.com/docs/guide/operations/list/remove_by_rank_range.html)
- [`remove_all_by_value()`](https://docs.aerospike.com/docs/guide/operations/list/remove_all_by_value.html), [`remove_all_by_value_list()`](https://docs.aerospike.com/docs/guide/operations/list/remove_all_by_value_list.html), [`remove_by_value_interval()`](https://docs.aerospike.com/docs/guide/operations/list/remove_by_value_interval.html), [`remove_by_value_rel_rank_range()`](https://docs.aerospike.com/docs/guide/operations/list/remove_by_value_rel_rank_range.html)

Aerospike 数据库 4.6 版本中添加了对嵌套 list/map 元素的操作。

Aerospike 4.3 版本中添加了相对排序操作(relative rank operations)。

##### Superseded operations

这些操作呗 `remove_by_*()` 和 `get_by_*()` 取代。这些操作已被弃用，并将在未来版本中删除。

- `get()`, `get_range()`, `get_range_from()`
- `pop()`, `pop_range()`
- `remove()`, `remove_range()`,
- `trim()`

---

### <span id="development-guidelines-and-tips"> Development Guidelines and Tips

- 对 List、Map 和 scalar data types 的多个操作可以组合成一个 single-record transaction。
- 将 list value写入 bin 或使用 list `append`, `insert`, `set` or `increment` 操作时，将创建 list bin。
- 在 list 的任一端插入不需要元素遍历并且通常很快。
- 对于 data-in-memory，内部结构体针对具有偏移索引的随机访问读取进行了优化。
- 如果命名空间保存到磁盘存储，磁盘写入可能会造成瓶颈。例如，如果没有足够的节点或设备，或者 key 使用分布不均。对于接近最大记录大小的bin尤其如此。

---

### <span id="performance"> Performance </span>

list操作的操作复杂度(operational complexity)分析参见 [List performance](https://docs.aerospike.com/docs/guide/cdt-list-performance.html) 。

---

### <span id="known-limitations"> Known Limitations </span>

- list 受最大记录大小限制。对于 SSD 上的数据，最大记录大小受 `write-block-size` 配置参数的限制。
- Lua UDF 不支持列表操作。

---

### <span id="language-specific-client-operations"> Language Specific Client Operations </span>

特定语言列表操作示例：

- [Java](https://github.com/aerospike/aerospike-client-java/tree/master/client/src/com/aerospike/client/cdt)
- [C](https://github.com/aerospike/aerospike-client-c/blob/master/examples/basic_examples/list/src/main/example.c)
- [C#](https://github.com/aerospike/aerospike-client-csharp/tree/master/Framework/AerospikeDemo)
- [GO](https://github.com/aerospike/aerospike-client-go/tree/master/examples)
- [Python](https://github.com/aerospike-examples/aerospike-operations-examples/tree/master/python/list)
- [Node.js](https://github.com/aerospike/aerospike-client-nodejs/tree/master/examples)
- [Rust](https://github.com/aerospike/aerospike-client-rust/)

---

### <span id="chinese-list-api"> Chinese List API </span>

- [`set_order()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20set_order.md)
- [`sort()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20sort.md), [`clear()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20clear.md)
- [`append()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20append.md), [`append_items()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20append_items.md)
- [`insert()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20insert.md), [`insert_items()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20insert_items.md)
- [`set()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20set.md)
- [`increment()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20increment.md)
- [`size()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20size.md)
- [`get_by_index()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20get_by_index.md), [`get_by_index_range()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20get_by_index_range.md)
- [`get_by_rank()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20get_by_rank.md), [`get_by_rank_range()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20get_by_rank_range.md)
- [`get_all_by_value()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20get_all_by_value.md), [`get_all_by_value_list()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20get_all_by_value_list.md),       [`get_by_value_interval()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20get_by_value_interval.md), [`get_by_value_rel_rank_range()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20get_by_value_rel_rank_range.md)
- [`remove_by_index()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20remove_by_index.md), [`remove_by_index_range()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20remove_by_index_range.md)
- [`remove_by_rank()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20remove_by_rank.md), [`remove_by_rank_range()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20remove_by_rank_range.md)
- [`remove_all_by_value()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20remove_all_by_value.md), [`remove_all_by_value_list()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20remove_all_by_value_list.md), [`remove_by_value_interval()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20remove_by_value_interval.md), [`remove_by_value_rel_rank_range()`](2.3.1.5 Outside/2.3.1.5.1 Chinese List API/List%20-%20remove_by_value_rel_rank_range.md)