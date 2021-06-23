## Batch

使用 Aerospike 批处理请求快速有效的检索大量记录。

批处理请求与查询不同，因为它们基于主键。该应用程序在批处理请求中传递主键列表。返回一系列的结果。

使用批处理请求可以：

- 有效地实现客户端联接。
- 存储并在计算中检索多个数据点。
- 确定 key 到期或 key 是否仍然存在。

Aerospike支持一下批处理请求：

- `batchGet`
- `batchExists`
- `batchGetHeaders`：仅读取记录元数据（expiration/TTL and generation）。不读取bins。
- `batchGetComplex`：批量读取具有不同命名空间，bin名称和读取类型的记录。

`batchGetComplex` requires Aerospike Server version 3.6.0+

##### Example

以下Java代码示例在一个批处理调用中获取多个记录。

```java
    void batchRead(
        AerospikeClient client,
        BatchPolicy policy,
        String namespace,
        String set,
        String binName,
        int size
        )throws Exception{
        Key[]keys=new Key[size];
        for(int i=0;i<size; i++){
        keys[i]=new Key(namespace,set,i+1);
        }

        Record[]records=client.get(policy,keys,binName);

        for(int i=0;i<records.length;i++){
        Key key=keys[i];
        Record record=records[i];

        if(record!=null){
        Object value=record.getValue(binName);
        // Process value here.
        }
        }
        }
```

---

### Implementation

客户端确定哪些记录在哪里，并为服务器创建批处理请求。批处理请求列出了要检索的主键和可选的bin名称。批处理请求是串行还是并行发生，具体取决于批处理策略。同步模式下的并行请求需要额外的线程，这些线程是从线程池创建或获取的。

批处理请求对每个服务器使用 single network transaction ，从而使开发人员不必并行处理请求。多个 keys
使用一个网络请求，这对于大量的小记录非常有好处，但是在每台服务器的记录数很小或每条记录返回的数据很大时却没有那么大的好处。请注意，批处理请求可能会增加某些请求的等待时间，但这仅是因为客户端通常会等到从服务器节点检索到所有 key
后才将控制权返回给调用方。

一些客户端（例如 c 客户端）支持在每条记录到达时立即将其交付，从而允许客户端应用程序首先处理最快的服务器中的数据。

---

### Batch Protocols

从 3.6.0 版本开始，Aerospike服务器支持两种批处理协议：Batch Direct 和 Batch Index。

#### Batch Index

Batch Index 允许单个请求使用 keys，namespaces， bin name filters，和 read types （read bins，read header，exists）的任意组合来检索记录。服务器通过在批处理中将
keys 作为 single record transaction 分发来处理此协议。 由于 batch index 使用与 single record reads 相同的 transaction
path，因此在集群迁移期间支持到其他服务器的代理。批处理请求的运行优先级也与 single record transaction 相同。

