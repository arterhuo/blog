**KAFKA跨机房添加broker节点迁移**
前提条件：
 - 两个机房网络必须全部打通
 - zookeeper需要迁移，考虑网络导致脑裂问题，业务迁移完成之后再进行zk迁移（可提前加入readonly到新机房）

1、topics.txt添加要迁移的topic
2、生成topics-to-move.json
```#!/usr/bin/env python
import json
topic_file="topics.txt"
topic_to_move = {"topics":
[
],"version":1
}
with open("topics.txt",'rb') as f:
    for topic in f:
        topic_to_move["topics"].append({"topic": topic.strip()})

print json.dumps(topic_to_move)
```
3、先生成推荐topic去的broker节点对应的json
比如：
```
kafka-reassign-partitions --zookeeper 10.26.234.167:2181/kafka/ka --topics-to-move-json-file topics-to-move.json --broker-list "1,2,3" --generate
```
4、通过修改上面的json内容，来指定topic的partions分部在哪些机器上，并且可指定Leader在哪一个broker上(此步骤只有在3步骤生成推荐时，新broker数量少于旧broker数量时使用)
```
import json
f=open("to_balance2.json",'rb')
content=json.load(f)

for topic in content.get("partitions"):
    rep = topic["replicas"]
    if len(rep) == 1:
        topic["replicas"][0]=3
    elif len(rep) > 2:
        if 3 in rep:
            topic["replicas"].remove(3)
            topic["replicas"].insert(0,3)
        else:
            topic["replicas"][0] = 3
print json.dumps(content)
```
5、通过新的balance.json进行分区行动,可通过throttle进行限速，修改限速可以同样命令不同阀值重复执行
```
kafka-reassign-partitions --zookeeper 10.26.234.167:2181/kafka/ka --reassignment-json-file to_balance.json   --execute   --throttle 10000000
```
6、检查移动进度
```
kafka-reassign-partitions --zookeeper 10.26.234.167:2181/kafka/ka --reassignment-json-file to_balance.json  --verify 进行验证，并remove throttle
```
7、待分区移动完成时，新节点未接管为leader时，可以手动执行选举
```
kafka-preferred-replica-election  --zookeeper 10.26.234.167:2181/kafka/ka
```
