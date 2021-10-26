## Context for Operations on Nested Elements

Collection data type (CDT) 上下文允许针对嵌套在 [List](https://docs.aerospike.com/docs/guide/cdt-list.html) 或 [Map](https://docs.aerospike.com/docs/guide/cdt-map.html) 中的元素进行操作。

- 集合顶层的元素不需要访问上下文。
- 上下文标识您希望操作的元素的路径。
- 当所描述的路径不完整时，您还可以使用上下文创建元素。
- 在 Aerospike 数据库 4.6 版中添加了 CDT 上下文功能。

##### Contents

- [CDT Context API](#cdt-context-api)
- [Context Type](#context-type)
  - [Language-Specific Client APIs](#language-specific-client-apis)
  - [Context Example - Drilling Down](#context-example-drilling-down)
- [Operation Examples](#operations-examples)
  - [Examples with List append()](#examples-with-list-append)
  - [Examples for Selecting a List Element by Value](#examples-for-selecting-a-list-element-by-value)
- [Creating a Context](#creating-a-context)

---

### <span id="cdt-context-api"> CDT Context API </span>

下面以通用术语描述 CDT Context API。每种语言客户端可能使用略有不同的术语来表达相同的概念，例如 Java 客户端的 [`CTX` class](https://www.aerospike.com/apidocs/java/com/aerospike/client/cdt/CTX.html) 。List 和 Map API 中的大多数操作都采用可选的上下文参数。

Context 是针对特定元素的 list of selector pairs，该元素嵌套在 List 或 Map 中。每个元素选择器都包含一个 context type 和一个 value。

---

### <span id="context-type"> Context Type </span>

这指定了元素选择器的类型，它将从 list 的顶层应用，或上下文中上一步到达的点。

- `BY_LIST_INDEX(index)`
- `BY_LIST_RANK(rank)`
- `BY_LIST_VALUE(value)`
- `BY_MAP_INDEX(index)`
- `BY_MAP_RANK(rank)`
- `BY_MAP_KEY(key)`
- `BY_MAP_VALUE(value)`

Aerospike 4.9 版本中添加了两个新的 Context types，如果元素不存在，可以创建一个元素，然后选择它。这类似于
```shell
mkdir -p /creates/the/path/to/a/sub/directory
```

- `MAP_KEY_CREATE(key)`
- `LIST_INDEX_CREATE(index)`


#### <span id="language-specific-client-apis"> Language-Specific Client APIs </span>

[Java](https://www.aerospike.com/apidocs/java/com/aerospike/client/cdt/CTX.html) |
[C](https://www.aerospike.com/apidocs/c/dc/dd1/structas__cdt__ctx.html) |
[C#](https://www.aerospike.com/apidocs/csharp/html/AllMembers_T_Aerospike_Client_CTX.htm) | 
[Go](https://godoc.org/github.com/aerospike/aerospike-client-go#CDTContext) | 
[Python](https://aerospike-python-client.readthedocs.io/en/latest/aerospike_helpers.cdt_ctx.html) | 
[Node.js](https://www.aerospike.com/apidocs/nodejs/CdtContext.html) | 
PHP | 
Ruby | 
REST | 
[Rust](https://docs.rs/aerospike/0.6.0/aerospike/operations/cdt_context/index.html)

#### <span id="context-example-drilling-down"> Context Example - Drilling Down </span>

考虑以下 list ：

```
[0, 1, [2, [3, 4], 5, 6], 7, [8, 9]]
```

我们可以通过使用以下方法识别操作的上下文来对 List 元素 `[3, 4]` 进行操作
```
[BY_LIST_INDEX(2), BY_LIST_INDEX(1)]
```

- 第一个上下文类型 `BY_LIST_INDEX(2)` 选择顶级 List 的第三个元素。它选择的元素是 `[2, [3, 4], 5, 6]`。
- 第二个上下文类型 `BY_LIST_INDEX(1)` 选择索引位置 1 的元素。它选择的元素是 List `[3, 4]`。
- 现在可以将 List API 操作应用于此嵌套 List 中的元素之一。

---

### <span id="operation-examples"> Operation Examples </span>

#### <span id="examples-with-list-append"> Examples with List append() </span>

在每个示例中，我们将从一个 bin 'l' 开始，其中包含一个 List 值
```
[0, 1, [2, [3, 4], 5, 6], 7, [8, 9]]
```

这个列表可以被形象化为
```
[0, 1, [               ], 7, [    ]]  # top level (depth 0)
        2, [    ], 5, 6       8, 9    # depth 1 nested elements
            3, 4                      # depth 2 nested elements
```

在顶层我们有五个元素，其中三个是标量整数值，其中两个是 List 值。

索引位置 2 处的 List 值有四个元素 - Integer 值 2、一个 List 元素，然后是 Integer 值 5 和 6.它的索引位置 1 处的 List 元素有两个元素，即整数 3 和 4。

[List `append()`](https://docs.aerospike.com/docs/guide/operations/list/append.html) 操作将单个 item 添加到 List 的末尾。我们将在接下来的几个示例中使用它。

---

##### Append an item to a List at the last element

将值 100 附加到嵌套在顶层最后一个元素的 List 中。
```
# [0, 1, [2, [3, 4], 5, 6], 7, [8, 9]]
list_append('l', 100, context=[BY_LIST_INDEX(-1)])
```

**Result:**
```
[0, 1, [2, [3, 4, 100], 5, 6], 7, [8, 9, 100]]
```

没有上下文，我们将附加到 list 列表。
```
# [0, 1, [2, [3, 4], 5, 6], 7, [8, 9]]
list_append('l', 100)
```

**Result:**
```
[0, 1, [2, [3, 4], 5, 6], 7, [8, 9], 100]
```

---

##### Append an item to a non-List element

让我们将值 100 附加到索引位置 0 的 List 元素。提示：索引 0 处有一个整数值。
```
# [0, 1, [2, [3, 4], 5, 6], 7, [8, 9]]
list_append('l', 100, context=[BY_LIST_INDEX(0)])
# Error 26 and no change [0, 1, [2, [3, 4], 5, 6], 7, [8, 9]]
```

**Error:**
```
Error code 26 OP_NOT_APPLICABLE
```

这是因为 List 上下文类型需要标识一个 List 元素，而 Map 上下文类型需要标识一个 Map 元素。

---

##### Append an item to depth 2

让我们将值 100 附加到深度为 2 的最深的 List 中。这意味着我们需要一个具有两种上下文类型的 Context 来选择该 List 的方式。
```
# [0, 1, [2, [3, 4], 5, 6], 7, [8, 9]]
list_append('l', 100, context=[BY_LIST_INDEX(2), BY_LIST_INDEX(1)])
```

**Result:**
```
[0, 1, [2, [3, 4, 100], 5, 6], 7, [8, 9]]
```

---

#### <span id="examples-for-selecting-a-list-element-by-value">Examples for Selecting a List Element by Value </span>

在接下来的例子中，我们将从一个 bin 'l' 开始，它包含一个元组列表，每个元组表示为一个 List 元素。
```
[[1, 'a'], [2, 'b'], [4, 'd'], [2, 'bb']]
```

这个列表可以被形象化为
```
[[      ], [      ], [      ], [       ]]  # top level (depth 0)
  1, 'a'    2, 'b'    4, 'd'    2, 'bb'    # depth 1 nested elements
```

##### Append a Map to the first tuple starting with 2

我们将使用 [wildcard](https://docs.aerospike.com/docs/guide/cdt-ordering.html#wildcard) 按值选择元素。[List `get_all_by_value`](https://docs.aerospike.com/docs/guide/operations/list/get_all_by_value.html)  返回每个匹配项，上下文类型 `BY_LIST_VALUE` 将返回第一个，具体取决于 List 中元素的升值排序。
```
# [[1, 'a'], [2, 'b'], [4, 'd'], [2, 'bb']]
list_append('l', {3: {'c': '👍'}}, context=[BY_LIST_VALUE([2, *])])
```

**Result:**
```
[[1, 'a'], [2, 'b', {3: {'c': '👍'}}], [4, 'd'], [2, 'bb']]
```

请注意，这不是有效的 JSON，但它是有效的 Aerospike 集合。 Aerospike Maps 接受整数映射键（以及字符串、二进制数据（字节）和双精度值）。

在 List 上执行“按值”操作时，Aerospike 使用值顺序。如果这是一个无序列表，这个值的顺序是即时计算的。在这种情况下，在 List `append()` 操作之前的元组的值顺序是
```
[[1, 'a'], [2, 'b'], [2, 'bb'], [4, 'd']]
```

---

##### Disambiguate a 'by List value' selection

在前面的示例中，我们看到了按 List 值进行的选择可能会产生歧义。在下面的例子中，我们得到了更多的特殊。我们将从嵌套 Map 中读取 key 'c' 的值，方法是 [Map `get_by_key()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html) 操作提供一个按值选择 List 元素的上下文，然后通过 key 3 获取内部 Map。
```
# [[1, 'a'], [2, 'b', {3: {'c': '👍'}}], [4, 'd'], [2, 'bb']]
map_get_by_key('l', 'c', context=[BY_LIST_VALUE([2, 'b', *]), BY_MAP_KEY(3)])
```

**Result:**
```
'👍'
```

---

### <span id="creating-a-context"> Creating a Context </span>

有时，您要对其执行操作的上下文不存在。在 Aerospike 4.9 版之前，您需要首先创建每个元素，通常使用 `CREATE_ONLY | NO_FAIL` 写标志组合。从 4.9 版开始，新的 `MAP_KEY_CREATE` 上下文类型简化了这个过程。

在下面的例子中，我们想为演员的统计数据添加荣誉，accolades 是一个可以包含任意数据的 Map。数据在一个 bin `m`
```
{'name': 'chuck norris'}
```

如果我们想 Map 将 jokes 的赞誉增加 317，而 'accolades' 和 'jokes' 都不存在怎么办？
如果我们不确定这种方式是否存在，并希望操作总是成功怎么办？
```
map_increment('m', 'jokes', 317, context=[MAP_KEY_CREATE('stats'), MAP_KEY_CREATE('accolades')])
```

**Result:**
```
{'name': 'chuck norris', 'stats': {'accolades': {'jokes': 317}}}
```

在 Aerospike 4.9 版之前，您需要通过利用 [Map `put()`](https://docs.aerospike.com/docs/guide/cdt-map-ops.html) 在 increment 操作之前创建上下文元素来创建 multi-operation transaction 。
```
ops = [
    map_put('m', 'stats', {}, CREATE_ONLY|NO_FAIL),
    map_put('m', 'accolades', {}, CREATE_ONLY|NO_FAIL),
    map_increment('m', 'jokes', 317,
              context=[MAP_KEY_CREATE('stats'), MAP_KEY_CREATE('accolades')]),
]
```

**Result:**
```
{'name': 'chuck norris', 'stats': {'accolades': {'jokes': 317}}}
```