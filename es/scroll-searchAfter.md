# Elasticsearch from+size/seach after/scroll 

ES中 from/size、scroll、search after介绍

**QUERY_THEN_FETCH：** ES 默认的搜索方式，搜索分为两个阶段 Query阶段和Fetch阶段。

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

**场景：** 可以看出scroll不适用与实时和用户交互的前端分页工作，主要用于从ES中分批次拉取大量结果集的情况，一般都是离线的应用场景。在旧版本中，ES为深度分页有scroll search 的方式，官方的建议并不是用于实时的请求，因为**每一个 scroll_id 不仅会占用大量的资源（特别是排序的请求），而且是生成的历史快照，对于数据的变更不会反映到快照上**。

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

**原理：** 每个文档具有一个唯一值得字段用作排序规范得仲裁器,如：_id。否则相同排序值的文档其排序顺序是未定义的。官方推荐是 _id，或者自定义的文档唯一值。

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

**场景：** search_after缺点是不能够随机跳转分页，只能一页一页的翻页，并且需要至少指定一个唯一不重复字段来排序。它与滚动API非常相似，但与它不同，**search_after参数是无状态的，它始终针对最新版本的搜索器进行解析。因此，排序顺序可能会在步行期间发生变化，具体取决于索引的更新和删除。**非常多页的查询时，search after是一个常量查询延迟和开销，并无什么副作用。
