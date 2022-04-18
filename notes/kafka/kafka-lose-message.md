---
layout:     note
title:      "kafka 避免消息丢失"
subtitle:   " \"kafka 避免消息丢失\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- kafka
- note
---


### 消息丢失

**broker**

- `replication.factor` ，指定topic 副本数量 > 3

- `unclean.leader.election.enable=false`，禁止`ISR`中落后太多的副本成为`leader`，默认为`false`

- `min.insync.replicas`，当生产者设置`acks=all`时，必须满足此配置的写入副本数量消息才算发送成功，确保`replication.factor` > `min.insync.replicas`，典型的例子是`replication.factor=3`，`min.insync.replicas=2`

  ```
  #min.insync.replicas 在创建完topic后使用脚本配置
  $./bin/kafka-configs.sh --bootstrap-server localhost:9092 --alter --entity-type topics --entity-name topic-name --add-config min.insync.replicas=2
  ```



**producer**

- `acks=all/-1`，`min.insync.replicas`数量的副本写入成功才算成功
- `retries` > 0，消息发送重试次数
- 发送消息时要确认是否发送成功，使用带有回调的send，producer.send(msg, callback)




**consumer**

- 位移提交要注意，事前提交会丢失