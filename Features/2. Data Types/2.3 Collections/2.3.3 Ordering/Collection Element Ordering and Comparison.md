## Collection Element Ordering and Comparison

### Type Order (Ascending)

具有不同类型的元素根据其类型进行排序。

1. NIL
2. BOOLEAN
3. INTEGER
4. STRING
5. LIST
6. MAP
7. BYTES
8. DOUBLE
9. GEOJSON
10. INF

---

#### NIL

最低值的类型。 NIL 是一个单例。

---

#### BOOLEAN

`False < Ture`

---

#### INTEGER

按整数值排序。

---

#### STRING

按字符串中的每个字节排序。

`"aa" < "b"`

假定字符串具有 UTF-8 编码。

---

#### LIST

Order by:

1. 每个元素从索引 0 `[1, 2] < [1, 3]` 开始
2. 元素计数 `[1, 2] < [1, 2, 1]`

存在一个已知问题，即对于不同长度的 map/list，map/list 比较可能不正确。如果计划使用 comparisons，请更新到 Aerospike 4.3.1 或更高版本。

---

#### MAP

Order by:

1. 元素计数
2. 每个 key 按顺序存储

存在一个已知问题，即对于不同长度的 map/list，map/list 比较可能不正确。如果计划使用 comparisons，请更新到 Aerospike 4.3.1 或更高版本。

---

#### BYTES

按字符串中的每个字节排序。

---

#### DOUBLE

按浮点值排序。

---

#### INF

在 Aerospike 数据库版本 4.3.1 中引入。

值最高的类型。 INF 是一个单例。

不是存储类型。将 INF 存储在 bin 列表或 map 中具有未定义的行为。

---

### Comparison

#### Wildcard

单例 WILDCARD(*) 类型可以在某些操作中作为参数传递。 WILDCARD 不是存储类型。

Bin: `[ [1, 1], [1, 2], [1, 3], [2, 1], [2, 2], [2, 3] ]`

```
list_get_all_by_value([1, *]) -> [ [1, 1], [1, 2], [1, 3] ]
```
在 Aerospike 数据库版本 4.3.1 中引入。

---

#### Intervals

默认情况下，间隔(intervals)是包含-不包含(inclusive-exclusive)的 : start <= elements < end

Bin: `[ [1, 1], [1, 2], [2, 1], [2, 2], [3, 1] ]`

```
list_get_by_value_interval(start=[1, NIL], end=[2, NIL]) -> [ [1, 1], [1, 2] ]
```

---

### INF

使用 INF，我们可以在使用 2nd level 列表时获得包含-包含间隔(inclusive-inclusive interval)。

```
list_get_by_value_interval(start=[1, NIL], end=[2, INF]) -> [ [1, 1], [1, 2], [2, 1], [2, 2] ]
```

在 Aerospike 数据库版本 4.3.1 中引入。