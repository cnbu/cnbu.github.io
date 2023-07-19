# 一：常用

##### 相关的配置文件目录：

```
• /etc/cloudera-scm-agent
• /etc/cloudera-scm-server
```

##### 相关的工作目录：

```
• /var/lib/cloudera-scm-agent/
• /var/lib/cloudera-scm-server/
• /opt/cloudera/cm
• /opt/cloudera/cm-agent
• /opt/cloudera/csd
• /opt/cloudera/parcels
• /opt/cloudera/parcel-cache
• /opt/cloudera/parcel-repo
```

##### 相关的日志目录：

```
• /var/log/cloudera-scm-server
• /var/log/cloudera-scm-agent
• /var/log/cloudera-scm-firehose
• /var/log/cloudera-scm-alertpublisher
• /var/log/cloudera-scm-eventserver
```

##### 大数据组件作用：

```
1、 数据存储：HDFS，一个分布式文件系统
2.  MapReduce（分布式计算模型）离线计算
3.  Yarn（分布式资源管理器）
4.  Spark（内存计算）
5.  HBase（分布式列存储数据库）
6.  Hive（数据仓库）
7.  Oozie（工作流调度器）
8.  Sqoop 与 Pig   Sqoop（SQL-to-Hadoop）
9.  Flume（日志收集工具）
10. Kafka（分布式消息队列）
11. ZooKeeper（分布式协作服务）
12. Ambari（大数据运维工具）
13. Cloudera Manager
	cloudera manager有四大功能：
	（1）管理：对集群进行管理，如添加、删除节点等操作。
	（2）监控：监控集群的健康情况，对设置的各种指标和系统运行情况进行全面监控。
	（3）诊断：对集群出现的问题进行诊断，对出现的问题给出建议解决方案。
	（4）集成：对hadoop的多组件进行整合
```

# 二：部署

### 2.1: 所有节点安装依赖

```shell
yum -y install chkconfig python bind-utils psmisc libxslt zlib sqlite cyrus-sasl-plain cyrus-sasl-gssapi fuse fuse-libs redhat-lsb mod_ssl httpd
```

### 2.2: 安装java

```shell
rmp -ivh jdk-8u201-linux-x64.rpm

vim /etc/profile

##JAVA_HOME
export JAVA_HOME=/usr/java/jdk1.8.0_201-amd64
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin


. /etc/profile
```

### 2.3：所有节点创建cloudear-scm用户

```
useradd --system --home=/opt/cm-5.14.1/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm

参数说明
--system 创建一个系统账户
--home 指定用户登入时的主目录，替换系统默认值/home/<用户名>
--no-create-home 不要创建用户的主目录
--shell 用户的登录 shell 名
--comment 用户的描述信息

注意
Cloudera Manager默认用户为cloudera-scm，创建具有此名称的用户是最简单的方法。
安装完成后，将自动使用此用户。
```

### 2.4：修改机器hosts文件

```
vim /etc/hosts

172.19.0.224  ops-prod-ns-cdh-172-19-0-224
172.19.0.225  ops-prod-ns-cdh-172-19-0-225
172.19.0.226  ops-prod-ns-cdh-172-19-0-226
```

### 2.4:：服务器优化

###### 2.4.1: 关闭透明大页

```shell
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled

#添加至开机启动生效
echo 'echo never > /sys/kernel/mm/transparent_hugepage/defrag' >> /etc/rc.d/rc.local 
echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' >> /etc/rc.d/rc.local
```

###### 2.4.2: 调整内核参数

```shell
echo -e 'vm.swappiness = 1\nnet.core.somaxconn = 32768' >> /etc/sysctl.conf
```

###### 2.4.3：修改最大进程数和句柄数

```shell
echo -e '* soft nofile 131070\n* hard nofile 131070' >> /etc/security/limits.conf;
echo -e '* soft nproc unlimited\n* hard nproc unlimited' >> /etc/security/limits.d/20-nproc.conf

/sbin/sysctl -p
```

### 2.5：添加时间同步

