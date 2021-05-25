## Network Heartbeat Configuration

Aerospike的心跳协议负责维护集群的完整性。有两种受支持的心跳模式：
- Multicast (UDP)
- Mesh (TCP)

---

### Cloud Considerations

1. **Lack of Multicast Support :** 云提供商（例如Amazon，Google Compute Engine）不支持 multicast networking 。对于这些提供商，我们提供了 [Mesh heartbeats](https://docs.aerospike.com/docs/operations/configure/network/heartbeat/index.html#mesh-unicast-heartbeatd ) ，该心跳使用点对点 TCP 连接进行心跳。

2. **Network Variability :** 通常，云平台的网络延迟会随着时间的推移而不一致。这可能会导致心跳数据包传递时间出现问题。对于这些供应商，我们建议将 heartbeat `interval` 设置为 150，并将 heartbeat `timeout` 设置为 20 。

3. **Instance Pauses :** 有时，云提供商可能会在短时间内暂停您的云实例。例如，GCE 使用 [live migration](http://googlecloudplatform.blogspot.in/2015/03/Google-Compute-Engine-uses-Live-Migration-technology-to-service-infrastructure-without-application-downtime.html) ，可以在短时间内暂停实例以进行维护或软件更新。这些短暂的暂停可能会导致集群中的其他实例在暂停持续时间内将该实例视为 "死机(dead)"。因此，我们建议考虑升级服务器版本 3.8.1 及更高版本，其中 [paxos-recovery-policy](https://docs.aerospike.com/docs/reference/configuration/index.html#paxos-recovery-policy) 默认为 `auto-reset-master`，以帮助您的集群在网络中断或集群更改后迅速恢复。Aerospike服务器版本 3.7.0.1 中已引入了此策略，但需要进行显示配置 `auto-reset-master` ，直到 3.8.1 版本，不再需要显式配置。

---

### Multicast Heartbeat

我们建议您在可用时使用 multicast heartbeat 协议。由于各种原因，您的网络可能不支持 multicast。请参阅我们的 [troubleshooting guide](https://docs.aerospike.com/docs/operations/troubleshoot) ，以获取有关如何在您的环境中验证多播的信息。

如果要从3.10之前的版本升级，请参阅 [Upgrade-Network to 3.10](https://docs.aerospike.com/docs/operations/upgrade/network_to_3_10#multicast) 以获得更多详细信息。

#### Configuration Steps

在 heartbeat 子节中：

1. 将 `mode` 设置为 `multicast`.
2. 将 `multicast-group` 设置为有效的多播地址 (239.0.0.0 - 239.255.255.255) 。
3. (可选) 将 `address` 设置为用于集群内通信的接口的IP。此设置还控制 fabric 将使用的接口。将集群内流量隔离到特定网络接口时需要。
4. 设置 `interval` 和 `timeout`
    - `interval`（建议：150）控制发送心跳数据包的频率。
    - `timeout`（建议：10）控制 interval的数量，在此间隔之后，如果集群中的其余节点尚未收到丢失节点发出的心跳信号，则该节点被视为丢失。
    - 使用默认设置，一个节点将在1.5秒内知道另一个节点离开群集。
    
#### Example

```
...
  heartbeat {
    mode multicast                  # 使用 Multicast 发送 heartbeats
    multicast-group 239.1.99.2              # multicast address
    port 9918                       # multicast port
    address 192.168.1.100           # (可选) (默认 any) 用于发送 heartbeat 和绑定 fabric 端口的 NIC IP
    interval 150                    # heartbeats 间隔的毫秒数
    timeout 10                      # 节点超时前等待的 heartbeats 间隔数
  }
...
```

---

### Mesh (Unicast) Heartbeat

Mesh使用TCP点对点连接进行心跳。集群中的每个节点都保持与所有其他节点的心跳连接，从而导致 mesh 需要许多连接。因此，我们建议在可用时使用多播心跳协议。

如果要从3.10之前的版本升级，请参阅 [Upgrade-Network to 3.10](https://docs.aerospike.com/docs/operations/upgrade/network_to_3_10#mesh) 以获得更多详细信息。

#### Configuration Steps

在 heartbeat 子节中：

1. 将 `mode` 设置为 `mesh` .
2. （可选） 将 `address` 设置为用于集群内通信的本地接口的IP。此设置还控制 fabric 将使用的接口。将群集内流量隔离到特定网络接口时需要。
3. 将 `mesh-seed-address-port` 设置为集群中节点的IP地址（或从3.10版开始的合格DNS名称）和心跳端口。
4. 设置 `interval` 和 `timeout`
    - `interval`（建议：150）控制发送心跳数据包的频率。
    - `timeout`（建议：10）控制 interval的数量，在此间隔之后，如果集群中的其余节点尚未收到丢失节点发出的心跳信号，则该节点被视为丢失。
    - 使用默认设置，一个节点将在1.5秒内知道另一个节点离开群集。
    
在版本 4.3.1 和更早版本中使用完全限定的名称(fully qualified names)时，如果 DNS 服务器变慢并且名称解析花费较长时间才能失败，则无法DNS解析的名称可能会导致集群分裂。成功的 DNS 解析将用 IP 地址替换名称，直到随后重新启动为止。

#### Example

```
...
  heartbeat {
    mode mesh                   # 使用 Mesh(Unicast) 协议发送心跳
    address 192.168.1.100       # (可选) (默认 any) 节点监听 heartbeat 的 NIC IP 
    port 3002                   # 节点监控 heartbeat 的端口
    mesh-seed-address-port 192.168.1.100 3002 # 集群中种子节点的IP地址
    mesh-seed-address-port 192.168.1.101 3002 # 集群中种子节点的IP地址
    mesh-seed-address-port 192.168.1.102 3002 # 集群中种子节点的IP地址
    mesh-seed-address-port 192.168.1.103 3002 # 集群中种子节点的IP地址

    interval 150                # heartbeats 间隔的毫秒数
    timeout 10                  # 节点超时前等待的 heartbeats 间隔数
  }
...
```

---

### Where to Next ?

- 配置 [service, fabric, and info sub-stanzas](https://docs.aerospike.com/docs/operations/configure/network/general) ，这些子节定义将用于应用程序到节点通信的接口。
- 配置 [heartbeat sub-stanzas](https://docs.aerospike.com/docs/operations/configure/network/heartbeat) ，该节定义将用于集群内通信的接口。
- 学习更多关于 [Rack Aware Architecture](https://docs.aerospike.com/docs/architecture/rack-aware.html) 。
- 或返回 [Configure Page](https://docs.aerospike.com/docs/operations/configure) 。