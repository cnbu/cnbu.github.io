```shell
1.获取 elasticsearch_exporter
cd /data
wget https://github.com/justwatchcom/elasticsearch_exporter/releases/download/v1.1.0/elasticsearch_exporter-1.1.0.linux-amd64.tar.gz

2.安装
tar xf elasticsearch_exporter-1.1.0.linux-amd64.tar.gz
mv elasticsearch_exporter-1.1.0.linux-amd64 elasticsearch-exporter
cd elasticsearch-exporter
./elasticsearch-exporter --es.uri=http://172.21.0.150:9200 --web.listen-address=":9114"  > exporter.log 2>&1 &


3.注册
/usr/bin/curl -v  -X PUT -d '{"Meta":{"hostname":"service-rmz-18130", "exporter_type":"node_exporter"}, "id":"vn_9100_service-rmz-18130","name":"vm", "address":"192.168.18.130",  "port":9100, "tags":["vm-exporter"],"checks":[{"http":"http://192.168.18.130:9100", "interval":"5s"}]}' http://172.22.8.48:8500/v1/agent/service/register


哆啦A梦 告警配置

elasticsearch_cluster_health_status{color="yellow"}   
1 
wp_es5业务专用监控_集群状态yellow

elasticsearch_cluster_health_status{color="red"}
1
wp_es5业务专用监控_集群状态red

elasticsearch_cluster_health_number_of_nodes
5
wp_es5业务专用监控_集群有不健康节点



curl -X PUT -d '{"id": "pt-service-ns-41","name":"elasticsearch","address": "10.10.1.24","port": 9114,"tags": ["elasticsearch_exporter"],"checks": [{"http": "http://10.10.1.24:9114","interval": "5s"}]}' http://172.22.8.48:8500/v1/agent/service/register



http://172.21.9.108:8500/v1/agent/service/deregister
windows 注册

在浏览器地址拦里输入newyum.ha.com/soft/ansible-app
找到windows_exporter下载然后安装
windows服务里面将windows_exporter的状态改成自动延迟和2分钟恢复
管理员启动cmd输入lodctr /R

在linux上注册
/usr/bin/curl -v -X PUT -d '{"Meta":{"hostname":"dba-win2008-prod-ns-172-22-8-52", "exporter_type":"windows_exporter"}, "id":"dba-win2008-prod-ns-172-22-8-52","name":"windows", "address":"172.22.8.52",  "port":9182, "tags":["dba","windows-exporter"],"checks":[{"http":"http://172.22.8.52:9182", "interval":"5s"}]}' http://172.22.8.48:8500/v1/agent/service/register


取消注册：修改vm 开头的serverID
curl -X PUT http://172.22.8.48:8500/v1/agent/service/deregister/vm_10-4-1-49_do-ops-ns-41
```
