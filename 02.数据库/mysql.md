# 一：常用

```shell
#授权root给任何IP，对任何数据库.
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
#刷新授权(立刻生效)
flush privileges;
#初始化mysql密码
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';
#更改root密码
update user set Password=password('123456') where User='root'; 
#设置root密码为hzins618
./bin/mysqladmin -u root -p password hzins618
#查看慢查询
select * from information_schema.processlist where command<>'sleep';
#查看编码
show variables like 'character_%';
show variables like 'collation_%';
#登录
语法：mysql -u用户名 -p用户密码
举例：mysql -uroot -p123456
语法：mysql -u用户名 -p+回车，然后输入密码
举例：mysql -uroot -p
#查看所有链接
show full processlist;
#刷新
flush privileges;
#显示用户的授权
SHOW GRANTS FOR user_name;
#从库用户创建，授权
grant REPLICATION SLAVE,REPLICATION CLIENT on *.* to 'repl'@'172.30.211.15' identified by '123456' with grant option;
#建库
CREATE DATABASE `huize_bi` CHARACTER SET 'utf8mb4';
#查看程序运行所需的共享库
ldd `which sshd`
#mysql检查
数据一致性
checksum            mysql中一致性检查命令
mysqldiff           mysql-utilities中对比表结构
pt-table-checksum   主从数据一致性检查
# 查看执行sql状态
show full processlist;    
show processlist;         
# 查看主或者从节点状态
show slave status\G    
show master status\G   
# 查看系统和状态变量
show variables\G      
show status\G
# mysql授权prometheus
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO prometheus@'%' identified by 'sJdf9123i1aZl' ;
# 查看支持的存储引擎
SHOW ENGINES
# 查看默认存储引擎
SHOW VARIABLES LIKE 'storage_engine'
# 查看具体某一个表所使用的存储引擎，这个默认存储引擎被修改了！
show create table tablename
# 准确查看某个数据库中的某一表所使用的存储引擎
show table status like 'tablename'
show table status from database where name="tablename"
# 建表时指定存储引擎。默认的就是INNODB，不需要设置
CREATE TABLE t1 (i INT) ENGINE = INNODB;
CREATE TABLE t2 (i INT) ENGINE = CSV;
CREATE TABLE t3 (i INT) ENGINE = MEMORY;
#修改存储引擎
ALTER TABLE t ENGINE = InnoDB;
#修改默认存储引擎，也可以在配置文件my.cnf中修改默认引擎
SET default_storage_engine=NDBCLUSTER;
#修改表结构(添加索引)：
ALTER table tableName ADD [UNIQUE] INDEX indexName(columnName)
#查看索引
SHOW INDEX FROM table_name\G
#确定版本和是否开启了 binlog 日志，以及日志格式
show variables like 'binlog_format';
show variables like 'log_bin';
select version();
#按照登录用户+数据库+登录服务器查看登录信息
SELECT 
DB as database_name,
USER as login_user,
LEFT(HOST,POSITION(':' IN HOST)-1) AS login_ip,
count(1) as login_count
FROM `information_schema`.`PROCESSLIST` P 
WHERE P.USER NOT IN('root','repl','system user') 
GROUP BY DB,USER,LEFT(HOST,POSITION(':' IN HOST)-1)
ORDER BY COUNT(1) DESC;
```

## 查看mysql连接数
```yaml
1、MySQL> show status like '%connect%';

   Connections，试图连接到（不管是否成功）MySQL服务器的连接数。
   Max_used_connections，服务器启动后已经同时使用的连接的最大数量。
   Threads_connected，当前的连接数。

2、mysql> show variables like '%connect%';

   max_connections，最大连接数。

3、修改max_connections

   在配置文件（my.cnf或my.ini）在最下面，天加一句：
   max_connections=32000
   然后，用命令重启：/etc/init.d/mysqld restart
   虽然这里写的32000，实际MySQL服务器允许的最大连接数16384；
   添加了最大允许连接数，对系统消耗增加不大
4、临时生效 
 set GLOBAL max_connections=1000;
```
## mysql 客户端安装

```shell
#安装yum源
rpm -ivh https://repo.mysql.com//mysql57-community-release-el7-11.noarch.rpm

#可以通过yum搜索
yum search mysql-community
#安装的是64位
yum install mysql-community-client.x86_64 --nogpgcheck -y
```

## 事务的属性

事务具有以下四个标准属性，通常用缩略词 ACID 来表示：

```
- 原子性：保证任务中的所有操作都执行完毕；否则，事务会在出现错误时终止，并回滚之前所有操作到原始状态。
- 一致性：如果事务成功执行，则数据库的状态得到了进行了正确的转变。
- 隔离性：保证不同的事务相互独立、透明地执行。
- 持久性：即使出现系统故障，之前成功执行的事务的结果也会持久存在。
```

