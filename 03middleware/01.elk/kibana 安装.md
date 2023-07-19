#### 3.1：下载

[https://www.elastic.co/cn/downloads/past-releases#kibana](https://www.elastic.co/cn/downloads/past-releases#kibana)

#### 3.2：解压，修改kibana.yml

```
设定kibana端口
设定所在主机ip
指定Elasticsearch连接地址
设置界面为中文

server.port: 5601
server.host: "172.19.0.236"
elasticsearch.hosts: ["http://172.19.0.236:9200"]
i18n.locale: "zh-CN"
```

#### 3.3：启动kibana

```
nohup ./bin/kibana --allow-root & > /dev/null 2>&1
```
