## Map Examples

Aerospike [maps](https://docs.aerospike.com/docs/guide/cdt-map.html) 可用于实现以下用例：

- Event History Containers
- Document Store
- Leaderboards

### Modeling Concepts

嵌套 list 和 map 可以结合机器为复杂的用例建模。在下面的示例中，伪代码在 [map operations](https://docs.aerospike.com/docs/guide/cdt-map-ops.html) 中进行了描述。Aerospike 的每种语言客户端，例如 Java 或 Python，都将具有与这些伪代码相同的方法或功能。

Aerospike 支持 multi-operation transactions，这些 transactions 在单个记录的一个或多个 bin 上按顺序执行（在记录加锁情况下）。 在所有语言客户端中，这样的 transactions 都是使用 `operate()` 方法调用的。

从 Aerospike 4.6.0 版开始，list 和 map 操作可以应用于 [deeply nested structures](https://docs.aerospike.com/docs/guide/cdt-context.html) 。

熟悉其他 NoSQL 文档存储的开发人员将看到在具有嵌套 CDT 的 bin 上使用 map 和 list 操作的灵活性。结合（operate）single record transactions，Aerospike 提供了强大的功能来对各种用例进行建模。

---

### Examples

#### Event Containers with Unique Timestamps

在这个例子中，我们要存储和查询用户事件数据。每条记录都包含特定用户最近的 N 个事件，以该用户的唯一标识作为 key 。

假设事件不会在同一毫秒发生，我们将使用毫秒时间戳作为不同事件的 map key。每个事件的数据将是一个元组 `[ event-type, { attr1: v1, attr2: v2, ... } ]` 。

我们的示例数据将是单个用户的以下事件：

```
{
  1523474230000: ['fav',     {'sku':1, 'b':2}],
  1523474231001: ['comment', {'sku':2, 'b':22}],
  1523474236006: ['viewed',  {'foo':'bar', 'sku':3, 'zz':'top'}],
  1523474235005: ['comment', {'sku':1, 'c':1234}],
  1523474233003: ['viewed',  {'sku':3, 'z':26}],
  1523474234004: ['viewed',  {'sku':1, 'ff':'hhhl'}]
}
```

##### Retrieving data for specific event types

我们将使用 `get_all_by_value` map操作检索特定事件类型的所有事件：

```
get_all_by_value(['comment', *]) 
{
  1523474231001: ['comment', {'sku':2, 'b':22}],
  1523474235005: ['comment', {'sku':1, 'c':1234}]
}
```

该操作的参数是元组 `['comment', *]`。 [wildcard singleton](https://docs.aerospike.com/docs/guide/cdt-ordering.html#wildcard) (`*`) 充当匹配零个或多个元素的 glob。由于 `get_all_by_value` 将不同的值相互比较（这不是范围操作），因此所有以 `'comment'` 开头的元组作为它们的第一个元素都匹配。

这种建模方法利用了 Aerospike list 相互比较的方式。我们目前只能依靠它来识别第一个元素是特定值的 list，然后是任意数量的 'trivial' 属性，我们无法按值查询这些属性。

扩展该示例，我们将检索多种事件类型的所有事件：

```
get_all_by_value_list([['comment', *], ['fav', *]])
{
  1523474230000: ['fav',     {'sku':1, 'b':2}],
  1523474231001: ['comment', {'sku':2, 'b':22}],
  1523474235005: ['comment', {'sku':1, 'c':1234}]
}
```

##### Counting Events

我们将通过指定 `resultType=count` 来获取针对上述示例数据的特定事件类型的计数。默认行为是 `resultType = KeyValue`：
```
get_all_by_value(['viewed', *], resultType=count) 
3
get_all_by_value(['comment', *], resultType=count) 
2
```

##### Trimming the Map

通常，我们希望限制 map 中事件的数量。我们将使用 `remove_by_index_range` map操作来保留最近的 1000 个事件：

```
remove_by_index_range(-1000, 1000, INVERTED, resultType=none)
```


#### Event Containers with Unique UUIDS

在某些用例中，时间戳会导致频繁的冲突。我们可以使用唯一标识符作为map key进行建模。

在此示例中，对话线程存储在一个单记录中。map key 是 message UUID，map value 是 list 元组 `[timestamp, msg-string, username]` 。

我们的示例数据将是单个对话线程中的消息：

```
{
  '0edf5b73-535c-4be7-b653-c0513dc79fb4': [1523474230, "Billie Jean is not my lover", "MJ"],
  '29342a0b-e20f-4676-9ecf-dfdf02ef6683': [1523474241, "She's just a girl who", "MJ"],
  '31a8ba1b-8415-aab7-0ecc-56ee659f0a83': [1523474245, "claims that I am the one", "MJ"],
  '9f54b4f8-992e-427f-9fb3-e63348cd6ac9': [1523474249, "...", "Tito"],
  '1ae56b18-7a3c-4f64-adb7-2e845eb5094e': [1523474257, "But the kid is not my son", "MJ"],
  '08785e96-eb1b-4a74-a767-7b56e8f13ea9': [1523474306, "ok...", "Tito"],
  '319fa1a6-0640-4354-a426-10c4d3459f0a': [1523474316, "Hee-hee!", "MJ"]
}
```

我们将使用 `get_by_value_interval` map操作检索时间戳内的所有消息：

```
get_by_value_interval([1523474240, nil], [1523474246, nil])
{
  '29342a0b-e20f-4676-9ecf-dfdf02ef6683': [1523474241, "She's just a girl who", "MJ"],
  '31a8ba1b-8415-aab7-0ecc-56ee659f0a83': [1523474245, "claims that I am the one", "MJ"]
}
```

该操作的参数是 `[timestamp, nil]` 的最小和最大元素。 [NIL singleton](https://docs.aerospike.com/docs/guide/cdt-ordering.html) 的 value 低于字符串。`get_by_value_interval` 检查每个 map value （一个 list） 是否在操作的两个 list 参数之间。

通过 [interval comparison rules](https://docs.aerospike.com/docs/guide/cdt-ordering.html#intervals) 评估以下内容：

```
[1523474240, nil] ≰ [1523474230, "Billie Jean is not my lover", "MJ"] < [1523474246, nil] (false)
[1523474240, nil] ≤ [1523474241, "She's just a girl who", "MJ"]       < [1523474246, nil] (true)
[1523474240, nil] ≤ [1523474245, "claims that I am the one", "MJ"]    < [1523474246, nil] (true)
[1523474240, nil] ≤ [1523474249, "...", "Tito"]                       ≮ [1523474246, nil] (false)
[1523474240, nil] ≤ [1523474257, "But the kid is not my son", "MJ"]   ≮ [1523474246, nil] (false)
[1523474240, nil] ≤ [1523474306, "ok...", "Tito"]                     ≮ [1523474246, nil] (false)
[1523474240, nil] ≤ [1523474316, "Hee-hee!", "MJ"]                    ≮ [1523474246, nil] (false)
```

同样，这种建模方法利用了 Aerospike list 相互比较的方式。我们目前只能依靠它来识别第一个元素在指定值范围内的 list。无法按值查询后续的 `trivial` 属性。