## 事务控制

有四个命令用于控制事务：

```
- COMMIT：提交更改；
- ROLLBACK：回滚更改；
- SAVEPOINT：在事务内部创建一系列可以 ROLLBACK 的还原点；
- SET TRANSACTION：命名事务；
```

## SQL 语句及其种类

```shell
#DDL（数据定义语言）
create ==> 创建数据库或者表等对象
drop ==> 删除数据库或者表等对象
alter ==> 修改数据库或者表等对象的结构
#DML（数据操作语言）
select ==> 查询表中数据
insert ==> 向表中插入数据
update ==> 更新表中数据
delete ==> 删除表中数据
#DCL（数据控制语言）
commit ==> 决定对数据库中的数据进行变更
rollback ==> 取消对数据库中的数据进行变更
grant ==> 赋予用户操作权限
revoke ==> 取消用户的操作权限
```

# 二：备份与恢复

###### 2.1: 备份与恢复 表

```shell
#方法1
create table 备份 like 主表（备份结构）
INSERT INTO 备份表 SELECT * FROM user;（备份数据）

#方法2
备份channel_cpp库的c_trip_renewal表
mysqldump -udba -p -S /tmp/mysql3310.sock channel_cpp c_trip_renewal > /data/backup/20210304.sql
```

###### 2.2: 备份与恢复 库

```shell
#备份库到当前目录
$mysqldump –u root –p*** dbname > filename.sql

#还原库  前提是数据库必须存在
$mysql  –u root –p***　dbname <filename.sql
#或者
mysql>use dbname;

mysql>source (fullpath)filename.sql

mysqldump -udba -p -S /tmp/mysql3311.sock --single-transaction product_travel_ps >/data/mysqlbackup/product_travel_ps_210225.sql

mysqldump -udba -p -S /tmp/mysql3310.sock channel_cpp c_trip_renewal > /data/backup/20210304.sql

 #备份某个数据库
mysqldump -uroot -p  database_name >/tmp/mysql.bak.sql;
#备份数据库中的某张表
mysqldump -uroot -p  database_name table_name >/tmp/mysql.table.sql;
#还原某个数据库
mysql -uroot -p  database_name < /tmp/mysql.bak.sql;
#还原数据库中的某一张表
mysql -uroot -p  database_name < /tmp/mysql.table.sql;
```

# 三：记录

###### 查看表容量

```shell
#查看mysql数据库容量大小
#第一种情况：查询所有数据库的总大小，方法如下：
mysql> use information_schema;
mysql> select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from TABLES;
+-----------+
| data      |
+-----------+
| 3052.76MB |
+-----------+
1 row in set (0.02 sec)
#统计一下所有库数据量 
每张表数据量=AVG_ROW_LENGTH*TABLE_ROWS+INDEX_LENGTH
SELECT
SUM(AVG_ROW_LENGTH*TABLE_ROWS+INDEX_LENGTH)/1024/1024 AS total_mb
FROM information_schema.TABLES 
#统计每个库大小：
SELECT
table_schema,SUM(AVG_ROW_LENGTH*TABLE_ROWS+INDEX_LENGTH)/1024/1024 AS total_mb
FROM information_schema.TABLES group by table_schema;  
# 第二种情况：查看指定数据库的大小，比如说：数据库test，方法如下：
mysql> use information_schema;
mysql> select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from TABLES where table_schema='test';
+----------+
| data     |
+----------+
| 142.84MB |
+----------+
1 row in set (0.00 sec)
#查看所有数据库各容量大小
select
table_schema as '数据库',
sum(table_rows) as '记录数',
sum(truncate(data_length/1024/1024, 2)) as '数据容量(MB)',
sum(truncate(index_length/1024/1024, 2)) as '索引容量(MB)'
from information_schema.tables
group by table_schema
order by sum(data_length) desc, sum(index_length) desc;
#查看所有数据库各表容量大小
select
table_schema as '数据库',
table_name as '表名',
table_rows as '记录数',
truncate(data_length/1024/1024, 2) as '数据容量(MB)',
truncate(index_length/1024/1024, 2) as '索引容量(MB)'
from information_schema.tables
order by data_length desc, index_length desc;
#查看指定数据库容量大小
#例：查看mysql库容量大小
select
table_schema as '数据库',
sum(table_rows) as '记录数',
sum(truncate(data_length/1024/1024, 2)) as '数据容量(MB)',
sum(truncate(index_length/1024/1024, 2)) as '索引容量(MB)'
from information_schema.tables
where table_schema='mysql';　
#查看指定数据库各表容量大小
#例：查看mysql库各表容量大小
select
table_schema as '数据库',
table_name as '表名',
table_rows as '记录数',
truncate(data_length/1024/1024, 2) as '数据容量(MB)',
truncate(index_length/1024/1024, 2) as '索引容量(MB)'
from information_schema.tables
where table_schema='mysql'
order by data_length desc, index_length desc;
```

