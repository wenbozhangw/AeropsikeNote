# mod-lua 配置说明

## 一、配置说明
| name | Default | tag  | Comment |
| --- | --- | --- | --- |
| cache-enabled | true |      | 是否为每个已注册的 Lua 模块启用 Lua states 的缓存，以提高性能。<br/>注意：启用缓存后，最初会为每个节点上的每个Lua模块缓存 10 个 [Lua states](luaState/LuaStates.md)，并且缓存会在运行时根据需要扩展，每个模块最多可容纳 128 个实体。 |
|~~system-path~~| /opt/aerospike/sys/udf/lua| Removed:4.3.1 | Aerospike进程用于存储默认UDF文件的目录。从 4.3.1 版本开始删除，该版本通过将代码存储在 mod-lua 模块中的 C 字符串中并直接从中加载来简化过程。这消除了任何客户端/服务器对 lua-core 模块的依赖。从 4.3.1 版本开始，/udf/lua/external 下的目录不再是安装的一部分。升级后，可以删除清理这些目录。如果用户指定此目录，则Aerospike进程必须对该目录具有读/写权限。|
|user-path| /opt/aerospike/usr/udf/lua | |Aerospike进程将使用该目录来存储用户生成的UDF文件。如果用户指定此目录，则Aerospike进程必须对该目录具有读/写权限。|
