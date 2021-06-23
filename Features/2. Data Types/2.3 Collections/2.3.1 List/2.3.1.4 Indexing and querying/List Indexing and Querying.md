## List Indexing and Querying

Aerospike 允许用户在数据类型为 list 的 bin
上创建 [secondary index](https://docs.aerospike.com/docs/guide/query.html#secondary-index-key-data-type) 。

---

### Indexing on List Elements

- 与基本索引类型，可索引的列表元素数据类型是 numeric, string, and GeoJSON。
- List 索引仅针对顶级元素，而不是嵌套元素。
- 创建索引时，用户需要明确指定应该对 list bin 进行索引，以及要索引的数据类型。
- 查询时，用户需要指定查询应用 CDT 数据类型。
- 与基本 [querying](https://docs.aerospike.com/docs/guide/query.html) 类似，支持 equality, range（for numeric and string data
  type）, points-within-region, region-containing-points (for GeoJSON data type)。

在本例中，我们使用 [aql](https://docs.aerospike.com/docs/tools/aql) 工具创建两个源类型为 LIST 的索引，一个用于数字数据类型的值，一个用于字符串数据类型的值。

```
CREATE LIST INDEX foo_list_ints ON test.demo (foo) NUMERIC
CREATE LIST INDEX foo_list_strs ON test.demo (foo) STRING
```

索引 list 的元素经过类型检查，因此 foo bin 包含 `[ 1, "2", 3, [4], 5 ]` 的记录会产生以下索引：

| Index On | Key Type | Index Type | Eligible Secondary Index Key |
| --- | --- | --- | --- |
| foo | string | LIST | "2" |
| foo | numeric | LIST | 1, 3, 5 |

---

#### List Index Queries

以下示例使用 aql 工具查询索引 list。

```
aql> INSERT INTO test.demo (PK, username, emails) VALUES ("u1", "Bob Roberts", LIST('["bob.roberts@gmail.com", "bob@yahoo.com"]'))
OK, 1 record affected.

aql> INSERT INTO test.demo (PK, username, emails) VALUES ("u2", "rocketbob", LIST('["bigb@gmail.com", "bob@yahoo.com"]'))
OK, 1 record affected.

aql> INSERT INTO test.demo (PK, username, emails) VALUES ("u3", "samunwise", "pppreciousss@gmail.com")
OK, 1 record affected.

aql> CREATE LIST INDEX email_idx ON test.demo (emails) STRING
OK, 1 index added.

aql> show indexes
+--------+----------+-----------+--------+-------+-------------+----------+----------+
| ns     | bin      | indextype | set    | state | indexname   | path     | type     |
+--------+----------+-----------+--------+-------+-------------+----------+----------+
| "test" | "emails" | "LIST"    | "demo" | "RW"  | "email_idx" | "emails" | "STRING" |
+--------+----------+-----------+--------+-------+-------------+----------+----------+
[127.0.0.1:3000] 1 row in set (0.001 secs)

OK

aql> SELECT username FROM test.demo IN LIST WHERE emails = "bigb@gmail.com"
+-------------+
| username    |
+-------------+
| "rocketbob" |
+-------------+
1 row in set (0.001 secs)

OK

aql> SELECT username FROM test.demo IN LIST WHERE emails = "bob@yahoo.com"
+---------------+
| username      |
+---------------+
| "Bob Roberts" |
| "rocketbob"   |
+---------------+
2 rows in set (0.001 secs)

OK

aql> SELECT username FROM test.demo IN LIST WHERE emails = "pppreciousss@gmail.com"
0 rows in set (0.001 secs)

OK

aql> SELECT username FROM test.demo WHERE emails = "pppreciousss@gmail.com"
0 rows in set (0.001 secs)

Error: (201) AEROSPIKE_ERR_INDEX_NOT_FOUND
```

---

### Known Limitations

-  List 索引仅适用于顶级元素。嵌套列表上没有索引。
- 在 list 上使用范围查询时，如果 list 包含多个落在范围内的值，则可以多次返回记录。

