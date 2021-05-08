## logging 参数配置

| name | default | subcontext | tag | note |
|--- | --- | --- | --- |---|
| context | any critical | file | | 指定要记录的日志的上下文和级别。您可以结合使用 contexts 和 logging level。有关更改日志级别的详细信息，请参阅 [Changing Log Levels](https://docs.aerospike.com/docs/operations/manage/log/index.html#changing-logging-levels) . <br/> 可以从命令中获得不同的 contexts 及其 logging levels ： <br/> `asinfo -v log/0` <br/> 简要输入: <br/> `misc:CRITICAL;alloc:CRTITICAL;arenax:CRITICAL;...` <br/> 在 Aerospike Server 4.9 之前，默认的日志级别为 `INFO`: <br/> `misc:INFO;alloc:INFO;arenax:INFO;...` <br/> 支持以下日志级别 : <br/> - context any info <br/> - context any debug <br/> - context any warning <br/> - context any critical <br/> - context any detail <br/> 有关常见日志消息的详细信息和 contexts 的完整列表，参见 [Server Log Messages Reference Manual](https://docs.aerospike.com/docs/reference/serverlogmessages) .|