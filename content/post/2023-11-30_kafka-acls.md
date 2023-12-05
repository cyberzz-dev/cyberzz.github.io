---
title: kafka acls的一些设置
description: 
date: 2023-11-29
slug: kafka_acls
---

根据confluent的介绍



## 资源类型分为三种：

- cluster
- topic
- group



## 集群资源操作 Cluster resource operations

| Operation       | Resource | APIs allowed                                                 |
| :-------------- | :------- | :----------------------------------------------------------- |
| Alter           | Cluster  | AlterReplicaLogDirs, CreateAcls, DeleteAcls                  |
| AlterConfigs    | Cluster  | AlterConfigs                                                 |
| ClusterAction   | Cluster  | Fetch (for replication only), LeaderAndIsr, OffsetForLeaderEpoch, StopReplica, UpdateMetadata, ControlledShutdown, WriteTxnMarkers |
| Create          | Cluster  | CreateTopics, Metadata                                       |
| Describe        | Cluster  | DescribeAcls, DescribeLogDirs, ListGroups                    |
| DescribeConfigs | Cluster  | DescribeConfigs                                              |



## topic资源操作 Topic resource type operations

| Operation       | Resource | APIs allowed                                             |
| :-------------- | :------- | :------------------------------------------------------- |
| Alter           | Topic    | CreatePartitions                                         |
| AlterConfigs    | Topic    | AlterConfigs                                             |
| Create          | Topic    | CreateTopics, Metadata                                   |
| Delete          | Topic    | DeleteRecords, DeleteTopics                              |
| Describe        | Topic    | ListOffsets, Metadata, OffsetFetch, OffsetForLeaderEpoch |
| DescribeConfigs | Topic    | DescribeConfigs                                          |
| Read            | Topic    | Fetch, OffsetCommit, TxnOffsetCommit                     |
| Write           | Topic    | Produce, AddPartitionsToTxn                              |



## 组资源操作 Group resource type operations

| Operation | Resource | APIs allowed                                                 |
| :-------- | :------- | :----------------------------------------------------------- |
| Delete    | Group    | DeleteGroups                                                 |
| Describe  | Group    | DescribeGroup, FindCoordinator, ListGroups                   |
| Read      | Group    | AddOffsetsToTxn, Heartbeat, JoinGroup, LeaveGroup, OffsetCommit, OffsetFetch, SyncGroup, TxnOffsetCommit |



## operation 操作分以下几类：

- Alter

- AlterConfigs

- Describe

- DescribeConfigs

- Read

- Write

- Create

- Delete

- ClusterAction

- IdempotentWrite

- All

  

## resource-pattern-type 类型分为以下：

- ANY
- MATCH
- LITERAL
- PREFIXED

完全匹配  LITERAL，Kafka 将尝试将完整资源名称（例如主题或消费者组）与 ACL 中指定的资源进行匹配。

前缀匹配 PREFIXED，Kafka 会尝试将资源名称的前缀与 ACL 中指定的资源进行匹配

在某些情况下，您可能需要使用星号 ( `*`) 来指定所有资源。



设置Alice对所有topic 和消费组的 的所有权限，如下： 

注：*必须加单引号

```shell
kafka-acls --bootstrap-server localhost:9092 \
--command-config adminclient-configs.conf \
--add \
--allow-principal User:Alice \
--operation All \
--topic '*' \
--group '*'  \
--cluster '*'
```



注意点，当 resource type 匹配为*时，可以不指定 resource-pattern-type 默认为(LITERAL)