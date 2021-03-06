## Map Performance

- [Legend](#legend)
- [Notes](#notes)
- [Performance Analysis Example](#performance-analysis-examplle)

δ»₯δΈ π θΏθ‘ζΆζ§θ½εζζ―ζεζε΅γ

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

ζ―δΈͺδΏ?ζΉζδ½ι½ζδΈδΈͺ `+C` η¨δΊεε₯ζΆε€εΆοΌδ»₯εθ?Έε¨ε€±θ΄₯ζΆεζ»γ

---

### <span id="notes"> Notes </span>

- ε¨ in-memory ε½εη©Ίι΄δΈ­οΌ K-ordered map ζδΈδΈͺ offset indexοΌKV-ordered map ζδΈδΈͺ full indexγ
- ε½ε½εη©Ίι΄ζ― data-on-SSD ζ data-in-PMEM ζΆοΌε―ΉδΊζζmapζδ½οΌιζ©ε(2) **K-ordered gives the best performance**οΌδ»£δ»·ζ―η£ηδΈη 4 δΈͺι’ε€ε­θγθΏζ―ε δΈΊθΏδΊε½εη©Ίι΄δΈ­η map δΈζΊεΈ¦ indexοΌδ»₯θηη£ηη©Ίι΄οΌοΌεΉΆδΈ K-ordered ε―Ήζζ map ζδ½ι½ζ²‘ζζ§θ½ηΌΊι·γ
- εΊδΊ interval ηζδ½ε·ζεζ° [start, stop)οΌεΆδΈ­θ΅·ε§εΌεε«ε¨εοΌεζ­’εΌδΈεε«ε¨εγε½ζ²‘ζ map ordering and/or data is not in memory ζΆοΌη±δΊηΌΊε° map index metadataοΌοΌε?δ»¬ιεΈΈζ―εΊδΊ range ηζδ½εΏ«γ
- ε―ΉδΊ data-in-memory ε½εη©Ίι΄δΈ­η ordered mapοΌεΈ¦ζεζ° (rank, count) ζ (index, count) ηεΊδΊ range ηζδ½ιεΈΈεΏ«γ
- εΊδΊ Key ηζδ½ζ―εͺδΊεΈ¦ζεηΌ `by_index`γ`by_index_range` εεηΌεθ― `key` ηζδ½γε¨θΏε `rank` η±»εηη»ζζΆοΌθΏδΊζδ½ιεΈΈδΌε―Όθ΄ι’ε€ηζ§θ½ζε€±γε?δ»¬ιεΈΈιθΏ K-ordered ε offset index metadataοΌε½ε½εη©Ίι΄ζ― data-in-memory ζΆε―η¨οΌζ₯ε ιγ
- εΊδΊ Key ηζδ½ζ―εͺδΊεΈ¦ζεηΌ `by_rank`γ`by_rank_range` εεηΌεθ― `value` ηζδ½γε¨θΏε `index` η±»εηη»ζζΆοΌθΏδΊζδ½ιεΈΈδΌε―Όθ΄ι’ε€ηζ§θ½ζε€±γε?δ»¬ιεΈΈιθΏ KV-ordered ε full index metadataοΌε½ε½εη©Ίι΄ζ― data-in-memory ζΆε―η¨οΌζ₯ε ιγ
- [for version < 3.16.0](https://docs.aerospike.com/docs/guide/cdt-map-complex0.html) οΌ ζδΈεη map ζ§θ½θ‘¨γ
- ζιοΌζεζε΅οΌεΉ³εζε΅εζε₯½ζε΅ηζ§θ½ζ―δΈεηη±»ε«γ

---

### <span id="performance-analysis-example"> Performance Analysis Example </span>

θθδΈδΈͺε­ε¨ζ¨‘εοΌεΆδΈ­ map key ζ― nameοΌmap value ζ―δΈ name ε³θη scoreγ

```
map: {name1: score1, name2: score2, ...}
```

εθ?Ύζ¨ζ³δΌεζ§θ½δ»₯θ·εεΎεε 10 ηεη§°γθ·εΎε 10 εηζδ½ε°ζ― `get_by_rank_range`γ

#### Data in memory

δ»ε€ζεΊ¦θ‘¨δΈ­ε―δ»₯ηεΊοΌδΈΊ `get_by_rank_range` ιζ© `KV-Ordered(4)` ε°ε―Όθ΄ O(M) ζ§θ½γη±δΊθΏζ― data-in-memoryοΌε ζ­€ KV-Ordered ε·ζ full map indexingγζ­€ε€οΌε δΈΊζδ»¬θ·ε top 10οΌη΄’εΌζ§θ½ζ― O(10) ζζε?ζΆι΄γ

η±δΊζ°ζ?εΊηεε₯ε€εΆζ§θ΄¨οΌζι’ε€η `+C` ζζ¬οΌε¦ζζ­€ε½εη©Ίι΄ζη£ηζ―ζοΌε `+W` ζζ¬γ

ζ§θ½: O(1) + C [+ W]

#### Return type extra cost

εζ¬‘ζ₯η `get_by_rank_range`οΌε¦ζη»ζη±»εζ― indexοΌεδΌζι’ε€ηζ§θ½ζζ¬οΌε¦ `get_by_rank_range` η `Result Type` ζ `+Index` γ

δΎε¦οΌε―ΉδΊ K-ordered data-in-memory mapοΌ`get_by_rank_range` ηεΊζ¬ζ§θ½ζ―οΌO((R + M) log N)

ε―ΉδΊ `get_by_rank_range(rank, count, opFlags=index)`οΌO((R + M) log N + M)