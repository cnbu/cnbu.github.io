#### 修改副本分片数量

```
PUT /索引/_settings
{
  "number_of_replicas": 0
}
```

#### 修改最大shards数量

```yaml
es7以后单节点1000分片

方式①：配置文件 elasticsearch.yml
    增加配置项：

    cluster.max_shards_per_node: 2000


方式②：动态修改该配置项：
_cluster/settings GET、PUT请求
    {
      "transient": {
        "cluster": {
          "max_shards_per_node":2000
        }
      }
    }

恢复之前默认：

_cluster/settings GET、PUT请求
    {
      "transient": {
        "cluster": {
          "max_shards_per_node":null
        }
      }
    }
```

#### 快照相关

```
GET _snapshot/_all	查看所有快照。
GET _snapshot/<snapshot_name>/_status	查看指定快照的进度。
GET _snapshot/_all	查看所有快照。
GET _snapshot/<snapshot_name>/_status	查看指定快照的进度。
```

#### 上传索引格式

```
curl -XPUT http://192.168.53.32:9200/es_saas_seller_xb -H “Content-Type:application/json” -d "{}"
```

#### 恢复快照

```
POST _snapshot/apm-7.2.1-transaction/apm-7.2.1-transaction-2021.04.05/_restore
```

#### 备份索引

```shell
PUT _snapshot/back/huntian-2020
{
  "indices": "huntian-2020",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

```
curl http://10.4.0.223:9201/_cat/indices/apm-7.2.1-metric*  >/root/test.txt

awk '{print $3}'  test.txt > 1.txt
```

```shell
#!/bin/bash
for i in `cat /root/1.txt`
do
        curl -XPUT http://10.4.0.223:9201/_snapshot/apm-7.2.1-transaction/${i}?wait_for_completion=true -H 'Content-Type: application/json' -d '
                {
                  "indices": "'${i}'",
                  "ignore_unavailable": true,
                  "include_global_state": false
                }'
  sleep 10
done
```

```
curl -u user:password   -XPOST "http://ip_new:port/_reindex" -H 'Content-Type:application/json'  -d '  #这里的user，port，ip，password都是新集群的
{
  "conflicts": "proceed",
  "source": {
    "remote": { #这是2.4.1的配置
      "host": "http://ip:port/",  
      "username": "user",
      "password": "password"
    },
    "index": "index_name_source", #2.4.1的要迁移的索引
    "query": {
       "term": {
        "_type":"type_name"  #查询单个type的数据
}
    },
    "size": 6000  #这个参数可以根据实际情况调整
  },
  "dest": {
    "index": "index_name_dest",  #这里是新es的索引名字
  }
}'
```

#### X-Pack配置

```shell
1#elasticsearch-certutil命令生成证书
bin/elasticsearch-certutil ca -out config/elastic-certificates.p12 -pass ""

2#配置加密通信
cd elasticsearch/config
vi elasticsearch.yml

xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate 
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12 
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12

3#设置集群密码
#执行设置用户名和密码的命令,这里需要为4个用户分别设置密码，elastic, kibana, logstash_system,beats_system，apm_system

bin/elasticsearch-setup-passwords interactive     --手动配置每个用户密码模式
bin/elasticsearch-setup-passwords auto            --自动配置每个用户密码模式

密码设置为 123456
```

#### 新建索引

```
#指定3分片，1副本

PUT ads_ev_qx_content_audit_di
{
	"settings": {
		"number_of_shards": 3,
		"number_of_replicas": 1
	}
}
```

#### 修改默认副本

```
PUT /_template/index_defaults 
{
  "template": "*", 
  "settings": {
    "number_of_shards": 4
  }
}
```

#### 修改节点分片上限

```
PUT /_cluster/settings
{
  "persistent": {
    "cluster": {
      "max_shards_per_node":10000
    }
  }
}
```

#### Rollover Index 别名滚动指向新创建的索引

```shell
#Rollover Index 示例：创建一个名字为logs-0000001 、别名为logs_write 的索引：

PUT /logs-000001
{
  "aliases": {
    "logs_write": {}
  }
}

#添加1000个文档到索引logs-000001，然后设置别名滚动的条件

POST /logs_write/_rollover
{
  "conditions": {
    "max_age":   "7d",
    "max_docs":  1000,
    "max_size":  "5gb"
  }
}


