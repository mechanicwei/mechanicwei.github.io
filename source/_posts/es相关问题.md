---
title: es问题总结
date: 2025.04.30
tags:
---

## 新增字段，但未更新索引了，导致部分数据未索引
这部分数据会存到_source里
```json
POST /index/_update_by_query?conflicts=proceed
{
    "script": {
        "source": "ctx._source['new_field'] = ctx._source['new_field']",
        "lang": "painless"
    },
    "query": {
        "exists": {
            "field": "new_field"
        }
    }
}
```
这种做法是没有效果的，因为值没有变，不会触发更新索引，用sdk的update也是一样的效果
**解决方案**：可以先把数据置空，再把数据更新为正确的值
notes: 未索引的字段，aggs里也是无效的

## es的连接数一直长
新系统上线后，es的连接数一直长，最终导致es写入失败
```
x.x.x.x:9200: connect: cannot assign requested address
```
原因：create后忘了加上`defer res.Body.Close()`


## 写入失败
私有化的项目，查看 es-cluster.json 的日志，有下面的报错
```
Caused by: org.elasticsearch.common.breaker.CircuitBreakingException: [parent] Data too large, data for [<transport_request>] would be [986412652/940.7mb], which is larger than the limit of [986061209/940.3mb], real usage: [986407712/940.7mb], new bytes reserved: [4940/4.8kb], usages [request=0/0b, fielddata=6212/6kb, in_flight_requests=4940/4.8kb, accounting=13694164/13mb]
```
原因：客户的机器配置不高，当时运维配置的堆内存太低了，jvm.options
```
-Xms1g
-Xmx1g
```
超过`indices.breaker.total.limit`(默认95%)就会报错

**解决方案**：增大配置，重启es
