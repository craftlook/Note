# 两阶段提交2PC

2PC即Innodb对于事务的两阶段提交机制。当MySQL开启binlog的时候，会存在一个内部XA的问题：事务在存储引擎层（redo）commit的顺序和在binlog中提交的顺序不一致的问题。如果不使用两阶段提交，那么数据库的状态有可能用它的日志恢复出来的库的状态不一致。

事务的commit分为prepare和commit两个阶段：

1. **prepare阶段**：redo持久化到磁盘（redo group commit），并将回滚段置为prepared状态，此时binlog不做操作。

<img src="https://github.com/craftlook/Note/blob/master/image/db/mysql-2pc-prepare.png" width="50%" heigth="50%"/>

2. **commit阶段**：innodb释放锁，释放回滚段，设置提交状态，binlog持久化到磁盘，然后存储引擎层提交。

<img src="https://github.com/craftlook/Note/blob/master/image/db/mysql-2pc-commit.png" width="50%" heigth="50%"/>

> 问题：为什么需要二阶段提交？

**解答**：由于存在redo log 和 binlog ，而他们两是相互独立的。而事务提交必须确保两者同时有效。不然会出现不一致的情形。我们对redo log和binlog不进行二阶段提交的顺序进行假设。

1. **先写redolog再写binlog**：redolog写了，binlog还没写，数据库崩了。通过redolog恢复数据库能将这条事务执行，但是binlog没有记录，从数据库就不能执行这条事务（或者在对数据库回到某个点时会没有这条事务），造成不一致的情况。
2. **先写binlog再写redolog**：binlog写了，redolog还没写，数据库崩了。通过redolog恢复数据库没有这条事务，但是binlog记录了，从数据库会执行这条事务（或者在对数据库回到某个点时会有这条事务的执行），造成不一致的情况。

> 问题：二阶段提交怎么解决问题？

<img src="https://github.com/craftlook/Note/blob/master/image/db/mysql-2pc-liu.png" width="40%" heigth="30%"/>


> 上图的①时出现问题怎么解决？

这个时候redo log已经到磁盘了。binlog没有刷到磁盘所以会消失。服务器从故障中恢复时，读取磁盘中的redo log ，但是由于对应的redo log项还是prepare状态，就要判断binlog 是否完整，**如果binlog完整则提交事务，如果binlog不完整则回滚事务**。

> 上图的②时出现问题怎么解决？

这个时候redo log 和 binlog都已经存磁盘，服务器从redo log恢复就好了。

> 怎么判断binlog的完整性?

- statement 模式的 binlog，最后会有 COMMIT；
- row 模式的 binlog，最后会有一个 XID event

注：binlog有三种模式：statement、row和mixed(statement和row的混合)

> 找到redo log后怎么找到binlog？

redo log和binlog有一个共同的数据字段，叫 XID。崩溃恢复的时候，会按顺序扫描 redo log：如果碰到既有 prepare、又有 commit 的 redo log，就直接提交；如果碰到只有 parepare、而没有 commit 的 redo log，就拿着 XID 去 binlog 找对应的事务。
