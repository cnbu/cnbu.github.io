# 常用命令

```shell
#登录客户端 -a 密码 -p 端口
redis-cli -a a5791E2a3043T -p 6379

#查看集群状态
redis-cli -a a5791E2a3043T info

# 使用 1 号数据库
redis 127.0.0.1:6379> SELECT 1

#删除指定key
del key名

#设置默认数据为0
SET db_number 0

#获取4号库的key
redis-cli -h 172.22.8.187 -a a5791E2a3043T -n 4  keys CR:planet_company* > /tmp/keys.txt


#选择DB
select 5
#Redis统计信息
info
#在线修改配置
192.168.0.238:6379> config get maxmemory
1) "maxmemory"
2) "1073741824"
192.168.0.238:6379> config set maxmemory 5000000
OK
192.168.0.238:6379> config get maxmemory
1) "maxmemory"
2) "5000000"
```

# 批量删除脚本

```shell
删除4号库的key
#!/bin/bash
for i in `cat /tmp/keys.txt`
do
 redis-cli -h 172.22.8.187 -a a5791E2a3043T -n 4 del $i
done
```

# 哨兵

```
哨兵的三个作用：
监控：监控谁？支持主从结构的工作一个是主节点一个是从节点，那肯定就是监控这俩个了。监控主节点和从节点是否正常运行；检测主节点是否存活，主节点和从节点运行情况。

通知：哨兵检测的服务器出现问题时，会向其他的哨兵发送通知，哨兵之间就相当于一个微信群，每个哨兵发现的问题都会发在这个群里。

自动转移故障：当检测到主节点宕机后，断开与宕机主节点连接的所有从节点，在从节点中选取一个作为主节点，然后将其他的从节点连接到这个最新主节点的上。并且告知客户端最新的服务器地址。

这里有一个注意点，哨兵也是一台 Redis 服务器，只是不对外提供任何服务。配置哨兵时配置为单数。
```

# 一、版本

3.2.9

# 二、模式

主从+哨兵模式
哨兵模式至少要求3节点，所以下面的步骤需要在每个节点都操作

# 三、环境信息
| IP | 系统版本 | 角色 |
| --- | --- | --- |
| 172.20.1.171 | Centos7.5 | master |
| 172.20.1.172 | Centos7.5 | slave |
| 172.20.1.173 | Centos7.5 | slave |


# 四、部署步骤

#### 1、下载安装包

（1）服务器执行命令下载：

```
cd /usr/local/src/
wget https://hz-package.hzins.com/redis-3.2.9.tar.gz
```

#### 2、解压安装包

```
tar zxf redis-3.2.9.tar.gz
```

#### 3、编译安装Redis

```shell
cd redis-3.2.9
yum install gcc -y  # 安装gcc编译环境
yum install jemalloc # 安装内存分配器，jemalloc在处理内存碎片化上优于Redis默认内存分配器libc
make MALLOC=/usr/lib64/libjemalloc.so.1  # 通过rpm -ql jemalloc可以查看so文件路径
make install        # make的时候安装已经完成，make install会把redis命令放在/usr/local/bin下
```

#### 4、服务器内核参数调整

（1）服务器内存分配策略，减少OOM

```
echo "vm.overcommit_memory=1" >> /etc/sysctl.conf
sysctl vm.overcommit_memory=1
```

（2）关闭THP，减少延迟

```
echo never > /sys/kernel/mm/transparent_hugepage/enabled   #该命令执行后还需写入到/etc/rc.local
```

#### 5、启动测试

运行redis-server进行启动测试，观察是否有报错

#### 6、Redis主从配置

※ 创建/etc/redis.conf配置文件
※ 通常情况下只关注和修改带有注释项的即可，正式配置时请删掉注释信息

