## Scalar Data Types

Aerospike 记录具有一个或多个 [bins](https://docs.aerospike.com/docs/architecture/data-model.html#bins) 。每个 bin 将保存不同的标量数据类型(scalar data type)(例如 Integer 或 String)，集合数据类型(collection data type)(例如 [List](https://docs.aerospike.com/docs/guide/cdt-list.html) 或 [Map](https://docs.aerospike.com/docs/guide/cdt-map.html) )，概率数据类型(probabilistic data type)(例如 [HyperLogLog](https://docs.aerospike.com/docs/guide/hyperloglog/) )，或地理空间 GeoJSON 数据类型( [geospatial](https://docs.aerospike.com/docs/guide/geospatial.html) GeoJSON data type )。

以下是标量数据类型：

- [Integer](#integer)
- [Double](#double)
- [String](#string)
- [Boolean](#boolean)
- [Blob/Bytes](#blob-bytes)
- [Language-Specific Serialized Blobs](#language-specific-serialized-blobs)

---

### <span id="integer"> Integer </span>

Integers 是 64 位数值。

- 每个 integer 需要 8 bytes（64 bits）的存储空间。
- 对于数据库中的 触点(touchpoints)，其中整数需要假设为有符号，例如二级索引，该整数是有符号的，范围为 `[ −(2^63) to 2^63 − 1 ]` 。
- 原子的 `increment` 运算可以应用于整数值。这可以用来实现计数器。
- 对于 equality 和 range queries，整数数据类型均支持[Secondary indexes](https://docs.aerospike.com/docs/architecture/secondary-index.html) 。

---

### <span id="double"> Double </span>

double 数据类型以 64 位 IEEE-745 格式存储。

- 原子的 `increment` 运算可以应用于 double 。
- 支持在Aerospike版本3.6.0中 add double数据类型。

---

### <span id="string"> String </span>

字符串是不透明的字节数组。它们未绑定编码，因此您可以对字符串使用任何字符编码。

- 对于数据库中字符串需要编码的触点，它将采用 UTF-8。
- 客户端库可以强制执行编码（例如，Java和C＃客户端始终转换为UTF-8）。有关实现的信息，请参考客户端库。
- 跨语言兼容性取决于应用程序。该应用程序设置读取和写入读取的编码。
- 记录大小限制为 128KB。不要存储巨大的字符串。
- 原子的 `prepend` and `append` 操作可以应用于字符串值。
- 对于 equality ，字符串数据类型均支持[Secondary indexes](https://docs.aerospike.com/docs/architecture/secondary-index.html) 。

---

### <span id="boolean"> Boolean </span>

boolean 数据类型的值为 `true` or `false` 。

- 作为 single byte 存储在服务器上。
- 支持在 Aerospike 版本 5.6.0 中 add boolean bin 值。
- 在5.6.0版之前，只有List和Map数据类型可以包含 boolean 值。

---

### <span id="blob-bytes"> Blob/Bytes </span>

[Blobs](https://docs.aerospike.com/docs/guide/blob/) 是特定大小的字节数组。

- 可以存储任何类型的任何二进制数据。请注意，字节不是以 NULL 结果的。
- 从 4.6.0 版开始，可以将各种 [bitwise operations](https://docs.aerospike.com/docs/guide/blob/#bitwise-operations) 的 API 应用于字节值。

---

### <span id="language-specific-serialized-blobs"> Language-Specific Serialized Blobs </span>

以下 Aerospike 客户端使用其本机语言序列化机制将不支持的数据类型序列化为 Blob。这些 Blob 在读取时会自动反序列化。

- Java
- C# .NET
- Node.js
- Python
- C
- PHP