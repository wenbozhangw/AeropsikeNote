## Map Performance

- [Legend](#legend)
- [Notes](#notes)
- [Performance Analysis Example](#performance-analysis-examplle)

以下 𝓞 运行时性能分析是最坏情况。

| Operation | ResultType | Unordered(1) | K/KV-Ordered(2) | K-Ordered(3) | KV-Ordered(4) |
| --- | --- | --- | --- | --- | --- |
| **Set Type Ops** | | | | | |
| `set_type` to unordered | | 1 | 1 | 1 | 1 |
| `set_type` to k_ordered | | N log N | 1 | 1 | 1 |
| `set_type` to kv_ordered | | N log N | N log N | N log N | 1 |
| **Modify Ops** | | | | | | 
| `put` |  | N | N | log N | log N |
| `put_items` | | (N + M) log M + N | M (log M + log N) + N | M (log M + log N) + N | M (log M + log N) + N |
| `increment` <br/> `decrement` | | N | N | log N | log N |
| `clear` |  | 1 | 1 | 1 | 1 |
| `remove_by_key` | Base | N | N | log N | log N |
| | +Index | +N | +N | +N | +log N |
| `remove_by_index` | Base | L log N + N | N | 1 | 1 |
| | +Rank | +N | +N | +N | +log N |
| `remove_by_rank` | Base | R log N + N | R log N + N | R log N | 1 | 
| | +Index | +N | +1 | +1 | +1 |
| `remove_by_key_interval` | N + M | N + M | log N + M | log N + M |
| | +Rank | +N log N |  +N log | +N log N | +N log N |
| `remove_by_index_range` | Base | L log N + N | N + M | M | M |
|  | +Rank | +N log N |  +N log | +N log N | +N log N |
| `remove_by_value_interval` <br/> `remove_all_by_value` | Base | N + M | N + M | N + M | log N + M |
| | +Index | +N +M | +M | +M | +M |
| `remove_by_rank_range` | Base | R log N + N | R log N + N | R log N | M |
| | +Index | +N log N | +M | +M | +M |
| `remove_all_by_key_list` | Base |	(M+N)logM + N | M log N + N | M log N | M log N |
| `remove_all_by_value_list` | Base | (M+N)logM + N | (M+N)log M + N | (M+N)log M | M log N |
| **Read Ops** | | | | | |
| `size` | | 1 | 1 | 1 | 1 |
| `get_by_index` | Base | N | N | log N | log N |
| | +Index | +N | +N | +N | +log N |
| `get_by_index_range` | Base | L log N + N | N + M | M | M |
| | +Rank | +N log N | +N log | +N log N | +N log N |
| `get_by_rank` | Base | R log N + N | R log N + N | R log N | 1 |
| | +Index | +N | +1 | +1 | +1 |
| `get_by_key_interval` | Base | N | N | log N + M | log N + M |
| | +Rank | +N log N | +N log | +N log N | +N log N |
| `get_by_value_interval` <br/> `get_all_by_value` | Base | N + M | N + M | N + M | log N + M |
| | +Index | +N +M | +M | +M | +M |
| `get_by_rank_range` | Base | R log N + N | R log N + N | R log N | M |
| | +Index | +N log N | +M | +M | +M |
| `get_all_by_key_list` | Base | (M+N)logM + N | M log N + N | M log N | M log N |
| `get_all_by_value_list` | Base | (M+N)logM + N | (M+N)log M + N | (M+N)log M | M log N |
| **Mode Modifiers** | | | | | |
| Not-in-memory On-Disk | | +D +W | +D +W | +D +W | +D +W |
| Data-in-memory On-Disk | | +W | +W | +W | +W |

---

### <span id="legend"> Legend </span>

- `1` : Unordered map
- `2` : K-Ordered or KV-Ordered map, namespace stores its data on SSD or PMEM
- `3` : K-Ordered map, namespace stores its data in memory
- `4` : KV-Ordered map, namespace stores its data in memory
- `D` : Cost of load from disk
- `W` : Cost of write to disk 
- `N` : element count of map
- `M` : element count of parameter, range or interval
- `R` : for rank, R = min(r, N - r - 1)
- `L` : for index, L = min(r, N - i - 1)
- `C` : Cost of memcpy on packed map

每个修改操作都有一个 `+C` 用于写入时复制，以允许在失败时回滚。

---

### <span id="notes"> Notes </span>

- 在 in-memory 命名空间中， K-ordered map 有一个 offset index，KV-ordered map 有一个 full index。
- 当命名空间是 data-on-SSD 或 data-in-PMEM 时，对于所有map操作，选择列(2) **K-ordered gives the best performance**，代价是磁盘上的 4 个额外字节。这是因为这些命名空间中的 map 不携带 index（以节省磁盘空间），并且 K-ordered 对所有 map 操作都没有性能缺陷。
- 基于 interval 的操作具有参数 [start, stop)，其中起始值包含在内，停止值不包含在内。当没有 map ordering and/or data is not in memory 时（由于缺少 map index metadata），它们通常比基于 range 的操作快。
- 对于 data-in-memory 命名空间中的 ordered map，带有参数 (rank, count) 或 (index, count) 的基于 range 的操作非常快。
- 基于 Key 的操作是哪些带有后缀 `by_index`、`by_index_range` 和后缀单词 `key` 的操作。在返回 `rank` 类型的结果时，这些操作通常会导致额外的性能损失。它们通常通过 K-ordered 和 offset index metadata（当命名空间是 data-in-memory 时可用）来加速。
- 基于 Key 的操作是哪些带有后缀 `by_rank`、`by_rank_range` 和后缀单词 `value` 的操作。在返回 `index` 类型的结果时，这些操作通常会导致额外的性能损失。它们通常通过 KV-ordered 和 full index metadata（当命名空间是 data-in-memory 时可用）来加速。
- [for version < 3.16.0](https://docs.aerospike.com/docs/guide/cdt-map-complex0.html) ， 有不同的 map 性能表。
- 提醒：最坏情况，平均情况和最好情况的性能是不同的类别。

---

### <span id="performance-analysis-example"> Performance Analysis Example </span>

考虑一个存储模型，其中 map key 是 name，map value 是与 name 关联的 score。

```
map: {name1: score1, name2: score2, ...}
```

假设您想优化性能以获取得分前 10 的名称。获得前 10 名的操作将是 `get_by_rank_range`。

#### Data in memory

从复杂度表中可以看出，为 `get_by_rank_range` 选择 `KV-Ordered(4)` 将导致 O(M) 性能。由于这是 data-in-memory，因此 KV-Ordered 具有 full map indexing。此外，因为我们获取 top 10，索引性能是 O(10) 或恒定时间。

由于数据库的写入复制性质，有额外的 `+C` 成本，如果此命名空间有磁盘支持，则 `+W` 成本。

性能: O(1) + C [+ W]

#### Return type extra cost

再次查看 `get_by_rank_range`，如果结果类型是 index，则会有额外的性能成本，如 `get_by_rank_range` 的 `Result Type` 有 `+Index` 。

例如，对于 K-ordered data-in-memory map，`get_by_rank_range` 的基本性能是：O((R + M) log N)

对于 `get_by_rank_range(rank, count, opFlags=index)`：O((R + M) log N + M)