## User-Defined Functions

User-Defined Function(UDF)是由 Aerospike 数据库服务器内运行的用户（或开发人员）编写的代码。UDF 可以在功能和性能上极大扩展 Aerospike 数据库引擎的功能。Aerospike当前仅支持将 Lua 作为 UDF 语言。

更多请参考, [UDF feature guide](https://docs.aerospike.com/docs/guide/udf.html) 。

### Why Lua ?

与其他脚本语言相比，Lua非常简单且功能强大。

Lua是一种功能强大，快速，轻巧，可嵌入的脚本语言。 Lua将简单的过程语法与基于关联数组和可扩展语义的强大数据描述结构结合在一起。 - [Lua.org](http://www.lua.org/about.html)

Lua已在许多类似的项目中发挥了巨大作用，例如Apache Squid代理中的任意重写规则，Redis和Postgres中的UDF以及其他项目。

Aerospike创建了一个Lua上下文队列 —— 每个线程，每个注册的UDF不超过一个 —— 极大地减少了延迟并提高了性能。

---

### Record UDFs

Record UDF 在单个数据库记录上执行。他们可以创建，更新或删除记录。

Record UDF API提供：
- 访问记录对象，包括其 bins 和 metadata （generation，TTL）
- 轻松处理诸如Map和List之类的复杂数据类型
- 轻松二进制访问Blob data types

Record UDF 在 transaction main flow 中执行。每个 record UDF 都是流的一部分，该流访问数据库中的一行，创建 Lua Record 记录，并调用 Lua 函数以允许UDF进行操作。有时，该记录不存在，这允许UDF创建记录，并允许对记录进行操作。

请参阅 [Record UDFs](https://docs.aerospike.com/docs/guide/record_udf.html) 上的 feature guide 部分。

---

### Stream UDFs

Stream UDFs 对一组记录执行只读操作。Stream UDF 系统允许使用 *MapReduce* 样式的编程，该样式用于常见的高并发的 *MapReduce* 作业，例如 *word count* —— 访问每一行，并发出单词列表及其计数。最终结果是在 reduce 阶段中计算的。对于简单的聚合，简单地在上下文中增加计数器，而不是连续创建和销毁对象，Aerospike提供了最佳实现。

Aerospike Stream UDF 系统与典型的 *MapReduce* 框架之间的重要区别在于 Aerospike Stream UDF 的延迟非常低，并且在 shared-nothing 架构中具有很高的可靠性。

Stream UDF 查询不是协调作业控制(coordinated job control)，而是由发出请求的客户端发送到所有集群节点。它们由每个服务器管理并确定优先级。结果返回到发出请求的客户端，客户端在将结果返回个客户端之前执行最终操作（例如 reduce 或 final聚合）。

尽管这可能会导致客户端使用大量内存，但是服务器集群不会因格式不正确的查询而中断。如果轻量级应用服务器需要发出要求大量最终缩减(large final reduce)的请求，则可以添加中间应用服务器以使用 REST 或 SOAP API。这就像协调代理一样，该服务器可以具有适合应用程序负载的内存配置文件。

请参阅 [Record UDFs](https://docs.aerospike.com/docs/guide/record_udf.html) 上的 feature guide 部分。

---

### Managing UDF Code in a Cluster

UDF 由集群的系统元数据(system metadata, SMD)组件管理。注册 Lua 模块后，它仅发送到一个节点。该节点将请求转发给当前的集群主体，后者将该传入版本与以前的版本进行比较。如果版本较新，它将文件持久保存到本地存储（在用户目录中），并将其发送到其余的群集节点。

在注册期间，文件被解释。这样可以立即检测到简单的分析错误。这些错误由注册API返回。

请参阅 [Managing UDFs](https://docs.aerospike.com/docs/udf/managing_udfs.html) 。

---

### Invoking UDFs

UDF 被组织成模块，其中模块是一个包含在数据库中注册的 Lua 代码的文件。要客户端调用 UDF，开发人员将
1. 指定UDF模块名称。
2. 在要执行的模块中指定函数名称。
3. 指定User-Defined Function的参数（可选）。

#### Multicore

许多其他解释性语言每个进程只能运行一个执行线程。例如，CPython其代码库中使用全局变量，这大大降低了每个进程运行多个Python上下文的能力。

#### Gateway to C

Lua代码可以直接调用C函数。尽管这样做的开销是可以测量的，但这很简单。 Aerospike提供了以下示例：编译和注册共享的C对象，然后直接从Lua UDF模块调用C函数之一。

请参阅 [Using C Functions in UDF Modules](https://docs.aerospike.com/docs/udf/developing_lua_modules.html#using-c-functions-in-udf-modules) 。

---

### Protection and Sandboxing 

Aerospike UDF在过程中运行。这样可以最大限度地提高性能，但是编写不正确的 UDF 可能会导致性能问题。Aerospike 提供了防止 UDF 中类似错误的功能。通过限制在 UDF 中花费的时间来防止无限循环。

---

### Stored Procedures vs. UDFs

存储过程通常在数据库系统中使用。存储过程与 UDF 相似，因为它们是一个用户应用程序，存储在关系型数据库管理系统（RDBMS）中并在其中运行。存储过程可以读取或写入一个或多个记录，通用机制也是如此。

UDF受到更多限制。它们对单个记录 (**Record UDFs**) 或选定的记录流 (**Stream UDFs**) 进行操作。Record UDF 的行为类似于传统 UDF。Stream UDF 的行为有点像存储过程 ———— 它们管理多个记录。