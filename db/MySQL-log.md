## <span id="mysql-log">MySQL 事务日志简介</span>

事务日志可以帮助提高事务的效率。使用事务日志，存储引擎在修改表的数据时只需要修改其内存拷贝，再把该修改行为记录到持久在硬盘上的事务日志中，而不用每次都将修改的数据本身持久到磁盘。事务日志采用的是追加的方式，因此写日志的操作是磁盘上一小块区域内的顺序I/O，而不像随机I/O需要在磁盘的多个地方移动磁头，所以采用事务日志的方式相对来说要快得多。事务日志持久以后，内存中被修改的数据在后台可以慢慢地刷回到磁盘。目前大多数存储引擎都是这样实现的，我们通常称之为预写式日志（Write-Ahead Logging），修改数据需要写两次磁盘。
 如果数据的修改已经记录到事务日志并持久化，但数据本身还没有写回磁盘，此时系统崩溃，存储引擎在重启时能够自动恢复这部分修改的数据。

MySQL Innodb中跟数据持久性、一致性有关的日志，有以下几种：

- Bin Log: mysql server层产生的日志。binlog是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。常用来进行数据恢复、数据库复制，常见的mysql主从架构，就是采用slave同步master的binlog实现的。binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。
- Redo Log:redo log 是 InnoDB 引擎特有的。redo log 是物理日志，记录的是“在某个数据页上做了什么修改”。记录了数据操作在物理层面的修改，mysql中使用了大量缓存，修改操作时会直接修改内存，而不是立刻修改磁盘，事务进行中时会不断的产生redo log，在事务提交时进行一次flush操作，保存到磁盘中。当数据库或主机失效重启时，会根据redo log进行数据的恢复，如果redo log中有事务提交，则进行事务提交修改数据。redo log是循环写的，空间固定会用完。
- Undo Log: 除了记录redo log外，当进行数据修改时还会记录undo log，undo log用于数据的撤回操作，它记录了修改的反向操作，比如，插入对应删除，修改对应修改为原来的数据，通过undo log可以实现事务回滚，并且可以根据undo log回溯到某个特定的版本的数据，实现MVCC
- Relay Log: 中继日志，一般情况下它在MySQL主从同步读写分离集群的从节点才开启。主节点一般不需要这个日志。master主节点的binlog传到slave从节点后，被写道relay log里，从节点的slave sql线程从relaylog里读取日志然后应用到slave从节点本地。从服务器I/O线程将主服务器的二进制日志读取过来记录到从服务器本地文件，然后SQL线程会读取relay-log日志的内容并应用到从服务器，从而使从服务器和主服务器的数据保持一致。
  它的作用可以参考如下图，从图片中可以看出，它是一个中介临时的日志文件，用于存储从master节点同步过来的binlog日志内容，它里面的内容和master节点的binlog日志里面的内容是一致的。然后slave从节点从这个relaylog日志文件中读取数据应用到数据库中，来实现数据的主从复制 ![avartar](https://github.com/craftlook/Note/blob/master/image/db/relay-log.png)
