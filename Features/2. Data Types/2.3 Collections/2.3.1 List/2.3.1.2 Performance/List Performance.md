## List Performance

- [Legend](#legend)
- [Notes](#notes)

δ»₯δΈ π θΏθ‘ζΆζ§θ½εζζ―ζεζε΅γ

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
| ADD_UNIQUE append_items/insert_items |  | N* Γ M | (M+N)log M + N | (M+N)log M + M |
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

ζ―δΈͺδΏ?ζΉζδ½ι½ζδΈδΈͺ `+C` η¨δΊεε₯ζΆε€εΆοΌδ»₯εθ?Έε¨ε€±θ΄₯ζΆεζ»γ

---

### <span id="Notes"> Notes </span>

- listζ§θ½ιεΈΈη±εη΄ θ?Ώι?ε³ε?
- εε­δΈ­ namespace(3) δΈ­ηεθ‘¨ζδΈδΈͺεη§»η΄’εΌοΌδ»η»εΊδΊζεζε΅ π (1) ηεη΄ θ?Ώι?γ
- ε―ΉδΊ SSD δΈζζ°ζ?ηε½εη©Ίι΄οΌδΌδΊ§ηι’ε€η π(n) memcpy ζ§θ½ζζ¬γδΈ element walk π(N) ηΈζ―οΌθΏε―δ»₯εΏ½η₯δΈθ?‘γ
- ε€εΆζζ¬ζ― π(n)οΌε δΈΊε?ζ΄ηθ?°ε½θ’«ε€εΆεΉΆδΏε­ε¨ε―ζ¬δΈγ
- ζιοΌζεζε΅γεΉ³εζε΅εζε₯½ζε΅ηζ§θ½ζ―δΈεηη±»ε«γδ½ εΊθ―₯ζ―θΎεζ¬’γ