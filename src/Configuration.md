# Aerospike配置

## 配置Aerospike数据库
Aerospike仅使用单个文件配置数据库节点。配置文件默认位置为`/etc/aerospike/aerospike.conf`。

配置文件划分为不同的 ***contexts（上下文）***，其中有些是可选的。这些上下文可以在配置文件中排列顺序是任意的。上下文可以进一步分为 ***sub-contexts（子上下文）***。

配置文件中一行最大长度为 ***1024*** 个字符。（在Aerospike Server 5.0 之前，行长度限制为 ***256*** 个字符）。

下面是常见上下文和子上下文的配置文件结构示例。下面显示的大多数上下文都是空的，它们没有实际参数，用大括号表示`{}`。子上下文缩进。

```
service {}                      # 调整参数和进程所有者

network {                       # 用于配置集群内和应用程序节点通讯
  service {}                    # 工具/应用程序通讯协议
  fabric {}                     # 集群内通讯协议
  info {}                       # 管理员telnet控制台协议
  heartbeat {}                  # 集群形成协议
}

security {                      # （可选，仅限企业版）在集群上启用 ACL
  enable-security true
}

logging {}                      # 日志配置

xdr {                           # （可选，仅限企业版）配置 Cross-Datacenter（跨数据中心） 复制
  dc <name> {}                  # 远程数据中心节点列表
}

namespace <name> {              # 定义命名空间记录策略和存储引擎
  storage {}                    # 配置持久化或缺乏持久化
  set {}                        # （可选）设置特定的记录策略
}

mod-lua {                       # UDF模块的位置
  user-path /opt/aerospike/usr/udf/lua
}
```

## 配置步骤

1. [配置 Network service 和 heartbeat sub-contexts](https://www.aerospike.com/docs/operations/configure/network/)
2. [配置 Namespaces](https://www.aerospike.com/docs/operations/configure/namespace/)
3. 配置 [Logging](https://www.aerospike.com/docs/operations/configure/log/) 和 [Log Rotate](https://www.aerospike.com/docs/operations/configure/log/index.html#log-rotate)
4. （可选）配置 [Security](https://www.aerospike.com/docs/operations/configure/security/)
5. （可选）配置 [Rack-Aware](https://www.aerospike.com/docs/operations/configure/network/rack-aware/)
6. （可选）配置 [Cross-Datacenter Replication](https://www.aerospike.com/docs/operations/configure/cross-datacenter/)