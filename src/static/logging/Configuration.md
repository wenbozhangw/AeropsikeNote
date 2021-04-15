# logging上下文参数配置

## 一、配置说明
| name | Default                          | tag  | Comment                                                      |
| ---- | -------------------------------- | ---- | ------------------------------------------------------------ |
| file | /var/log/aerospike/aerospike.log |      | 指定用于记录的服务器日志文件的路径。不要与命名空间上下文中的文件混淆。你可以针对不同的上下文使用多个文件。 |

## 二、示例

```
logging {
  file /var/log/aerospike/aerospike.log {
    context any info
  }
  
  file /var/log/aerospike/aerospike_debug.log {
    context any debug
  }
}
```

上下文指定要记录的日志的上下文和级别。你可以结合使用上下文和日志记录级别。此配置可以与 `file` 或 `console`类型的的日志记录一起使用。可以从命令中获得不同的上下文及其日志记录级别。

```
asinfo -v log/0
```

输出：

```
misc:INFO;
alloc:INFO;
arenax:INFO;
hardware:INFO
msg:INFO;
rbuffer:INFO;
socket:INFO;
tls:INFO;
vault:INFO;
vmapx:INFO;
xmem:INFO;
aggr:INFO;
appeal:INFO;
as:INFO;
batch:INFO;
bin:INFO;
config:INFO;
clustering:INFO;
drv_pmem:INFO;
drv_ssd:INFO;
exchange:INFO;
exp:INFO;
fabric:INFO;
flat:INFO;
geo:INFO;
hb:INFO;
health:INFO;
hlc:INFO;
index:INFO;
info:INFO;
info-port:INFO;
job:INFO;
migrate:INFO;
mon:INFO;
namespace:INFO;
nsup:INFO;
particle:INFO;
partition:INFO;
paxos:INFO;
proto:INFO;
proxy:INFO;
proxy-divert:INFO;
query:INFO;
record:INFO;
roster:INFO;
rw:INFO;
rw-client:INFO;
scan:INFO;
security:INFO;
service:INFO;
service-list:INFO;
sindex:INFO;
skew:INFO;
smd:INFO;
storage:INFO;
truncate:INFO;
tsvc:INFO;
udf:INFO;
xdr:INFO;
xdr-client:INFO
```

## 三、日志记录级别：

- context any info

- context any debug

- context any warning

- context any critical

- context any detail

有关常用日志消息的详细信息和上下文的完整列表，请参阅[服务器日志消息参考手册](manual/ServerLogMessagesReferenceManual.md)，官方链接地址 [Server Log Messages Reference Manual](https://www.aerospike.com/docs/reference/serverlogmessages/) .