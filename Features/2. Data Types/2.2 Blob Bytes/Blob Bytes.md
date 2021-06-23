## Blob/Bytes

Aerospike 支持两种主要类型的 Blob/Bytes bins 。有通用的 Blob bins 和特定语言的 Blob bins。

通用 Blob bins是特定大小的字节数组。任何类型的任何二进制数据都可以存储在 Blob bin中。它们还支持 [Bitwise Operations](https://docs.aerospike.com/docs/guide/blob/#bitwise-operations) 。

特定语言的 Blob bins 只能从编写它们的客户端库语言访问。这些对于将使用该语言的对象序列化到 Aerospike 数据库很有用。当前，Aerospike 支持以下特定语言的 Blob 类型：[C#](https://docs.aerospike.com/docs/client/csharp/) , [Java](https://docs.aerospike.com/docs/client/java/), [PHP](https://docs.aerospike.com/docs/client/php/) , [Python](https://docs.aerospike.com/docs/client/python/) , and [Ruby](https://docs.aerospike.com/docs/client/ruby/) 。

##### Content

- [Bitwise Operations](#bitwise-operations)
  - [Modify flags](#modify-flags)
  - [Modify Operations](#modify-operations)
    - [resize](#resize)
    - [insert](#insert)
    - [remove](#remove)
    - [set](#set)
    - [or](#or)
    - [xor](#xor)
    - [and](#and)
    - [not](#not)
    - [lshift](#lshift)
    - [rshift](#rshift)
    - [add](#add)
    - [subtract](#subtract)
    - [set_int](#set-int)
  - [Read Operations](#read-operations)
    - [get](#get)
    - [count](#count)
    - [lscan](#lscan)
    - [rscan](#rscan)
    - [get_integer](#get-integer)

### <span id="bitwise-operations"> Bitwise Operations </span>

Aerospike 支持一组丰富的按位运算，可用于 Blob 数据类型。这些操作允许应用程序在服务器上操纵大型 Blob bin，而无需将整个 Blob 拉到客户端，这可以节省客户端到服务器的带宽。

#### title

| Name | Value | Description | 
| --- | --- | --- |
| create_only | 0x01 | 禁止更新此 bin 的现有值。 |
| update_only | 0x02 | 禁止创建新的 Blob bin。 |
| no_fail | 0x04 | 如果操作失败，请像成功一样继续操作。 |
| partial | 0x08 | 如果从偏移量到现有 Blob bin 的末尾的字节数少于指定的字节数，则仅应用从偏移量到末尾的操作。 |

### <span id="modify-operations"> Modify Operations </span>

#### <span id="resize"> resize </span>

```
resize(policy, bin_name, n_bytes, resize_flags)
```

将 Blob bin 的大小指定为 **n_bytes**。此操作（默认情况下）可以创建一个新的 Byte bin，或将现有的 Byte bin 扩展或 trim 为指定的 **n_bytes** 大小。默认情况下，调整大小操作将从 Blob bin 的末端开始扩展或 trim。 

**Flags** : create_only, update_only, no_fail.

**Arguments :**
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **n_bytes** (integer) : Number of bytes to resize to.
- **resize_flags** (integer)

| Name | Value | Description |
| --- | --- | --- |
| from_front | 0x01 | Extend or trim the Blob bin from the beginning instead of the end. |
| grow_only | 0x02 | Disallow trimming existing objects. |
| shrink_only | 0x04 | Disallow extending existing objects. |

**Returns** (none)

---

#### <span id="insert"> insert </span>

```
insert(policy, bin_name, byte_offset, value)
```

将 **buffer** 中的 bytes 内容插入指定的 **byte_offset**。

**Flags** : create_only, update_only, no_fail.

**Arguments** :
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **byte_offset** (integer) : Offset to location of insertion.
- **value** (bytes) : Bytes to be inserted.

**Returns** (none)

---

#### <span id="remove"> remove </span>

```
remove(policy, bin_name, byte_offset, n_bytes)
```

删除从指定的 **byte_offset** 开始的 **n_bytes** 个字节。

**Flags** : update_only, no_fail, partial.

**Arguments** :
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **byte_offset** (integer) : Offset to location of removal.
- **n_bytes** (integer) : Number of bytes to remove.

**Returns** (none)

---

#### <span id="set"> set </span>

```
set(policy, bin_name, bit_offset, n_bits, value)
```

用 **value** 的前 **n_bits** 覆盖指定 **offset** (in bits) 的 **n_bits** 位。

**Flags** : update_only, no_fail, partial.

**Arguments** :
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first bit to overwrite.
- **n_bits** (integer) : Offset (in bits) to the first bit to overwrite.
- **value** (bytes) : Buffer containing at least **n_bits** bits to be written. Bits are taken in order from the beginning of the buffer.

**Returns** (none)

---

#### <span id="or"> or </span>

```
or(policy, bin_name, bit_offset, n_bits, value)
```

**buffer** 的 **n_bits** 与从指定 **offset** (in bits) 开始的 Blob bin 的前 **n_bits** 的按位 OR。

**Flags** : update_only, no_fail, partial.

**Arguments** :
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first bit to apply operation.
- **n_bits** (integer) : Number of bits to apply operation to.
- **value** (bytes) : Buffer containing at least **n_bits** bits.


**Returns** (none)

---

#### <span id="xor"> xor </span>

```
xor(policy, bin_name, bit_offset, n_bits, value)
```

从指定的 **offset** (in bits) 开始， **buffer** 的 **n_bits** 与 Blob bin 的前 **n_bits** 的按位 XOR。

**Flags** : update_only, no_fail, partial.

**Arguments** :
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first bit to apply operation.
- **n_bits** (integer) : Number of bits to apply operation to.
- **value** (bytes) : Buffer containing at least **n_bits** bits.


**Returns** (none)

---

#### <span id="and"> and </span>

```
and(policy, bin_name, bit_offset, n_bits, value)
```

**buffer** 的 **n_bits** 与从指定 **offset** (in bits) 开始的 Blob bin 的前 **n_bits** 的按位 AND 。

**Flags** : update_only, no_fail, partial.

**Arguments** :
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first bit to apply operation.
- **n_bits** (integer) : Number of bits to apply operation to.
- **value** (bytes) : Buffer containing at least **n_bits** bits.


**Returns** (none)

---

#### <span id="not"> not </span>

```
not(policy, bin_name, bit_offset, n_bits)
```

从指定的 **offset** (in bits) 开始的 Blob bin 的 **n_bits** 的按位 NOT 。

**Flags** : update_only, no_fail, partial.

**Arguments** :
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first bit to apply operation.
- **n_bits** (integer) : Number of bits to apply operation to.

**Returns** (none)

---

#### <span id="lshift"> lshift </span>

```
lshift(policy, bin_name, bit_offset, n_bits, shift)
```

从指定的 **offset** (in bits) 开始向左按位移动 Blob bin 的 **n_bits** 位

**Flags** : update_only, no_fail, partial.

**Arguments** :
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first bit to apply operation.
- **n_bits** (integer) : Number of bits to apply operation to.
- **shift** (integer) : Number of bits to shift.

**Returns** (none)

---

#### <span id="rshift"> rshift </span>

```
lshift(policy, bin_name, bit_offset, n_bits, shift)
```

从指定的 **offset** (in bits) 开始向右按位移动 Blob bin 的 **n_bits** 位

**Flags** : update_only, no_fail, partial.

**Arguments** :
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first bit to apply operation.
- **n_bits** (integer) : Number of bits to apply operation to.
- **shift** (integer) : Number of bits to shift.

**Returns** (none)

---

#### <span id="add"> add </span>

```
add(policy, bin_name, bit_offset, n_bits, value, signed, action)
```

将 Blob bin 中从 **offset** 开始的 **n_bits** 位视为 **n_bits** 位整数，并将整数 **value** add 到其中 —— 整数 **value** 将转换为 **n_bits** 位整数。默认情况下，如果结果溢出则失败。Blob bin中的整数倍存储并解释为 big-endian 整数。

**Flags** : update_only, no_fail.

**Arguments** :
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first bit to apply operation.
- **n_bits** (integer) : Size of the integer in bits (maximum of 64 bits).
- **value** (integer) : The unsigned integer value to be added.
- **signed** (boolean) : Read the integer from the Blob bin as signed (true) or unsigned (false).
- **action** (client_specific) : How to handle integer overflow.
  - (Default) : Fail transaction on overflow.
  - Set maximum value on overflow.
  - Wrap the value from the min value.
  
**Returns** (none)

---

#### <span id="subtract"> subtract </span>

```
subtract(policy, bin_name, bit_offset, n_bits, value, signed, action)
```

将 Blob bin 中从 **offset** 开始的 **n_bits** 位视为 **n_bits** 位整数，并 subtract 整数 **uint64** —— 整数 **uint64** 将转换为 **n_bits** 位整数。默认情况下，如果结果溢出则失败。Blob bin中的整数倍存储并解释为 big-endian 整数。

**Flags** : update_only, no_fail.

**Arguments** :
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first bit to apply operation.
- **n_bits** (integer) : Size of the integer in bits (maximum of 64 bits).
- **value** (integer) : The unsigned integer value to be added.
- **signed** (boolean) : Read the integer from the Blob bin as signed (true) or unsigned (false).
- **action** (client_specific) : How to handle integer overflow.
  - (Default) : Fail transaction on overflow.
  - Set maximum value on overflow.
  - Wrap the value from the min value.

**Returns** (none)

---

#### <span id="set-int"> set_int </span>

```
set_int(policy, bin_name, bit_offset, n_bits, value)
```

将 **uint64** 转换为 **n_bits** 为 big_endian 整数，覆盖偏移量 **offset** 处的 **n_bits** 位。

**Flags** : update_only, no_fail.

**Arguments** :
- **policy** (library_specific) : Bitwise modify policy.
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first bit to apply operation.
- **n_bits** (integer) : Size of the integer in bits (maximum of 64 bits).
- **value** (integer) : The unsigned integer value to be added.

**Returns** (none)

---

### <span id="read-operations"> Read Operations </span>

#### <span id="get"> get </span>

```
get(bin_name, bit_offset, n_bits)
```

从偏移量 **offset** 开始检索 **n_bits** 位。如果 **n_bits** 不是 8 的倍数，那么将有 **n_bits** `modulo 8` 零位填充末尾。

**Flags** : update_only, no_fail.

**Arguments** :
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first bit to be retrieved .
- **n_bits** (integer) : Number of bits to retrieve.

**Returns** (bytes)

---

#### <span id="count"> count </span>

```
count(bin_name, bit_offset, n_bits)
```

计算从 **offset** 开始的 **n_bits** 中设置为 1 的位数。

**Flags** : update_only, no_fail.

**Arguments** :
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first bit to be checked .
- **n_bits** (integer) : Number of bits to check.

**Returns** (bytes)

---

#### <span id="lscan"> lscan </span>

```
lscan(bin_name, bit_offset, n_bits, value)
```

返回相对于第一个 bit 的 **offset** 的位置，设置为从 **offset + n_bits** 到 **offset** 的 **value** 搜索。如果没有找到该 **value**，返回 -1。

**Flags** : update_only, no_fail.

**Arguments** :
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first (leftmost) bit to be scanned .
- **n_bits** (integer) : Number of bits from the **offset** to scan.
- **value** (boolean) : If true, search for the first set bit. If false, search for the first unset bit.

**Returns** (Integer)

---

#### <span id="rscan"> rscan </span>

```
lscan(bin_name, bit_offset, n_bits, value)
```

返回相对于第一个 bit 的 **offset** 的位置，设置为从 **offset** 到 **offset + n_bits** 的 **value** 搜索。如果没有找到该 **value**，返回 -1。

**Flags** : update_only, no_fail.

**Arguments** :
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first (leftmost) bit to be scanned .
- **n_bits** (integer) : Number of bits from the **offset** to scan.
- **value** (boolean) : If true, search for the first set bit. If false, search for the first unset bit.

**Returns** (Integer)

---

#### <span id="get-integer"> get_integer </span>

```
get_integer(bin_name, bit_offset, n_bits, signed)
```

检索从偏移量 **offset** 开始的 **n_bits** 位 big-endian 整数，作为 64 位整数。

**Flags** : update_only, no_fail.

**Arguments** :
- **bin_name** (string) : Name of bin.
- **bit_offset** (integer) :  Offset (in bits) to the first (leftmost) bit to be retrieved .
- **n_bits** (integer) : Number of bits to retrieve.
- **value** (boolean) : If true, treat the value at offset as a signed **n_bit** bit integer, otherwise treat value as unsigned.

**Returns** (Integer)