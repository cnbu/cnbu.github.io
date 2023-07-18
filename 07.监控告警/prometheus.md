# prometheus数据类型
```
1.Counter计数器
Counter类型，Counter类型好比计数器，只增不减(除非系统发生了重置)，用于统计类似于：CPU运行时间，API访问总次数，异常发生次数等等场景。这些指标的特点就是增加不减少。

2.Gauge(仪表盘类型)(天穹目前主要用这个)
Gauge是可增可减的指标类，可以用于反应当前应用的状态。比如在监控实例时，主机当前的内存大小，可用内存大小

3.Histogram(直方图类型)

4.Summary(摘要类型)
```
# 一：常用命令

```shell
#更新配置文件
前置启动需要添加
--web.enable-lifecycle
curl -X POST http://localhost:9090/-/reload
```

###### consul注册
```shell
#windows
在浏览器地址拦里输入newyum.ha.com/soft/ansible-app
找到windows_exporter下载然后安装
windows服务里面将windows_exporter的状态改成自动延迟和2分钟恢复
管理员启动cmd输入lodctr /R


#linux
/usr/bin/curl -v -X PUT -d '{"Meta":{"hostname":"dba-win2008-prod-ns-172-22-8-52", "exporter_type":"windows_exporter"}, "id":"dba-win2008-prod-ns-172-22-8-52","name":"windows", "address":"172.22.8.52",  "port":9182, "tags":["dba","windows-exporter"],"checks":[{"http":"http://172.22.8.52:9182", "interval":"5s"}]}' http://172.22.8.48:8500/v1/agent/service/register
```

###### 匹配标签器

```shell
=：选择与提供的字符串完全相同的标签。       				     #等值比较
!=：选择不等于提供的字符串的标签。		   				         #不等值比较
=~：选择与提供的字符串进行正则表达式匹配的标签。	          #等值正则匹配
!~：选择不与提供的字符串进行正则表达式匹配的标签。         #不等值正则匹配
```


告警文件
```
groups:
- name: kafka
  rules:
  - alert: kafka-accumulation
    expr: (sum(kafka_consumergroup_lag{consumergroup=~"SYNC_binlog-collect-task-test|LAWAGELOGMONITOR_binlog-collect-task-test|ACCOUNTLOGMONITOR_binlog-collect-task-test"}) by (topic,instance,hostname,job,consumergroup)) > 2000
    for: 30s
    labels:
      severity: kafka-accumulation-warning
#    annotations:
      value: "{{ $value }}"
```

# 二：Thanos

## 2.1: Thanos 的主要特性

```
1. 全局视图：与现有 Prometheus 设置无缝集成，能够跨集群联合，跨所有连接的 Prometheus 服务器的全局查询视图，很好的对 HA 中的 Prometheus 进行容错路由查询。
2. 不受限的保留数据：支持各种对象存储。
3. 压缩和降准采样：对历史数据进行自定义的降准采样以大幅提高查询速度。
4. 实现包括 Prometheus 在内的各个组件高可用。
5. 能够记录规则，实现告警。
```

## 2.2: Thanos架构图

```shell
#Sidecar
Sidecar 必须与 Prometheus 一起部署，实现将 Prometheus 监控数据上传到对象存储，并允许 Querier 查询器高效的查询 Prometheus 数据。

#Bucket
Bucket 是用来检测对象存储（Object Storage）的一组工具,以及提供了一个 web 界面，来查看对象存储中的块（Blocks）。对象存储可以选择 GCS(Google Cloud Storage),AWS/S3,Azure Storage Account,OpenStack Swift,Tencent COS,AliYun OSS 等，本文部署实践使用的 S3 作为对象存储。

#Store
Store 组件在对象存储上实现 Store API，充当了对象存储的网关，使其与对象存储同步，本地只保留对象存储中所有块的少量的源数据信息。

#Querier/Query
Querier 组件实现了 Prometheus http v1 API，完全兼容 Promql 查询，他可以连接 Store 组件和 Sidecar 组件实现从对象存储和 Prometheus 中查询所需数据，并可以从任何实现 Store API 的对象中查询数据。
Querier 组件是完全无状态的查询器，可以水平扩展实现高可用。

#Compact
Compact 组件为 Thanos 的压缩器。负责压缩在对象存储中的数据，还负责数据的降准采样。

例：超过 30 天的数据创建 5m 的降准采样（降准采样并不是为了减少存储，而是为了在进行长时间范围查询的时候更快的返回结果）
```

## 2.3: minio对象存储部署

172.20.1.143

```shell
wget -P /data/  http://dl.minio.org.cn/server/minio/release/linux-amd64/minio
chmod +x minio
./minio server /data/miniodata/   #启动服务并指定数据存储路径


vim /lib/systemd/system/minio.service
[Unit]

[Service]
User=thanos
Restart=always
RestartSec=1
Restart=on-failure
LimitNOFILE=655350
LimitNPROC=655350

ExecStart=/data/minio /data/miniodata/

[Install]
WantedBy=multi-user.target
```

