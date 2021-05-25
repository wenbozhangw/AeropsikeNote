## IPv6 Configuration

从 3.10 版本开始，可以将 Aerospike 集群配置为将 IPv6 用于节点到节点和客户端到集群的连接。

本文档介绍了前提条件和相关配置。如果您打算将网络迁移到仅IPv6的环境，请联系Aerospike支持以进行平滑迁移。

---

### Required Network Setup for IPv6 :

1. 支持 IPv6 的Linux操作系统。
2. 接口上的 Site-local or global IPv6 地址。Aerospike IPv6不支持 link-local addresses。

在启用IPv6的接口上的示例配置：
```
[citrusleaf@localhost serverlocal]$ ifconfig
eth1      Link encap:Ethernet  HWaddr 00:0C:29:A9:DF:10 
          inet addr:172.16.1.128  Bcast:172.16.1.255  Mask:255.255.255.0
          inet6 addr: 2001::1111/64 Scope:Global
          inet6 addr: fe80::20c:29ff:fea9:df10/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:387 errors:0 dropped:0 overruns:0 frame:0
          TX packets:324 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:34012 (33.2 KiB)  TX bytes:31786 (31.0 KiB)
```

对于上述情况，地址“ 2001 :: 1111”将起作用，因为它是全局的。地址“ fe80 :: 20c：29ff：fea9：df10”将不会，因为它是本地链接。如果您的网络未启用IPv6，因此没有为您分配IPv6地址，则仍然可以通过手动将IPv6地址分配给计算机来使用IPv6。为此，机器（客户端以及群集节点）需要驻留在同一以太网段中。

---

### Required Client :

- Java - version 3.3 and above. 
- C - version 4.10 and above.
- C# - version 3.3 and above.

---

### Additional Requirement -

- 仅在企业版构建中可用。
- `advertise-ipv6` 必须为 true。
- 必须使用心跳协议版本v3。请联系Aerospike支持以升级到心跳v3协议。

---

### Sample Aerospike configuration :

```
service {
    ....
    advertise-ipv6 true
}

network {
    heartbeat {
        ....
        protocol v3 // MUST follow instruction on how to switch over to hbv3 protocol.
    }
}
```