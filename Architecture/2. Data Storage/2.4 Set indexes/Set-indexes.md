## Set indexes

从 Aerospike 数据库 5.6 版本开始，您可以通过有选择地创建 set index 来将记录的成员索引到它们的 set 。使用 set index，可以提高 set 级别操作的性能，例如扫描 set 中的所有记录。

### Fast restarts

与二级索引相反， set index 与 [warm restart](https://docs.aerospike.com/docs/operations/manage/aerospike/fast_start/) 兼容。

---

### Memory consumption

Set indexes 在 set 中的每个记录大约消耗 16 字节。有关更多信息，请参阅 [capacity planning guide](https://docs.aerospike.com/docs/operations/plan/capacity/index.html) 。

---

### Effectiveness

如果一个 set 与包含它的 namespace 相比很小，并且没有创建 set index，则需要遍历整个主索引来查找属于该 set 的记录。Set index 可以提高获取这些记录的效率。

相对于 namespace 的 set 越小，set index 将提供的优势更大。当 set 相对于 namespace 变得足够大时，这种优势将消失，并且 set index 的内存成本将变得不值得。

作为广泛的指导原则，如果 set 大于 namespace 的 1%，则不太可能提供显著的优势。截止点将取决于用例和配置。例如，较大的记录（不在内存中）可能意味着较低的截止点，因为与索引查找相比，设备 I/O 将占主导地位。因此，建议仅尝试 set index 以最好地评估其有效性。

即使在单个节点上，也可以动态配置 set index。然后，可以将具有和没有 set index 的扫描进行相互比较。如果 set index 没有提供明显的优势，则可以将其丢弃。即使在进行扫描时，也可以启用和禁用 set indexes。

扫描将使用 set index (如果存在)，否则将使用 primary index。

---

### Set indexes make the secondary index hack obsolete

在大型 namespace 中查找小型 set 中所有记录的常见解决方法是在包含 set 名称的 bin 上创建二级索引。使用此技术的用户应考虑切换 set indexes。

1. 您无需使用额外的 bin 来修改记录。
2. 启动时间收到的影响要小的多。
3. Set indexes 的管理更加简单。他只是一个动态配置。
4. 您可以扫描 indexed set 中的所有记录，而无需过滤器。
5. 扫描始终正确，并支持分页。