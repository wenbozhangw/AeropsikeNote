## Rack Aware Configuration

在3.13中，Aerospike大大简化了机架感知型部署的操作易用性。如果您的部署版本早于3.13版本，并且想要切换到机架配置，我们建议您首先 [Upgrade to 3.13](https://docs.aerospike.com/docs/operations/upgrade/cluster_to_3_13) 。如果您必须使用较旧的版本来配置 rackaware，请参见“3.13之前的版本”部分以获取说明。

从版本4.0开始，机架感知功能是仅Aerospike Enterprise Edition Server的功能。

当具有不平衡的机架（每个机架具有不同数量的节点）时，主分区仍将在集群中的每个节点上均匀分布（根据`prefer-uniform-balance`配置）。副本分区将分布在不同的机架上，在这种特殊情况下，将产生节点数量较少的机架上数量更多的副本分区。因此，master write 将均匀分布，但副本写入将不会均匀分布。跨机架的节点上的一般负载的不平衡将取决于工作负载的性质（例如，记录的更新部分与其总大小之比，因为副本写入将承载完整的记录）。

---

### Configuration a new rackaware cluster

可以在命名空间级别指定 Rackaware。对于应该在同一机架上的节点，只需为这些节点指定相同的 `rack-id` 
```
namespace {
    ...
    rack-id 1
    ...
}
```
Rack-id 必须为 1 到 1000000 之间的整数。

---

### Upgrade a cluster for rack awareness

可以通过滚动重启或动态启用或更新现有集群的Rackaware配置（对于运行新集群协议的3.13及更高版本）。

#### Dynamic

- 在每个节点上，使用asinfo命令将rack-id更改为所需的值：
```
asinfo -v "set-config:context=namespace;id=namespaceName;rack-id=1"
```
- 将rack-id配置参数添加到配置文件中的 namespace 节中，以确保在以后重新启动后配置仍然保留。
- 触发集群的重新平衡，以使迁移与新的机架ID配置一起进行：
    - 对于Aerospike版本3.14.1.1及更高版本，请发出recluster命令：`asadm -e“ asinfo -v recluster：”`
    - 或者，对于运行新群集协议的aerospike版本3.13，请重新启动其中一个节点以触发迁移（无需完全滚动重新启动所有节点）。
    

#### Rolling restart method

- 将一个节点下线。
- 将 `rack-id` 添加到所需的命名空间。
- 重新启动节点
- 对下一个节点重复上述步骤，直到所有节点/命名空间都具有所需的 `rack-id` 。

Note - 使用滚动重启方法时，请确保集群具有足够的容量。当第一个节点启动时，它将具有与集群中所有其他节点不同的 `rack-id`，因此将接收整个机架的流量，并尝试为其他机架感知的命名空间保留每个分区的副本。建议使用动态方法。

---

### Display rack group settings

要对机架进行分组-

```
asinfo -v "racks:""
ns=test:rack_1=BCD10DFA9290C00,BB910DFA9290C00:rack_2=BD710DFA9290C00,BC310DFA9290C00
```

对于上面的示例，对于 test 命名空间，rack-id 1包括节点BCD10DFA9290C00，BB910DFA9290C00，rack-id 2包括节点BD710DFA9290C00，BC310DFA9290C00

---

### Versions earlier than 3.13

#### Configure a new rackaware cluster

要配置机架感知，需要完成以下工作：
- 配置机架感知协议。
- 配置每个服务器节点及其组。

这些配置更改将在下面详细描述。

#### Configuring the Rack-Awareness Protocol

将服务节中的paxos-protocol参数更新为v4，如下所示：

```
service {
    ...
    paxos-protocol v4
    ...
}
```

#### Configuring the Group (Rack)

在非机架感知的情况下，每个节点都是由一个节点值 —— 一个 64 bit的无符号整数标识，该值包括： 16-bit port ID + 48-bit hardware MAC address。集群中的每个节点值都必须是唯一的。

使用机架感知时，节点值包括： 16-bit port ID + 16-bit group ID + 32-bit node ID : 

- 32-bit node ID 可以从节点的 IP 地址自动生成，也可以明确指定。当您想要对 node ID 值进行更多控制以验证或集群调试时，可以显示指定 node ID。
- 必须明确指定 group ID。

##### Explicit Node ID

通过在节点的配置文件中插入以下顶级节来设置每个节点的ID和组:
```
cluster {
    mode static
    self-node-id [32-bit unsigned integer node ID]
    self-group-id [16-bit unsigned integer group ID]
}
```

##### Auto Generate Node ID

将以下顶级节插入每个节点的配置文件中：（节点ID是根据IP地址计算的）
```
cluster {
    mode dynamic
    self-group-id [16-bit unsigned integer group ID]
}
```

##### Caution:

选择 `dynamic` 选项并使用 IP 地址作为节点 ID 的一部分时，需要一些额外的 bookkeeping。您必须注意避免为集群中的节点重用IP地址，因为这会影响节点ID的唯一性。不同数据库节点的节点ID不得冲突。这样的冲突将导致群集无法正确形成。

数据库节点的节点ID在集群实例中节点生命周期的开始处仅计算一次，因此，即使该节点的实际 IP 可能随时发生变化，其节点 ID 在其声明周期或节点声明周期中都是固定的（whichever is shorter）。因此，即使集群的节点IP地址在任何时候都不会发生冲突，为了在集群的整个生命周期内确保他们的唯一性，任何活动的IP地址都不得与任何先前启动的节点IP地址重叠。

##### Note:

1. 如果节点值（port+group+node）在集群中不是唯一的，则具有重复节点值的节点将无法加入集群。
2. 节点ID和组ID必须为正（非零）整数。
3. 要关闭 rack-aware 支持，用户可以注释掉或删除 "mode" 行，也可以使用值 "none" : e.g. mode none。这将恢复默认（non-rack-aware）集群行为。
4. 处于机架感知模式的集群可以混合使用具有自动生成的节点ID和显式节点ID的节点，也就是说，您可以混合并配置如何制定节点ID。

例如，要将两个节点配置为一个组，第一个节点的配置文件中可能包含以下内容：

```
service {
    ...
    paxos-protocol v4
    ...
}

cluster {
    mode static
    self-node-id  101
    self-group-id 201
}
```

第二个节点的配置文件中可能包含以下内容：

```
service {
    ...
    paxos-protocol v4
    ...
}

cluster {
    mode dynamic
    self-group-id 201
}
```

启动时，当节点加入集群时，集群中的现有节点将发现组拓扑并相应地分配集群副本。

---

### Upgrade a Cluster to Rackaware

要升级群集以使用机架感知：

1. 停止所有集群节点
2. 升级其配置文件，类似于启动新集群。
3. 重启集群。
    - 由于所有集群节点都必须统一副本的位置，因此当您首次启用机架感知功能时，必须重新启动整个集群。
    
如果启用机架感知，则集群中的所有节点都必须使用机架感知；也就是说，按照上述方法配置所有节点。对机架感知至少需要两个组。如果只有一个节点或只有一个组，则默认（非机架感知）行为将自动接管。如果集群启动时存在多个组（机架），但故障仅导致单个组（机架）启动，也会出现这种情况。单个剩余组（机架）中的其余节点会自动形成默认（非机架感知）集群。

---

### Scale a Rackware Cluster

启用机架感知后，无需重新启动整个集群，即可将节点添加到组或从组中删除节点。要将节点添加到集群，只需在启动节点之前配置组即可。集群重新平衡。

节点值（即由 port + group + node 组成的 64-bit 标识符）*必须*唯一。当您第一次启用机架感知，这种唯一性很容易配置；然而，随着硬件升级和 IP 地址的变化，唯一性可能会更难维护，特别是在混合使用 static and dynamic 节点值时。最佳做法是创建并仔细维护当前节点值的列表。当一台新机器进入集群，并且发生节点值冲突时，由于节点通信出现故障，可能会发生奇怪的集群行为。

---

### Display Rack Group Settings

要查看组中的节点：

```
asinfo -v dump-ra:verbose=true
```

在此 3-node/3-group 集群示例中，使用节点ID 101的查询的输出为：

```
May 28 2013 18:39:00 GMT: INFO (info): (base/cluster_config.c:267) Rack Aware is enabled.  Mode: static.
May 28 2013 18:39:00 GMT: INFO (paxos): (base/cluster_config.c:281) SuccessionList[0]: Node bcd00cb00000069 : Port 3021 ; GroupID 203 ; NodeID 105 [Master]
May 28 2013 18:39:00 GMT: INFO (paxos): (base/cluster_config.c:281) SuccessionList[1]: Node bc300ca00000067 : Port 3011 ; GroupID 202 ; NodeID 103
May 28 2013 18:39:00 GMT: INFO (paxos): (base/cluster_config.c:281) SuccessionList[2]: Node bb900c900000065 : Port 3001 ; GroupID 201 ; NodeID 101 [Self]
```

---

### Where to Next ?

### Where to Next ?
- 配置 [service, fabric, and info sub-stanzas](https://docs.aerospike.com/docs/operations/configure/network/general) ，这些子节定义将用于应用程序到节点通信的接口。
- （可选）配置 [Rack Aware](https://docs.aerospike.com/docs/operations/configure/network/rack-aware) ，使 Aerospike 支持 top-of-rack switch failure 。
- 学习更多关于 [Clustering Architecture](https://docs.aerospike.com/docs/architecture/clustering.html) 。
- 或返回 [Configure Page](https://docs.aerospike.com/docs/operations/configure) 。