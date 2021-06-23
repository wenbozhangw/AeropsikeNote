## Collection Data Types

Aerospike 记录有一个或多个 [bins](https://docs.aerospike.com/docs/architecture/data-model.html#bins) 。每个 bin 将包含不同的 [scalar data type](https://docs.aerospike.com/docs/guide/data-types.html) , 例如 Integer or String, a collection data type, 例如 [List](https://docs.aerospike.com/docs/guide/cdt-list.html) or [Map](https://docs.aerospike.com/docs/guide/cdt-map.html), a probabilstic data type, 例如 [HyperLogLog](https://docs.aerospike.com/docs/guide/hyperloglog/) 或 [geospatial](https://docs.aerospike.com/docs/guide/geospatial.html) GeoJSON data type。

[Collection](https://en.wikipedia.org/wiki/Collection__%28abstract_data_type%29) data types (CDTs) 是灵活的(flexible)，无模式的容器(schema-free containers)，可以保存标量数据类型或在其中嵌套其他集合。CDT 中的元素可以是混合类型。

CDT 是 JSON 的超集，支持更多数据类型作为集合元素，例如整数的 Map keys，以及二进制数据的 List values。

```
{ 'scores': { 'ACE': [ 34500,
                       { 'awards': {'🏆': 1},
                         'dt': '1979-04-01 09:46:28',
                         'ts': 291807988156}],
              'CFO': [ 17400,
                       { 'awards': {'🦄': 1},
                         'dt': '2017-11-19 15:22:38',
                         'ts': 1511104958197}],
              'CPU': [9800, {'dt': '2017-12-05 01:01:11', 'ts': 1512435671573}],
              'EIR': [ 18400,
                       {'dt': '2018-03-18 18:44:12', 'ts': 1521398652483}],
              'ETC': [9200, {'dt': '2018-05-01 13:47:26', 'ts': 1525182446891}],
              'SOS': [ 24700,
                       {'dt': '2018-01-05 01:01:11', 'ts': 1515114071923}]},
  'valid': {1: 'a', 2: 'b', 3: 'c', 26: 'z'}}
```

集合带有广泛的 API，用于单记录 (single record) transaction 中执行多个操作，并带有策略来控制 transaction 的任何阶段的错误是否导致回滚、跳到下一个操作，或者在移动到下一个操作之前部分成功。

目前，不能对集合中包含的blob(二进制数据)执行HyperLogLog和按位操作，只能在bin级别上执行。