#说明：如果别名logs_write指向的索引是7天前（含）创建的或索引的文档数>=1000或索引的大小>= 5gb，则会创建一个新索引 logs-000002，并把别名logs_writer指向新创建的logs-000002索引
```

#### 将缓存在内存中的索引数据刷新到持久存储中

```
POST twitter/_flush
```

#### 修改es磁盘容量使用

```shell
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "80%",
    "cluster.routing.allocation.disk.watermark.high": "50gb",
    "cluster.info.update.interval": "1m"
  }
}

#cluster.routing.allocation.disk.watermark.low：控制磁盘使用的低水位。默认为85%，意味着如果节点磁盘使用超过85%，则ES不允许在分配新的分片。当配置具体的大小如80MB时，表示如果磁盘空间小于80MB不允许分配分片。

#cluster.routing.allocation.disk.watermark.high：控制磁盘使用的高水位。默认为90%，意味着如果磁盘空间使用高于90%时，ES将尝试分配分片到其他节点

#cluster.routing.allocation.disk.watermark.flood_stage
#默认为磁盘容量的95％。Elasticsearch对每个索引强制执行只读索引块（index.blocks.read_only_allow_delete）。这是防止节点耗尽磁盘空间的最后手段。只读模式待磁盘空间充裕后，需要人工解除
```

#### 修改索引只读状态

```
PUT /_all/_settings
{
  "index.blocks.read_only_allow_delete": null
}
```

#### 修改es查询最大返回页数

```shell
PUT beidou_meta_index3/_settings
 {
  "max_result_window" : "20000"
 }
 
 
 #max_result_window 设置的越大，越往后翻意味着：内存耗费越大、耗时越长。
```

#### 获取索引容量

```
curl -s http://10.4.0.223:9201/_cat/indices?v | awk '{print $3,$9}' | sort -n |grep -v '^\.' > /tmp/es.xlsx
```

#### 清空索引

```shell
POST my_index/_delete_by_query
{
  "query":{
    "match_all":{}
  }
}

#delete_by_query是通过查询删除数据，数据不会立即删除，而是形成新段，待段合并时机再删除

#DELETE myindex

#DELETE 索引会索引层面删除索引及数据，并释放空间。
```

#### 添加别名

```
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "myindex",
        "alias": "myindex_alias"
      }
    }
  ]
}
```

#### 嵌套查询返回数修改

```shell
PUT _all/_settings
{
  "index.mapping.nested_objects.limit":20000
}


index.mapping.total_fields.limit
#索引中的最大字段数。字段和对象映射以及字段别名都计入此限制。默认值为1000。
index.mapping.depth.limit
#字段的最大深度，以内部对象的数量来衡量。例如，如果所有字段都在根对象级别定义，则深度为1.如果有一个对象映射，则深度为2，等等。默认值为20。
index.mapping.nested_fields.limit
#索引中嵌套字段的最大数量，默认为50.索引1个包含100个嵌套字段的文档实际上索引101个文档，因为每个嵌套文档都被索引为单独的隐藏文档。
index.mapping.nested_objects.limit
#跨所有嵌套字段的单个文档中嵌套json对象的最大数量，默认为10000.在嵌套字段中使用100个对象的数组索引一个文档，实际上将创建101个文档，因为每个嵌套对象将作为单独的索引编制索引隐藏文件。
index.mapping.field_name_length.limit
#设置字段名称的最大长度。默认值为Long.MAX_VALUE（无限制）。此设置实际上不是解决映射爆炸的问题，但如果要限制字段长度，则可能仍然有用。通常不需要设置此设置。默认是可以的，除非用户开始添加大量具有真正长名称的字段。
动态mappingedit
#在使用之前不需要定义字段和映射类型。由于动态映射，只需索引文档即可自动添加新的字段名称。可以将新字段添加到顶级映射类型，内部对象和嵌套字段。
```

#### Kibana server is not ready yet

```
curl -XDELETE http://localhost:9200/.kibana
curl -XDELETE http://localhost:9200/.kibana*
curl -XDELETE http://localhost:9200/.kibana_2
curl -XDELETE http://localhost:9200/.kibana_1
```

#### 下线节点驱逐节点数据

```
PUT _cluster/settings
{"transient":{"cluster.routing.allocation.exclude._ip":"10.10.1.35"}}
```

#### 查询索引id后删除

```yaml
POST /open_product/_delete_by_query
{
  "query": {
    "bool": {
      "filter": [
        {
              "term": {
                "productId": {
                  "value": "823"
                }
              }
            }
      ]
    }
  }
}
```
# 
