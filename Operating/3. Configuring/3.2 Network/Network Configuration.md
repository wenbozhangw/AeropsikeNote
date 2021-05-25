## Network Configuration

Aerospike 数据库的 Network 节配置了供其他节点，应用程序和工具使用的关键网络端口。下表列出并描述了Aerospike数据库和 XDR 使用的端口：

| Name | Default Port | Description |
| --- | --- | --- |
| Service | 3000 | 应用程序，工具 和 远程XDR 使用 Service 端口进行数据库操作和集群状态。 |
| Fabric | 3001 | 集群内通信端口。副本写入，迁移和其他节点的通信都使用 Fabric 端口。 |
| Mesh Heartbeat | 3002 | Heartbeat protocol 端口用于形成和维护集群。（只能配置一个心跳端口。） |
| Multicast Heartbeat | 9918 | Heartbeat protocol 端口用于形成和维护集群。（只能配置一个心跳端口。） |
| Info | 3003 | Telnet 端口，该端口实现了纯文本协议，供管理员发布 info 命令。有关更多信息，请参见 [asinfo documentation](https://docs.aerospike.com/docs/tools/asinfo) 。|

确保所有 Application 和 XDR 节点都可以与所有 Aerospike 节点上的 **service** 端口通信。还要确保每个节点都可以通过配置的 **heartbeat** 和 **fabric** 进行通信。

对于 3.8 之前的版本，端口 3004 是用于查询 Cross-Datacenter Replication (XDR) 客户端的运行状况的默认端口。

### Where to Next ?

- 配置 [service, fabric, and info sub-stanzas](https://docs.aerospike.com/docs/operations/configure/network/general) ，这些子节定义将用于应用程序到节点通信的接口。
- 配置 [heartbeat sub-stanzas](https://docs.aerospike.com/docs/operations/configure/network/heartbeat) ，该节定义将用于集群内通信的接口。
- （可选）配置 [Rack Aware](https://docs.aerospike.com/docs/operations/configure/network/rack-aware) ，使 Aerospike 支持 top-of-rack switch failure
- 或返回 [Configure Page](https://docs.aerospike.com/docs/operations/configure) 。