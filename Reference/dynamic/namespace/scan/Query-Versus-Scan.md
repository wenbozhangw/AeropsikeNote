## Query Versus Scan

### Usage

扫描和查询的用法之间的区别是 predicate 的用法。扫描不使用 "WHERE" 子句，而查询使用它。

*Scan*
```
aql> select * from test.testset
```

*Query*
```
aql> SELECT a, b FROM test.testset WHERE b = 123
```

### Internals

为了检索 set 中的所有记录，扫描将爬取(或 reduces)该节点拥有主副本的命名空间的所有分区，并选择属于该 set 的记录。扫描在内部查看每个记录的元数据，并识别记录是否属于该集合。对于整个命名空间的扫描，将返回该节点对于给定命名空间拥有的所有主分区的记录。但是，如果使用二级索引查询，则在查找它们的值之前，将通过二级索引直接在内存中选择匹配的记录（根据命名空间的存储引擎配置，在内存中还是在持久存储中）。

### Influence of result set size on performance

如上所述，无论 set 大小如何，在执行扫描时，都必须对整个命名空间进行爬取。因此，扫描小 set 所花费的时间取决于它所属的命名空间的大小。对于二级索引查询，花费的时间与返回的记录数成正比。

通常，二级索引查询至少与扫描给定结果集大小一样快。

### Scan or Query ?

考虑以下情形，在这种情况下，用户考虑将二级索引 bin 放入 set 的每条记录中，并且所有索引值都相等，以便在该容器上进行查询以检索整个 set。有趣的问题是关于这种查询与该集合的正常扫描之间的性能差异。

### Answer

在个别情况下对这两种方案进行基准测试可以揭示确切的性能结果，但是通常，即使在这种情况下，由于二级索引标识和相应的以及索引检测都在内存中，因此通过二级索引查询获取记录的速度应该比扫描所有分区以检索属于该 set 的记录更快，而且差异将非常大，因为该集合所属的名称空间非常大。

指出二级索引当前存在的一些弱点也很重要：

 1. 快速重启会减慢速度，因为它需要通过扫描所有记录值来重新构建二级索引。
 2. 将使用内存。二级索引存储器的大小也比较困难。
 3. 可能会减慢某些写入 transactions 的速度（替换 transactions 以及prole write transactions 将需要查询先前的值以清理二级索引）。
 4. 迁移期间潜在的不一致，因为分区可能在二级索引查询期间向右移动（此行为在扫描中适用）。已作为改进请求AER-3291提交。 `query-pre-reserve-partitions` 选项可以帮助减少某些用例的不一致窗口。
 5. 如果用例涉及在二级索引垃圾收集运行之前删除和重新添加具有不同 bin 值（`data-in-memroy false`）的记录，则二级索引垃圾收集器不会清理分配的内存。这已作为改进请求AER-1126提交。