```
sed -ri 's/^server/#server/g' /etc/ntp.conf 

#所有机器执行
ntpdate pool.ntp.org

#第一台机器修改 /etc/ntp.conf
#授权192.168.77.0网段上的机器能和本机做时间同步
restrict 192.168.77.0 mask 255.255.255.0 nomodify notrap
#server 0.rhel.pool.ntp.org iburst
#server 1.rhel.pool.ntp.org iburst
#server 2.rhel.pool.ntp.org iburst
#server 3.rhel.pool.ntp.org iburst
#让本机的ntpd和本地硬件时间同步
server 127.127.1.0
fudge  127.127.1.0 stratum 10 

#启动ntp服务
systemctl enable ntpd
systemctl start ntpd

#其他机器执行
vim /etc/ntp.conf
#server 0.rhel.pool.ntp.org iburst
#server 1.rhel.pool.ntp.org iburst
#server 2.rhel.pool.ntp.org iburst
#server 3.rhel.pool.ntp.org iburst
server hadoop001

#其他机器设置
systemctl enable ntpd
systemctl start ntpd
```

### 2.6 ：启动并配置MariaDB

```
yum -y install mariadb
yum -y install mariadb-server

systemctl start mariadb
systemctl enable mariadb

/usr/bin/mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] Y
New password: 
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] n
 ... skipping.

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

### 2.7：建立CM，Hive等需要的表

```shell
mysql -u root -p 123456


CREATE DATABASE IF NOT EXISTS `hive` default character set utf8;  CREATE USER 'hive'@'%' IDENTIFIED BY 'password';   GRANT ALL PRIVILEGES ON hive. * TO 'hive'@'%';   FLUSH PRIVILEGES;  
create database cm default character set utf8;  CREATE USER 'cm'@'%' IDENTIFIED BY 'password';   GRANT ALL PRIVILEGES ON cm. * TO 'cm'@'%';   FLUSH PRIVILEGES;  
create database am default character set utf8;   CREATE USER 'am'@'%' IDENTIFIED BY 'password';    GRANT ALL PRIVILEGES ON am. * TO 'am'@'%';    FLUSH PRIVILEGES;    
create database rm default character set utf8;   CREATE USER 'rm'@'%' IDENTIFIED BY 'password';    GRANT ALL PRIVILEGES ON rm. * TO 'rm'@'%';    FLUSH PRIVILEGES;
create database hue default character set utf8;   CREATE USER 'hue'@'%' IDENTIFIED BY 'password';    GRANT ALL PRIVILEGES ON hue. * TO 'hue'@'%';    FLUSH PRIVILEGES;
create database oozie default character set utf8;   CREATE USER 'oozie'@'%' IDENTIFIED BY 'password';    GRANT ALL PRIVILEGES ON oozie. * TO 'oozie'@'%';    FLUSH PRIVILEGES;
create database sentry default character set utf8;   CREATE USER 'sentry'@'%' IDENTIFIED BY 'password';    GRANT ALL PRIVILEGES ON sentry. * TO 'sentry'@'%';    FLUSH PRIVILEGES;
create database nav_ms default character set utf8;   CREATE USER 'nav_ms'@'%' IDENTIFIED BY 'password';    GRANT ALL PRIVILEGES ON nav_ms. * TO 'nav_ms'@'%';    FLUSH PRIVILEGES;
create database nav_as default character set utf8;  CREATE USER 'nav_as'@'%' IDENTIFIED BY 'password';   GRANT ALL PRIVILEGES ON nav_as. * TO 'nav_as'@'%';   FLUSH PRIVILEGES;




#修改hive元数据字符集
#HIVE 安装完成后需要修改字符集编码，否则表注释信息会显示乱码

#-- 修改字段注释字符集
alter table COLUMNS_V2 modify column COMMENT varchar(256) character set utf8;
#-- 修改表注释字符集
alter table TABLE_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;
#-- 修改分区参数，支持分区建用中文表示
alter table PARTITION_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;
alter table PARTITION_KEYS modify column PKEY_COMMENT varchar(4000) character set utf8;
#-- 修改表名注释，支持中文表示
alter table INDEX_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;
#-- 修改视图，支持视图中文
ALTER TABLE TBLS modify COLUMN VIEW_EXPANDED_TEXT mediumtext CHARACTER SET utf8;
ALTER TABLE TBLS modify COLUMN VIEW_ORIGINAL_TEXT mediumtext CHARACTER SET utf8;
```

### 2.8： 安装jdbc驱动(所有机器)

```
#安装java

mkdir -p /usr/share/java/
mv mysql-connector-java-8.0.22.jar /usr/share/java/
cd /usr/share/java/
ln -s mysql-connector-java-8.0.20.jar mysql-connector-java.jar
```

```
http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/cloudera-manager-agent-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm
http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/cloudera-manager-daemons-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm
http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/cloudera-manager-server-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm
http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/cloudera-manager-server-db-2-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm
http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/enterprise-debuginfo-5.16.1-1.cm5161.p0.1.el7.x86_64.rpm
http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/jdk-6u31-linux-amd64.rpm
http://archive.cloudera.com/cm5/redhat/7/x86_64/cm/5.16.1/RPMS/x86_64/oracle-j2sdk1.7-1.7.0+update67-1.x86_64.rpm
```

### 2.9： 配置免密登陆

```shell
#所有机器执行，直接按回车
ssh-keygen -t rsa

