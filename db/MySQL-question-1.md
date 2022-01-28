


# MySQL半同步延迟导致数据不一致-线上问题

* [MySQL半同步延迟导致数据不一致-线上问题](#mysql半同步延迟导致数据不一致-线上问题)
  * [问题描述](#问题描述)
    * [背景](#背景)
  * [线上问题分析](#线上问题分析)
  * [发现上述问题后改进](#发现上述问题后改进)

[toc]

## 问题描述

### 背景

**订单流程概述**：

1. 数据保存DB库后，DB库发送binlog同步给Binlog同步服务;
2. Binlog同步服务反查MySQL库进行异构数据；
3. Binlog同步服务保存订单信息到ES集群；

**线上问题现象**：订单数据保存MySQL库后异构数据同步ES集群流程中，在同步ES集群服务从Mysql反查出的数据与binlog同步的数据不一致。从而导致ES库保存的数据值与DB库中的不一致。

## 线上问题分析

问题订单流程,如图：

<img src="https://github.com/craftlook/Note/blob/master/image/db/Mysql-question-1.png" width="80%" heigth="80%">

**机器分布情况：**

* **MySQL-Master**: 数据库主库部署在A机房；

* **MySQL-Master-ReadOnly**:数据库从库部署在A机房，提供大数据抽数、隔离读流量；(与主库同属A机房，不配置半同步) 

* **MySQL-Master-Replica**:数据库从库部署在B机房，提供大数据抽数、隔离读流量；(与主库分属不同机房，作为主库的备库存在，当A机房异常可切换B机房机器，保证高可用，所以配置了半同步相关配置。半同步已B机房的从库为准)  

* **Binlake同步转发服务**：接收DB库的binlog信息，转发JMQ给应用系统；

* **Binlog同步服务**：接收Binlog信息，反查DB主库（反查：尽可能保存ES集群的数据是最新的），异构数据保存ES集群

**问题数据详细流程：（序号对应图中内容）**

1. MySQL-Master送binlog信息。这里Binlog同步服务和MySQL-Slave-Replica接收binlog信息进行数据同步；
2. Binlake同步转发服务发送消息和MySQL-Slave-Replica进行数据复制同步进行；
3. Binlog同步服务反查主库异构数据；
4. MySQL-Slave-Replica复制数据 ACK完毕；

**流程问题分析**：问题的主要原因是由于，binlog消息发送到同步服务较快而半同步的ACK响应延迟（导致主库对应的数据落磁盘延迟-参考[半同步机制详情](https://github.com/craftlook/Note/blob/master/db/MySQL-semi.md)与[更新流程](https://github.com/craftlook/Note/blob/master/db/MySQL-update.md)）。同步服务反查主库时数据不是最新的，导致保存ES数据与DB数据不一致。

**流程结构问题**：Binlog同步服务接收的binlog数据是从主库获取，主库的binlog发送消息比较快速。线上正常应接收从库的binlog消息（目的：减缓消息的速度）

## 发现上述问题后改进

对流程结构不合理的配置进行调整：

<img src="https://github.com/craftlook/Note/blob/master/image/db/Mysql-question-1-answer.png" width="80%" heigth="80%">

**流程改进**：Binlake同步转发服务对接readOnly的从库消息，Binlog同步服务应用进行binlog消息的延迟消费。（通过减缓binlog的消息消费速度，尽量避免半同步ACK延迟的问题）

**改进后的效果**：通过上述改进，订单同步不一致的问题有所减少，并未完全解决线上问题。半同步ACK不间断的发生性能抖动和延迟。对上述的现象，曾考虑将binlog 迁移到B机房的从库上，但由于保证binlog信息同步的一致性必须要正在一个机房内（binlog从库与对外消息放在同一个机房避免网络延迟和断联问题）。

**最终方案**：binlog同步服务接收binlog信息后，获取binlog信息的时间戳与反查DB的数据的时间戳进行对比，如果binlog时间戳大于DB时间戳，则进行消息重试；反之进行ES集群的数据同步