```shell
bind 172.20.1.171           #redis服务需要监听在服务器哪个IP上，每个节点修改为本机IP即可
protected-mode yes
daemonize yes 
port 6379                   #redis服务端口
tcp-backlog 511  
timeout 0
tcp-keepalive 300
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile "/data/redis/logs/redis.log"    #日志存放路径，该目录需要手动创建
databases 16
requirepass 123456           #redis密码
maxclients 10000             #最大客户端连接数
maxmemory 1GB                #redis最大占用内存，建议配置为服务器最大内存的60%
maxmemory-policy volatile-lru  

# RDB持久化配置 
# save 900 1
# save 300 10
# save 60 10000
stop-writes-on-bgsave-error yes  #如果快照保存失败，所有节点停止写入
rdbcompression yes    
rdbchecksum yes       
dbfilename dump.rdb   #RDB文件名
dir /data/redis/rdb/  #RDB存放目录，需要手动创建


# 主从配置
slaveof <masterip> <masterport>  #该项只在从节点配置，指定主库的ip和端口
masterauth 123456                #主库的密码，所有节点配置一致
slave-serve-stale-data yes       #如果从库与主库失联，是否继续响应客户端请求，No则响应SYNC with master in progress
slave-read-only yes              #该项只在从节点配置，从库开启只读
repl-ping-slave-period 10        #从库向主库发送ping请求的间隔
repl-timeout 60                  #主从超时时间
repl-disable-tcp-nodelay no      #是否用nodelay方式传输数据，no可以降低数据传输到从库的延迟，但使用更多的带宽
repl-backlog-size 10mb           #从服务离线之后，主服务器会把离线之后的写入命令存储在一个特定大小的队列中，避免短时间断开服务却进行全量同步的问题
repl-backlog-ttl 3600            #当所有从库都与主库断开连接后达到多少秒释放backlog
slave-priority 100               #从库优先级，数字越小优先级越高，0代表不会被哨兵选为主
# min-slaves-to-write 3          #如果从库少于N个，主库就停止写入，需配合min-slaves-max-lag一起使用。比如至少需要3个从库、并且延时小于等于10秒的，主库才能写入
# min-slaves-max-lag 10          #如果从库延迟小于N秒，主库才能写入数据，需配合min-slaves-to-write一起使用。比如至少需要3个从库、并且延时小于等于10秒的，主库才能写入
repl-diskless-sync yes           #开启无盘复制，主从全量同步时，主库并不会在本地创建RDB 文件，而是创建一个子进程通过Socket将RDB文件写入到从服务器，节约IO资源
repl-diskless-sync-delay 5

# AOF持久化配置
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
# appendfsync always
# appendfsync no
no-appendfsync-on-rewrite no     #如果正在导出rdb数据则停止aof的写入，aof将保存在一个队列中，等rdb备份完成后再执行队列，不会丢失数据
auto-aof-rewrite-percentage 100  #aof文件体积与上次相比增长率达到100%就进行重写（重写相当于记总账，比如对同一个key做了100次操作，我们只需要最后一次的操作，重写就会把多余的操作给忽略掉，节省内存
auto-aof-rewrite-min-size 64mb   #和auto-aof-rewrite-percentage组合使用，aof文件达到64M时进行重写
aof-load-truncated yes
aof-rewrite-incremental-fsync yes  #当Redis保存AOF文件时，每生成32MB数据就执行一次fsync操作，这样分批提交可以避免高延迟

#慢日志配置
slowlog-log-slower-than 1000000  #慢查询的评定时间，单位为微妙，这里代表1秒
slowlog-max-len 1000  #慢日志最大记录条数



#客户端输出缓冲区配置，内网环境可以全部为0
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 0 0 0
client-output-buffer-limit pubsub 0 0 0

hz 10  #通常使用默认10即可（可选范围1到500）。该值设置Redis每秒处理后台任务的频率，如清理过期Key、关闭超时连接等。调高该值可以让CPU空闲时间更频繁的去处理后台的任务，但通常不建议调高到100以上
```

#### 7、启动Redis

```
redis-server /etc/redis.conf
```

#### 8、连接redis，检查三个节点主从状态

```
redis-cli -a 123456 -h 172.20.1.171 info
redis-cli -a 123456 -h 172.20.1.172 info
redis-cli -a 123456 -h 172.20.1.173 info
```

执行info命令后关注Replication字段，确定主从节点正常，即master下能看到两个slave节点

#### 9、哨兵配置

（1）创建/etc/sentinel.conf配置文件

```shell
bind 172.20.1.171           #哨兵服务需要监听在服务器哪个IP上，修改为本机IP即可
daemonize yes
protected-mode yes  
port 26379                  #哨兵服务端口
logfile /usr/local/redis/log/sentinel.log    #哨兵日志路径
dir /usr/local/redis/data/

sentinel monitor mymaster 172.20.1.171 6379 2     #配置要监控的主节点名字、IP、端口以及选举票数。哨兵会通过master自动发现其他slave服务器。选举票数建议为节点数/2+1，3节点下配置为2
sentinel down-after-milliseconds mymaster 30000   #判断master节点是否存活，超过这个时长就将其下线。单位是毫秒，默认是30秒
sentinel parallel-syncs mymaster 1                #哨兵发生切换后允许多少台slave服务器对新master进行同时同步，该项用于减轻压力
sentinel failover-timeout  mymaster 180000        #主节点出现故障时，故障转移超时时间，超出这个时间范围代表切换失败，默认是3分钟
sentinel auth-pass mymaster 123456                #redis主节点的密码
```

（2）启动哨兵

```
redis-sentinel /etc/sentinel.conf
```

（3）连接哨兵

```
 redis-cli -h 172.20.1.171 -a 123456 -p 26379 info
 redis-cli -h 172.20.1.172 -a 123456 -p 26379 info
 redis-cli -h 172.20.1.173 -a 123456 -p 26379 info
```

执行info命令后关注Sentinel字段，确定status=ok，并且adress信息和slave数量都是正常的

# 五、服务启停

#### 1、关闭Redis

使用redis-cli客户端连接服务后发送shutdown指令即可关闭Redis

```
redis-cli -h 172.20.1.171 -a 123456  shutdown
```

#### 2、关闭哨兵

使用redis-cli客户端连接哨兵后发送shutdown指令即可关闭哨兵服务

```
redis-cli -h 172.20.1.171 -a 123456 -p 26379 shutdown
```

#### 3、启动Redis

```
redis-server /etc/redis.conf
```

#### 4、启动哨兵

```
redis-sentinel /etc/sentinel.conf
```