访问地址 [http://172.20.1.143:9000/minio/login](http://172.20.1.143:9000/minio/login)

默认用户 minioadmin

默认密码 minioadmin





## 2.4: Thanos部署

所有机器都需要执行一下步骤

```
二进制包下载
https://github.com/thanos-io/thanos/releases

useradd thanos
cd /data/
tar xf thanos-0.20.1.linux-amd64.tar.gz
mv thanos-0.20.1.linux-amd64 thanos
chown thanos.thanos thanos
```

#### 2.4.1: 机器规划

```
ip                     部署的组件
172-20-9-107           prometheus     Sidecar
172-20-1-141           thanos store   thanos query                                                            
172-20-1-142           prometheus     Sidecar                                                    
172-20-1-143		   minio

minio 对象存储
```

#### 2.4.2: 修改Prometheus配置文件

172.20.9.107  172.20.1.142 2台

###### 4.2.1: prometheus.yml

```yaml
172.20.9.107

# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  external_labels:
    regino: 'prod'
    replica: 0

172.20.1.142
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  external_labels:
    regino: 'prod'
    replica: 1


地区和副本必须有
external_labels，标注你的地域。如果你是多副本运行，需要声明你的副本标识，如 0号，1，2 三个副本采集一模一样的数据，另外2个 Prometheus 就可以同时运行，只是 replica 值不同而已
```

prometheus.service

```shell
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=prometheus
Restart=always
RestartSec=1
Restart=on-failure
LimitNOFILE=655350
LimitNPROC=655350

ExecStart=/data/prometheus/prometheus \
  --config.file=/data/prometheus/prometheus.yml \
  --storage.tsdb.path=/data/prometheus/data \
  --storage.tsdb.max-block-duration=2h \
  --storage.tsdb.min-block-duration=2h \
  --storage.tsdb.wal-compression \
  --storage.tsdb.retention.time=24h \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target



--storage.tsdb.retention.time=24h   本地日志保留时间
--web.enable-lifecycle  通过HTTP请求启用关闭和重新加载
--storage.tsdb.max-block-duration=2h  在持久化之前数据块的最大保存期
--storage.tsdb.min-block-duration=2h  在持久化之前数据块的最短保存期
```

#### 2.4.3: 启动 thanos-sidecar

机器地址：172.20.9.107  172.20.1.142

###### 2.4.3.1: thanos-sidecar.service

```shell
[Unit]


[Service]
User=thanos
Restart=always
RestartSec=1
Restart=on-failure
LimitNOFILE=655350
LimitNPROC=655350

ExecStart=/data/thanos/thanos sidecar \
  --tsdb.path /data/prometheus/data \
  --prometheus.url http://172.20.1.142:9090 \
  --objstore.config-file /data/thanos/bucket_config.yaml \
  --shipper.upload-compacted

[Install]
WantedBy=multi-user.target



##--tsdb.path 此路径必须与prometheus的--storage.tsdb.path相同，否则无法上传数据到存储
##启动默认端口10901 10902
##--http-address              0.0.0.0:19191              # HTTP endpoint for collecting metrics on the Sidecar
##--grpc-address              0.0.0.0:19090              # GRPC endpoint for StoreAPI
```

###### 2.4.3.2: bucket_config.yaml

对象存储连接信息

vim /data/thanos/bucket_config.yaml

```shell
type: S3
config:
    bucket: "prometheus"            #s3桶文件名
    endpoint: "172.20.1.143:9000"   #s3连接地址
    access_key: "minioadmin"        #s3用户名
    insecure: true                  #s3禁用https
    secret_key: "minioadmin"        #s3用户密码
```

#### 2.4.4:启动 store  query

机器地址：172.20.1.141

###### 2.4.4.1: thanos-store.service

```shell
[Unit]

[Service]
User=thanos
Restart=on-failure
Restart=always
RestartSec=1
LimitNOFILE=655350
LimitNPROC=655350

ExecStart=/data/thanos/thanos store \
  --data-dir /data/thanos/store \
  --objstore.config-file /data/thanos/bucket_config.yaml \
  --http-address 0.0.0.0:19191 \
  --grpc-address 0.0.0.0:19090


[Install]
WantedBy=multi-user.target


#--objstore.config-file /data/thanos/bucket_config.yaml  连接对象存储信息
#--http-address 0.0.0.0:19191  thanos-store连接地址 http://172.20.1.141:19191/loaded
```

###### 2.4.4.2: thanos-query.service

```shell
[Unit]

[Service]
User=thanos
Restart=always
RestartSec=1
Restart=on-failure
LimitNOFILE=655350
LimitNPROC=655350

ExecStart=/data/thanos/thanos query \
  --http-address=0.0.0.0:8090 \
  --query.replica-label "replica" \
  --query.replica-label "replica1" \
  --store=172.20.1.142:10901 \
  --store=172.20.9.107:10901 \
  --store=172.20.1.141:19090


[Install]
WantedBy=multi-user.target


--store=172.20.9.107:10901   调用sidecar远程端口
--store=172.20.1.142:10901   调用sidecar远程端口
--store=172.20.1.141:19090   调用store 端口

按标签去除重复数据
--query.replica-label "replica" 
--query.replica-label "replica1"
```

#### 2.4.5:  thanos-query连接地址

[http://172.20.1.141:8090/graph](http://172.20.1.141:8090/graph)




