## List Operations

以下是通用操作名称，可能与每个语言客户端中使用的方法名称不同。有关更多信息，请参阅每个操作的文档页面。

| Operation | Description |
| --- | --- |
| **Set Type Operations** | |
| [set_order](https://docs.aerospike.com/docs/guide/operations/list/set_order.html) | 将 list 设置为有序或无序。 |
| **Read Operations** | |
| [size](https://docs.aerospike.com/docs/guide/operations/list/size.html) | 获取 list 中元素的个数。 |
| [get_by_index](https://docs.aerospike.com/docs/guide/operations/list/get_by_index.html) | 获取指定索引位置的元素。 |
| [get_by_index_range](https://docs.aerospike.com/docs/guide/operations/list/get_by_index_range.html) | 获取由起始索引和 count 指定的范围内的元素。 |
| [get_by_rank](https://docs.aerospike.com/docs/guide/operations/list/get_by_rank.html) | 获取指定 rank 位置的元素。 |
| [get_by_rank_range](https://docs.aerospike.com/docs/guide/operations/list/get_by_rank_range.html) | 获取由起始 rank 和 count 指定的范围内的元素。 |
| [get_all_by_value](https://docs.aerospike.com/docs/guide/operations/list/get_all_by_value.html) | 获取与值或值模式匹配的所有元素。 |
| [get_all_by_value_list](https://docs.aerospike.com/docs/guide/operations/list/get_all_by_value_list.html) | 获取与 value list 或值模式之一匹配的所有元素。 |
| [get_by_value_interval](https://docs.aerospike.com/docs/guide/operations/list/get_by_value_interval.html) | 获取两个值或值模式之间的区间内的元素。 |
| [get_by_value_rel_rank_range](https://docs.aerospike.com/docs/guide/operations/list/get_by_value_rel_rank_range.html) | 从给定 value 的 relative rank 开始，获取范围内的元素。 |
| **Modify Operations** | | 
| [clear](https://docs.aerospike.com/docs/guide/operations/list/clear.html) | 清空 list 中的元素。 |
| [sort](https://docs.aerospike.com/docs/guide/operations/list/sort.html) | 对无序列表中的元素进行排序。 |
| [append](https://docs.aerospike.com/docs/guide/operations/list/append.html) | 将元素添加到无序列表的末尾，或按 rank 添加到有序列表。 |
| [append_items](https://docs.aerospike.com/docs/guide/operations/list/append_items.html) | 将元素添加到无序列表的末尾，或按 rank 添加到有序列表。 |
| [insert](https://docs.aerospike.com/docs/guide/operations/list/insert.html) | 在无序列表的指定索引位置插入一个元素。 |
| [insert_items](https://docs.aerospike.com/docs/guide/operations/list/insert_items.html) | 将元素插入到无序列表的指定索引位置。 |
| [set](https://docs.aerospike.com/docs/guide/operations/list/set.html) | 设置或覆盖指定索引位置的元素。 |
| [increment](https://docs.aerospike.com/docs/guide/operations/list/increment.html) | 在索引位置（无序列表）或 rank（有序）处增加数值。 |
| [remove_by_index](https://docs.aerospike.com/docs/guide/operations/list/remove_by_index.html) | 移除指定索引位置的元素。 |
| [remove_by_index_range](https://docs.aerospike.com/docs/guide/operations/list/remove_by_index_range.html) | 移除由起始索引和count指定的范围内的元素。 |
| [remove_by_rank](https://docs.aerospike.com/docs/guide/operations/list/remove_by_rank.html) | 移除指定 rank 的元素。 |
| [remove_by_rank_range](https://docs.aerospike.com/docs/guide/operations/list/remove_by_rank_range.html) | 移除由指定 rank 和count指定的范围内的元素。 |
| [remove_all_by_value](https://docs.aerospike.com/docs/guide/operations/list/remove_all_by_value.html) | 移除与 value 或 value pattern 匹配的所有元素。 |
| [remove_all_by_value_list](https://docs.aerospike.com/docs/guide/operations/list/remove_all_by_value_list.html) | 移除与 values list 或 value pattern 匹配的所有元素。 |
| [remove_by_value_interval](https://docs.aerospike.com/docs/guide/operations/list/remove_by_value_interval.html) | 删除两个 values 之间或 value patterns 间隔中的元素。 |
| [remove_by_value_rel_rank_range](https://docs.aerospike.com/docs/guide/operations/list/remove_by_value_rel_rank_range.html) | 从给定 value 的 relative rank 开始删除范围内元素。 |

对 List、Map 和标量数据类型的多个操作可以组合成一个 single-record transaction 。