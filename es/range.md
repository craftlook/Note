# ElasticSearch Range查询过滤原理

## range查询介绍

### 支持的数据类型

* 数值类型:integer、long等
* 日期类型：date等
* 字符串类型：keyword、text等

range查询对应数据类型底层存储数据结构以最典型的两种类型拆解

### 对应的数据结构： Block K-d Tree （数值类型）

ES 5.0开始引入Block K-d Tree这种新数值类型索引数据结构，不在采用倒排索引，该结构更适合范围查找

**K-d Tree的基本思想：** [K-d 详解](https://zhuanlan.zhihu.com/p/127022333)

将一个N维的数值空间，不断选定**包含值最多的维度**做2分切割，反复迭代，直到切分出来的空间单元`cell` 包含的值，数量小于某个数值。

对于单维度的数据，实际上就是简单的对所有值做一个排序，然后反复从中间做切分，生成一个类似于B-tree的结构。与传统的B-tree不同的时，他的叶子结点存储的不是单值，而是一组值得集合Block。每个Block内部包含的值控制在521-1024个，保证值的数量在block之间精良均匀分布。

![avatar](https://github.com/craftlook/Note/blob/master/image/es/kd-tree-1.png)

### 对应的数据结构：LSM 树 （字符串类型）

Elasticsearch、HBase、Cassandra、RockDB 等都是基于LSM Tree来构建索引

LSM Tree 是一种分层、有序、面向磁盘的数据结构，核心思想是充分利用磁盘`批量`**顺序写**比**随机写**性能高出很多的特点。[LSM介绍]()

#### LSM Tree的核心特点：

1. 将索引分为内存和磁盘两部分，并在内存达到阈值时启动树合并（Merge Trees）;
2. 用批量写入代替随机写入，并且用预写日志**WAL 技术**（Elasticsearch 中威translog事务日志）保证内存数据，在系统崩溃后可以被恢复 [WAL介绍]()

3. 数据采取类似日志追加写的方式写入（Log Structured）磁盘，以顺序写的方式提高写入效率；

![avatar](https://github.com/craftlook/Note/blob/master/image/es/es-merge-tree.png)

## range原理

针对range查询，通过增加profile：true能看到底层逻辑

**数值整型以及日期类型**的range查询后台使用luence的`IndexOrDocValuesQuery`

date类型实际会转换成时间戳：

``` java
"description": "+orderCreateDate:[1569859200000 TO 1601481600000] +erpOrderStatus:[6 TO 2147483647]"
```

从ES 5.4版本开始 indexOrDocValuesQuery 将Range查询过程中的顺序访问扔给

Block K-D Tree索引，将随机访问任务交给doc Values。

**字符串类型**的range查询后台使用的是luence的`MultiTermQueryConstantScoreWrapper`

`MultiTermQueryConstantScoreWrapper`本质是：多个term query 组合

#### 数值类型range原理

Lucene将单维度数据返回从中间做切分后，KD-tree的非叶子结点部分放在内存里，耳叶子节点紧紧相邻存放在磁盘上。当做range查询时候，内存里的KD-tree可以帮助快速定位到满足查询条件的叶子结点块在磁盘上的位置，之后对叶子节点块的读取几乎是顺序的。(重点)

这个跟Mysql的B+ 树很类似，索引与数据分离、顺序存盘的机制很适合做范围查询。

如果要基于 B+ 树做 Range 查询，举例查询（a，b）之间的所有元素， 实际操作如下：

步骤1：通过二分查找找到 a 元素；
步骤2：依次向后遍历叶子节点对应的链表元素，直到数值 > b 停止。
这样就能得到（a, b) 之间的所有元素值。

#### 字符串类型range原理

针对LSM-Tree核心的数据结构是SSTable（Sorted String Table），SSTables的概念其实来源于Google的Bigtable论文。

SSTable是一种拥有持久化，有序且不可变的键值存储结构，它的key和balue都是认以的字节数组，并且提供了按指定key查找和指定范围的key区间迭代遍历的功能。

SSTable内部包含了一系列可配置大小的Block块，典型大小64kb，冠以这些Block块的索引存储在SSTable的尾部，用于帮助快速查找特定的Block。

查询原理：当一个SSTable被打开的时候，索引会被加载到内存，然后根据key在内存索引里面进行二分查找，查找到该key对应的磁盘的offset后，然后去磁盘把响应的块数据读取出来。

range 的原理：在查询基础上，基于 SStable 文件（已有序）向后遍历找到直到找到大于右区间值停止遍历。

当然如果内存足够大的话，可以直接把 SSTable 直接通过 MMap 的技术映射到内存中，从而提供更快的查找。

![avatar](https://github.com/craftlook/Note/blob/master/image/es/lsm1.png)
