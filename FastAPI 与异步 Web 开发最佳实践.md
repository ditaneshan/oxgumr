# Elasticsearch 全文检索性能优化实战

## 引言
在高并发搜索场景中，Elasticsearch 的瓶颈通常不在“能否搜到”，而在“能否稳定、快速地搜到”。全文检索链路涉及分词、倒排索引、相关性评分、段合并与网络传输，任何一个环节配置不当都会放大延迟。要做好性能优化，必须从查询模型、索引设计和[架构设计](https://about-ayx-app.com.cn)三方面协同治理，而不是单点调参。

## 核心原理分析
全文检索的性能核心是倒排索引。查询时，ES 先根据 term 定位文档集合，再执行 BM25 评分与排序。性能损耗主要来自三类操作：  
1. `must` 触发评分，成本高于 `filter`；  
2. `from + size` 深分页会扫描大量无效文档；  
3. 高基数字段聚合、脚本排序、动态 mapping 会增加 CPU 和内存压力。

优化思路应优先落在“减少参与计算的数据量”。例如，能过滤的条件尽量放到 `filter`，不需要精确总数时关闭 `track_total_hits`，分页场景使用 `search_after` 替代深分页。同时，索引侧要控制字段类型，避免把大文本字段都设置为 `text` 并开启不必要的 `fielddata`。

## 代码示例
下面是一个实际可落地的查询优化示例：将可过滤条件移入 `filter`，关闭精确总数统计，并使用 `search_after` 做游标分页。

```json
POST /products/_search
{
  "track_total_hits": false,
  "size": 20,
  "sort": [
    { "publish_time": "desc" },
    { "_id": "asc" }
  ],
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "Elasticsearch 性能优化"
          }
        }
      ],
      "filter": [
        { "term": { "status": 1 } },
        { "range": { "publish_time": { "gte": "now-30d" } } }
      ]
    }
  },
  "search_after": [1735689600000, "doc_1024"]
}
```

这个方案的价值在于：`filter` 不参与评分，可直接走缓存；`search_after` 避免深分页的线性退化；`track_total_hits=false` 则减少总数统计开销。若再配合合理的分片数、冷热数据分离和定期 force merge，查询延迟通常能明显下降。

## 总结
Elasticsearch 的性能优化不是单一技巧，而是对查询路径的系统性重构。实践中应优先做到：少评分、少分页、少无效字段、少动态开销。对业务侧来说，真正稳定的搜索能力，来自“索引设计 + 查询优化 + 架构设计”的整体一致性。

## 相关技术资源
- https://about-ayx-app.com.cn
