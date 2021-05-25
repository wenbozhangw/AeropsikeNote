## General Network Configuration

Aerospike配置的 network 节需要以下子节 :
- service
- heartbeat
- fabric
- info

通常，仅根据需要修改 service 和 heartbeat 子节，具体取决于您是否需要将 fabric (节点间复制，迁移)流量与 heartbeat 和 service 隔开。

如果要从3.10之前的版本升级，请参阅将 [Upgrade-Network to 3.10](https://docs.aerospike.com/docs/operations/upgrade/network_to_3_10) 以获得更多详细信息。

这是“ service”子节1中的配置的一般定义：

- `address` 配置是要绑定和监听的接口或 IP 地址。对于 3.10 及更高版本，您可以配置要绑定的多个 IP 地址。
- `access-address` 配置是指要为客户端（通常是同一子网/DC内的客户端）发布的接口或IP地址。
- `alter-access-address` 配置是指要为无法连接到访问地址接口或IP地址的客户端发布的接口或IP地址。如果此处指定的接口/IP是实际接口（而不是通过NAT映射），则还必须指定相应的 `address` 配置（除非设置了 `address`）。

---

### For server version 3.10 and above :

##### Service

用例示例：

1 - 主机具有 2 个网络接口 x.x.x.x 和 y.y.y.y ，其中 x.x.x.x 用于同一个子网/DC（专用IP）中的客户端，y.y.y.y用于不同子网/DC（公用IP）中的客户端。IP地址 y.y.y.y 未通过NAT映射：
```
service {
    address x.x.x.x
    address y.y.y.y
    access-address x.x.x.x
    alternate-access-address y.y.y.y
}
```

`access-address` x.x.x.x 是为了防止 y.y.y.y IP 也被广播（如果未指定`access-address`，则指定为 `address` 的所有 IP 都将被发布/广播）。

2 - 如果y.y.y.y IP是通过NAT映射的：
```
service {
    address x.x.x.x
    access-address x.x.x.x
    alternate-access-address y.y.y.y
}
```

或者，因为默认情况下将发布`address`（除非通过 `access-address` 覆盖）：
```
service {
    address x.x.x.x
    alternate-access-address y.y.y.y
}
```

3 - 在大多数情况下有效的替代配置 ：`address any` (将绑定到所有可用接口)，然后发布特定的 `access-address` 和 `alternate-access-address` 。
```
service {
    address any
    access-address x.x.x.x
    alternate-access-address y.y.y.y
}
```

##### Info and Fabric

通过在 fabric 和 info 相应的子节中指定 `address` 配置，可以将集群内通信流量与常规客户端流量隔离开。默认情况下，它们设置为 `any`.
```
fabric {
  address any
  port 3001   # Intra-cluster communication port (migrates, replication, etc).
}

info {
  address any
  port 3003   # Plain text telnet management port.
}
```

### For server versions prior to 3.10 :

在 service 子节中，确保将 `access-address` 配置为应用程序将阈值通信的网络接口。对于服务器版本 3.3.26 至 3.9.1.1，`access-address` 应为物理网络接口地址之一。如果是虚拟访问地址，请在最后使用 `virtual` 关键字，例如 `access-address 192.168.1.100 virtual` 。

```
...
network {
  service {
    address any                  # service 监听的 NIC 的 IP
    port 3000                    # service 监控的端口
    access-address 192.168.1.100 # 客户端访问 service 导出的 IP 地址
  }

  fabric {
    address any
    port 3001   # 集群内通信端口（迁移，复制等）
  }

  info {
    address any
    port 3003   # 纯文本 telnet 管理端口
  }

  heartbeat {
    mode multicast                  # 使用 Multicast 发送 heartbeat
    address 239.1.99.2              # multicast address
    port 9918                       # multicast port
    interface-address 192.168.1.100 # 发送 heartbeat 和绑定 
                                    # fabric 端口的 NIC IP
    interval 150                    # heartbeats 间隔的毫秒数
    timeout 10                      # 节点超时前等待的 heartbeats 间隔数
  }

#  heartbeat {
#    mode mesh                   # 使用 Mesh (Unicast) 协议发送  heartbeats
#    address 192.168.1.100       # 节点监听 heartbeat 的 NIC IP 
#    port 3002                   # 节点监控 heartbeat 的端口
#    mesh-seed-address-port 192.168.1.101 3002 # 集群中种子节点的IP地址
#    mesh-seed-address-port 192.168.1.102 3002 # 集群中种子节点的IP地址
#    interval 150                # heartbeats 间隔的毫秒数
#    timeout 20                  # 节点超时前等待的 heartbeats 间隔数
#  }
}
...
```

---

### What is access-address ?

`access-address` 参数确定发布供客户端访问的地址。当 Aerospike 节点具有多个 IP 地址时，建议在配置中发布指向所需 IP 的 `access-address` 。

#### When to use access-address ?

当您希望客户端只在一个 IP 上通信，但希望服务器兼听多个 IP 时，应该指定 `access-address` 。在设置 XDR destination 时可能需要这样做。`access-address` 应该被指定客户端应该使用的 IP 。通常，当服务器具有公网和私网 IP 时，私网 IP 用于客户端，而公网 IP 用于 XDR 。

如果无法从 XDR 的源集群访问目标集群的私有访问地址，并且目标具有其他正在监听Aerospike的公网地址（并且这些公网地址可以从源集群访问），则重要的是指定一下内容之一：

- `dc-int-ext-map` 在源集群的配置上，用于将私网 IP 映射到公网节点 。
- 目标集群上的 `alternate-access-address` 以及源集群上的 `dc-use-alternate-services` 。

#### What happens when access-address is missing ?

如果节点具有多个 IP 地址，则每个节点将会看到多个服务器 IP。某些客户端（如 Java） 可以基于节点 ID 对重复的 IP 进行删除，但是某些工具 （如 [asadm](https://docs.aerospike.com/docs/tools/asadm) ）可能无法进行重复删除。注入 asadm 之类的工具可能会报告此类情况，并显示集群可见性为 false error，因为它看到集群大小与服务器 IP 数量之间不匹配。

这不会影响客户端的工作，只会导致工具出现问题。

---

### Next Steps

- 配置 [heartbeat sub-stanzas](https://docs.aerospike.com/docs/operations/configure/network/heartbeat) ，该节定义将用于集群内通信的接口。
- 配置 [Rack Aware](https://docs.aerospike.com/docs/operations/configure/network/rack-aware) ，使 Aerospike 支持 top-of-rack switch failure 。
- 或返回 [Configure Page](https://docs.aerospike.com/docs/operations/configure) 。