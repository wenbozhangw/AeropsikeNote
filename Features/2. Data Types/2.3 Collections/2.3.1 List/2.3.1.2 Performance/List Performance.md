## List Performance

- [Legend](#legend)
- [Notes](#notes)

以下 𝓞 运行时性能分析是最坏情况。

| Operation | ResultType | Unordered(1) | Ordered(2) | Ordered(3) |
| --- | --- | --- | --- | --- |
| **Set Type Ops** | | | | |
| [set_type](https://docs.aerospike.com/docs/guide/operations/list/set_order.html) to unordered | | 1 | 1 | 1 |
| [set_type](https://docs.aerospike.com/docs/guide/operations/list/set_order.html) to ordered | | N log N | 1 | 1 |
| **Read Ops** | | | | |
| [size](https://docs.aerospike.com/docs/guide/operations/list/size.html) | | 1 | 1 | 1 |
| [get_by_index](https://docs.aerospike.com/docs/guide/operations/list/get_by_index.html) | Base | N | N | 1 |
| [get_by_rank](https://docs.aerospike.com/docs/guide/operations/list/get_by_rank.html) | Base | R log N + N | N | 1 |
| [get_by_rank_range](https://docs.aerospike.com/docs/guide/operations/list/get_by_rank_range.html) | Base | N + M | N + M | M |
| | +Rank | +L*N | | |
| [get_by_value_interval](https://docs.aerospike.com/docs/guide/operations/list/get_by_value_interval.html) <br/> [get_all_by_value](https://docs.aerospike.com/docs/guide/operations/list/get_all_by_value.html) | Base | N + M | log N + N | log N + M |
| [get_by_rank_range](https://docs.aerospike.com/docs/guide/operations/list/get_by_rank_range.html) | Base | R log N + N | N + M | M |
| [get_by_value_rel_rank_range](https://docs.aerospike.com/docs/guide/operations/list/get_by_value_rel_rank_range.html) | Base | R log N + N | N + M + log N | M + log N |
| [get_all_by_value_list](https://docs.aerospike.com/docs/guide/operations/list/get_all_by_value_list.html) | Base | (M+N)log M + N | (M+N)log M + N | (M+N)log M |
| **Modify Ops** | | | | |
| [set(0, v)](https://docs.aerospike.com/docs/guide/operations/list/set.html) <br/> [append(v)](https://docs.aerospike.com/docs/guide/operations/list/append.html) <br/> [insert(0, v)](https://docs.aerospike.com/docs/guide/operations/list/insert.html) | | 1 | log N + N | log N  |
| [insert(i > 0, v)](https://docs.aerospike.com/docs/guide/operations/list/insert.html) <br/> [set(i > 0, v)](https://docs.aerospike.com/docs/guide/operations/list/set.html) | | N | - | - |
| ADD_UNIQUE set/append/insert | | N | log N + N | log N |
| [append_items(m)](https://docs.aerospike.com/docs/guide/operations/list/append_items.html) <br/> [insert_items(0, m)](https://docs.aerospike.com/docs/guide/operations/list/insert_items.html) | | M | (M+N)log M + N |(M+N)log M + M |
| [insert_items(i > 0, m)](https://docs.aerospike.com/docs/guide/operations/list/insert_items.html) | | M + N | - | - |
| ADD_UNIQUE append_items/insert_items |  | N* × M | (M+N)log M + N | (M+N)log M + M |
| [increment](https://docs.aerospike.com/docs/guide/operations/list/increment.html) |  | N | - | - |
| [sort](https://docs.aerospike.com/docs/guide/operations/list/sort.html) | | N log N | 1 | 1 |
| UNIQUE sort | | N log N | N | N |
| [clear](https://docs.aerospike.com/docs/guide/operations/list/clear.html) | | 1 | 1 | 1 |
| [remove_by_index](https://docs.aerospike.com/docs/guide/operations/list/remove_by_index.html) | Base | N | N | 1 |
| [remove_by_rank](https://docs.aerospike.com/docs/guide/operations/list/remove_by_rank.html) | Base | R log N + N | N | 1 |
| [remove_by_index_range](https://docs.aerospike.com/docs/guide/operations/list/remove_by_index_range.html) | Base | N + M | N + M | M |
| | +Rank | +L*N | | |
| [remove_by_value_interval](https://docs.aerospike.com/docs/guide/operations/list/remove_by_value_interval.html) <br/> [remove_all_by_value](https://docs.aerospike.com/docs/guide/operations/list/remove_all_by_value.html) | Base | N + M | log N + N | log N + M |
| [remove_by_rank_range](https://docs.aerospike.com/docs/guide/operations/list/remove_by_rank_range.html) | Base | R log N + N | N + M | M |
| [remove_by_value_rel_rank_range](https://docs.aerospike.com/docs/guide/operations/list/remove_by_value_rel_rank_range.html) | Base | R log N + N |N + M + log N | M + log N |
| [remove_all_by_value_list](https://docs.aerospike.com/docs/guide/operations/list/remove_all_by_value_list.html) | Base |(M+N)log M + N | (M+N)log M + N | 	(M+N)log M |
| **Mode Modifiers** | | | | |
| Data On SSD | | +D + W | +D + W | +D + W |
| Data-in-memory with persistence | | + W | + W | + W |

---

### <span id="legend"> Legend </span>

- `1` : Unordered list
- `2` : Ordered list, namespace stores its data on SSD
- `3` : Ordered list, namespace stores its data in memory
- `D` : Cost of load from disk
- `W` : Cost of write to disk
- `N` : Element count of list
- `M` : Element count of parameter, range or interval
- `R` : For rank r, R = min(r, N - r - 1)
- `L` : For index i, L = min(i, N - i - 1)
- `C` : Cost of memcpy on packed list
- `n` : The total byte size of list elements

每个修改操作都有一个 `+C` 用于写入时复制，以允许在失败时回滚。

---

### <span id="Notes"> Notes </span>

- list性能通常由元素访问决定
- 内存中 namespace(3) 中的列表有一个偏移索引，他给出了最坏情况 𝓞 (1) 的元素访问。
- 对于 SSD 上有数据的命名空间，会产生额外的 𝓞(n) memcpy 性能成本。与 element walk 𝓞(N) 相比，这可以忽略不计。
- 复制成本是 𝓞(n)，因为完整的记录被复制并保存在副本上。
- 提醒：最坏情况、平均情况和最好情况的性能是不同的类别。你应该比较喜欢。