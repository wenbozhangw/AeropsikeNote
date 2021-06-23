## Context for Operations on Nested Elements

Collection data type (CDT) 上下文允许针对嵌套在 [List](https://docs.aerospike.com/docs/guide/cdt-list.html) 或 [Map](https://docs.aerospike.com/docs/guide/cdt-map.html) 中的元素进行操作。

- 集合顶层的元素不需要访问上下文。
- 上下文标识您希望操作的元素的路径。
- 当所描述的路径不完整时，您还可以使用上下文创建元素。
- 在 Aerospike 数据库 4.6 版中添加了 CDT 上下文功能。

##### Contents

- [CDT Context API]()