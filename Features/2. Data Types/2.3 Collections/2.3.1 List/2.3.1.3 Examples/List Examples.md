## List Examples

Aerospike [lists](https://docs.aerospike.com/docs/guide/cdt-list.html) 可用于实现用例，例如：
- Queues
- Unique ordered data
- Time series containers
- Leaderboards
- Membership lists for user profiles

每个 list [operations](https://docs.aerospike.com/docs/guide/cdt-list-ops.html) 都有详细的代码示例说明。

---

### Modeling Concepts

Aerospike list 可用于在单个记录中表达多对一的关系。通过对关系数据库进行比较，这种关系将跨越两个表中的多行。

例如，用户和他的车辆可能在关系数据库中表示为用户表和车辆表，其中 _vehicles.user_id_ 列被索引并声明为用户的外键。

在 Aerospike 中，用户 set 将包含用户记录。每个用户的车辆可以表示为一个 map 元素列表，每个 map 元素包含一个车辆的信息。没有任何车辆的用户根本不会有车辆的bin。在 Aerospike 中，set 是无模式(no schema)的，记录的结构可以彼此不同。没有必要在关系型数据库中使用诸如 `NULL` 之类的东西来放置车辆。

---

### Example

#### List as Queues

我们有一个无序列表 `[ 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233 ]` 。

使用 [aql](https://docs.aerospike.com/docs/tools/aql) 工具，我们将使用两个 operations 的 transaction 一次将量新元素加入到队列。

```
aql> OPERATE LIST_APPEND(lqueue, 377), LIST_APPEND(lqueue,477) on test.demo where PK = 'lk1'
+---------+
| lqueue |
+---------+
| 16      |
+---------+
1 row in set (0.001 secs)

OK

aql>  select * from test.demo where PK='lk1'
+-----------------------------------------------------------------------+
| lqueue                                                               |
+-----------------------------------------------------------------------+
| LIST('[0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 477]') |
+-----------------------------------------------------------------------+
1 row in set (0.000 secs)
```

然后我们将使用一个原子列表操作将前两个值出列。

```
OPERATE LIST_REMOVE_RANGE (lqueue, 0, 2) on test.demo where PK = 'lk1')
+---------+
| lqueue |
+---------+
| 2       |
+---------+
1 row in set (0.000 secs)

OK

aql> select * from test.demo where PK='lk1'
+-----------------------------------------------------------------+
| lqueue                                                         |
+-----------------------------------------------------------------+
| LIST('[1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 477]') |
+-----------------------------------------------------------------+
1 row in set (0.001 secs)
```

我们将使用单个 transaction 将记录入队，同时使队列中的第一条记录出队。

```
aql> OPERATE LIST_APPEND(lqueue, 125), LIST_REMOVE_RANGE (lqueue, 0, 1) on test.demo where PK = 'lk1'
+---------+
| lqueue |
+---------+
| 1       |
+---------+
1 row in set (0.000 secs)

OK

aql> select * from test.demo where PK='lk1'
+-------------------------------------------------------------------+
| lqueue                                                           |
+-------------------------------------------------------------------+
| LIST('[2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 477, 125]') |
+-------------------------------------------------------------------+
1 row in set (0.000 secs)
```

#### Lists as Containers for Time Series Data

使用 aql 工具，我们将创建一个包含 element pairs 列表的记录。

```
aql> insert into test.demo (PK, pairs) values ('ll2', 'JSON[[1523474230000, 39.04],[1523474231001, 39.78],[1523474236006, 40.07],[1523474235005, 41.18],[1523474233003, 40.89],[1523474234004, 40.93] ]')
OK, 1 record affected.

aql> select * from test.demo where PK='ll2'
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| pairs                                                                                                                                                |
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| LIST('[[1523474230000, 39.04], [1523474231001, 39.78], [1523474236006, 40.07], [1523474235005, 41.18], [1523474233003, 40.89], [1523474234004, 40.93]]') |
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.002 secs)
```

我们将使用 Java 客户端获取指定时间戳值范围内的 pairs

```java
Record record = client.operate(new WritePolicy(), key,
                               ListOperation.getByValueRange(binName,
                                      new ListValue(new ArrayList<>(Arrays.asList(1523474231000L, null))),
                                      new ListValue(new ArrayList<>(Arrays.asList(1523474234000L, null))),
                                      ListReturnType.VALUE));

List<?> list = record.getList(binName);

for (Object value : list) {
    System.out.println(value);
    //console.info("Received: " + value);
}
```

输出

```
[[1523474231001, 39.78]
[1523474233003, 40.89]
```