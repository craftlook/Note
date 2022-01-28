# MySQL 半同步
* [MySQL 半同步](#mysql-半同步)
  * [MySQL支持的复制方式](#mysql支持的复制方式)
  * [主从复制原理](#主从复制原理)
  * [MySQL主从复制模式](#mysql主从复制模式)
    * [异步模式](#异步模式)
    * [半同步模式](#半同步模式)
      * [半同步隐患](#半同步隐患)
    * [半同步增强-新半同步](#半同步增强-新半同步)
    * [GTID模式](#gtid模式)
      * [GTID的工作原理](#gtid的工作原理)
      * [GTID的优劣势](#gtid的优劣势)

[toc]

## MySQL支持的复制方式

MySQL支持三种复制方式：

- 基于语句的复制（也称为逻辑复制）主要是指，在主数据库上执行的SQL语句，在从数据库上会重复执行一遍 。MySQL默认采用的就是这种复制，效率比较高。但是也是有一定的问题的，如果SQL中使用uuid()、rand()等函数，那么复制到从库的数据就会有偏差。
- 基于行的复制，指将更新处理后的数据复制到从数据库，而不是执行一边语句 。从MySQL5.1的版本才被支持。
- 混合复制，默认采用语句复制，当发现语句不能进行精准复制数据时（例如语句中含有uuid()、rand()等函数），采用基于行的复制 。

## 主从复制原理

MySQL的复制原理概述上来讲大体可以分为这三步

1. 在主库上把数据更改，记录到二进制日志（Binary Log）中。
2. 从库将主库上的日志复制到自己的中继日志（Relay Log）中。
3. 备库读取中继日志中的事件，将其重放到备库数据之上。

如图：

<img src="https://github.com/craftlook/Note/blob/master/image/db/mysql-binlog-syn.png" width="80%" heigth="80%">

复制的这三步：

**第一步**：是在主库上记录二进制日志，首先主库要开启binlog日志记录功能，并授权Slave从库可以访问的权限。这里需要注意的一点就是binlog的日志里的顺序是按照事务提交的顺序来记录的而非每条语句的执行顺序。

**第二步**：从库将binLog复制到其本地的RelayLog中。首先从库会启动一个工作线程，称为I/O线程，I/O线程跟主库建立一个普通的客户端连接，然后主库上启动一个特殊的二进制转储（binlog dump）线程，此转储线程会读取binlog中的事件。当追赶上主库后，会进行休眠，直到主库通知有新的更新语句时才继续被唤醒。

这样通过从库上的I/O线程和主库上的binlog dump线程，就将binlog数据传输到从库上的relaylog中了。

**第三步**：从库中启动一个SQL线程，从relaylog中读取事件并在备库中执行，从而实现备库数据的更新。

## MySQL主从复制模式

MySQL的主从复制其实是支持， **全同步复制**、**异步复制** 、 **半同步复制** 、 **GTID复制** 等多种复制模式的。

**全同步复制**：主库提交事务之后，所有的从库节点必须收到，APPLY并且提交这些事务，然后主库线程才能继续做后续操作。这里面有一个很明显的缺点就是，主库完成一个事务的时间被拉长，性能降低。

* **缺点**:等到所有从库都执行完，执行过程中会被阻塞，等待返回结果，所以性能上会有很严重的影响。

**异步复制**：主库将事务Binlog事件写入到Binlog文件中，此时主库只会通知一下Dump线程发送这些新的Binlog，然后主库就会继续处理提交操作，而此时不会保证这些Binlog传到任何一个从库节点上。

* **缺点**：主库执行的binlog还没同步到从库时，主库挂了，这个时候从库就就会被强行提升为主库，这个时候就有可能造成数据丢失。

**半同步复制**：半同步复制，是介于全同步复制和异步复制之间的一种，主库只需要等待至少一个从库节点收到并且Flush Binlog到Relay Log文件即可，主库不需要等待所有从库给主库反馈。同时，这里只是一个收到的反馈，而不是已经完全执行并且提交的反馈，这样就节省了很多时间。

* **对比**：半同步复制模式，比异步模式提高了数据的可用性，但是也产生了一定的性能延迟，最少要一个TCP/IP连接的往返时间

### 异步模式

MySQL的默认复制模式就是异步模式，主要是 指MySQL的主服务器上的I/O线程，将数据写到binlong中就直接返回给客户端数据更新成功，不考虑数据是否传输到从服务器，以及是否写入到relaylog中 。在这种模式下，复制数据其实是有风险的，一旦数据只写到了主库的binlog中还没来得及同步到从库时，就会造成数据的丢失。

但是这种模式确也是效率最高的，因为变更数据的功能都只是在主库中完成就可以了，从库复制数据不会影响到主库的写数据操作。

<img src="https://github.com/craftlook/Note/blob/master/image/db/mysql-binlog-asyn.png" width="80%" heigth="80%">

上面我也说了，这种异步复制模式虽然效率高，但是数据丢失的风险很大，所以就有了后面要介绍的半同步复制模式。

### 半同步模式

MySQL从 **5.5** 版本开始通过以插件的形式开始支持半同步的主从复制模式。什么是半同步主从复制模式呢？

半同步复制模式，可以说是介于异步和同步之间的一种复制模式，主库在执行完客户端提交的事务后，要等待至少一个从库接收到binlog并将数据写入到relay log中才返回给客户端成功结果。

半同步复制模式，可以很明确的知道，在一个事务提交成功之后，此事务至少会存在于两个地方一个是主库一个是从库中的某一个。 **主要原理是**，在master的dump线程去通知从库时，增加了一个ACK机制，也就是会确认从库是否收到事务的标志码，master的dump线程不但要发送binlog到从库，还有负责接收slave的ACK。**当出现异常时，Slave没有ACK事务，那么将自动降级为异步复制，直到异常修复后再自动变为半同步复制**

流程图：

<img src="https://github.com/craftlook/Note/blob/master/image/db/mysql-binlog-semiA.png" width="80%" heigth="80%">

#### 半同步隐患

半同步复制模式也存在一定的数据风险，当事务在主库提交完后等待从库ACK的过程中，如果Master宕机了，这个时候就会有两种情况的问题。

* **事务还没发送到Slave上** ：若事务还没发送Slave上，客户端在收到失败结果后，会重新提交事务，因为重新提交的事务是在新的Master上执行的，所以会执行成功，后面若是之前的Master恢复后，会以Slave的身份加入到集群中，这个时候，之前的事务就会被执行两次，第一次是之前此台机器作为Master的时候执行的，第二次是做为Slave后从主库中同步过来的。

* **事务已经同步到Slave上** ：因为事务已经同步到Slave了，所以当客户端收到失败结果后，再次提交事务，你那么此事务就会在当前Slave机器上执行两次。

### 半同步增强-新半同步

为了解决上面的隐患，MySQL从5.7版本开始，增加了一种新的半同步方式。新的半同步方式的执行过程是将“ **Storage Commit** ”这一步移动到了“ **Write Slave dump** ”后面。这样保证了 **只有Slave的事务ACK后，才提交主库事务** 。MySQL 5.7.2版本新增了一个参数来进行配置： rpl_semi_sync_master_wait_point ，此参数有两个值可配置：

- **AFTER_SYNC** ：参数值为AFTER_SYNC时，代表采用的是新的半同步复制方式。
- **AFTER_COMMIT** ：代表采用的是之前的旧方式的半同步复制模式。

流程如图：

<img src="https://github.com/craftlook/Note/blob/master/image/db/mysql-binlog-semiB.png" width="80%" heigth="80%">

MySQL从5.7.2版本开始，默认的半同步复制方式就是 AFTER_SYNC 方式了，但是方案不是万能的，因为 AFTER_SYNC 方式是在事务同步到Slave后才提交主库的事务的，若是当主库等待Slave同步成功的过程中Master挂了，这个Master事务提交就失败了，客户端也收到了事务执行失败的结果了，但是Slave上已经将binLog的内容写到Relay Log里了，这个时候，Slave数据就会多了，但是多了数据一般问题不算严重，多了总比少了好。MySQL，在没办法解决分布式数据一致性问题的情况下，它能保证的是不丢数据，多了数据总比丢数据要好。

这里说几个的半同步复制模式的参数：

```text
mysql> show variables like '%Rpl%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | ON         |
| rpl_semi_sync_master_timeout              | 10000      |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
| rpl_stop_slave_timeout                    | 31536000   |
+-------------------------------------------+------------+
```

* **rpl_semi_sync_master_enabled**:半同步复制模式开关
* **rpl_semi_sync_master_timeout**:半同步复制，超时时间，单位毫秒，当超过此时间后，自动切换为异步复制模式。
* **rpl_semi_sync_master_wait_for_slave_count**：MySQL 5.7.3引入的，该变量设置主需要等待多少个slave应答，才能返回给客户端，默认为1。
* **rpl_semi_sync_master_wait_no_slave**：此值代表当前集群中的slave数量是否还能够满足当前配置的半同步复制模式，默认为ON，当不满足半同步复制模式后，全部Slave切换到异步复制，此值也会变为OFF。
* **rpl_semi_sync_master_wait_point**：代表半同步复制提交事务的方式，5.7.2之后，默认为AFTER_SYNC。

### GTID模式

MySQL从5.6版本开始推出了GTID复制模式，GTID即全局事务ID (global transaction identifier)的简称，GTID是由UUID+TransactionId组成的，UUID是单个MySQL实例的唯一标识，在第一次启动MySQL实例时会自动生成一个server_uuid, 并且默认写入到数据目录下的auto.cnf(mysql/data/auto.cnf)文件里。TransactionId是该MySQL上执行事务的数量，随着事务数量增加而递增。这样保证了 **GTID在一组复制中，全局唯一** 。

**优点**：这样通过GTID可以清晰地看到，当前事务是从哪个实例上提交的，提交的第多少个事务。

```text
mysql> show master status;
+-----------+----------+--------------+------------------+-------------------------------------------+
| File      | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                         |
+-----------+----------+--------------+------------------+-------------------------------------------+
| on.000003 |      187 |              |                  | 76147e28-8086-4f8c-9f98-1cf33d92978d:1-322|
+-----------+----------+--------------+------------------+-------------------------------------------+
1 row in set (0.00 sec)
```

* **GTID**：76147e28-8086-4f8c-9f98-1cf33d92978d:1-322

* **UUID**：76147e28-8086-4f8c-9f98-1cf33d92978d

* **TransactionId**:1-322

#### GTID的工作原理

由于GTID在一组主从复制集群中的唯一性，从而保证了每个GTID的事务只在一个MySQL上执行一次。

当从服务器连接主服务器时，把自己执行过的GTID（ **Executed_Gtid_Set: 即已经执行的事务编码** ）以及获取到GTID（ **Retrieved_Gtid_Set: 即从库已经接收到主库的事务编号** ）都传给主服务器。主服务器会从服务器缺少的GTID以及对应的transactionID都发送给从服务器，让从服务器补全数据。当主服务器宕机时，会找出同步数据最成功的那台conf服务器，直接将它提升为主服务器。若是强制要求某一台不是同步最成功的一台从服务器为主，会先通过change命令到最成功的那台服务器，将GTID进行补全，然后再把强制要求的那台机器提升为主。

主要数据同步机制可以分为这几步：

- master更新数据时，在事务前生产GTID，一同记录到binlog中。
- slave端的i/o线程，将变更的binlog写入到relay log中。
- sql线程从relay log中获取GTID，然后对比Slave端的binlog是否有记录。
- 如果有记录，说明该GTID的事务已经执行，slave会忽略该GTID。
- 如果没有记录，Slave会从relay log中执行该GTID事务，并记录到binlog。
- 在解析过程中，判断是否有主键，如果没有主键就使用二级索引，再没有二级索引就扫描全表。

**初始结构如下图**:

<img src="https://github.com/craftlook/Note/blob/master/image/db/mysql-gtid.png" width="80%" heigth="80%">

**当Master出现宕机后，就会演变成下图**:

<img src="https://github.com/craftlook/Note/blob/master/image/db/mysql-gtid.png" width="80%" heigth="80%">

通过上图我们可以看出来，当Master挂掉后，Slave-1执行完了Master的事务，Slave-2延时一些，所以没有执行完Master的事务，这个时候提升Slave-1为主，Slave-2连接了新主（Slave-1）后，将最新的GTID传给新主，然后Slave-1就从这个GTID的下一个GTID开始发送事务给Slave-2。 这种自我寻找复制位置的模式减少事务丢失的可能性以及故障恢复的时间。

#### GTID的优劣势

**GTID的优势**：

- 每一个事务对应一个执行ID，一个GTID在一个服务器上只会执行一次;
- GTID是用来代替传统复制的方法，GTID复制与普通复制模式的最大不同就是不需要指定二进制文件名和位置;
- 减少手工干预和降低服务故障时间，当主机挂了之后通过软件从众多的备机中提升一台备机为主机;

**GTID的缺点**：

- 首先不支持非事务的存储引擎；
- 不支持create table ... select 语句复制(主库直接报错);(原理: 会生成两个sql, 一个是DDL创建表SQL, 一个是insert into 插入数据的sql; 由于DDL会导致自动提交, 所以这个sql至少需要两个GTID, 但是GTID模式下, 只能给这个sql生成一个GTID)
- 不允许一个SQL同时更新一个事务引擎表和非事务引擎表;
- 在一个MySQL复制群组中，要求全部开启GTID或关闭GTID。
- 开启GTID需要重启 (mysql5.7除外);
- 开启GTID后，就不再使用原来的传统复制方式（不像半同步复制，半同步复制失败后，可以降级到异步复制）;
- 对于create temporary table 和 drop temporary table语句不支持;
- 不支持sql_slave_skip_counter;



最后说几个开启GTID的必备条件：

- MySQL 5.6 版本，在my.cnf文件中添加:

```text
gtid_mode=on (必选)                    #开启gtid功能
log_bin=log-bin=mysql-bin (必选)       #开启binlog二进制日志功能
log-slave-updates=1 (必选)             #也可以将1写为on
enforce-gtid-consistency=1 (必选)      #也可以将1写为on
```

- MySQL 5.7或更高版本，在my.cnf文件中添加:

```text
gtid_mode=on    (必选)
enforce-gtid-consistency=1  （必选）
log_bin=mysql-bin           （可选）    #高可用切换，最好开启该功能
log-slave-updates=1     （可选）       #高可用切换，最好打开该功能
```

