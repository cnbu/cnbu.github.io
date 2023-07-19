### 一、版本

5.7.33

### 二、模式

一主两从

### 三、环境信息
| IP | 系统版本 | 角色 |
| --- | --- | --- |
| 172.19.1.120 | Centos7.5 | 主 |
| 172.19.1.121 | Centos7.5 | 从 |
| 172.19.1.122 | Centos7.5 | 从 |


### 四、部署步骤

#### 1、下载安装包

（1）服务器执行命令下载：

```
mkdir -p /usr/local/src
cd /usr/local/src
wget https://hz-package.hzins.com/mysql-5.7.33-linux-glibc2.12-x86_64.tar.gz
```

#### 2、部署前系统环境设置

（1）关闭 SELinux

```
vim /etc/selinux/config
SELINUX=disabled
```

（2）关闭 firewalld防火墙

```shell
systemctl stop firewalld
systemctl disable firewalld
systemctl status firewalld     ##查看状态
```

（3）设置swap分区，默认为1

```shell
vim /proc/sys/vm/swappiness     ##设置为1
cat  /proc/sys/vm/swappiness    ##查看是否为1
```

（4）文件系统建议使用xfs
（5）修改操作系统限制

```shell
vim /etc/security/limits.conf   ##在末尾添加
* soft nofile 65535
* hard nofile 65535
* soft nproc 65535
* hard nproc 65535
```

检查命令ulimit -a
（6）关闭NUMA

```shell
vi /etc/default/grub   ##在 GRUB_CMDLINE_LINUX 参数的末尾增加 ： numa=off
```

以上操作完成后，重启操作系统使配置生效。

#### 3、部署MYSQL

（1）建立mysql用户

```
groupadd mysql
useradd -g mysql mysql -s /sbin/nologin
```

（2）解压、软链接

```shell
cd /usr/local/src
tar -xvzf  mysql-5.7.33-linux-glibc2.12-x86_64.tar.gz
ln -s /usr/local/src/mysql-5.7.33-linux-glibc2.12-x86_64 /usr/local/mysql
```

（3）建立数据和日志目录

```shell
mkdir -p /data/mysql3306/
mkdir -p /data/tmp/mysql3306
mkdir -p /data/dblog/mysql3306/{binlog,relaylog}
touch  /data/dblog/mysql3306/error.log
chown -R mysql:mysql /data/mysql3306/
chown -R mysql:mysql  /data/tmp/mysql3306
chown -R mysql:mysql /data/dblog/
```

（4）编辑配置文件

