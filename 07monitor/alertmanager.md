# 统一安装目录

```
/data/
```

# 服务端口

```
grafana        3000
prometheus     9090
pushgateway    9091
alertmanager   9093
```

# 一、下载

```
wget https://hz-package.hzins.com/monitor.tar.gz
```



# 二、kaka-exporter安装

```
cd /data/
tar xf kafka_exporter-1.4.2.linux-amd64.tar.gz
mv kafka_exporter-1.4.2.linux-amd64 kafka_exporter
```

#### kafka-exporter服务文件

```
vim /lib/systemd/system/kafka_exporter.service

[Unit]
Description=Kafka Exporter
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
Restart=on-failure
ExecStart=/data/kafka_exporter/kafka_exporter \
  --kafka.server=10.1.28.96:9092 \
  --web.listen-address=0.0.0.0:9308

[Install]
WantedBy=multi-user.target
```

#### 启动服务

```
systemctl start kafka_exporter

systemctl stop kafka_exporter
```

# 三、安装jdk配置

```shell
cd /data/

rpm -ivh jdk-8u201-linux-x64.rpm

vim /etc/profie

#末尾添加如下内容
export JAVA_HOME=/usr/local/jdk1.8.0_201
 export JRE_HOME=${JAVA_HOME}/jre
 export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
 export PATH=${JAVA_HOME}/bin:$PATH
 
 #使配置生效
 source /etc/profie
```

# 四、安装prometheus，grafana，alertmanager，pushgateway

#### 4.1 、grafana

```shell
tar xf grafana-enterprise-8.3.3.linux-amd64.tar.gz
mv grafana-enterprise-8.3.3.linux-amd64 grafana

#启动
cd grafana/
nohup ./grafana-server start &
```

#### 4.2、prometheus

###### 4.2.1 、解压

```
tar xf prometheus-2.15.2.linux-amd64.tar.gz
mv prometheus-2.15.2.linux-amd64 prometheus
```

###### 4.2.2、修改配置文件

```yaml
vim prometheus/prometheus.yml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - localhost:9093
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
# 自定义告警规则文件
rule_files:
   - "aler.yml"
   - "hz-flink-task-errors.yml"
   - "flink.yml"
   - "flink-live.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["10.1.28.104:9090"]
  - job_name: 'node1'
    file_sd_configs:
    static_configs:
    - targets:
      - 10.1.28.104:9100
  - job_name: 'pushgateway'
    static_configs:
    - targets: ['10.1.28.104:9091']
  - job_name: 'kafka'
    scrape_interval: 60s
    scrape_timeout: 60s
    static_configs:
    - targets: ['10.1.28.96:9308']
    - targets: ['10.1.28.97:9308']
    - targets: ['10.1.28.98:9308']
```

###### 4.2.3、启动prometheus

```shell
vim /lib/systemd/system/prometheus.service

[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=user
Restart=on-failure
LimitNOFILE=655350
LimitNPROC=655350

ExecStart=/data/prometheus/prometheus \
  --config.file=/data/prometheus/prometheus.yml \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target


#启停命令
systemctl start prometheus
systemctl stop prometheus
```

#### 4.3、alertmanager

###### 4.3.1、解压

```
tar xf alertmanager-0.23.0.linux-amd64.tar.gz
mv alertmanager-0.23.0.linux-amd64 alertmanager
```

###### 4.3.2、修改alertmanager,yml
```yaml
cat > alertmanager.yml << 'EOF'
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.163.com':25'
  smtp_from: 'shd_stars@163.com''
  smtp_auth_username: 'xxx@163.com'' #告警发送邮箱地址
  smtp_auth_password: 'xxxxx' #授权码！不是邮箱的登陆密码
  smtp_require_tls: false

route:
  group_by: ['alertname'] # 分组标签
  group_wait: 10s # 分组等待时间，同一组内在10秒钟内还有其它告警，如果有则一同发送
  group_interval: 10s # 上下两组间隔时间
  repeat_interval: 300s # 重复告警间隔时间，间隔时间不要设置太短，容易出现告警轰炸
  receiver: 'mail' # 接收者是谁

receivers: # 定义接收者，将告警发送给谁
- name: 'mail'
  email_configs:
  - to: 'xxx@163.com'' #接收邮箱地址
    send_resolved: true  #是否接收恢复信息，false|true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    #确保这个配置下的标签内容相同才会抑制，也就是说警报中必须有这三个标签值才会被抑制。
    equal: ['alertname', 'dev', 'instance']    
EOF
```

