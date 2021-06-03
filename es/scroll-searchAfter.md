# Elasticsearch from+size/seach after/scroll 

ES中 from/size、scroll、search after介绍

**QUERY_THEN_FETCH：**ES 默认的搜索方式，搜索分为两个阶段 Query阶段和Fetch阶段。

* Query阶段比较轻量级，通过查询到排序索引，获取慢速查询结果的文档ID列表
* Fetch阶段比较重，需要将每个shard的结果取回，在coordinate node进行全局排序。通过From+size这种方式分批获取数据的时候，随着from加大，需要全局排序并丢弃的结果数量随之上升，**性能越来越差**。

## From/Size 浅分页

ES 可以使用from/size的参数对结果进行分页查询

**原理：** 在from+size的查询时，coordinate node（协调节点）向目标index对应的shards发送同样的请求，每个shard取出 from+size 条数据，等汇总 shard*（from+size）条数时coordinate node再做一次排序，取最后的size条数据作为结果返回。

eg：from=10000，size=10时 es会从每个分片取出（10000+10）条记录，如果有10个分片，则总共要取出（10000+10）*10条数据，协调节点再内存中对这些数据进行排序，最终返回10条数据。这种方式会耗费大量的系统资源，包括时间和空间。

注意from + size 不能超过 index.max_result_window 索引设置，默认为 10,000。 有关深入滚动的更有效方法，请参阅 Scroll 或 Search After API。

## Scroll 深分页

scroll类似SQL中的cursor，使用scroll每次只能获取一页的内容，然后返回一个scroll_id。根据返回的scroll_id不断的获取下一页的内容，所以scroll并不适用与分页列表。

**原理：** 先做轻量级的query阶段后，免去的了**全局排序的过程**。它只是将查询结果集，也就是doc_id 列表保留在一个（快照）上下文里（也就是说，在query阶段初始化之后对索引的插入、删除、更新数据不会影响遍历结果）。之后每次分批取得时候，只需根据请求得size值，在每个shard内部参照已经顺序（默认doc_id），取回size数量得文档即可。

**场景：** 可以看出scroll不适用与实时和用户交互的前端分页工作，主要用于从ES中分批次拉取大量结果集的情况，一般都是离线的应用场景。在旧版本中，ES为深度分页有scroll search 的方式，官方的建议并不是用于实时的请求，因为**每一个 scroll_id 不仅会占用大量的资源（特别是排序的请求），而且是生成的历史快照，对于数据的变更不会反映到快照上** 。
启用游标查询可以通过在查询的时候设置参数 scroll 的值为我们期望的游标查询的过期时间。 游标查询的过期时间会在每次做查询的时候刷新，所以这个时间只需要足够处理当前批的结果就可以了，而不是处理查询结果的所有文档的所需时间。 这个过期时间的参数很重要，因为保持这个游标查询窗口需要消耗资源，所以我们期望如果不再需要维护这种资源就该早点儿释放掉。 设置这个超时能够让 Elasticsearch 在稍后空闲的时候自动释放这部分资源。

#### keeping the search context alive

