```shell
# 避免发生OOM，发生OOM对集群影响很大的
indices.breaker.total.limit: 80%

# 有了这个设置，最久未使用（LRU）的 fielddata 会被回收为新数据腾出空间   
indices.fielddata.cache.size: 10%

# fielddata 断路器默认设置堆的  作为 fielddata 大小的上限。
indices.breaker.fielddata.limit: 60%

# request 断路器估算需要完成其他请求部分的结构大小，例如创建一个聚合桶，默认限制是堆内存
indices.breaker.request.limit: 60%
```
