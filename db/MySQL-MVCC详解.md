# Mysql-MVCC 详解

## MVCC介绍

全称 Multi-Version Concurrency Control  多版本控制，主要目的为提高数据库的并发性能(InnoDB引擎下)。对同一行的数据发生并发读写时，为了避免数据冲突，可以通过上锁阻塞实现。MVCC则不用增加锁，用更好的方式处理了读-写请求的并发问题。（此处的读是快照读，读分为快照读、当前读）


