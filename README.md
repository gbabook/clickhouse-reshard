# clickhouse-reshard
Clickhouse 增加分片后，官方文档建议把新分片的权重设大。这样新数据会优先写入新的分片，整体数据逐渐平衡后再把权重设置回来。

加新机器的目的大都是集群负载变高，但是按照官方建议操作，显而易见集群上的老数据并未发生任何变化，短时间内并未达到降低集群负载的目的。

如何平均 reshard 到新机器上的问题，以下是我的思路，抛砖引玉：