```yaml
vim alertmanager/alertmanager.yml

global:
 resolve_timeout: 5m
 smtp_smarthost: 'smtp.exmail.qq.com:25'
 smtp_from: 'pythonalert@huize.com'
 smtp_auth_username: 'pythonalert@huize.com'
 smtp_auth_password: 'fA1ffg22d'
 smtp_require_tls: false
templates:
- '/data/alertmanager/template/*.tmpl'
route:
 group_by: ['alertname']
 group_wait: 20s
 group_interval: 5m
 repeat_interval: 60m
 receiver: 'default'
 routes:
 - match_re:
     severity: kafka-accumulation-.*
   receiver: 'receiver-kafka'
 - match_re:
     flink: flink-warning
   receiver: 'receiver-flink'
 - match:
     severity: kafka-topic-error
   receiver: 'receiver-kafka-error'
receivers:
- name: 'default'
  email_configs:
  - to: 'dengjun@huize.com'
    send_resolved: true
- name: 'receiver-kafka'
  email_configs:
  - to: 'dengjun@huize.com,yusongze@huize.com'
    send_resolved: true
    headers: { Subject: '长城项目, flink任务消息堆积，请及时处理'}
    html: '{{ template "email.to.html" .}}'
- name: 'receiver-kafka-error'
  email_configs:
  - to: 'dengjun@huize.com,yusongze@huize.com'
    headers: { Subject: '长城项目, flink任务出现异常数据，请及时处理'}
    html: '{{ template "email.to.html" .}}'
- name: 'receiver-flink'
  email_configs:
  - to: 'dengjun@huize.com,,yusongze@huize.com'
    headers: { Subject: '长城项目, flink任务已停止，请及时处理'}
    html: '{{ template "email.to.flink.html" .}}'
```

###### 4.3.3、添加邮件模板

```shell
mkdir template
#kafka邮件告警模板
vim go-mail.tmpl

{{ define "email.to.html" }}
{{ range .Alerts }}
=========start==========<br>
  TOPIC:     {{ .Labels.topic }}<br>
  消费组：   {{ .Labels.consumergroup }}<br>
  积压数值:  {{ .Labels.value }}<br>
  告警级别:  {{ .Labels.severity }}<br>
  告警类型:  {{ .Labels.alertname }}<br>
  所在主机:  {{ .Labels.instance }}<br>
  触发时间:  {{ .StartsAt.Format "2021-01-02 15:04:05" }}<br>
=========end==========<br>
{{ end }}
{{ end }}

#flink邮件告警模板
vim go-mail-flink.tmpl

{{ define "email.to.flink.html" }}
{{ range .Alerts }}
=========start==========<br>
  任务名称：{{ .Labels.job_name }}<br>
  任务ID:   {{ .Labels.job_id }}<br>
  告警类型: {{ .Labels.alertname }}<br>
  所在主机: {{ .Labels.instance }}<br>
  触发时间: {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }} <br>
=========end==========<br>
{{ end }}
{{ end }}
```

###### 4.3.4、启动alertmanager

```shell
vim /lib/systemd/system/alertmanager.service

[Install]
WantedBy=multi-user.target

[Unit]
After=network-online.target

[Service]
User=user
Restart=on-failure
LimitNOFILE=655350
LimitNPROC=655350

ExecStart=/data/alertmanager/alertmanager \
  --config.file=/data/alertmanager/alertmanager.yml

[Install]
WantedBy=multi-user.target

#启停命令
systemctl start alertmanager
systemctl stop alertmanager
```

#### 4.4、pushgateway

###### 4.4.1、解压

```
tar xf pushgateway-1.4.2.linux-amd64.tar.gz
mv pushgateway-1.4.2.linux-amd64 pushgateway
```

###### 4.4.2、启动

```shell
vim /lib/systemd/system/pushgateway.service

[Install]
WantedBy=multi-user.target

[Unit]
Description=pushgateway Server
After=network-online.target

[Service]
User=user
Restart=on-failure
LimitNOFILE=655350
LimitNPROC=655350

ExecStart=/data/pushgateway/pushgateway

[Install]
WantedBy=multi-user.target


#启停命令
systemctl start pushgateway
systemctl stop pushgateway
```
