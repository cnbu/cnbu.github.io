# 一：环境

#### 1：系统

```
CentOS Linux release 7.5.1804 (Core)
```

#### 2：es版本

```
elasticsearch-7.2.1-linux-x86_64.tar.gz
```

#### 3：下载地址

```
链接：https://pan.baidu.com/s/1Z4mwlC4sfvC_B6RR8rUOxw 
提取码：nbck
```

#### 4：机器

```
172.21.0.236
172.21.0.237
172.21.0.238
```

# 二：安装步骤(所有机器)

#### 2.1：系统优化

原因1： Elasticsearch 在节点和 HTTP 客户端之间进行通信也使用了大量的套接字（注：sockets）。 所有这一切都需要足够的文件描述符。

原因2：许多现代的 Linux 发行版本，每个进程默认允许一个微不足道的 1024 文件描述符。这对一个小的 Elasticsearch 节点来说实在是太低了，更不用说一个处理数以百计索引的节点。

```shell
echo "* soft nofile 65535" >> /etc/security/limits.conf

echo "* hard nofile 65535" >> /etc/security/limits.conf

echo "vm.max_map_count=262144" >> /etc/sysctl.conf

sysctl -p
```

#### 2.2：上传安装包(省略)

#### 2.3：解压（解压目录/data/）

```
tar xf elasticsearch-7.2.1-linux-x86_64.tar.gz -C /data/
```

#### 2.4：创建用户以及修改目录权限

```
useradd elasticsearch

chown -R elasticsearch:elasticsearch /data/elasticsearch-7.2.1
```

#### 2.5：elasticsearch.yml核心配置解释

```yaml
cluster.name: cluster1
node.name: node1
node.master: true
node.data: true
path.data: /data/elasticsearch-7.2.1/data
path.logs: /data/elasticsearch-7.2.1/logs
network.host: 172.19.0.236
discovery.zen.minimum_master_nodes: 2
gateway.recover_after_nodes: 1
cluster.initial_master_nodes: ["172.21.0.236:9300","172.21.0.237:9300""172.21.0.238:9300"]
discovery.seed_hosts: ["172.21.0.236:9300","172.21.0.237:9300""172.21.0.238:9300"]
transport.tcp.port: 9300
http.port: 9200
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-credentials: true
xpack.security.enabled: false


cluster.name: 集群名称，唯一确定一个集群。
node.name：节点名称，一个集群中的节点名称是唯一固定的，不同节点不能同名。
node.master: 主节点属性值
node.data: 数据节点属性值
path.data:  es数据存放位置
path.logs： es日志存放位置 
network.host： 本节点的ip

discovery.zen.minimum_master_nodes: 2
设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点。默认为1，对于大的集群来说，可以设置大一点的值（2-4）

gateway.recover_after_nodes: 1 设置集群中N个节点启动时进行数据恢复，默认为1。
cluster.initial_master_nodes：指定集群初次选举中用到的具有主节点资格的节点，称为集群引导，只在第一次形成集群时需要。
discovery.seed_hosts:节点发现需要配置一些种子节点，与7.X之前老版本：disvoery.zen.ping.unicast.hosts类似，一般配置集群中的全部节点
transport.port：9300——集群之间通信的端口，若不指定默认：9300
http.port: 本节点的http端口
http.cors.enabled: true 是否开启跨域访问
http.cors.allow-origin: “*” 开启跨域访问后的地址限制，*表示无限制
http.cors.allow-credentials	是否返回设置的跨域Access-Control-Allow-Credentials头，如果设置为true,那么会返回给客户端
xpack.security.enabled: false 是否开启xpack组件
```

#### 2.6：所有节点es内存修改

```
vim  /data/elasticsearch-7.2.1/config/jvm.options

推荐为服务器内存的一半
```

#### 2.7：172.21.0.236  yml文件配置

