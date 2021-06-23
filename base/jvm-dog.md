# JVM 线上配置内容介绍

## 应用使用场景

对外提供查询、修改服务，通过调用代理层应用进行数据获取和操作。

机器规格  8C12G

```java
-Xms6144m
-Xmx6144m
-XX:NewSize=4096m
-XX:MaxNewSize=4096m
```

8C16G 

```java
-Xms8192m
-Xmx8192m
-XX:NewSize=6144m
-XX:MaxNewSize=6144m
```

公用部分

```java
-XX:MaxMetaspaceSize=512m
-XX:SurvivorRatio=8
-XX:ParallelGCThreads=8
-XX:+UseConcMarkSweepGC
-XX:+UseParNewGC
-XX:MaxTenuringThreshold=8
-XX:+UseCMSInitiatingOccupancyOnly
-XX:CMSInitiatingOccupancyFraction=75
-XX:CMSFullGCsBeforeCompaction=0
-XX:+UseCMSCompactAtFullCollection
-XX:+CMSClassUnloadingEnabled
-XX:+CMSPermGenSweepingEnabled
-XX:-UseBiasedLocking
-XX:AutoBoxCacheMax=20000
-XX:+AlwaysPreTouch
-XX:+ExplicitGCInvokesConcurrent
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
```

已12G进行解释：

* jvm的堆内存设置为总内存的1/2，-Xms6144m  -Xmx6144m 为了冗余和保证线上非堆内存的使用（设置的过于保守 8G左右比较合适，但是公司规定）
* 该应用在之前的gc回收图标观察时，发现主要是年轻代的对象比较频繁，-XX:NewSize=4096m -XX:MaxNewSize=4096m 年轻代占总内存2/3
* -XX:ParallelGCThreads={value} 这个参数是指定并行 GC 线程的数量，一般最好和 CPU 核心数量相当。默认情况下，当 CPU 数量小于8， ParallelGCThreads 的值等于 CPU 数量，当 CPU 数量大于 8 时，则使用公式：ParallelGCThreads = 8 + ((N - 8) * 5/8) = 3 +（（5*CPU）/ 8）；同时这个参数只要是并行 GC 都可以使用，不只是 ParNew
* XX:+UseConcMarkSweepGC Concurrent Mark Sweep并发标记清除,即使用CMS收集器
* UseParNewGC：并发串行收集器，它是工作在新生代的垃圾收集器，它只是将串行收集器多线程化，除了这个并没有太多创新之处，而且它们共用了相当多的代码。它与串行收集器一样，也是独占式收集器，在收集过程中，应用程序会全部暂停。但它却是许多运行在Server模式下的虚拟机中首选的新生代收集器，其中有一个与性能无关但很重要的原因是，除了Serial收集器外，目前只有它能与CMS收集器配合工作。
* 新生代是使用copy算法来进行垃圾回收的。默认情况下,当新生代执行了15次young gc后,如果还有对象存活在Survivor区中,那么就可以直接将这些对象晋升到老年代,但是由于新生代使用copy算法,如果Survivor区存活的对象太久的话,Survivor区存活的对象就越多,这个就会影响copy算法的性能,使得young gc停顿的时间加长,建议设置成8
* -XX:+UseCMSInitiatingOccupancyOnly  -XX:CMSInitiatingOccupancyFraction=75。那么当老年代堆空间的使用率达到75%的时候就开始执行垃圾回收,CMSInitiatingOccupancyFraction默认值是92%,这个就太大了。
  CMSInitiatingOccupancyFraction参数必须跟下面两个参数一起使用才能生效的
* 配置-XX:+UseCMSCompactAtFullCollection（默认）前提下 CMSFullGCsBeforeCompaction 说的是，在上一次CMS并发GC执行过后，到底还要再执行多少次full GC才会做压缩。默认是0，也就是在默认配置下每次CMS GC顶不住了而要转入full GC的时候都会做压缩。 把CMSFullGCsBeforeCompaction配置为10，就会让上面说的第一个条件变成每隔10次真正的full GC才做一次压缩（而不是每10次CMS并发GC就做一次压缩，目前VM里没有这样的参数）。这会使full GC更少做压缩，也就更容易使CMS的old gen受碎片化问题的困扰。 本来这个参数就是用来配置降低full GC压缩的频率，以期减少某些full GC的暂停时间。CMS回退到full GC时用的算法是mark-sweep-compact，但compaction是可选的，不做的话碎片化会严重些但这次full GC的暂停时间会短些；这是个取舍
*  -XX:+CMSClassUnloadingEnabled 相对于并行收集器，CMS收集器默认不会对永久代进行垃圾回收。如果希望对永久代进行垃圾回收，可用设置标志-XX:+CMSClassUnloadingEnabled。在早期JVM版本中，要求设置额外的标志
*  -XX:+CMSPermGenSweepingEnabled。注意，即使没有设置这个标志，一旦永久代耗尽空间也会尝试进行垃圾回收，但是收集不会是并行的，而再一次进行Full GC。
*  -XX:-UseBiasedLocking参数来禁用偏向锁。引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令（由于一旦出现多线程竞争的情况就必须撤销偏向锁，所以偏向锁的撤销操作的性能损耗必须小于节省下来的CAS原子指令的性能消耗）。在存在大量锁对象的创建并高度并发的环境下禁用偏向锁能够带来一定的性能优化
*  -XX:AutoBoxCacheMax=20000 JAVA进程启动的时候,会加载rt.jar这个核心包的,rt.jar包里的Integer自然也是被加载到JVM中,Integer里面有一个IntegerCache缓存。如果是这个范围内的数字,则直接从缓存取出
* -XX:+AlwaysPreTouch JAVA进程启动的时候,虽然我们可以为JVM指定合适的内存大小,但是这些内存操作系统并没有真正的分配给JVM,而是等JVM访问这些内存的时候,才真正分配,这样会造成以下问题。1.GC的时候,新生代的对象要晋升到老年代的时候,需要内存,这个时候操作系统才真正分配内存,这样就会加大young gc的停顿时间;2.可能存在内存碎片的问题。这样JVM就会先访问所有分配给它的内存,让操作系统把内存真正的分配给JVM.后续JVM就可以顺畅的访问内存了。
* +ExplicitGCInvokesConcurrent 如果系统使用堆外内存,比如用到了Netty的DirectByteBuffer类,那么当想回收堆外内存的时候,需要调用System.gc()，而这个方法将进行full gc,整个应用将会停顿,如果是使用CMS垃圾收集器,那么可以设置，这个参数来改变`System.gc()`的行为,让其从full gc –> CMS GC,CMS GC是并发收集的,且中间执行的过程中,只有部分阶段需要stop the world。设置了ExplicitGCInvokesConcurrent,那就不要设置DisableExplicitGC参数来禁掉`System.gc()`
