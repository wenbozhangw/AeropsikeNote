# xdr 

## 配置说明
| name | default | Subcontext | Introduced | comment|
| ---- | ---- | ---- | ----| ----|
| ~~datecenter~~ | none | | `Enterprise` Removed : 5.0.0 | 此参数指定 XDR 的远程数据中心的名称。远程数据中心的名称是用户定义的。在 Aerospike 5.0 版本中，已被 dc 取代。 <br/> Example : <br/> 一下是数据中心参数的示例 ： <br/> `xdr {` <br/> `enable-xdr true` <br/> `datacenter DC1 {` <br/> `dc-node-address-port xx.xx.xx.xx 3000` <br/> `dc-node-address-port yy.yy.yy.yy 3000` <br/> `dc-node-address-port zz.zz.zz.zz 3000` <br/> `}` <br/> `}` |
| dc-connections | 64 | Subcontext : datacenter | 