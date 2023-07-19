# 一. ogg源端操作

### 1.1  oracle配置

###### 1.1.1  开启oracle归档模式

登录到oracle用户

```sql
sqlplus / as sysdba
```

查看归档日志是否打开，如果是Disable，则需要打开

```shell
SQL> archive log list 
Database log mode	       No Archive Mode
Automatic archival	       Disabled
Archive destination	       USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     22
Current log sequence	       24
```

打开归档模式步骤

```shell
#立即关闭数据库
shutdown immediate
#启动实例并加载数据库
startup mount
#更改数据库为归档模式
alter database archivelog
#打开数据库
alter database open
#启用自动归档
alter system archive log start
```

再次确认归档日志是否开启

```sql
SQL> archive log list 
Database log mode	       Archive Mode
Automatic archival	       Enabled
Archive destination	       USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     22
Next log sequence to archive   24
Current log sequence	       24
```

Automatic archival	       Enabled   说明已经打开归档日志

###### 1.1.2 开启oracle强制日志以及附加日志

查看是否开启

```sql
select force_logging, supplemental_log_data_min from v$database;
```

如果返回为NO，则需要进行更改

```shell
FORCE_ SUPPLEMENTAL_LOG
------ ----------------
NO     NO
```

更改开启

```shell
#强制日志
alter database force logging;
#附加日志
alter database add supplemental log data;
```

再次确认是否开启

```sql
SQL> select force_logging, supplemental_log_data_min from v$database;

FOR SUPPLEME
--- --------
YES YES
```

###### 1.1.3 创建ogg用户并授权

```shell
#创建ogg用户，并设置默认名称空间
create user ogg identified by ogg default tablespace ogg_data;

#赋予ogg用户权限
grant connect,resource,dba,create table,create sequence to ogg;
```

### 1.2 源端ogg安装

本次采集的是oracle版本为11G

下载源端安装包

```
wget https://hz-package.hzins.com/ogg/11G/V861007-01.zip
```

解压

```shell
mkdir /app/ogg
cd /app/ogg
unzip V861007-01.zip
chown -R oracle:oinstall /app/ogg
```

切换用户修改配置

```shell
sudo su oracle
cd /app/ogg/fbo_ggs_Linux_x64_shiphome/Disk1/response
```

```shell
[oracle@master response]$ grep -Ev "^$|[#;]" oggcore.rsp

oracle.install.responseFileVersion=/oracle/install/rspfmt_ogginstall_response_schema_v19_1_0
INSTALL_OPTION=ORA11g
SOFTWARE_LOCATION=/app/ogg
START_MANAGER=
MANAGER_PORT=
DATABASE_LOCATION=/app/ora/oracle/product/11.2.0
INVENTORY_LOCATION=/app/ora/oraInventory
UNIX_GROUP_NAME=oinstall
```

修改完成后执行安装

注：不能使用root用户安装，配置文件路径必须是绝对路径

```
cd /app/ogg/fbo_ggs_Linux_x64_shiphome/Disk1
./runInstaller -silent -nowait -responseFile /app/ogg/fbo_ggs_Linux_x64_shiphome/Disk1/response/oggcore.rsp
```

```shell
#出现一下语句，说明安装完成
cessfully Setup Software.
```

### 1.3 添加环境变量

```shell
vim /home/oracle/.bash_profile

增加：
export OGG_HOME=/app/ogg
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export PATH=.:$ORACLE_HOME/bin:$ORACLE_HOME/OPatch:$ORACLE_HOME/jdk/bin:$OGG_HOME:$PATH
```

使变量生效

```shell
source /home/oracle/.bash_profile
```

### 1.4 初始化ogg

```
cd /app/ogg

./ggsci


Oracle GoldenGate Command Interpreter for Oracle
Version 19.1.0.0.4 OGGCORE_19.1.0.0.0_PLATFORMS_191017.1054_FBO
Linux, x64, 64bit (optimized), Oracle 11g on Oct 17 2019 23:13:12
Operating system character set identified as US-ASCII.

Copyright (C) 1995, 2019, Oracle and/or its affiliates. All rights reserved.
GGSCI (mirror-node1) 1>
```

初始化目录

```
GGSCI (master) 1> create subdirs

Creating subdirectories under current directory /app/ogg

Parameter file                 /app/ogg/dirprm: created.
Report file                    /app/ogg/dirrpt: created.
Checkpoint file                /app/ogg/dirchk: created.
Process status files           /app/ogg/dirpcs: created.
SQL script files               /app/ogg/dirsql: created.
Database definitions files     /app/ogg/dirdef: created.
Extract data files             /app/ogg/dirdat: created.
Temporary files                /app/ogg/dirtmp: created.
Credential store files         /app/ogg/dircrd: created.
Masterkey wallet files         /app/ogg/dirwlt: created.
Dump files                     /app/ogg/dirdmp: created.


GGSCI (master) 2>
```

配置源端mgr

```
GGSCI (master) 1> edit params mgr
```

添加如下内容

```
PORT 7809
DYNAMICPORTLIST 7810-7909
AUTORESTART EXTRACT *,RETRIES 5,WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 3
```

