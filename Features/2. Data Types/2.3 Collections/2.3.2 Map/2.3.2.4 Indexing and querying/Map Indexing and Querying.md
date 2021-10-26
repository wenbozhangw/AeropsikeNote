## Map Indexing and Querying

Aerospike 允许用户在数据类型为 map 的 bin 上创建 [secondary index](https://docs.aerospike.com/docs/guide/query.html#secondary-index-key-data-type) 。map key 或 map value 都可以被索引。

### Indexing on Map Elements

- 与基本索引类似，可索引的 map 数据类型是 numeric, string, and GeoJSON.
- Map 索引仅针对顶级元素，而不是嵌套元素。
- 创建索引时，用户需要明确指定应该对 map bin 进行索引，以及要索引的数据类型。
- 查询时，用户需要制定查询应用于 CDT 数据类型。
- 与基本查询类似，支持 equality, range (for numeric and string data type), points-within-region and region-containing-points(for geoJSON data type)。

在本例中，我们使用 [aql](https://docs.aerospike.com/docs/tools/aql) 工具创建了两个索引，一个是源类型 `MAPKEYS` 和字符串类型，一个是源类型 `MAPVALUES`  和数字类型：

```
CREATE MAPKEYS INDEX foo_mapkey_idx ON test.demo (foo) STRING
CREATE MAPVALUES INDEX foo_mapval_idx ON test.demo (foo) NUMERIC
```

索引 list 的元素经过类型检查，因此 foo bin 包含 `{ a:1, b:"2", c:3, d:[4], e:5, 666:"zzz" }` 的记录将产生以下索引：

| Index On | Key Type | Index Type | Eligible Secondary Index Key |
| --- | --- | --- | --- |
| foo | string | MAPKEYS | a, b, c, d, e |
| foo | numeric | MAPVALUES | 1, 3, 5 |

---

### Map Index Queries

此示例说明如何使用 aql 脚本使用 map 索引。该实例在 bin `shopping_carts` 上创建索引，该索引是产品和订购它的状态的 map，并查询基于产品名称或状态的记录。

```
aql> CREATE MAPKEYS INDEX foo_mapkey_idx ON test.demo (foo) STRING
OK, 1 index added.

aql> CREATE MAPVALUES INDEX foo_mapval_idx ON test.demo (foo) NUMERIC
OK, 1 index added.

aql> show indexes
+--------+-------+-------------+--------+-------+------------------+-------+-----------+
| ns     | bin   | indextype   | set    | state | indexname        | path  | type      |
+--------+-------+-------------+--------+-------+------------------+-------+-----------+
| "test" | "foo" | "MAPKEYS"   | "demo" | "RW"  | "foo_mapkey_idx" | "foo" | "STRING"  |
| "test" | "foo" | "MAPVALUES" | "demo" | "RW"  | "foo_mapval_idx" | "foo" | "NUMERIC" |
+--------+-------+-------------+--------+-------+------------------+-------+-----------+
[127.0.0.1:3000] 2 rows in set (0.001 secs)

OK

aql> INSERT INTO test.demo (PK, username, foo) VALUES ("u1", "Bob Roberts", JSON('{"a":1, "b":"2", "c":3, "d":[4]}'))
OK, 1 record affected.

aql> INSERT INTO test.demo (PK, username, foo) VALUES ("u2", "rocketbob", MAP('{"c":3, "e":5}'))
OK, 1 record affected.

aql> INSERT INTO test.demo (PK, username, foo) VALUES ("u3", "samunwise", JSON('{"x":{"z":26}, "y":"yyy"}'))
OK, 1 record affected.

aql> SELECT username FROM test.demo IN MAPVALUES WHERE foo = 2
0 rows in set (0.001 secs)

OK

aql> SELECT username FROM test.demo IN MAPVALUES WHERE foo = "2"
0 rows in set (0.000 secs)

Error: (201) AEROSPIKE_ERR_INDEX_NOT_FOUND

aql> SELECT username FROM test.demo IN MAPVALUES WHERE foo = 1
+---------------+
| username      |
+---------------+
| "Bob Roberts" |
+---------------+
1 row in set (0.001 secs)

OK

aql> SELECT username FROM test.demo IN MAPVALUES WHERE foo = 3
+---------------+
| username      |
+---------------+
| "Bob Roberts" |
| "rocketbob"   |
+---------------+
2 rows in set (0.001 secs)

OK

aql> SELECT username FROM test.demo IN MAPVALUES WHERE foo BETWEEN 1 AND 5
+---------------+
| username      |
+---------------+
| "Bob Roberts" |
| "Bob Roberts" |
| "rocketbob"   |
| "rocketbob"   |
+---------------+
4 rows in set (0.001 secs)

aql> SELECT username FROM test.demo IN MAPKEYS WHERE foo = "y"
+-------------+
| username    |
+-------------+
| "samunwise" |
+-------------+
1 row in set (0.001 secs)

OK

aql> SELECT username FROM test.demo IN MAPKEYS WHERE foo = "x"
+-------------+
| username    |
+-------------+
| "samunwise" |
+-------------+
1 row in set (0.001 secs)

OK

aql> SELECT username FROM test.demo IN MAPKEYS WHERE foo = "c"
+---------------+
| username      |
+---------------+
| "Bob Roberts" |
| "rocketbob"   |
+---------------+
2 rows in set (0.001 secs)

OK

aql> SELECT username FROM test.demo IN MAPKEYS WHERE foo = "z"
0 rows in set (0.001 secs)

OK
```

aql工具不允许使用数组 map key，因此本示例中不使用这些 key。但是，这不是 Aerospike 服务器或 Java、Python、Go 等 Aerospike 客户端的限制。

---

### Known Limitations

- Map 索引仅适用于顶级元素。嵌套元素没有索引。
- 在 map 上使用范围查询时，如果 map 包含多个落在范围内的值，则可以多次返回记录。