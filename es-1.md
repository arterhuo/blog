# ES集群是所有数据的存储和查询

**问题**

1.严格控制集群shard的个数，shard的大小以及文档数量

es在执行index的创建删除的过程中的时候，时间复杂度应当是和shard的数量是正相关的，所以如果shard数量过大，将会导致的是索引的管理
耗时十分长，管理起来十分的困难。目前使用下来shard的总数尽量不要超过3w左右。同时shard数量过多，直接导致的是segement过多，查询
的时候需要遍历的shard直接变多，导致查询效率显著下降，所以希望还是严格控制shard的数量，保证每个索引shard数量处于一个合理的范围，
如果索引总体较大可以加大shard数量，如果较小那么就降低shard数量，使用下来单个shard大小在30-50G左右比较合适，过大写入较慢，I/O问题
如果shard小于50G的，最好至少分3个shard，过少shard，可能会导致热点数据查询集中在一台机器，导致内存溢出
同时要注意单个shard的文档数量不能超过lucene一个实例的文档数限制。

2.注意index的mapping

我们使用mapping的时候要注意字段不能过多，会加载到内存当中，否则有可能出现OOM的情况，同时最主要的要关注json格式的数据，
因为如果json的字段不断发生变化，那么有可能在不断的update_mapping，可以通过_cat/pending_task观察下当前的任务，如果有大量的update_mapping
就有可能是某个索引的字段在不断发生变化。

3.如何正确的做es集群的维护

ES集群的维护过程中，我们先了解下ES数据恢复的流程，ES在进行数据恢复的过程当中，首先primary shard是直接从本地磁盘进行恢复，replica shard的
恢复则不一定，因为Recovery的恢复是对比主副本的segment flush id来判断这两个shard是不是一致的，如果一致那么这个replica shard也将从本地恢复，如果
发现不一致将会从网络恢复，但是不同节点的segment merge其实是独立运行的，所以会导致一些shard的数据看上去主副本其实一样，但其实segment flush id
不同，只能够副本从网络恢复。其实在ES中，一个shard一段时间不更新后，会自动进行一次synced flush来同步一个synced flush id，所以我们在做集群维护的
过程中，最重要的就是要手动同步一次。那么完整的操作如下：

       1.停止消费hangout，不再向ES写数据。

       2.关闭集群shard allocation

       3.POST /_flush/synced强制flush

       4.重启ES

       5.打开集群shard allocation

       6.等待health变为green，打开hangout写数据。 

4.  rebalance和recovery条件限制:

    1.有的时候我们发生故障重启了集群，但是集群中节点的加入是有时间的先后顺序，那么很有可能数据直接开始recovery，但是这个时候数据并不完整，节点之间很有可能
       已经开始进行网络恢复，然后导致的是数据分布不均匀，再出发rebalance导致数据一致漂移，造成没有意义的网络和磁盘I/O。
       那么对应这种情况，我们需要调整recovery的相关参数和rebalance的相关参数来限制一下recovery和rebalance的条件，比如gateway的设置:

          gateway.expected_nodes
          gateway.expected_master_nodes
          gateway.expected_data_nodes
          gateway.recover_after_nodes
          gateway.recover_after_master_nodes
          gateway.recover_after_data_nodes
          gateway.recover_after_time: 5m
       具体参数不详细讲解，主要就是限制加入的节点数和时间，再达到一定的条件后再开始进行recovery，给定一定的缓冲时间。

    2.再一个问题就是如果由于网络原因，可能有一些节点暂时脱离集群，这个时候我们并不希望直接开始进行recovery，因为节点很有可能很快加回节点，那么做下如下配置:

          index.unassigned.node_left.delayed_timeout: 5m
       等待5分钟后再开始进行recovery。

    3.限制的一些具体参数可以参考如下配置，比如恢复速率和并发数等等:

         "transient" : {
             "indices.recovery.max_bytes_per_sec" : "500mb",
             "indices.recovery.concurrent_streams" : "20",
             "cluster.routing.allocation.node_concurrent_recoveries": "20"
         }
         "transient" : {
             "cluster.routing.allocation.cluster_concurrent_rebalance": "20",
             "cluster.routing.rebalance.enable": "all"
        }
5.写入的优化

  目前我们所做的写入优化主要是针对translog进行的一些优化，具体的配置参数可以参考如下:

     "translog": {  "interval": "30s",  "flush_threshold_size": "3g",  "durability": "async",  "sync_interval": "30s" }    
