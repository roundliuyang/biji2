# Query DSL



## Term-level queries

You can use **term-level queries** to find documents based on precise values in structured data. Examples of structured data include date ranges, IP addresses, prices, or product IDs.

Unlike [full-text queries](https://www.elastic.co/guide/en/elasticsearch/reference/current/full-text-queries.html), term-level queries do not analyze search terms. Instead, term-level queries match the exact terms stored in a field.





### Term

term 的搜索是不分词的，搜索给定字符串的全部内容，比如对于我们插入的 id=4 的那条数据，address 的内容是 `read a book`，它被分词为三个，read、a、book，所以我们使用 term 方法搜索下面三个都可以搜到这条数据：

```python
GET /exam/_search
{"query": {"term": {"address": "read"}}}

GET /exam/_search
{"term": {"address": "a"}}

GET /exam/_search
{"term": {"address": "book"}}
```

但是，如果我们 address 后面的值如下这种就搜索不到了，因为 term 操作并不会给搜索的内容进行分词，而是作为一个整体进行搜索：

```python
GET /exam/_search
{"query": {"term": {"address": "read a"}}}

GET /exam/_search
{"query": {"term": {"address": "a book"}}}

GET /exam/_search
{"query": {"term": {"address": "read a book"}}}
```

但是还有一种情况，那就是对于搜索的 text 字段后加上 `.keyword` 字段的操作，这个相当于将 address 不分词进行搜索，将 address 这个字段看作是一个 keyword 来操作，可以理解成是使用 term 来搜索 keyword 字段，就是上一个类型的操作。

所以下面的这个操作就是可以搜索到 `address='read a book'` 的数据

```python
GET /exam/_search
{"query": {"term": {"address.keyword": "read a book"}}}
```



### Exists

Returns documents that contain an indexed value for a field.

An indexed value may not exist for a document’s field due to a variety of reasons:

- The field in the source JSON is `null` or `[]`
- The field has `"index" : false` and `"doc_values" : false` set in the mapping
- The length of the field value exceeded an `ignore_above` setting in the mapping
- The field value was malformed and `ignore_malformed` was defined in the mapping



#### Example request

```
GET /_search
{
  "query": {
    "exists": {
      "field": "user"
    }
  }
}
```



## Full text queries

The full text queries enable you to search [analyzed text fields](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html) such as the body of an email. The query string is processed using the same analyzer that was applied to the field during indexing.



首先我们创建这样一个 index 和下面几条数据：

```
PUT /exam

PUT /exam/_mapping
{
  "properties": {
    "address": {
      "type": "text",
      "fields": {
        "keyword": {
          "type": "keyword",
          "ignore_above": 256
        }
      }
    },
    "name": {
      "type": "keyword"
    }
  }
}
```

我们创建了 name 字段为 keyword，address 字段是 text，接下来我们先创建几条数据来测试 keyword 字段类型的搜索。



### Match phrase 

匹配短语，使用这个方法不加其他参数的情况下，可以看作是会匹配包含这个短语、且顺序一致的数据。

比如说对于 address="read a book" 的数据，搜索 `read a`，`a book`，`read a book` 都可以筛选到这条数据。

```python
GET /exam/_search
{"query": {"match_phrase": {"address": "read a"}}}

GET /exam/_search
{"query": {"match_phrase": {"address": "a book"}}}

GET /exam/_search
{"query": {"match_phrase": {"address": "read a book"}}}
```

但是如果搜索 `book a`，因为顺序不一致，所以下面的搜索是无法搜素到该数据的：

```python
GET /exam/_search
{"query": {"match_phrase": {"address": "book a"}}}
```

但是 match_phrase 有一个 slop 参数可以用于忽略这种顺序，也就是允许搜索的关键词错位的个数，比如 'book a'，分词后的 'book' 和 'a' 如果允许错位两个顺序（a 往前挪一个，book 往后挪一个，这是我理解的 slop 的操作用法），那么就可以筛选到我们这条数据，示例如下：

```python
GET /exam/_search
{
  "query": {
    "match_phrase": {
      "address": {
        "query": "book a",
        "slop": 2
      }
    }
  }
}
```





### Match

match，模糊匹配，在匹配前会将搜索的字符串进行分词，然后将匹配上的数据按照匹配度（在 es 里有一个 _score 字段用于表示这种匹配程度）倒序返回。

比如我们对 address 字段搜索字符串 `a`，会返回两条数据，id 为 4 和 5 的，因为 address 字段进行分词存储后都包含这个字符串。

```python
GET /exam/_search
{"query": {"match": {"address": "a"}}}
```

或者我们搜索内容为 `read a`，match 搜索会先将其分词，变成 `read` 和 `a`，然后匹配分词后包含这两个字符串一个或者两个的数据，在这里也会返回两条，一条的结果是 `read a book`，一条是 `you can get a good job`，因为这两条数据都包含字符串 `a`，但是因为前者分别满足了两个搜索的条件，所以前者的匹配度会更高，所以作为第一条数据返回：

```python
GET /exam/_search
{"query": {"match": {"address": "read a"}}}
```







#### match 的其他用法

**匹配分词后的全部结果**

对于 match，前面我们介绍过会先将搜索的字符串分词，然后去筛选包含分词结果一至多个的结果。

比如前面介绍的搜索 'read a'，会搜索出 'read a book' 以及 'you can get a good job'，因为他们都包含分词的结果 'a'，这种操作就类似于用 should 去对分词结果进行进一步的搜索操作，

但是如果我们想要更精确，搜索的内容必须包含分词的全部结果 'read' 和 'a'，我么可以加上 operator 参数：

```python
GET /exam/_search
{
  "query": {
    "match": {
      "address": {
        "query": "read a",
        "operator": "and"
      }
    }
  }
}
```

这样操作结果就是筛选了包含全部搜索词分词后结果的数据。



**匹配的模糊处理**

我们可以通过 fuzziness 字段来打开字符模糊匹配的开关，最简单的一个例子就是比如我们搜索 'read'，打字不小心打成了 'raed'，这种就可以实现他的模糊匹配：

```python
GET /exam/_search
{
  "query": {
    "match": {
      "address": {
        "query": "raed a",
        "operator": "and",
        "fuzziness": 1
      }
    }
  }
}
```



### **match_phrase_prefix**

匹配前缀，比如对于 address 值为 'read a book' 的数据，我们只知道的值是 'read a bo'，想要根据这个搜索词搜索完整的数据，就可以用到 match_phrase_prefix。

他的用法是这样的，先将检索词分词，然后将最后一个分词结果单独去匹配，所以这个搜索词的过程就是先根据 'read a' 的分词结果搜索到一些数据，然后根据剩下的 'bo' 去匹配满足这个前缀的数据：

```python
GET /exam/_search
{"query": {"match_phrase_prefix": {"address": "read a bo"}}}
```





### multi-match

前面我们的 match 参数操作的都是针对于单个字段，multi_match 则可以针对于多个字段进行 match 操作，这个需要都能匹配上搜索的关键字，使用示例如下：

```python
GET /exam/_search
{
  "query": {
    "multi_match": {
      "query": "python",
      "fields": ["name", "address"]
    }
  }
}
```

其中，fields 是一个数组，里面是需要搜索的字段。





## Compound queries

Compound queries wrap other compound or leaf queries, either to combine their results and scores, to change their behaviour, or to switch from query to filter context.



### Boolean

A query that matches documents matching boolean combinations of other queries. The bool query maps to Lucene `BooleanQuery`. It is built using one or more boolean clauses, each clause with a typed occurrence. The occurrence types are:

| Occur      | Description                                                  |
| ---------- | ------------------------------------------------------------ |
| `must`     | The clause (query) must appear in matching documents and will contribute to the score. |
| `filter`   | The clause (query) must appear in matching documents. However unlike `must` the score of the query will be ignored. Filter clauses are executed in [filter context](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html), meaning that scoring is ignored and clauses are considered for caching. |
| `should`   | The clause (query) should appear in the matching document.   |
| `must_not` | The clause (query) must not appear in the matching documents. Clauses are executed in [filter context](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html) meaning that scoring is ignored and clauses are considered for caching. Because scoring is ignored, a score of `0` for all documents is returned. |

The `bool` query takes a *more-matches-is-better* approach, so the score from each matching `must` or `should` clause will be added together to provide the final `_score` for each document.

```
POST _search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "user.id" : "kimchy" }
      },
      "filter": {
        "term" : { "tags" : "production" }
      },
      "must_not" : {
        "range" : {
          "age" : { "gte" : 10, "lte" : 20 }
        }
      },
      "should" : [
        { "term" : { "tags" : "env1" } },
        { "term" : { "tags" : "deployed" } }
      ],
      "minimum_should_match" : 1,
      "boost" : 1.0
    }
  }
}
```





## 参考文档

[[es笔记三之term，match，match_phrase 等查询方法介绍](https://segmentfault.com/a/1190000043636007)](https://segmentfault.com/a/1190000043636007#item-2)