```shell
vim  /etc/my3306.cnf
[client]
port = 3306
socket = /tmp/mysql.sock


[mysql]
no-auto-rehash
prompt = "\\u@\\d \\R:\\m> "
default-character-set = utf8


[mysqld]
########basic settings########
server-id = 120
port = 3306
user            = mysql
basedir         = /usr/local/mysql
datadir = /data/mysql3306
tmpdir = /data/tmp/mysql3306
socket = /tmp/mysql.sock
skip-external-locking
skip-name-resolve
back_log        = 600
connect_timeout = 20
character_set_server            = utf8
collation-server                = utf8_general_ci
skip-character-set-client-handshake=1
default-storage-engine          = InnoDB
character-set-client-handshake  = FALSE
init_connect                    ='set names utf8'
skip_name_resolve               = 1
max_connections                 = 1000
max_connect_errors              = 5000
interactive_timeout             = 400
connect_timeout                 = 20
wait_timeout                    = 400
max_allowed_packet              = 512M
group_concat_max_len            = 10240
transaction_isolation           = READ-COMMITTED
log_bin = /data/dblog/mysql3306/binlog/mysql-bin
explicit_defaults_for_timestamp = 0
#transaction_write_set_extraction=MURMUR32
#sql_mode       = "STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER"
sql_mode      = "NO_ENGINE_SUBSTITUTION"
optimizer_switch="index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on,index_condition_pushdown=on,mrr=on,mrr_cost_based=on,block_nested_loop=on,batched_key_access=off,materialization=on,semijoin=on,loosescan=on,firstmatch=on,duplicateweedout=on,subquery_materialization_cost_based=on,use_index_extensions=on,condition_fanout_filter=on,derived_merge=off"
max_execution_time=60000
log_timestamps=SYSTEM
#######per_thread_buffers############
join_buffer_size                = 134217728
tmp_table_size                  = 67108864
read_buffer_size                = 16777216
read_rnd_buffer_size            = 33554432
sort_buffer_size                = 33554432
table_open_cache                = 8000
table_definition_cache          = 4096


########slow log settings########
slow_query_log                  = 1
slow_query_log_file = /data/dblog/mysql3306/mysql.slow
long_query_time                 = 1
#log_queries_not_using_indexes  = 1
#log_slow_admin_statements      = 1
#log_slow_slave_statements      = 1
#log_throttle_queries_not_using_indexes = 10
#min_examined_row_limit         = 100


#####error log settings########
log_error = /data/dblog/mysql3306/error.log


####binlog settings####
log_bin = /data/dblog/mysql3306/binlog/mysql-bin
expire_logs_days                = 20
binlog_format                   = row
max_binlog_size                 = 1024M
binlog_cache_size               = 4M
binlog_stmt_cache_size          = 1M
binlog_rows_query_log_events=ON

########replication settings########
skip-slave-start                = 1
slave-net-timeout               = 10
#######mysql5.7###########
slave_parallel-type = LOGICAL_CLOCK
slave_parallel_workers = 8
#########################
master_info_repository          = TABLE
relay_log_info_repository       = TABLE
log_slave_updates               = 1
relay_log_recovery              = 1
relay_log_purge                 = 0
relay-log = /data/dblog/mysql3306/relaylog/relay-bin
sync_master_info                = 1
sync_relay_log_info             = 1
sync_relay_log                  = 1
#read_only = 1
#super_read_only = 1


########innodb settings########
innodb_data_home_dir = /data/mysql3306
innodb_data_file_path           = ibdata1:1024M:autoextend:max:10G
#innodb_data_file_path          = ibdata1:1024M:autoextend
innodb_page_size                = 16384
innodb_buffer_pool_size = 48G
innodb_buffer_pool_instances    = 16
innodb_lru_scan_depth           = 2000
innodb_lock_wait_timeout        = 10
innodb_read_io_threads          = 16
innodb_write_io_threads         = 16
innodb_io_capacity              = 4000
innodb_io_capacity_max          = 10000
innodb_flush_method             = O_DIRECT
innodb_file_per_table           = 1
innodb_file_format              = Barracuda
innodb_file_format_max          = Barracuda
innodb_log_file_size            = 4G
innodb_log_buffer_size          = 16777216
innodb_log_files_in_group       = 3
innodb_undo_logs                = 128
innodb_undo_tablespaces         = 0
innodb_sort_buffer_size         = 67108864
innodb_autoinc_lock_mode        = 2
innodb_flush_neighbors          = 0
innodb_large_prefix             = 1
innodb_purge_threads            = 8
innodb_purge_batch_size         = 300
innodb_thread_concurrency       = 64
innodb_print_all_deadlocks      = 1
innodb_strict_mode              = 1
innodb_max_dirty_pages_pct      = 75
innodb_old_blocks_pct           = 37
innodb_old_blocks_time          = 1000
innodb_stats_on_metadata        = off
innodb_buffer_pool_load_at_startup      = 1
innodb_buffer_pool_dump_at_shutdown     = 1

#mysql5.7 new
#innodb_buffer_pool_dump_pct    = 40
#innodb_page_cleaners           = 4
#innodb_undo_log_truncate       = 1
#innodb_max_undo_log_size       = 2G
#innodb_purge_rseg_truncate_frequency = 128

#双1参数
innodb_flush_log_at_trx_commit  = 1
sync_binlog                     = 1
innodb_support_xa               = 1


########gtid settings########
gtid_mode                       = on
enforce_gtid_consistency        = 1
#binlog_gtid_simple_recovery    = 1


########semi sync replication settings########
#plugin_dir      = /usr/lib64/mysql/plugin/
#plugin_load     = "rpl_semi_sync_master=semisync_master.so;rpl_semi_sync_slave=semisync_slave.so"
#loose_rpl_semi_sync_master_enabled     = 1
#loose_rpl_semi_sync_slave_enabled      = 1
#loose_rpl_semi_sync_master_timeout     = 5000


[mysqldump]
quick
max_allowed_packet      = 512M


[myisamchk]
key_buffer_size         = 20M
sort_buffer_size        = 20M
read_buffer             = 16M
write_buffer            = 16M


[mysqlhotcopy]
interactive-timeout

[mysqld_safe]
###numa####
#flush_caches = 1
#numa_interleave = 1
open-files-limit = 65535
#malloc-lib = /usr/local/lib/libtcmalloc.so
#malloc-lib= /usr/local/mysql/lib/mysql/libjemalloc.so
```

**注意：server-id在同一主从环境中，每个节点server-id 需要不一样。**
（5）启动MySQL

```shell
cd /usr/local/mysql/bin
##初始化mysql
./mysqld --defaults-file=/etc/my3306.cnf --basedir=/usr/local/mysql --datadir=/data/mysql3306 --user=mysql --initialize
###查看root初始密码
vim  /data/dblog/mysql3306/error.log   
##可以看到如下信息
[Note] A temporary password is generated for root@localhost: /#T#mZhZQ7gp   ##此即root初始密码
##启动mysql
/usr/local/mysql/bin/mysqld_safe  --defaults-file=/etc/my3306.cnf  &
##查看mysql是否启动成功
ps -ef|grep mysql
##复制二进制文件
cp mysql /usr/bin
##用root初始密码登录
mysql -uroot -p      
#需要先修改密码才能进行其他操作
mysql>alter user 'root'@'localhost' identified by "1qaz@2wsx";
```

#### 4、搭建主从

按以上步骤，在两台从库服务器172.19.1.121、122上安装MySQL，注意配置修改配置文件中**server-id**的值**。**
（1）在主库上分别对从库进行授权

```shell
mysql>use mysql;
mysql>GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'172.19.1.121'  IDENTIFIED BY 'repl888#@999';
mysql>GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'172.19.1.122'  IDENTIFIED BY 'repl888#@999';
```

（2）分别在从库进行change master，执行如下操作

```shell
mysql -uroot -p
mysql>use mysql;
mysql>CHANGE MASTER TO MASTER_HOST='172.19.1.120',MASTER_USER='repl',MASTER_PASSWORD='repl888#@999',MASTER_PORT=3306,MASTER_AUTO_POSITION=1;
##启动主从同步
mysql>start slave;
##查看主从同步是否正常
mysql>show slave status\G;
```

### 五、服务启停

#### 1、MySQL启动

```shell
/usr/local/mysql/bin/mysqld_safe  --defaults-file=/etc/my3306.cnf  &
```

#### 2、MySQL停止

```shell
/usr/local/mysql/bin/mysqladmin -uroot -p --socket=/tmp/mysql.sock  shutdown
```
