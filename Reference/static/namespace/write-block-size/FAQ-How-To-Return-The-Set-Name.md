## 如何使用其摘要返回记录的 set 名称 

### FAQ How to return the Set name using the Digest

### Context

在 Aerospike 日志中，特定的警告确实会打印受影响记录的摘要。为了解决这个问题，我可能需要确定包含摘要数据的 set name。

### Method

在Aerospike日志中，可能会观察到以下输出：
```log
May 07 2018 07:07:07 GMT: WARNING (rw): (write.c:1508) {namespace} write_master: failed as_storage_record_write() <Digest>:0x2263c4bbe8f3ecfdfe1f67391f3627c8a3b7eada
```
这是使用UDF返回设置名称的解决方案。

#### Step 1:
创建lua文件（例如setname.lua）
```lua
function get(rec)
return record.setname(rec)
end
```

#### Step 2:
使用aql注册UDF模块。
```lua
aql> register module 'setname.lua'
OK, 1 module added.
```

#### Step 3:
使用执行UDF模块lua函数get的aql获取setname。 
注意：为获得正确的DIGEST值，您将需要删除 hash value 之前的0x。

```log
aql> execute setname.get() on test where DIGEST="AE6A515F37DE4ADACC4496E2DA1FEA848C3A9270"
+--------+
|  get   |
+--------+
| "demo" |
+--------+
1 row in set (0.001 secs)
OK
```

### Notes
 - AQL – UDF Management documentation is located here: https://www.aerospike.com/docs/tools/aql/udf_management.html 
 - UDF API for database record: https://www.aerospike.com/docs/udf/api/record.html 

