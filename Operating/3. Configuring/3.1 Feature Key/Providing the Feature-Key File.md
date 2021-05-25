## Providing the Feature-Key File

The feature-key 文件是启用服务器功能的加密签名列表。从企业版(Enterprise Edition, EE) 4.6 开始，服务器需要 feature-key 才能启动。 Community Edition (CE) 的用户可以跳过此部分。

### Defining the location of the feature key file

如果服务器找不到 feature key 文件，它将在启动顺序中提前退出并发出以下日志消息：
```log
Apr 09 2021 06:35:12 GMT: CRITICAL (config): (features_ee.c:142) failed to get feature key /etc/aerospike/features.conf
```

如您所见，功能密钥文件的默认路径是 `/etc/aerospike/features.conf`。满足此要求的最简单方法是将您的文件复制到此位置。

仅对于企业版，可以将 `feature-key-file` 配置参数添加到 `service` 节中。
```
service {
    feature-key-file /opt/aerospike/evaluation-features.conf
}
```

- 在EE版本5.4中，增加了对
    - `vault:secret_in_vault`，以从 HashiCorp Vault 中获取 feature key 的内容。请参阅 [Optional security with Vault integration](https://docs.aerospike.com/docs/operations/configure/security/vault/) 。
    - 从环境变量 (例如 `env-64: FEATURES`) 读取 feature key。
- 在EE 5.5版本中，添加了对组合多个 feature-key 文件的支持。现在，该路径可以指示，目录，其中包含的所有文件都是 feature-key 文件。服务器将检查每个服务器的有效性，有效期，并将有效的服务器合并到功能集合中。这支持对新功能的限时试用。

如果要使用 Docker, Kubernetes 或 Helm 部署 Aerospike Enterprise Edition，它们都提供了用于将功能密钥传递到容器中的标志。

### Base64 - encoding the feature key file in an environment variable

您可以将 feature key 作为 secret 传递给环境变量，而不是将 feature key 防止在系统路径中。
```shell
export FOO=$(base64 ~/evaluation-features.conf)
```
然后在配置文件中：
```
service {
    feature-key-file env-b64:FOO
}
```

在启动时加载数据库功能时，将从命名的环境变量中读取 base64 编码的 feature key 并将其解码为二进制形式。然后您可以清除环境变量，直到下次启动数据库为止。