PORT即mgr的默认监听端口；

DYNAMICPORTLIST动态端口列表，当指定的mgr端口不可用时，会在这个端口列表中选择一个，最大指定范围为256个；

AUTORESTART重启参数设置表示重启所有EXTRACT进程，最多5次，每次间隔3分钟；

PURGEOLDEXTRACTS即TRAIL文件的定期清理

查看状态

```
GGSCI (master) 3> start mgr
Manager started.


GGSCI (master) 4> info all

Program     Status      Group       Lag at Chkpt  Time Since Chkpt
MANAGER     RUNNING                                           


GGSCI (master) 5> sh netstat -ntpl | grep 7809
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp6       0      0 :::7809                 :::*                    LISTEN      13666/./mgr
```

如果出现：

MANAGER STOPPED

使用：view report mgr

查看mgr报错

### 1.5  添加复制表

```shell
GGSCI (master) 6> dblogin userid ogg,password ogg
Successfully logged into database.

GGSCI (master as ogg@ORCL) 7> add trandata test_ogg.test_ogg

#如果需要添加整个库，可以使用 add trandata test_ogg.*
```

查看是否成功

```
GGSCI (master as ogg@ORCL) 8> info trandata test_ogg.test_ogg

Logging of supplemental redo log data is enabled for table TEST_OGG.TEST_OGG.
Columns supplementally logged for table TEST_OGG.TEST_OGG: ID.
Prepared CSN for table TEST_OGG.TEST_OGG: 2590115
```

### 1.6  配置Extract进程

名称可以按自己想法配置

```
GGSCI (master as ogg@ORCL) 9> edit params extkaf01
```

添加如下内容

```
EXTRACT extkaf01
USERID ogg_test, PASSWORD ogg_test
EXTTRAIL /app/ogg/dirdat/to
TABLE test_ogg.test_ogg;
```

说明

```
第一行指定extract进程名称，注意不能超过8个字符
userid ggs,password ggs即OGG连接Oracle数据库的帐号密码；
exttrail定义trail文件的保存位置以及文件名，注意这里文件名只能是2个字母，其余部分OGG会补齐；
table即复制表的表名，支持*通配，必须以 ; 结尾；
```

添加进程

```
GGSCI (master  as ogg@ORCL) 10> add extract extkaf01,tranlog,begin now

EXTRACT added.
```

添加trail文件的定义与extract进程绑定

```
GGSCI (master as ogg@ORCL) 11> add exttrail /app/ogg/dirdat/to,extract extkaf01

EXTTRAIL added.
```

1.7 配置Pump进程

作用仅仅是把trail文件传递到目标端，配置过程和extract进程类似

```
GGSCI (master  as ogg@ORCL) 12> edit param pukaf01
```

```
EXTRACT pukaf01
PASSTHRU
userid ogg_test,password ogg_test
RMTHOST 192.168.10.180 MGRPORT 7809
RMTTRAIL ./dirdat/to
TABLE test_ogg.test_ogg;
```

第一行指定extract进程名称；

passthru即禁止OGG与Oracle交互，我们这里使用pump逻辑传输，禁止即可；

userid ogg,password ogg即OGG连接Oracle数据库的帐号密码

rmthost和mgrhost即目标端(kafka)OGG的mgr服务的地址以及监听端口；

rmttrail即目标端trail文件存储位置以及名称

分别将本地trail文件和目标端的trail文件绑定到extract进程

```
GGSCI (master as ogg@ORCL) 13> add extract pukaf01,exttrailsource /app/ogg/dirdat/to
EXTRACT added.

GGSCI (master as ogg@ORCL) 14> add rmttrail /app/ogg/dirdat/to,extract pukaf01
RMTTRAIL added.
```

### 1.7  配置define文件

Oracle与MySQL，Hadoop集群（HDFS，Hive，kafka等）等之间数据传输可以定义为异构数据类型的传输，需要定义表之间的关系映射

```
GGSCI (master as ogg@ORCL) 15> edit param ogg_test
```

```
defsfile /app/ogg/dirdef/test_ogg.test_ogg
userid ogg,password ogg
table test_ogg.test_ogg;
```

在OGG_HOME目录下执行，注意这里是使用oracle用户

```
[oracle@master ogg]$ ./defgen paramfile dirprm/ogg_test.prm

***********************************************************************
        Oracle GoldenGate Table Definition Generator for Oracle
      Version 19.1.0.0.4 OGGCORE_19.1.0.0.0_PLATFORMS_191017.1054
   Linux, x64, 64bit (optimized), Oracle 11g on Oct 17 2019 14:27:47
 
Copyright (C) 1995, 2019, Oracle and/or its affiliates. All rights reserved.

                    Starting at 2021-04-21 10:28:43
***********************************************************************

Operating System Version:
Linux
Version #1 SMP Fri Dec 18 16:34:56 UTC 2020, Release 3.10.0-1160.11.1.el7.x86_64
Node: mirror-node1
Machine: x86_64
                         soft limit   hard limit
Address Space Size   :    unlimited    unlimited
Heap Size            :    unlimited    unlimited
File Size            :    unlimited    unlimited
CPU Time             :    unlimited    unlimited

Process id: 26393

***********************************************************************
**            Running with the following parameters                  **
***********************************************************************
defsfile /app/ogg/dirdef/test_ogg.test_ogg
userid ogg,password ***
table test_ogg.test_ogg;
Retrieving definition for TEST_OGG.TEST_OGG.


Definitions generated for 1 table in /app/ogg/dirdef/test_ogg.test_ogg.
```

