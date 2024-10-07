# Search your data

## The search API



### Paginate search results

By default, searches return the top 10 matching hits. To page through a larger set of results, you can use the [search API](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-search.html)'s `from` and `size` parameters. The `from` parameter defines the number of hits to skip, defaulting to `0`. The `size` parameter is the maximum number of hits to return. Together, these two parameters define a page of results.

```
GET /_search
{
  "from": 5,
  "size": 20,
  "query": {
    "match": {
      "user.id": "kimchy"
    }
  }
}
```

Avoid using `from` and `size` to page too deeply or request too many results at once. Search requests usually span multiple shards. Each shard must load its requested hits and the hits for any previous pages into memory. For deep pages or large sets of results, these operations can significantly increase memory and CPU usage, resulting in degraded performance or node failures.

By default, you cannot use `from` and `size` to page through more than 10,000 hits. This limit is a safeguard set by the [`index.max_result_window`](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#index-max-result-window) index setting. If you need to page through more than 10,000 hits, use the [`search_after`](https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html#search-after) parameter instead.

>Elasticsearch uses Lucene’s internal doc IDs as tie-breakers. These internal doc IDs can be completely different across replicas of the same data. When paging search hits, you might occasionally see that documents with the same sort values are not ordered consistently.





#### Search after

You can use the `search_after` parameter to retrieve the next page of hits using a set of [sort values](https://www.elastic.co/guide/en/elasticsearch/reference/current/sort-search-results.html) from the previous page.

Using `search_after` requires multiple search requests with the same `query` and `sort` values. The first step is to run an initial request. The following example sorts the results by two fields (`date` and `tie_breaker_id`):

```
GET twitter/_search
{
    "query": {
        "match": {
            "title": "elasticsearch"
        }
    },
    "sort": [
        {"date": "asc"},
        {"tie_breaker_id": "asc"}            1
    ]
}
```

>1    A copy of the `_id` field with `doc_values` enabled



The search response includes an array of `sort` values for each hit:

```
{
  "took" : 17,
  "timed_out" : false,
  "_shards" : ...,
  "hits" : {
    "total" : ...,
    "max_score" : null,
    "hits" : [
      ...
      {
        "_index" : "twitter",
        "_id" : "654322",
        "_score" : null,
        "_source" : ...,
        "sort" : [
          1463538855,
          "654322"
        ]
      },
      {
        "_index" : "twitter",
        "_id" : "654323",
        "_score" : null,
        "_source" : ...,
        "sort" : [                                
          1463538857,                        1
          "654323"
        ]
      }
    ]
  }
}
```

>1  Sort values for the last returned hit.



To retrieve the next page of results, repeat the request, take the `sort` values from the last hit, and insert those into the `search_after` array:

```
GET twitter/_search
{
    "query": {
        "match": {
            "title": "elasticsearch"
        }
    },
    "search_after": [1463538857, "654323"],
    "sort": [
        {"date": "asc"},
        {"tie_breaker_id": "asc"}
    ]
}
```

Repeat this process by updating the `search_after` array every time you retrieve a new page of results. If a [refresh](https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html) occurs between these requests, the order of your results may change, causing inconsistent results across pages. To prevent this, you can create a [point in time (PIT)](https://www.elastic.co/guide/en/elasticsearch/reference/current/point-in-time-api.html) to preserve the current index state over your searches.

```
POST /my-index-000001/_pit?keep_alive=1m
```

The API returns a PIT ID.

```
{
  "id": "46ToAwMDaWR5BXV1aWQyKwZub2RlXzMAAAAAAAAAACoBYwADaWR4BXV1aWQxAgZub2RlXzEAAAAAAAAAAAEBYQADaWR5BXV1aWQyKgZub2RlXzIAAAAAAAAAAAwBYgACBXV1aWQyAAAFdXVpZDEAAQltYXRjaF9hbGw_gAAAAA=="
}
```

To get the first page of results, submit a search request with a `sort` argument. If using a PIT, specify the PIT ID in the `pit.id` parameter and omit the target data stream or index from the request path.





### Sort search results

Allows you to add one or more sorts on specific fields. Each sort can be reversed as well. The sort is defined on a per field level, with special field name for `_score` to sort by score, and `_doc` to sort by index order.

Assuming the following index mapping:

```
PUT /my-index-000001
{
  "mappings": {
    "properties": {
      "post_date": { "type": "date" },
      "user": {
        "type": "keyword"
      },
      "name": {
        "type": "keyword"
      },
      "age": { "type": "integer" }
    }
  }
}
```

```
GET /my-index-000001/_search
{
  "sort" : [
    { "post_date" : {"order" : "asc", "format": "strict_date_optional_time_nanos"}},
    "user",
    { "name" : "desc" },
    { "age" : "desc" },
    "_score"
  ],
  "query" : {
    "term" : { "user" : "kimchy" }
  }
}
```

>`_doc` has no real use-case besides being the most efficient sort order. So if you don’t care about the order in which documents are returned, then you should sort by `_doc`. This especially helps when [scrolling](https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html#scroll-search-results).



####  Sort values

The search response includes `sort` values for each document. Use the `format` parameter to specify a [date format](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-date-format.html#built-in-date-formats) for the `sort` values of [`date`](https://www.elastic.co/guide/en/elasticsearch/reference/current/date.html) and [`date_nanos`](https://www.elastic.co/guide/en/elasticsearch/reference/current/date_nanos.html) fields. The following search returns `sort` values for the `post_date` field in the `strict_date_optional_time_nanos` format.

```
GET /my-index-000001/_search
{
  "sort" : [
    { "post_date" : {"format": "strict_date_optional_time_nanos"}}
  ],
  "query" : {
    "term" : { "user" : "kimchy" }
  }
}
```

