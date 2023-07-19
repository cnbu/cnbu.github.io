# 一、日常问题处理
###### kafka命令

```shell
##、启动kafka服务：
./kafka-server-start.sh -daemon config/server.properties

##、创建topic
./kafka-topics.sh --create --topic test-topic --partitions 2 --replication-factor 3 --zookeeper 192.168.103.46:2181

##、 删除Topic
./kafka-topics.sh --delete --topic test-topic  --zookeeper 192.168.103.46:2181
./kafka-topics.sh --delete --topic pingtai 30.4.98.53:2181,30.4.98.54:2181,30.4.98.55:2181 

##、查看topic列表
./kafka-topics.sh --list --zookeeper 127.0.0.1:2181

##、查看consumer组内消费的offset  
./kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper localhost:2181 --group test --topic testKJ1

##、kafka生产者客户端命令 （控制台发送消息）
./kafka-console-producer.sh --broker-list localhost:9092 --topic bi-nginx-log 

##、kafka消费者客户端命令 
./kafka-console-consumer.sh -zookeeper localhost:2181 --from-beginning --topic testKJ1 
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test1 --from-beginning

##、从头开始消费test的消息
./kafka-console-consumer.sh --bootstrap-server 30.4.66.23:9092,30.4.66.24:9092 --topic test --from-beginning

##、kafka从多少时间从新消费
./bin/kafka-consumer-groups.sh --bootstrap-server broker1:9092,broker2:9092 --group logstash_nginx --reset-offsets -topic nginxlog --to-datetime 2020-10-27T16:00:00.000 --execute

##、查看当前kafka的正在消费的组信息
./kafka-consumer-groups.sh --bootstrap-server 30.4.66.23:9092,30.4.66.24:9092 --list

##、查看当前kafka的消费组详细信息
./kafka-consumer-groups.sh --bootstrap-server 30.4.66.23:9092,30.4.66.24:9092,30.4.89.62:9092 --describe --group logstash2

./kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper localhost:2181 --group logstash --topic elis-smp-ubas-new

##、查看kafka的topic当前的offset值
./kafka-run-class.sh kafka.tools.GetOffsetShell --topic elis-smp-ubas-new  --time -1 --broker-list 30.4.66.23:9092,30.4.66.24:9092,30.4.89.62:9092 --partitions 0

注： time为-1时表示最大值，time为-2时表示最小值

##显示topic详细信息
./kafka-topics.sh --describe --topic jumi_partner.partner --zookeeper 10.5.3.7:12181

#重置topic消费者的offsets
./bin/kafka-consumer-groups.sh --bootstrap-server broker1:9092,broker2:9092 --group logstash_nginx --reset-offsets -topic nginxlog --to-datetime 2020-10-27T16:00:00.000 --execute

##查看当前kafka的正在消费的组信息
./kafka-consumer-groups.sh --bootstrap-server 172.22.8.132:9092 --list

##查看当前kafka的消费组详细信息
./kafka-consumer-groups.sh --bootstrap-server 30.4.98.52:9092,30.4.98.51:9092,30.4.98.50:9092,30.4.98.49:9092,30.4.98.44:9092,30.4.98.43:9092,30.4.98.42:9092,30.4.98.41:9092 --describe --group lops-las-wangxiao-new

./kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper 30.4.98.53:2181,30.4.98.54:2181,30.4.98.55:2181 --group lops-las-wangxiao-new --topic wangxiao

#查看kafka的topic当前的offset值：
./kafka-run-class.sh kafka.tools.GetOffsetShell --topic anquanbu --time -1 --broker-list 30.4.98.52:9092,30.4.98.51:9092,30.4.98.50:9092,30.4.98.49:9092,30.4.98.44:9092,30.4.98.43:9092,30.4.98.42:9092,30.4.98.41:9092 --partitions 0

--partitions 0表示topic分区为0

##控制台接收消息
bin/kafka-console-consumer.sh --zookeeper  192.168.197.170:2181,192.168.197.171:2181  --from-beginning --topic test

##控制台发送消息
bin/kafka-console-producer.sh --broker-list  192.168.197.170:9092,192.168.197.171: 9092    --topic test 

#查看log
./kafka-run-class.sh kafka.tools.DumpLogSegments --files /wls/apache/servers/kfk_lbdp-aip-prd-ins6796/data/kafkaUbasTopicProd-11/00000000010600377216.log --print-data-log >> /tmp/kafkaUbasTopicProd1.txt

##查看消费组列表
bin/kafka-consumer-groups.sh --bootstrap-server 10.4.1.16:9092,10.4.1.42:9092,10.4.1.36:9092 --list

#删除消费组
./kafka-consumer-groups.sh --bootstrap-server 172.22.8.132:9092 --delete --group event-risk-collect

#查看单个topic配置
./kafka-topics.sh --describe --topic travel_cpp.c_trip_partner --zookeeper 172.22.8.174:2181

#查看
./kafka-topics --zookeeper 172.22.8.132:2181 --describe --topic __consumer_offsets
./kafka-topics.sh --bootstrap-server 172.20.0.187:9092 --topic travel_cpp.c_trip_industry --describe

#消费topic信息到文件
./kafka-run-class.sh kafka.tools.DumpLogSegments --files /data/kafka/data/alertnginx-1/00000000000000000000.log --print-data-log >> /tmp/ceshi.txt


#修改单个topic日志保留时长毫秒
#1天86400000毫秒   2天172800000  3天259200000
./kafka-topics.sh --topic ogg_data --zookeeper 10.75.5.191:2181 --alter --config retention.ms=86400000
./kafka-topics.sh --zookeeper 10.4.1.79:2181 -topic JMETER_METRICS --alter --config retention.ms=259200000


#修改单个topic消息体大小3M
#动态修改
./kafka-topics.sh --zookeeper 10.5.3.7:12181 --alter --topic jumi_partner --config max.message.bytes=10485760     


#配置文件修改
server.properties中添加配置项

#broker能接收消息的最大字节数
message.max.bytes=20000000
#broker可复制的消息的最大字节数
fetch.message.max.bytes=20485760


message.max.bytes=104857600  100M
```

# 二、kafka简介
###### kafka特点

```
offset.reset=1
0从头消费 1从上次提交记录消费 2从末尾消费

可靠性：具有副本及容错机制。

可扩展性：kafka无需停机即可扩展节点及节点上线。

持久性：数据存储到磁盘上，持久性保存。

性能：kafka具有高吞吐量。达到TB级的数据，也有非常稳定的性能。

速度快：顺序写入和零拷贝技术使得kafka延迟控制在毫秒级。
```
