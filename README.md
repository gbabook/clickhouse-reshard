# clickhouse-reshard
Clickhouse 增加分片后，官方文档建议把新分片的权重设大。这样新数据会优先写入新的分片，整体数据逐渐平衡后再把权重设置回来。

加新机器的目的，大都是集群负载变高。但是按照官方建议来操作显而易见集群上的老数据并未发生任何变化，短时间内并未达到降低集群负载的目的。

如何把旧数据较为平均的 reshard 到新机器上？

洗澡的时候思考了一阵，以下是我的思路，抛砖引玉：

*PS：副本是另一个方向，是 ck 和 zookeeper 交互的事情，故以下不会提到副本。*

单分片转 2 分片
----------
也就是原来是单机，现在用两台机器，每台机器一个分片。

假设线上有一台 ck 单机，名字是 node-1，本地数据都放在数据库 local 下。现在需要增加一台 node-2。

首先在 node-2 根据 node-1 的结构建立同样的本地表。

然后在 node-1、node-2 的 `config.xml` 中，配置 `remote_servers`：
```xml
<remote_servers>
    <!-- 集群命名 cluster_2s_1r，随意 -->
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
目前这个所谓的集群，暂时就只有 node-1 自己，目的是在 node-2 通过分布式表从网络上直接读取 node-1 的数据。

接下来在 node-2 的 default 数据库建立分布式表：
```SQL
CREATE TABLE default.cs_100_1
(字段...)
ENGINE = Distributed('cluster_2s_1r', 'local', 'cs_100_1', rand())
```
`Distributed` 第一个参数：`cluster_2s_1r` 是 `config.xml` 中集群的名字；

第二个参数：`local` 是本机本地数据库名；

第三个参数：`cs_100_1` 是对应 `local` 库本地表的表名。

*PS：ck 每张表的数据，大部分都是事件日志，都会有时间字段，拆分数据就依靠这个字段。*

准备工作完毕，接下来就开始把 node-1 的数据尽量平分到 node-2：

在 node-2 的 clickhouse-client 中执行：
```SQL
INSERT INTO local.cs_100_1 SELECT * FROM default.cs_100_1 WHERE (toSecond(time) % 2) != 0
```
`node-2` 上面的 `default.cs_100_1` 是分布式表，数据源就只有 `node-1`

`toSecond` 函数会把时间字段转化为以秒为单位的数字，范围是 0-59。

很显然，切割的逻辑很简单粗暴：node-1 保存秒数是偶数的数据，node-2 保存秒数是奇数的数据。

处理很快，单机 cs_100_1 数据接近一亿，十秒不到就完成了（当然了，这是在我本地模拟的两台 ck，CPU、内存容量、硬盘都还不错）。

完成之后就要把 node-1 的 cs_100_1 表中，时间是奇数秒的数据给删了：

在 node-1 的 clickhouse-client 中执行
```SQL
ALTER TABLE local.cs_100_1 DELETE WHERE (toSecond(time) % 2) != 0
```
因为是 ck 的删除是异步操作，立即就会返回完成，所以别忘了适时检查下是不是删除都完成了。
```SQL
SELECT COUNT(*) FROM local.cs_100_1 WHERE (toSecond(time) % 2) != 0;
```

至此 node-1 单机的 local.cs_100_1 表数据就还算平均地分到 node-2 上了。

接下来就是配置 `remote_servers`，设置第二个分片，把 `node-2` 加进去。

同样 node-1 的 default 库也建立好分布式表，完成之后两台机器就可以随意挑一台来用了。