主要是换成异步写入，并且增大flush的大小和间隔时间。因为本身日志是可丢的，异步存在丢失日志的风险，这个觉得是可以接受的。

6.段文件segment

如果存在一些保留时间很长的索引，由于segment的个数很多，对于这类索引，如果索引大小很大，可以定期进行一次optimize来强制进行合并，
原则上segment其实是个数越少所占内存越小，并且查询效率会显著提高，因为不需要去遍历多个segment，但是合并的过程也是很占资源，所以
在实际使用中不是很推荐，因为我们保存的日志时间很短，并没有太大的意义，但是对于保留时间较长的日志还是建议定期进行一次optimize，
需要注意的一点是，这个操作最好只对不再更新的索引执行。

7.发现discovery

  由于在某些负载较高的情况下，data节点很有可能直接脱离集群，那么建议提高discovery的检测时间，降低这种风险，主要配置可以参考:

    discovery.zen.fd.ping_timeout: 120s
    discovery.zen.fd.ping_retries: 6
    discovery.zen.fd.ping_interval: 30s

8.如何去检查一下index的状况

  这个可以直接用底层的lucene的去检查一下index的一些状态，比如index是否发生了丢失数据，数据是否完整等等。
```
  java -cp lucene-core-5.5.0.jar -ea:org.apache.lucene… org.apache.lucene.index.CheckIndex /data/elasticsearch/wg-lpd-es-apollo-dqs/nodes/0/indices/shipping_order_2016_12_05
```
9.索引的路由

  因为目前我们的场景下索引的格式非常多，超过4000个，那么索引的划分以及如果发生recovery或者rebalance的情况下，index中shard的漂移
是不可接受的。所以我们的建议是在物理集群中每10台机器打上一个tag划分成一个逻辑集群，然后再进行创建索引的过程中，通过allocation的
include将这个索引限制在这个逻辑集群当中，比如可以如果我们有200个索引，40台机器，我们可以将前20个最大的索引include到10台机器中，
将60个索引include到10台机器中。依次类推，只要保证每个逻辑集群中的总index的大小差异不是很大，限制住了索引的漂移，降低recovery和
rebalance的成本，部分配置可以参考:

    curl -XPUT localhost:9200/test/_settings -d '{
        "index.routing.allocation.include.tag" : "value1,value2"
    }'
    curl -XPUT localhost:9200/test/_settings -d '{
        "index.routing.allocation.include.group1" : "xxx",
        "index.routing.allocation.include.group2" : "yyy",
        "index.routing.allocation.exclude.group3" : "zzz",
    }'
    curl -XPUT localhost:9200/_cluster/settings -d '{
      "transient" : {
      "cluster.routing.allocation.exclude._ip" : "10.0.0.1"
        }
    }'

10. 跨域配置：

```
  http.cors.enabled: true
  http.cors.allow-origin: "*"
  http.cors.allow-methods: OPTIONS, HEAD, GET, POST
  http.cors.allow-headers: X-Requested-With, Content-Type, Content-Length, X-Auth-Token
```

11.节点发生异常

  如果某个节点你发现发生了异常，比如memory异常等等，可以尝试进行索引的手动迁移，将部分占用资源的索引迁移出去，降低
该节点的资源占用率，或者可以尝试通过api将该节点exclude出去，迁移所有的index，等待恢复后再include进来。

12.索引发生异常

  如果一个索引异常了，比如之前遇到的shard个数分的过少，我们可以尝试新建shard数目足够的索引进行reindex或者从kafka重新消费过滤出该索引的数据写入ES。

**改进**

1.双写问题

  考虑到未来集群可能会进行拆分，那么建议是两个集群中间通过tribe节点实现互联，这样前端不需要再做过多的配置，tribe节点会去找对应的索引的数据，
但是不要同一个索引两个集群中都有，因为tribe节点不会去做merge，但是考虑到我们的使用情况，我们只需要根据appid来进行es集群的拆分。

2.索引路由

  目前的索引路由还不完善，未来需要根据数据的大小，shard的数量来进行一定的索引路由，将对应的索引限制在对应的机器中，减少数据漂移。

3.双实例

  由于未来是单机双实例，所以要考虑到限制每个实例占用的资源以及同一个shard主副本不能够在同一台机器上，监控每个index的
segment的占用，一旦发生一个实例出现异常，很有可能出现雪崩效应导致整个集群挂掉。