将生成的 /app/ogg/dirdef/test_ogg.test_ogg 发送的目标端kafka机器ogg目录下的dirdef里

```
scp -r /app/ogg/dirdef/test_ogg.test_ogg root@192.168.10.180:/app/ogg/dirdef/
```

# 二.  目标端配置kafka机器

下载包

```
wget https://hz-package.hzins.com/ogg/11G/OGG_BigData_Linux_x64_19.1.0.0.5.zip
```

解压

```
cd /app
mkdir ogg
unzip OGG_BigData_Linux_x64_19.1.0.0.5.zip -d ./ogg
tar xf OGG_BigData_Linux_x64_19.1.0.0.5.tar
```

### 2.1  修改环境变量

```
/vim /etc/profile
```

```
export OGG_HOME=/app/ogg
export LD_LIBRARY_PATH=$JAVA_HOME/jre/lib/amd64:$JAVA_HOME/jre/lib/amd64/server:$JAVA_HOME/jre/lib/amd64/libjsig.so:$JAVA_HOME/jre/lib/amd64/server/libjvm.so:$OGG_HOME/lib
export PATH=$OGG_HOME:$PATH
```

```
source /etc/profile
```

### 2.2 创建目录

```
cd /app/ogg
./ggsci

GGSCI (kafka) 1> create subdirs
```

### 2.3 配置mgr

```
GGSCI (kafka) 1> edit param mgr
```

```
PORT 7809
DYNAMICPORTLIST 7810-7909
AUTORESTART EXTRACT *,RETRIES 5,WAITMINUTES 3
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 3
```

### 2.4 配置replicate

```
GGSCI (kafka) 2> edit param rekaf01
```

```
REPLICAT rekaf01
sourcedefs /app/ogg/dirdef/test_ogg.test_ogg
TARGETDB LIBFILE libggjava.so SET property=dirprm/kafka01.props
REPORTCOUNT EVERY 1 MINUTES, RATE 
GROUPTRANSOPS 10000
MAP test_ogg.test_ogg, TARGET test_ogg.test_ogg;
```

REPLICATE 定义rep进程名称；

sourcedefs 是在源端服务器上做的表映射文件；

TARGETDB LIBFILE即定义kafka一些适配性的库文件以及配置文件，配置文件位于OGG主目录下的dirprm/kafka01.props （需要自己编辑生成）；

REPORTCOUNT即复制任务的报告生成频率；

GROUPTRANSOPS为以事务传输时，事务合并的单位，减少IO操作；

MAP即源端与目标端的映射关系；

### 2.5 配置kafka

```
cd /app/ogg/dirprm
vim kafka01.props
```

```
#handler类型
gg.handlerlist=kafkahandler
gg.handler.kafkahandler.type=kafka
#kafka相关配置
gg.handler.kafkahandler.KafkaProducerConfigFile=custom_kafka_producer_01.properties
#kafka的topic
gg.handler.kafkahandler.topicName=test_oracle_ogg
#传输文件的格式，支持json，xml等
gg.handler.kafkahandler.format=json
#OGG for Big Data中传输模式，即op为一次SQL传输一次，tx为一次事务传输一次
gg.handler.kafkahandler.mode=op
gg.classpath=dirprm/:/app/soft/kafka_2.12-2.0.0/libs/*:/app/ogg/:/app/ogg/lib/*
javawriter.bootoptions=-Xmx2048m -Xms1024m -Djava.class.path=./ggjava/ggjava.jar
```

```
vim custom_kafka_producer_01.properties
```

```
#kafka 地址
bootstrap.servers=172.19.2.139:9092,172.19.2.138:9092,172.19.2.140:9092
acks=1
#压缩类型
compression.type=gzip
#重连延迟
reconnect.backoff.ms=1000
value.serializer=org.apache.kafka.common.serialization.ByteArraySerializer
key.serializer=org.apache.kafka.common.serialization.ByteArraySerializer
batch.size=102400
linger.ms=10000
```

添加trail文件到replicate进程

```
GGSCI (kafka) 2> add replicat rekaf01 exttrail /app/ogg/dirdat/to,checkpointtable OGG.checkpoint

REPLICAT added.
```

# 三  启动进程

源端

```
cd /app/ogg
./ggsci

start mgr
start extkaf01
start pukaf01
```

目标端

```
cd /app/ogg
./ggsci

start mgr
start rekaf01
```

完成后到kafka创建topic，然后查看是否有数据过来

topic 名称在目标端kafka01.props

```
test_oracle_ogg
```