#所有机器，安装
yum install -y sshpass expect

#编写脚本
cd /usr/local/bin


vim aoto-ssh.sh
#!/bin/bash
IP="
192.168.77.51
192.168.77.52
"
for node in ${IP};do
   sshpass -p 123456 ssh-copy-id ${node} -o stricthostkeychecking=no                                                                                         
   if [ $? -eq 0 ];then
     echo "${node} 密钥copy完成"
   else
     echo "%{node} 密钥copy失败"
   fi
done
```

### 2.10： 配置本地repo源（仅在hadoop001配置即可）

```
scp root@10.7.16.91:/data/deploy/CDH5/5.16.2-centos7.tar.gz ./

tar -zxvf 5.16.2-centos7.tar.gz -C /var/www/html

启动httpdsystemctl start httpd,浏览器访问查看是否生效http://hadoop001/cm/

配置repo文件，并放置到所有服务器的/etc/yum.repos.d/目录下

#cm.repo
#内容如下：
[cloudera-cm]
name=cm5.16.2
baseurl=http://hadoop001/cm/5
gpgkey =http://hadoop001/cm/RPM-GPG-KEY-cloudera
gpgcheck = 1
enabled=1


yum repolist查看yum源是否生效
```

### 2.11：安装server

```
yum -y install cloudera-manager-server
```

### 2.12：修改server配置文件

```
vim/etc/cloudera-scm-server/db.properties

com.cloudera.cmf.db.type=mysql
com.cloudera.cmf.db.host=[mysqlhosts]:[port]
com.cloudera.cmf.db.name=cm
com.cloudera.cmf.db.user=cm
com.cloudera.cmf.db.setupType=EXTERNAL
com.cloudera.cmf.db.password=password
```

### 2.13：所有服务器安装Cloudera-manager-agent

```
yum install -y cloudera-manager-agent cloudera-manager-daemons
```

### 2.14：**修改cloudera-manager-agent的配置 server_host**

```
 [General]
# Hostname of the CM server.
server_host=hadoop001-77-21

# Port that the CM server is listening on.
server_port=7182
```

### 2.15：启动服务

```
#所有集群服务器执行命令启动cloudera-scm-agent
systemctl start cloudera-scm-agent
#所有集群服务器cloudera-scm-agent设置开机自启（默认）
systemctl enable cloudera-scm-agent
#server节点启动cloudera-scm-server
systemctl start cloudera-scm-server
#cloudera-scm-server开机自启
systemctl enable cloudera-scm-server
```

### 2.16：日志目录

```
#cloudera-scm-server
/var/log/cloudera-scm-server/cloudera-scm-server.log
#cloudera-scm-agent
/var/log/cloudera-scm-agent/cloudera-scm-agent.log
```
### 2.17：将parcels包放到指定目录

```
# 下载至hadoop001
# 解压
tar -zxvf cdh5.tar.gz 
cd cdh5/
# 拷贝至/opt/cloudera/parcel-repo目录
cp CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel /opt/cloudera/parcel-repo
cp CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel.sha /opt/cloudera/parcel-repo
chown -R cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo
```
# 三：问题处理

```
Canary 测试无法为 /tmp/.cloudera_health_monitoring_canary_files 创建父目录

sudo -uhdfs hdfs dfs -mkdir /tmp

sudo -u hdfs hadoop fs -mkdir /user/root
```

# 四：账号

```
hue  hzins618
```

### 分发文件脚本

```shell
#!/bin/bash
#1. 判断参数个数 
if [ $# -lt 1 ]
then
   echo Not Enough Arguement!
   exit;
fi
#2. 遍历集群所有机器
for host in hadoop100 hadoop101 hadoop102                                                                                                             
do
   echo ==================== $host ==================== #3. 遍历所有目录，挨个发送
   for file in $@
   do
#4. 判断文件是否存在 
      if [ -e $file ]
         then
           #5. 获取父目录
           pdir=$(cd -P $(dirname $file); pwd)
           #6. 获取当前文件的名称 
           fname=$(basename $file)
           ssh $host "mkdir -p $pdir"
           rsync -av $pdir/$fname $host:$pdir
         else
           echo $file does not exists!
      fi
   done
done
```
