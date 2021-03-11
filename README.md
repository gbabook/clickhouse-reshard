# clickhouse-reshard
Clickhouse 增加分片后，官方文档建议把新分片的权重设大。这样新数据会优先写入新的分片，整体数据逐渐平衡后再把权重设置回来。

加新机器的目的，大都是集群负载变高。但是按照官方建议来操作显而易见集群上的老数据并未发生任何变化，短时间内并未达到降低集群负载的目的。

如何把旧数据较为平均的 reshard 到新机器上？

洗澡的时候思考了一阵，以下是我的思路，抛砖引玉：

PS：副本是另一个方向，是 ck 和 zookeeper 交互的事情，故以下不会提到副本。

单分片转 2 分片
----------
也就是原来是单机，现在用两台机器，每台机器一个分片。

假设线上有一台 ck 单机，名字是 node-1，本地数据都放在数据库 local 下。现在需要增加一台 node-2。

首先在 node-2 根据 node-1 的结构建立同样的本地表。

然后在 node-1、node-2 的 config.xml 中，配置 remote_servers：
```xml
<remote_servers>
    <cluster_2s_1r>
        <shard>
            <internal_replication>true</internal_replication>
            <replica>
                <host>node-1</host>
                <port>9000</port>
                <user>default</user>
                <password></password>
            </replica>
        </shard>
    </cluster_2s_1r>
</remote_servers>
```
这个所谓的集群，就只有 node-1 自己，目的是在 node-2 建立分布式表，从网络上直接读取 node-1 的数据。

