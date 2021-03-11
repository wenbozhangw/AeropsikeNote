## FAST RESTART
Aerospike Enterprise Edition 具有一项功能，使节点可以更快速地启动并重新加入集群。拥有10亿条记录的节点将在大约 10 秒内重新启动（而没有此功能的在 40 分钟以上）。这使集群升级和其他各种操作变得更快。

### 它是如何工作的？
此功能使服务器节点可以使用 Linux 系统共享内存。服务器节点的主索引以及各种其他关键数据都分配在系统共享内存中，即时服务器进程停止，该共享内存仍将保留。重新启动后，该进程将重新连接到包含索引和其他数据的持久性内存，从而快速重建某些内部状态并重新加入其集群。他不需要从存储驱动器中读取所有记录数据即可重建索引。

### 我们如何做到这一点？
没有。对于配置了 flash storage 或 Data In Index 的命名空间，该节点的默认行为是始终尝试快速重启，因此只需要正常启动即可：

`sudo systemctl start aerospike`

您可以检查日志以查看节点是否对命名空间执行快速重启（也称为 WARM 重启）：

`INFO(namespace): (namespace_ee.c:361) {test} beginning warm restart`

### 何时不发生 fast restart?

在某些情况下，服务器节点将尝试快速重启，但确定无法做到时，将切换到 `cold restart`，在这种情况下，将从存储驱动器中读取所有记录数据以重建索引：

- 命名空间被配置为 `data-in-memory`。由于记录数据为存储在系统共享内存中，因此必须读取所有记录数据才能将数据恢复到内存中。但是，从 Enterprise Edition 版本 3.15.1.3 开始，不会重建索引，避免了在这种情况，从存储中恢复已删除记录的情况（当节点不在集群中时，为持久化删除的记录将被带回）。<br>
在`data-in-memory true` 3.15.1.3之前的服务器版本上使用。
  ```log
  INFO (namespace): (namespace_ee.c:162) {test} can't warm restart if data-in-memory
  INFO (namespace): (namespace_ee.c:243) {test} beginning COLD start
  ```
  从 3.15.1.3和更高版本的服务器开始，
  ```log
  INFO (namespace): (namespace_ee.c:360) {test} beginning cool restart
  ```
  
- 更改的值[partition-tree-sprigs](../Configuration.md#partition-tree-sprigs)。例如，原始值 512 更改为 1024：<br>
  ```log
  WARNING (namespace): (namespace_ee.c:495) {nsSSD} persistent memory partition-tree-sprigs 512 doesn't match config 1024
  ```
  
- 服务器节点关闭时，存储驱动器被删除干净。偶尔会执行此操作，例如，在修复具有物理故障的驱动器时。显然，如果记录数据被删除，则持久索引无效。幸运的是，在这种情况下，`cold restart`将很快 ———— 因为没有要读取的数据。

- 先前的关机是不可信的。如果节点意外停止，并且我们不能确定持久索引的可靠性和存储数据的一致性，那么最好重建索引。

- 机器重新启动。系统共享内存可以在服务器进程停止（但机器不重新启动）后幸免于难。

- 在3.13.0之前的版本中，如果更改了名称空间在配置中的顺序。

如果节点确实切换到“cold restart”，则日志文件将提及该节点，并应指出切换的原因。

请注意，在定义二级索引时，节点的启动时间会大大减慢，具体取决于定义的二级索引的数量及其大小。如果名称空间满足快速重新启动的条件但定义了二级索引，则在快速重新启动（仅主索引）之后，名称空间将必须重建定义的每个辅助索引，这将需要从持久性存储中查找记录值。

### 我可以强制 "cold restart" 吗？
是的，尽管不是必须的，但是您可以使用以下命令手动强制"cold restart":
```shell
sudo /etc/init.d/aerospike coldstart
```

### 可以监控 Aerospike 使用的系统共享内存吗？
可以使用以下命令查看系统的共享内存块：
```shell
sudo ipcs -m
```
列出的所有带有 key 开头的块：
```shell
0xae...
```
"ae"（如Aerospike一样）是Aerospike共享内存块。对于每个命名空间，会有一个或多个 1GB 内存块。例如，队友具有两个命名空间的节点，您可能会看到：
```shell
------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      
0xae001100 0          root       666        1073741824 1                       
0xae002100 32769      root       666        1073741824 1                       
...
```

### fast restart for data-in-memory namespace
如果您的内存中数据命名空间是仅包含数字数据（整数或双精度）的单bin命名空间，则它也可以利用快速丑重启功能。
为此，将"data-in-index"配置添加到您的命名空间中：
```
namespace <namespace-name> {
...
    single-bin true
    data-in-index true
...
}
```
然后，只需重新启动Aerospike，就应该发生 fast restart。

如果数据库中存在非数字数据类型-在减少快速重启索引的同时，我们将检查每个记录的确是整数还是双精度。如果找到的记录不存在，Aerospike进程将退出并显示一条消息。成功重启后，任何尝试写入非数字数据的尝试都将被拒绝，错误代码为12（INCOMPATIBLE_TYPE）。