# 一：系统信息

### 1.1  软件信息

```
1.系统版本
CentOS Linux release 7.5.1804 (Core)

2.kafka版本
kafka_2.12-2.2.0

3.oracle版本
11G

4.zookeeper
zookeeper-3.4.14.tar.gz

5.debezium
debezium-connector-oracle-1.7.0.Beta1-plugin.tar.gz
instantclient-basic-linux.x64-21.3.0.0.0.zip

安装参考链接
https://blog.csdn.net/weixin_40548182/article/details/117956341?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-9.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-9.control

软件链接
链接：https://pan.baidu.com/s/13c6tuCDi1tmWYXgMlsYpMg 
提取码：gkmd

debezium官网oracle链接
https://debezium.io/documentation/reference/connectors/oracle.html#how-the-oracle-connector-works
```

### 1.2    debezium模式

```
默认情况下，Debezium Oracle 连接器使用本机 Oracle LogMiner 摄取更改。可以切换连接器以改用 Oracle XStream。要将连接器配置为使用 Oracle XStream，您必须应用与用于 LogMiner 的数据库和连接器配置不同的特定数据库和连接器配置。

先决条件
要使用 XStream API，您必须拥有 GoldenGate 产品的许可证。不需要安装 GoldenGate。
```

# 二：安装

### 2.1 oracle配置

##### 2.1.1 切换到oracle用户

```
su - oracle
```

2.1.2  登录oracle数据库

```
sqlplus / as sysdba
```

2.1.3 开启归档日志

需要在mount状态下开始数据库归档，重启至mount

```
shutdown immediate
startup mount
alter database open;
shutdown immediate
startup mount
alter database archivelog;  #开启数据库archivelog模式
alter database add supplemental log data (all) columns;   #启动日志补充记录
```

创建用户赋予权限使kafka-connect可以访问：

```
create user kafka02 identified by kafkapass;
GRANT CREATE SESSION TO kafka02;
GRANT SELECT ON V_$DATABASE to kafka02;
GRANT FLASHBACK ANY TABLE TO kafka02;
GRANT SELECT ANY TABLE TO kafka02;
GRANT SELECT_CATALOG_ROLE TO kafka02;
GRANT EXECUTE_CATALOG_ROLE TO kafka02;
GRANT SELECT ANY TRANSACTION TO kafka02;


GRANT CREATE TABLE TO kafka02;
GRANT LOCK ANY TABLE TO kafka02;
GRANT ALTER ANY TABLE TO kafka02;
GRANT CREATE SEQUENCE TO kafka02;

GRANT EXECUTE ON DBMS_LOGMNR TO kafka02;
GRANT EXECUTE ON DBMS_LOGMNR_D TO kafka02;

GRANT SELECT ON V_$LOG TO kafka02;
GRANT SELECT ON V_$LOG_HISTORY TO kafka02;
GRANT SELECT ON V_$LOGMNR_LOGS TO kafka02;
GRANT SELECT ON V_$LOGMNR_CONTENTS TO kafka02;
GRANT SELECT ON V_$LOGMNR_PARAMETERS TO kafka02;
GRANT SELECT ON V_$LOGFILE TO kafka02;
GRANT SELECT ON V_$ARCHIVED_LOG TO kafka02;
GRANT SELECT ON V_$ARCHIVE_DEST_STATUS TO kafka02;
```

### 2.2 kafka-connect

##### 2.2.1.解压软件，所有软件都默认放在 /data

```
tar xf debezium-connector-oracle-1.7.0.Beta1-plugin.tar.gz
tar xf instantclient-basic-linux.x64-21.3.0.0.0.zip
```



##### 2.2.2.复制jar包到对应目录

```
cp /data/debezium-connector-oracle/*.jar /data/kafka/libs
cp /data/instantclient_21_3/*.jar /data/kafka/libs/
```

##### 2.2.3.kafka-connect以及kafka配置

注：3台都需要修改，

```
bootstrap.servers=172.19.1.116:9092,172.19.1.117:9092,172.19.1.118:9092  #注意
group.id=connect-cluster
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=true
value.converter.schemas.enable=true
offset.storage.topic=connect-offsets
offset.storage.replication.factor=1
config.storage.topic=connect-configs
config.storage.replication.factor=1
status.storage.topic=connect-status
status.storage.replication.factor=1
offset.flush.interval.ms=30000
plugin.path=/data/debezium-connector-oracle  #注意
```

kafka配置：注意不要禁用topic自动创建功能