滚动参数（传递到搜索请求和每个滚动请求）告诉Elasticsearch应该保持搜索上下文活动的时间。其值（例如，1m，请参阅 [“Time unit”](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/common-options.html#time-units) 一节）不需要足够长以处理所有数据 - 它只需要足够长的时间来处理前一批结果。每个滚动请求（具有滚动参数）设置新的到期时间。

```java
GET /_search/scroll
{
  "scroll": "1m",
  "scroll_id":"DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAC7EWa1U5THR6U0xSSXVOZjJTaUhpeHY2dw=="
}
```

## Search After 深分页

search after通过一个实时的游标来避免scroll的缺点，可用于实时请求和高并发场景。（ps：es设计之初不是高并发）

**原理：**每个文档具有一个唯一值得字段用作排序规范得仲裁器,如：_id。否则相同排序值的文档其排序顺序是未定义的。官方推荐是 _id，或者自定义的文档唯一值。

```java
GET order/_search
{
    "from": 0,
    "size": 10,
    "query": {
        "match" : {
            "title" : "es"
        }
    },
    "search_after": [124648691, "624812_124648691"],
    "sort": [
        {"orderId": "asc"},
        {"_id": "desc"}
    ]
}
```

上面的语句翻译成sql：

```sql
select * from order where orderId>124648691 and _id<"624812_124648691" order by orderId asc,_id desc limit 10
```

ps: 当我们使用search_after时，**from值必须设置为0或者-1**

**场景：**search_after缺点是不能够随机跳转分页，只能一页一页的翻页，并且需要至少指定一个唯一不重复字段来排序。它与滚动API非常相似，但与它不同，**search_after参数是无状态的，它始终针对最新版本的搜索器进行解析。因此，排序顺序可能会在步行期间发生变化，具体取决于索引的更新和删除。**非常多页的查询时，search after是一个常量查询延迟和开销，并无什么副作用。

## 线上问题记录

## scroll 查询问题

问题背景，订单导出接口包可用率的问题。经过排查ES代理层的系统日志发现，ES底层的代码报错有空指针的异常。问题日志：

```java
2021-05-31 17:26:29.692 [JSF-BZ-22001-21-T-14 ] ERROR com.jd.es.order.service.soa.proxy.PopOrderQueryServiceMainProxyImpl.queryByQueryBuilderByScroll(316) - doSearch error:[all shards failed] queryDetail:indices(order_pop) AppSearchRequestBuilderByScroll{boolQueryBuilder=AppBoolQueryBuilder{ filter:AppBoolQueryBuilder{ must:AppTermQueryBuilder{fieldName='venderId', value=10483004}AppRangeQueryBuilder{fieldName='orderCreateDate', from=1621440000000, to=1622390399000, includeLower=true, includeUpper=true}AppTermsQueryBuilder{fieldName='yn', values=[0, 1, 2]} mustNot:AppTermQueryBuilder{fieldName='erpOrderStatus', value=2}}}, scrollid='DXF1ZXJ5QW5kRmV0Y2gBAAAAAAZ-WMcWWTkyNE1HbTBSVjZtUTBBbEZKUkItZw==', searchAfterId='null', size=500, venderId='10483004', sortField='orderCreateDate', returnFields='all', sortType='desc'}
org.elasticsearch.action.search.SearchPhaseExecutionException: all shards failed
at org.elasticsearch.action.search.SearchScrollAsyncAction.onShardFailure(SearchScrollAsyncAction.java:269) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.action.search.SearchScrollAsyncAction$1.onFailure(SearchScrollAsyncAction.java:202) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.action.ActionListenerResponseHandler.handleException(ActionListenerResponseHandler.java:51) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.action.search.SearchTransportService$ConnectionCountingHandler.handleException(SearchTransportService.java:531) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.transport.TransportService$ContextRestoreResponseHandler.handleException(TransportService.java:1056) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.transport.TcpTransport.lambda$handleException$31(TcpTransport.java:1476) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.common.util.concurrent.EsExecutors$1.execute(EsExecutors.java:135) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.transport.TcpTransport.handleException(TcpTransport.java:1474) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.transport.TcpTransport.handlerResponseError(TcpTransport.java:1466) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.transport.TcpTransport.messageReceived(TcpTransport.java:1396) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.transport.netty4.Netty4MessageChannelHandler.channelRead(Netty4MessageChannelHandler.java:64) ~[transport-netty4-client-6.3.2.jar:6.3.2]
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:340) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.handler.codec.ByteToMessageDecoder.fireChannelRead(ByteToMessageDecoder.java:310) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.handler.codec.ByteToMessageDecoder.fireChannelRead(ByteToMessageDecoder.java:297) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:413) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.handler.codec.ByteToMessageDecoder.channelRead(ByteToMessageDecoder.java:265) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:340) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.handler.logging.LoggingHandler.channelRead(LoggingHandler.java:241) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:340) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.DefaultChannelPipeline$HeadContext.channelRead(DefaultChannelPipeline.java:1334) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:926) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:134) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:644) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.nio.NioEventLoop.processSelectedKeysPlain(NioEventLoop.java:544) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:498) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:458) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:858) ~[netty-all-4.1.16.Final.jar:4.1.16.Final]
at java.lang.Thread.run(Thread.java:748) [?:1.8.0_191]
Caused by: org.elasticsearch.transport.RemoteTransportException: [d_10.194.33.42:30011][10.194.33.42:30111][indices:data/read/search[phase/query+fetch/scroll]]
Caused by: java.lang.NullPointerException
at org.elasticsearch.search.fetch.subphase.MatchedQueriesFetchSubPhase.lambda$hitsExecute$0(MatchedQueriesFetchSubPhase.java:52) ~[elasticsearch-6.3.2.jar:6.3.2]
at java.util.TimSort.binarySort(TimSort.java:296) ~[?:1.8.0_191]
at java.util.TimSort.sort(TimSort.java:239) ~[?:1.8.0_191]
at java.util.Arrays.sort(Arrays.java:1438) ~[?:1.8.0_191]
at org.elasticsearch.search.fetch.subphase.MatchedQueriesFetchSubPhase.hitsExecute(MatchedQueriesFetchSubPhase.java:52) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.search.fetch.FetchPhase.execute(FetchPhase.java:170) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.search.SearchService.executeFetchPhase(SearchService.java:370) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.search.SearchService.executeFetchPhase(SearchService.java:467) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.action.search.SearchTransportService$9.messageReceived(SearchTransportService.java:424) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.action.search.SearchTransportService$9.messageReceived(SearchTransportService.java:421) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.transport.RequestHandlerRegistry.processMessageReceived(RequestHandlerRegistry.java:66) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.transport.TcpTransport$RequestHandler.doRun(TcpTransport.java:1554) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.common.util.concurrent.ThreadContext$ContextPreservingAbstractRunnable.doRun(ThreadContext.java:637) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.common.util.concurrent.AbstractRunnable.run(AbstractRunnable.java:37) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.common.util.concurrent.TimedRunnable.doRun(TimedRunnable.java:41) ~[elasticsearch-6.3.2.jar:6.3.2]
at org.elasticsearch.common.util.concurrent.AbstractRunnable.run(AbstractRunnable.java:37) ~[elasticsearch-6.3.2.jar:6.3.2]
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) [?:1.8.0_191]
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) [?:1.8.0_191]
... 1 more
```

重要的问题日志：

```java
Caused by: org.elasticsearch.transport.RemoteTransportException: [d_10.194.33.42:30011][10.194.33.42:30111][indices:data/read/search[phase/query+fetch/scroll]]
Caused by: java.lang.NullPointerException
at org.elasticsearch.search.fetch.subphase.MatchedQueriesFetchSubPhase.lambda$hitsExecute$0(MatchedQueriesFetchSubPhase.java:52) ~[elasticsearch-6.3.2.jar:6.3.2]
```

经过资料搜索发现相似问题的讨论：https://github.com/elastic/elasticsearch/issues/25820

其中论坛中的问题讨论，空指针的问题，有可能是使用了原始的scrollid去请求所有的scroll请求分片，或者是因为查询上下文的超时。

因为在此之前，我曾经切换过线上的流量到冷备集群（配置较差），我怀疑有可能是集群配置较差的原，经过监控的对比发现，在切换前后scroll查询可用率变化的较大，再切换后查询可用率问题较多。

切换前

![avatar](https://github.com/craftlook/Note/blob/master/image/image-20210531214531758.png)

切换后

![avatar](https://github.com/craftlook/Note/blob/master/image/image-20210531214634745.png)



并对请求进行对比，有问题的查询请求size 是500，正常的查询size是100。这也可能是导出查询失败的，即可能是查询超时的原因。
