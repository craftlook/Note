# ES 组合查询语句介绍

Elasticsearch 中的bool使用用场景很多，本文主要介绍bool中的查询条件与组合查询。Bool包含四个语句 must、must_not、filter、should

#### must

文档 **必须匹配** 这些条件才能被包含进来，相当于sql中的**and**。并且参与计算分值

#### must_not

文档 **必须不匹配** 这些条件才能被包含进来，相当于sql的**not**。不会影响评分，它的作用是将不相关文件排除

#### filter

必须匹配，但它以不评分、过滤模式来进行。这些语句**对评分没有贡献** ，只是根据过滤标准来排除或包含文档。可以使用缓存。 [高效的原理](#filter)

#### should

如果满足这些语句中的任意语句，将增加 `_score` ，否则，无任何影响。它们主要用于修正每个文档的相关性得分。相当于sql中**or**。

### 组合查询

前面的两个例子都是单个过滤器（filter）的使用方式。 在实际应用中，我们很有可能会过滤多个值或字段。比方说，用 Elasticsearch 来表达下面的 SQL 

```sql
SELECT product
FROM   products
WHERE  (price = 20 OR productID = "XHDK-A-1293-#fJ3")
  AND  (price != 30)
```

这种情况下，我们需要 `bool` （布尔）过滤器。 这是个 *复合过滤器（compound filter）* ，它可以接受多个其他过滤器作为参数，并将这些过滤器结合成各式各样的布尔（逻辑）组合。

用 Elasticsearch 来表示本部分开始处的 SQL 例子，将两个 `term` 过滤器置入 `bool` 过滤器的 `should` 语句内，再增加一个语句处理 `NOT` 非的条件：

```json
GET /my_store/products/_search
{
   "query" : {
      "filtered" : { 
         "filter" : {
            "bool" : {
              "should" : [
                 { "term" : {"price" : 20}}, 
                 { "term" : {"productID" : "XHDK-A-1293-#fJ3"}} 
              ],
              "must_not" : {
                 "term" : {"price" : 30} 
              }
           }
         }
      }
   }
}
```

1. 注意，我们仍然需要一个 `filtered` 查询将所有的东西包起来
2. 在 `should` 语句块里面的两个 `term` 过滤器与 `bool` 过滤器是父子关系，两个 `term` 条件需要匹配其一。
3. 如果一个产品的价格是 `30` ，那么它会自动被排除，因为它处于 `must_not` 语句里面。

我们搜索的结果返回了 2 个命中结果，两个文档分别匹配了 `bool` 过滤器其中的一个条件：

```json
"hits" : [
    {
        "_id" :     "1",
        "_score" :  1.0,
        "_source" : {
          "price" :     10,
          "productID" : "XHDK-A-1293-#fJ3" 
        }
    },
    {
        "_id" :     "2",
        "_score" :  1.0,
        "_source" : {
          "price" :     20, 
          "productID" : "KDKE-B-9947-#kL5"
        }
    }
]
```

1. 与 `term` 过滤器中 `productID = "XHDK-A-1293-#fJ3"` 条件匹配
2. 与 `term` 过滤器中 `price = 20` 条件匹配

### 嵌套布尔过滤器

尽管 `bool` 是一个复合的过滤器，可以接受多个子过滤器，需要注意的是 `bool` 过滤器本身仍然还只是一个过滤器。 这意味着我们可以将一个 `bool` 过滤器置于其他 `bool` 过滤器内部，这为我们提供了对任意复杂布尔逻辑进行处理的能力。

对于以下这个 SQL 语句：

```sql
SELECT document
FROM   products
WHERE  productID      = "KDKE-B-9947-#kL5"
  OR (     productID = "JODL-X-1937-#pV7"
       AND price     = 30 )
```

我们将其转换成一组嵌套的 `bool` 过滤器：

```json
GET /my_store/products/_search
{
   "query" : {
      "filtered" : {
         "filter" : {
            "bool" : {
              "should" : [
                { "term" : {"productID" : "KDKE-B-9947-#kL5"}}, 
                { "bool" : { 
                  "must" : [
                    { "term" : {"productID" : "JODL-X-1937-#pV7"}}, 
                    { "term" : {"price" : 30}} 
                  ]
                }}
              ]
           }
         }
      }
   }
}
```

1. 因为 `term` 和 `bool` 过滤器是兄弟关系，他们都处于外层的布尔逻辑 `should` 的内部，返回的命中文档至少须匹配其中一个过滤器的条件。
2. 这两个 `term` 语句作为兄弟关系，同时处于 `must` 语句之中，所以返回的命中文档要必须都能同时匹配这两个条件。

得到的结果有两个文档，它们各匹配 `should` 语句中的一个条件：

```json
"hits" : [
    {
        "_id" :     "2",
        "_score" :  1.0,
        "_source" : {
          "price" :     20,
          "productID" : "KDKE-B-9947-#kL5" 
        }
    },
    {
        "_id" :     "3",
        "_score" :  1.0,
        "_source" : {
          "price" :      30, 
          "productID" : "JODL-X-1937-#pV7" 
        }
    }
]
```

1. 这个 `productID` 与外层的 `bool` 过滤器 `should` 里的唯一一个 `term` 匹配。
2. 这两个字段与嵌套的 `bool` 过滤器 `must` 里的两个 `term` 匹配。

### 控制精度

所有 `must` 语句必须匹配，所有 `must_not` 语句都必须不匹配，但有多少 `should` 语句应该匹配呢？默认情况下，没有 `should` 语句是必须匹配的，只有一个例外：那就是当没有 `must` 语句的时候，至少有一个 `should` 语句必须匹配。

就像我们能控制 [`match` 查询的精度](https://www.elastic.co/guide/cn/elasticsearch/guide/current/match-multi-word.html#match-precision) 一样，我们可以通过 `minimum_should_match` 参数控制需要匹配的 `should` 语句的数量，它既可以是一个绝对的数字，又可以是个百分比：

```json
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title": "brown" }},
        { "match": { "title": "fox"   }},
        { "match": { "title": "dog"   }}
      ],
      "minimum_should_match": 2 
    }
  }
}
```



## <span id="filter">filter 高效原理</span>

filter查询高效的原因，引入ES的query context和filter context概念。

#### query context

关注文档有多匹配查询的条件，这匹配的程度是由相关性分数决定，分数越高自然就越匹配。所以这种查询除了关注文档是否满足查询条件，还要额外的计算相关性分数。

#### filter context

关注文档是否匹配插叙你条件，结果只有`是`和`否`。没有其他的额外计算。它常用的一个场景就是过滤时间范围。

对于bool查询，must是query context，而filter使用的就是filter context。我们应该根据自己的实际业务场景选择合适的查询语句，在某些不需要相关性算分的查询场景，尽量使用filter context可以让你的查询更加高效。