```yaml
rm -rf /data/elasticsearch-7.2.1/config/elasticsearch.yml

vim /data/elasticsearch-7.2.1/config/elasticsearch.yml


cluster.name: cluster1
node.name: node1
node.master: true
node.data: true
path.data: /data/elasticsearch-7.2.1/data
path.logs: /data/elasticsearch-7.2.1/logs
network.host: 172.19.0.236
discovery.zen.minimum_master_nodes: 2
gateway.recover_after_nodes: 1
cluster.initial_master_nodes: ["172.21.0.236:9300","172.21.0.237:9300""172.21.0.238:9300"]
discovery.seed_hosts: ["172.21.0.236:9300","172.21.0.237:9300""172.21.0.238:9300"]
transport.tcp.port: 9300
http.port: 9200
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-credentials: true
xpack.security.enabled: false
```

#### 2.8：172.21.0.237  yml文件配置

```yaml
rm -rf /data/elasticsearch-7.2.1/config/elasticsearch.yml

vim /data/elasticsearch-7.2.1/config/elasticsearch.yml


cluster.name: cluster1
node.name: node1
node.master: true
node.data: true
path.data: /data/elasticsearch-7.2.1/data
path.logs: /data/elasticsearch-7.2.1/logs
network.host: 172.19.0.237
discovery.zen.minimum_master_nodes: 2
gateway.recover_after_nodes: 1
cluster.initial_master_nodes: ["172.21.0.236:9300","172.21.0.237:9300""172.21.0.238:9300"]
discovery.seed_hosts: ["172.21.0.236:9300","172.21.0.237:9300""172.21.0.238:9300"]
transport.tcp.port: 9300
http.port: 9200
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-credentials: true
xpack.security.enabled: false
```

#### 2.9：172.21.0.238  yml文件配置

```yaml
rm -rf /data/elasticsearch-7.2.1/config/elasticsearch.yml

vim /data/elasticsearch-7.2.1/config/elasticsearch.yml


cluster.name: cluster1
node.name: node1
node.master: true
node.data: true
path.data: /data/elasticsearch-7.2.1/data
path.logs: /data/elasticsearch-7.2.1/logs
network.host: 172.19.0.238
discovery.zen.minimum_master_nodes: 2
gateway.recover_after_nodes: 1
cluster.initial_master_nodes: ["172.21.0.236:9300","172.21.0.237:9300""172.21.0.238:9300"]
discovery.seed_hosts: ["172.21.0.236:9300","172.21.0.237:9300""172.21.0.238:9300"]
transport.tcp.port: 9300
http.port: 9200
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-credentials: true
xpack.security.enabled: false
```

#### 2.10：service文件编写（所有主机相同）

```
vim /lib/systemd/system/elasticsearch.service
```

```shell
[Unit]
Description=Elasticsearch
Documentation=http://www.elastic.co
Wants=network-online.target
After=network-online.target

[Service]
RuntimeDirectory=elasticsearch
PrivateTmp=true

Environment=JAVA_HOME=/data/elasticsearch-7.2.1/jdk
Environment=ES_HOME=/data/elasticsearch-7.2.1
Environment=ES_PATH_CONF=/data/elasticsearch-7.2.1/config
Environment=PID_DIR=/var/run/elasticsearch

WorkingDirectory=/data/elasticsearch-7.2.1

User=elasticsearch
Group=elasticsearch

ExecStart=/data/elasticsearch-7.2.1/bin/elasticsearch -p ${PID_DIR}/elasticsearch.pid --quiet

# StandardOutput is configured to redirect to journalctl since
# some error messages may be logged in standard output before
# elasticsearch logging system is initialized. Elasticsearch
# stores its logs in /var/log/elasticsearch and does not use
# journalctl by default. If you also want to enable journalctl
# logging, you can simply remove the "quiet" option from ExecStart.
StandardOutput=journal
StandardError=inherit

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65535

# Specifies the maximum number of processes
LimitNPROC=4096

# Specifies the maximum size of virtual memory
LimitAS=infinity

# Specifies the maximum file size
LimitFSIZE=infinity

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=0

# SIGTERM signal is used to stop the Java process
KillSignal=SIGTERM

# Send the signal only to the JVM rather than its control group
KillMode=process

# Java process is never killed
SendSIGKILL=no

# When a JVM receives a SIGTERM signal it exits with code 143
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target

# Built for packages-7.2.1 (packages)
```

#### 2.11 启动服务

```shell
systemctl daemon-reload

systemctl start elasticsearch
```