```
broker.id=116
port=9092
listeners=PLAINTEXT://172.19.1.116:9092
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
default.replication.factor=2
log.dirs=/data/kafka/data
num.partitions=1
num.recovery.threads.per.data.dir=1
num.replica.fetchers=8
log.cleanup.policy=delete
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=172.19.1.116:2181,172.19.1.117:2181,172.19.1.118:2181
zookeeper.connection.timeout.ms=6000
#auto.create.topics.enable=false
unclean.leader.election.enable=false
auto.leader.rebalance.enable=false
message.max.bytes=10000120
offsets.topic.replication.factor=2
kafka.logs.dir=/data/kafka/logs
```

##### 2.2.4.启动kafka-connect

```
cd /data/kafka

bin/connect-distributed.sh config/connect-distributed.properties
```

##### 2.2.5.查看端口 kafka-connect端口：8083

```
[root@wp-prod-ns-kafka-172-19-1-116 data]# ss -tnl
State       Recv-Q Send-Q                                                 Local Address:Port                                                                Peer Address:Port              
LISTEN      0      32768                                                   172.19.1.116:9100                                                                           *:*                  
LISTEN      0      128                                                                *:22                                                                             *:*                  
LISTEN      0      50                                               ::ffff:172.19.1.116:9092                                                                          :::*                  
LISTEN      0      50                                                                :::2181                                                                          :::*                  
LISTEN      0      50                                                                :::27307                                                                         :::*                  
LISTEN      0      50                                               ::ffff:172.19.1.116:3888                                                                          :::*                  
LISTEN      0      50                                                                :::8083                                                                          :::*                  
LISTEN      0      128                                                               :::22                                                                            :::*                  
LISTEN      0      50                                                                :::28472                                                                         :::*                  
LISTEN      0      50                                                                :::13246                                                                         :::*
```

##### 2.2.6.创建连接器

```
curl -X POST http://172.19.1.116:8083/connectors -H "Content-Type: application/json" -d '{
"name": "inventory-connector",
"config": {
"connector.class" : "io.debezium.connector.oracle.OracleConnector",
"tasks.max" : "1",
"database.server.name" : "ORCL",
"database.hostname" : "119.23.67.177",
"database.port" : "1521",
"table.include.list" : "hz_oc.*",
"database.user" : "kafka02",
"database.password" : "kafkapass",
"database.dbname" : "ORCL",
"database.history.store.only.captured.tables.ddl" : "true",
"database.history.kafka.bootstrap.servers" : "172.19.1.116:9092,172.19.1.117:9092,172.19.1.118:9092",
"database.history.kafka.topic": "schema-changes.inventory"
}
}'
```



##### 2.2.7.查看连接器状态

```
curl -s localhost:8083/connectors/inventory-connector/status

#以json方式展示
curl -s localhost:8083/connectors/inventory-connector/status|jq


安装jq命令
yum install jq -y
```

```
[root@wp-prod-ns-kafka-172-19-1-116 data]# curl -s localhost:8083/connectors/inventory-connector/status|jq
{
  "name": "inventory-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "172.19.1.117:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "172.19.1.116:8083"
    }
  ],
  "type": "source"
}
```

##### 2.2.8.查看topic是否创建

```
[root@wp-prod-ns-kafka-172-19-1-116 bin]# cd /data/kafka/bin
[root@wp-prod-ns-kafka-172-19-1-116 bin]# ./kafka-topics.sh --list --zookeeper 172.19.1.116:2181
ORCL
ORCL.HZ_OC.CQJ_TEST10
__consumer_offsets
connect-configs
connect-offsets
connect-status
schema-changes.inventory
```

##### 2.2.9.topic创建顺序

1. 确定要捕获的表
2. 获取`ROW SHARE MODE`对每个受监控表的锁定，以防止在创建快照期间发生结构更改。Debezium 持有锁的时间很短。
3. 从服务器的重做日志中读取当前系统更改号 (SCN) 位置。
4. 捕获所有相关表的结构。
5. 释放在步骤 2 中获得的锁。
6. 在步骤 3 ( `SELECT * FROM … AS OF SCN 123`) 中读取的 SCN 位置扫描所有相关的数据库表和模式为有效，`READ`为每一行生成一个事件，然后将事件记录写入特定于表的 Kafka 主题。
7. 在连接器偏移中记录快照的成功完成。

重要：捕获表后会按照顺序获取完成第一个表的记录后，才会创建下一个表的对应topic
