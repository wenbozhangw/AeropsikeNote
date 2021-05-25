### 动态配置参数详解

#### 一、service配置参数

| name | Default | tag               | Comment                        |
| ----------------------------- | ------- | ----------------- | ------------------------------ |

| batch-index-threads | #cpu | dynamic | 批处理索引响应工作线程数。在版本3.12+中，默认情况下将其设置为可用的CPU内核数。在以前的版本中，默认值为4。每个线程都有自己的队列。这些线程只处理通过套接字将批处理响应缓冲区发送回客户端。值范围：1-256。<br />动态设置 batch-index-threads 为16：<br />`asinfo -v "set-config:context=service;batch-index-threads=16"`<br />Tip: 3.12版本之前允许最大值为64。 |
| batch-max-buffers-per-queue | 255 | dynamic | 每个批处理索引队列中被标记为已满之前允许的128KB响应缓冲区的数量。批处理索引队列（每个批处理索引线程一个）可以具有多于`batch-max-buffers-per-queue`缓冲区，但是直到它低于该数量，它才会接收任何新的批处理。当所有的队列都大于`batch-max-buffers-per-queue`时，新的批处理请求将被拒绝，并且将在服务器上记录错误："*Failed to find active batch queue that is not full*."。<br />动态设置batch-max-buffers-per-queue为512：<br />`asinfo -v "set-config:context=service;batch-max-buffers-per-queue=512"` |
| batch-max-requests | 5000 | dynamic | 每个节点允许的最大key个数。<br />动态设置batch-max-requests为6000：<br />`asinfo -v "set-config:context=service;batch-max-requests=6000"` |
| batch-max-unused-buffers | 256 | dynamic | 缓冲池中允许的最大128KB响应缓冲区的个数。如果达到限制，则已完成的缓冲区将在批处理请求结束时销毁。对于大量的批量工作负载，建议增加这个配置参数，以避免不必要的破坏和重新创建缓冲区，这会影响CPU负载。<br />动态设置 batch-max-unused-buffers为512：<br />`asinfo -v "set-config:context=service;batch-max-unused-buffers=512"` |
| ~~batch-priority~~ | 200 | removed:4.4/dynamic | 生成前的顺序读取数。数字越大，优先级越高。仅适用于旧的batch direct协议，该协议已在版本4.4中删除。<br />动态设置 batch-priority为300：<br />`asinfo -v "set-config:context=service;batch-priority=300"` |
| batch-threads | 4 | removed:4.4/dynamic | batch direct 工作线程的数量。batch direct是旧版的批处理协议。该线程处理完整的批处理请求。所有批处理线程只有一个批处理队列。值范围：0-256。注意旧的batch direct协议已经在4.4版本移除。<br />动态设置 batch-threads 为8：<br />`asinfo -v "set-config:context=service;batch-threads=8"`<br />tips: 3.12版本之前最大值为 64。 |