# 四：自动安装脚本

```shell
#!/bin/bash
#脚本说明部分
cat << EOF
1、请事先下载好MySQL二进制安装包并解压于/usr/local下
2、为二进制包做好软连接，如：
        ln -s /usr/local/mysql-5.7.27-linux-glibc2.12-x86_64/ /usr/local/mysql
3、确保上面两步都操作后，运行脚本完成服务安装和初始化，MySQL监听端口为3306，配置文件为/etc/my.cnf
EOF

#变量设置
mysql_dir=/data/mysql
data_dir=/data/mysql/data
log_dir=/data/mysql/log
echo ""
#初始化数据库（可反复执行，但会清空数据）
read -p "是否进行初始化，初始化将清空已有数据,输入y继续,其它按键退出脚本：" input
if [ $input == "y" ];then
  #创建mysql用户
  id mysql > /dev/null 2>&1
  if [ $? != 0 ];then
    useradd -r -s /sbin/nologin -M mysql && echo "成功创建MySQL用户"
    sleep 1
  else
    echo "MySQL用户已存在，无需创建"
    sleep 1
  fi

#创建数据和日志目录
mkdir -p $log_dir
mkdir -p $data_dir
chown -R mysql.mysql $mysql_dir

#创建数据库配置文件
if [ -e /etc/my.cnf ];then
  echo > /etc/my.cnf
else
  touch /etc/my.cnf
fi

#my.cnf配置文件内容
cat > /etc/my.cnf <<EOF
[mysqld]
basedir = /usr/local/mysql
datadir = $data_dir
port = 3306
socket = /tmp/mysql.sock 
character-set-server = utf8mb4
default-storage-engine = INNODB
skip-name-resolve
server_id = 10
# sql_safe_updates=1
sql_mode =STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION

#Log setting
log_error = $log_dir/error.log
slow_query_log = on
slow_query_log_file = $log_dir/slow.log
long_query_time = 3
log_bin = $log_dir/master-bin
log_bin_index = $log_dir/master-bin.index
binlog_format = row
expire_logs_days = 10
max_binlog_size = 1024M
binlog_cache_size = 32768

#双1参数，建议都设置为1提升数据安全
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1
# Innodb setting
innodb_buffer_pool_size = 512M 
innodb_flush_method=O_DIRECT 
innodb_log_file_size = 200M 
innodb_file_per_table = 1  
innodb_lock_wait_timeout = 10 
# innodb_buffer_pool_dump_at_shutdown = ON  
# innodb_buffer_pool_filename = ib_buffer_pool
# innodb_buffer_pool_load_at_startup = ON 

[mysql]  #作用于mysql客户端工具如果这里配置了先关选项，使用客户端命令就不需要填写了
socket = /tmp/mysql.sock
prompt= "[\\u@linuxe][\\d]>"  
# tee = "/data/dblog/tee.log"  
no-auto-rehash
max_connections=32000
EOF

#检查是否有3306端口占用
mysql_proc=`netstat -ntulp | grep 3306 | wc -l`
if [ $mysql_proc -gt 0 ];then
  echo "当前MySQL正在运行，请输入MySQL管理密码停止进程"
  /usr/local/mysql/bin/mysqladmin -uroot -p -S /tmp/mysql.sock shutdown && echo "MySQL服务已关闭，开始初始化"
    if [ $? -ne 0 ];then
      echo "密码错误，脚本退出" 
      exit 1
    fi
  else
      echo "没有MySQL进程正在运行，开始进行初始化"
      sleep 1
fi

rm -rf $data_dir/*  && echo "已清空数据目录"
rm -rf $log_dir/*  && echo "已清空日志目录"

#初始化数据库
/usr/local/mysql/bin/mysqld --defaults-file=/etc/my.cnf  --initialize --user=mysql
if [ $? -ne 0 ];then
  echo "初始化失败，脚本退出" 
  exit 1
else
  mysql_password=`grep "temporary password" $log_dir/error.log | awk '{print $NF}'`
  echo "初始化成功，当前MySQL临时管理密码：$mysql_password"
fi

#启动服务
  /usr/local/mysql/bin/mysqld_safe --defaults-file=/etc/my.cnf 2>&1 > /dev/null &
  echo "请运行/usr/local/mysql/bin/mysql_secure_installation 进行安全初始化"
else
  echo "脚本已退出"
fi
```