从 Aerospike 数据库 4.7 版本开始，[predicate expressions](https://www.aerospike.com/docs/guide/predicate.html) 可以应用于任何批处理操作。

Batch Index的过程为：

1. 接收来自客户端的批处理请求。
2. 将请求拆分为多个 single record 请求。
3. 在单个记录线程队列中分配这些单个记录请求。
4. 将单个响应在响应缓冲区中合并到128KB。
5. 将响应缓冲区放在批处理响应线程队列上，以使处理记录的线程不会阻塞。
6. 批处理响应线程将这些响应缓冲区返回给客户端。

如果连接的服务器不支持Batch Index，则直接使用Batch Direct协议。

服务器提供以下配置变量用于批处理索引性能调整。

| Name | Default | Max | Dynamic | Description | 
| --- | --- | --- | --- | --- |
| batch-max-request | 5000 |   | true | 每个节点允许的最大key数量。用于防止意外的大批量请求由于过多的内存消耗而导致服务器不稳定。 |
| batch-max-buffers-per-queue | 255  | | true | 每个batch index队列中允许的最大128KB响应缓冲区数。如果所有批处理索引队列都已满，则拒绝新的批处理请求。 |
| batch-max-unused-buffers | 256 | | true | 缓冲池中允许的最大128KB响应缓冲区数。如果达到限制，则将创建新缓冲区（如果在每个队列的 `batch-max-buffers-queue` 中），并在批处理请求结束时完成，则将其销毁。这基本上是缓冲池的大小。 |
| batch-index-threads | #cpu or 4 (3.12版本之前) | 64 | true | 批处理响应工作程序线程数。每个线程都有自己的队列。这些线程只处理通过套接字将批处理响应缓冲区发送回客户端。可以使用的最大内存计算为：`batch-index-threads * batch-max-buffer-per-queue * 128KB` 。 |

客户端可以在服务器运行时更改动态变量。

服务器还提供以下批处理索引统计信息变量（请参阅 [full metrics reference manual](https://docs.aerospike.com/docs/reference/metrics) ）：

| Name | Description |
| --- | --- |
| batch_index_initiate | 接受的批处理索引请求个数 |
| batch_index_queue | 每个批处理队列上剩余的批处理索引请求和响应缓冲区的数量。<br />格式: `<q1 requests> :<q1 buffers> ,<q2 requests> :<q2 buffers>,...` |
| batch_index_complete | 已完成的批处理索引数量 |
| batch_index_timeout | 批处理索引请求的超时时间 |
| batch_index_error | 由于错误而被拒绝的批处理索引请求数 |
| batch_index_unused_buffers | 缓冲池中可用的 128KB 响应缓冲区的数量 |
| batch_index_huge_buffers | 创建的临时响应缓冲区数量超过 128KB。当检索一个大于 128KB的记录时，将创建巨大的缓冲区。大量的记录不会从批处理中获益，而且会导致服务器上过多的内存混乱。 |
| batch_index_created_buffers | 创建的128KB响应缓冲区的数量。当池中没有缓冲区时，将创建响应缓冲区。如果此数目持续增加并且有可用的内存，那么将会增加 `batch-max-unused-buffers`。 |
| batch_index_destoryed_buffers | 销毁的128KB响应缓冲区数。当没有剩余的插槽可将响应缓冲区放回池时，响应缓冲区将被销毁。最大响应缓冲池大小是 `batch-max-unused-buffers`。 |
| batch-index | 批处理索引性能直方图。 |

对于 batch index sub transactions ：

| Name | Description | 
| --- | --- |
| batch_sub_proxy_complete | 已完成的代理 batch-index sub transactions 的数量。|
| batch_sub_proxy_error | 失败并发生错误的代理 batch-index sub transactions的数量。 |
| batch_sub_proxy_timeout | 代理的 batch-index sub transactions超时的数量。 |
| batch_sub_read_error | 因错误而失败的 batch-index read sub transactions数量。|
| batch_sub_read_not_found | 找不到结果的 batch-index read sub transactions的数量。 |
| batch_sub_read_success | 成功的 batch-index read sub transactions的数量。 |
| batch_sub_read_timeout | 超时的 batch-index read sub transactions的数量。 |
| batch_sub_tsvc_error | 在尝试处理 transaction 之前，由于 transaction service 中的错误而失败的 batch-index 读取sub transactions的数量。例如协议错误或安全权限不匹配。 |
| batch_sub_tsvc_timeout | 在尝试处理 transaction 之前，在 transaction service 中超时的 batch-index 读取sub transactions的数量。例如协议错误或安全权限不匹配。 |
| retransmit_all_batch_sub_dup_res | 正在 duplicate resolved 的batch sub transactions 期间发生的重传次数。注意，这包括在客户端和代理节点上发起的重传。 |
| retransmit_batch_sub_dup_res | 正在 duplicate resolved 的 batch sub transactions 期间发生的重传次数。在 4.5.1.5 版本被替换为`retransmit_all_batch_sub_dup_res` |
| early_tsvc_batch_sub_error | 在 transactions 早期 batch sub transactions 的错误数。例如，bad/unknown 命名空间或身份验证错误。 |


#### Batch Direct (legacy batch protocol supported on all server versions prior to 4.4)

bitch direct协议允许单个请求使用单个命名空间(single namespace)和 bin name filter 从多个key检索记录。服务器使用单独的批处理队列/线程 (separate batch queues/threads) 和单记录 transaction 队列/线程(single record transaction queues/threads )来处理此协议，这使批处理请求的优先级低于 single record transactions 。

请注意，服务器版本4.4或更高版本不支持批 batch direct 协议。

batch direct 过程为：

1. 从客户端接收批处理请求。
2. 将整个批处理请求放入批处理线程队列中。
3. 批处理线程处理整个批处理请求，包括 low-level 读取并将结果发送到客户端。

Aerospike使用直接的 low-level 数据库读取方式实现 batch direct 协议。当所有的 key 都在同一个命名空间中时，batch direct会更快；但是，有一个重要的缺点：Batch direct 不能把记录中 not found 错误代理到其他服务器节点。在将节点添加到集群或从集群中删除节点之后，以及正在迁移的记录和客户端分区映射更新（每秒一次）之间存在滞后时，可能会发生这种情况。

旧版客户端继续使用 batch direct 协议。较新的客户端默认为 batch index 协议（如果可用）。要在客户端中选择 batch direct，请改变批处理策略变量。例如，在 Aerospike Java 客户端中，启用 `BatchPolicy.useBatchDirect` 批处理策略变量。

服务器提供以下配置 batch direct 性能调整变量。

| Name | Default | Max | Dynamic | Description |
| --- | --- | --- | --- | --- |
| `batch-max-requests` | 5000 | - | true | 每个节点允许的最大 key 数量。防止由于大的批处理请求导致大量内存消耗而造成意外的服务器不稳定。 |
| `batch-priority` | 200 | - | true | Number of sequential reads before yielding.  数字越大，优先级越高。 | 
| `batch-threads` | 4 | 64 | true | batch direct 工作线程的数量，该线程处理完整的批处理请求。所有批处理线程只有一个批处理队列。 |

客户端可以在服务器运行时更改动态变量。

服务器还提供以下 batch direct 统计变量：

| Name | Description | 
| --- | --- |
| `batch-initiate` | 收到的 batch direct 请求数量。 |
| `batch-queue` | 队列中剩余的等待处理的 batch direct 请求数量。 |
| `batch_tree_count` | 所有的 batch direct 请求的tree查找数量。 |
| `batch_timeout` | batch direct请求超时的数量。 |
| `batch_errors` | 因错误而被拒绝的 batch direct 请求的数量。 |
| `batch_q_process` | batch direct 性能直方图。 |

---

### known limitations

批量写入不支持。

---

### References

有关特定于语言的示例，请参阅以下主题：

- [Java](https://docs.aerospike.com/docs/client/java/examples/application/batch.html)
- [C# .NET](https://docs.aerospike.com/docs/client/csharp/examples/application/batch.html)
- [C](https://docs.aerospike.com/docs/client/c/usage/kvs/batch.html)
- [Node.js](https://docs.aerospike.com/docs/client/nodejs/usage/kvs/read.html)
- [Go](https://docs.aerospike.com/docs/client/go/usage/kvs/batch.html)
- [Python](https://docs.aerospike.com/docs/client/python/usage/kvs/read.html#running-batch-operations)
- [PHP](https://docs.aerospike.com/docs/client/php/usage/kvs/read.html#batch-operations)
- [Ruby](https://docs.aerospike.com/docs/client/ruby/usage/kvs/batch.html)

---

### 提供一些批处理代码做参考

```
int
as_batch_init()
{
  // 加锁
	cf_mutex_init(&batch_resize_lock);

  // 如果没有指定，设置全局配置的 batch-index-threads 为 cpu的总个数。
	// Default 'batch-index-threads' can't be set before call to cf_topo_init().
	if (g_config.n_batch_index_threads == 0) {
		g_config.n_batch_index_threads = cf_topo_count_cpus();
	}

  // 日志
	cf_info(AS_BATCH, "starting %u batch-index-threads", g_config.n_batch_index_threads);

  // 初始as的批处理线程池
	int rc = as_thread_pool_init_fixed(&batch_thread_pool, g_config.n_batch_index_threads, as_batch_worker,
			sizeof(as_batch_work), offsetof(as_batch_work,complete));

	if (rc) {
		cf_warning(AS_BATCH, "Failed to initialize batch-index-threads to %u: %d", g_config.n_batch_index_threads, rc);
		return rc;
	}

  // 初始化批处理缓冲区池 ， BATCH_BLOCK_SIZE = 128K
	rc = as_buffer_pool_init(&batch_buffer_pool, sizeof(as_batch_buffer), BATCH_BLOCK_SIZE);

	if (rc) {
		cf_warning(AS_BATCH, "Failed to initialize batch buffer pool: %d", rc);
		return rc;
	}
	
  // 初始化批处理线程队列
	rc = as_batch_create_thread_queues(0, g_config.n_batch_index_threads);

	if (rc) {
		return rc;
	}

	return 0;
}

//-------------------------------------------------------------------------------------

int
as_thread_pool_init_fixed(as_thread_pool* pool, uint32_t thread_size, as_task_fn task_fn,
						  uint32_t task_size, uint32_t task_complete_offset)
{
  // 初始化互斥锁
	if (pthread_mutex_init(&pool->lock, NULL)) {
		return -1;
	}
	
  // 加锁
  if (pthread_mutex_lock(&pool->lock)) {
    return -2;
  }
	
	// Initialize queues.
  // 初始化调度队列，即任务队列
	pool->dispatch_queue = cf_queue_create(task_size, true);
  // 初始化完成队列
	pool->complete_queue = cf_queue_create(sizeof(uint32_t), true);
  // task function
	pool->task_fn = task_fn;
  // 完成回调
	pool->fini_fn = NULL;
  // task大小
	pool->task_size = task_size;
  // 偏移量
	pool->task_complete_offset = task_complete_offset;
  // 线程个数
	pool->thread_size = thread_size;
  // 已初始化
	pool->initialized = 1;
	
	// Start detached threads.
  // 启动线程
	pool->thread_size = as_thread_pool_create_threads(pool, thread_size);
	int rc = (pool->thread_size == thread_size)? 0 : -3;
	
  // 释放锁
	pthread_mutex_unlock(&pool->lock);
	return rc;
}

//-------------------------------------------------------------------------------------

static int
as_batch_create_thread_queues(uint32_t begin, uint32_t end)
{
	// Allocate one queue per batch response worker thread.
  // 每个批处理响应工作线程分配一个队列。
	int status = 0;

	as_batch_work work;
	work.complete = false;

	for (uint32_t i = begin; i < end; i++) {
		work.batch_queue = &batch_queues[i];
		work.batch_queue->response_queue = cf_queue_create(sizeof(as_batch_response), true);
		work.batch_queue->complete_queue = cf_queue_create(sizeof(uint32_t), true);
		work.batch_queue->tran_count = 0;
		work.batch_queue->delay_count = 0;
		work.batch_queue->active = true;
		
    // 初始化任务
		int rc = as_thread_pool_queue_task_fixed(&batch_thread_pool, &work);

		if (rc) {
			cf_warning(AS_BATCH, "Failed to create batch thread %u: %d", i, rc);
			status = rc;
		}
	}
	return status;
}

```