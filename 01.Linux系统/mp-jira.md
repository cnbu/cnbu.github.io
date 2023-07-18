# jira7.2 安装：

jira的安装很简单

###### 1、nginx  用我们标准的tengine剧本装即可

###### 2、MySQL  标准安装即可

###### 3、jira服务 照着下列链接操作就行

```
https://blog.csdn.net/qq_35957398/article/details/80388958
```

jira有两个目录，一个数据目录，一个系统程序目录

安装过程中会出现让你选择安装及数据目录的选项，记得选择磁盘空间大的目录就行。

中文插件、MySQL的`connection jar`包以及破解补丁包  都在这里 直接下载下来就可以用

```
http://10.5.1.43/soft/jira7.2.zip
```

jira的`License Key` 在 Jira-Server_ID-License_KEY.txt 文件中   直接复制粘贴进去就修行

# 维护：

目前 慧择使用的jira

**MP master节点：**`**172.20.0.230**`

所有常规操作在这台机器上操作即可

### 系统目录：`/opt/atlassian/jira/`

这个目录是jira运行系统所在的目录

jira重启：

```
su - jira     #在root用户下执行 su - jira 使之切换至jira用户

cd  /opt/atlassian/jira/bin/  #进入jira系统操作命令目录
 
bash stop-jira.sh    #执行停止jira系统脚本
 
ps -ef |grep java    #检查jira进程是否还存在

bash start-jira.sh    #执行启动jira系统脚本

然后访问 http://mp.oa.com   等界面加载完成  即启动成功
```

### 数据目录：`/var/atlassian/application-data/jira/`

这个目录是jira所有数据的目录地址，其中 `dbconfig.xml` 文件是数据库的核心配置文件

### crontab脚本目录：`/usr/local/shells/`

目前运行中的脚本任务如下

```
00 00 * * * root sleep $(($$\%10)) ;bash /usr/local/shells/jira_sql_bak.sh > /dev/null
00 03 1 * * root sleep $(($$\%10)) ;/bin/bash /usr/local/shells/auto_restart_jira.sh > /tmp/jira_restart_log.txt
00 00 * * * root sleep $(($$\%10)) ;bash /usr/local/shells/cut_nginx_log_zip.sh > /dev/null
30 01 * * * root sleep $(($$\%60)); bash  /usr/local/shells/find_nginx_log_rm.sh > /dev/null
40 02 * * * root sleep $(($$\%60)); bash /usr/local/shells/find_nginx_log_zip_rm.sh > /dev/null
```

注意：jira系统会在每月1号凌晨 1：30自动重启，第二天上班后记得观察jira状态。

**MP slave节点：**`**172.20.1.115**`

slave节点的搭建方法与Master 相同

slave节点部署好之后 停掉slave的jira服务，请路总做主从同步 彭浩同学做好主从同步告警即可

### Jira主备同步

在 主 备 两台机器的Jira都部署好正常可以访问之后

我这里使用lsync+rsync来实现 主备实时同步

主节点：172.20.0.230
备节点：172.20.1.115

步骤：

1、备节点部署rsync服务

172.20.1.115

关闭防火墙

```
systemctl disable firewalld
systemctl disable iptables
```

创建用来同步的用户

```
useradd -g jira -s /bin/bash   jirabak
echo 'r1_u8MdcA' | passwd --stdin jirabak
```

安装rsync服务

```
yum -y install rsync
```

配置rsync同步参数

```
cat  >/etc/rsyncd.conf << EOF
uid = 0
gid = 0
use chroot = no
list = no
read only = no
hosts allow = *
[appData]
comment = appData
path = /var/atlassian/application-data/jira
auth users = jirabak
secrets file = /etc/rsyncd.secrets

[jira]
comment = jira
path = /opt/atlassian/jira
auth users = jirabak
secrets file = /etc/rsyncd.secrets
EOF

echo "jirabak:r1_u8MdcA" > /etc/rsyncd.secrets

chmod 600  /etc/rsyncd.secrets
```

启动rsync服务

```
rsync --daemon --config=/etc/rsyncd.conf
```

2、主节点部署lsyncd服务及放置rsync密码文件

172.20.0.230

关闭防火墙

```
systemctl disable firewalld
systemctl disable iptables
```

创建用来同步的用户

```
useradd -g jira -s /bin/bash   jirabak
echo 'r1_u8MdcA' | passwd --stdin jirabak
```

添加rsync认证密码

```
echo "r1_u8MdcA" > /etc/rsyncd.secrets
chmod 600  /etc/rsyncd.secrets
```

安装lsync服务

```
yum -y install lsyncd
```

配置lsync参数

```
cat > /etc/lsyncd.conf << EOF 

settings {
        logfile = "/var/log/lsyncd/lsyncd.log",
        statusFile = "/var/log/lsyncd/lsyncd.status",
        inotifyMode = "CloseWrite",
        statusInterval = 3,
        maxProcesses = 8,
        maxDelays = 1,
        nodaemon = false,
}

-- 同步jira数据目录
sync {
        default.rsync,
        source = "/var/atlassian/application-data/jira",
        target = "jirabak@172.20.1.115::appData",
        delay = 15,
        rsync = {
                binary = "/usr/bin/rsync",               --rsync可执行文件路径，必须为绝对路径
                password_file = "/etc/rsyncd.secrets",       --密码认证文件
                archive = true,
                compress = true,
                verbose = true,
        }
}

-- 同步jira程序目录
sync {
        default.rsync,
        source = "/opt/atlassian/jira",
        target = "jirabak@172.20.1.115::jira",
        delay = 15,
        rsync = {
                binary = "/usr/bin/rsync",               --rsync可执行文件路径，必须为绝对路径
                password_file = "/etc/rsyncd.secrets",       --密码认证文件
                archive = true,
                compress = true,
                verbose = true,
        }
}

EOF

echo "fs.inotify.max_user_watches=99999999" >> /etc/sysctl.conf
sysctl -p
```

增加服务自起

```
systemctl enable lsyncd
```

启动lsync服务

```
systemctl start lsyncd
```

至此 主备同步部署完成
查看备节点同步效果

以上步骤即可完成主 备节点 系统及数据实时同步。

将来主备切换的时候 只需要dns将域名切到备节点   备节点数据库提升为主库 ，即可启动备节点
数据完全一致。

# jira6.1.1: 安装以及恢复

## 一：安装

##### 1：jira6.1.1安装包以及jdbc包

```
http://10.5.1.43/soft/jira6.1.1.zip
```

##### 2:  jdk使用7

```
http://10.5.1.43/soft/java/jdk-7u71-linux-x64.rpm
```

##### 3:  安装jira6.1参考7.2的安装教程

```
https://blog.csdn.net/qq_35957398/article/details/80388958
```

##### 4：备份存档找城哥要

## 二：恢复数据

##### 2.1：参考链接

```
https://www.cnblogs.com/kevingrace/p/8862531.html
```

##### 2.2：问题处理

###### 2.2.1：忘记管理员密码问题

```
1.先执行恢复流程，输入管理员密码这一步。

2.在数据库中修改管理员密码

-- Jira数据库中，用户信息都存放在表 cwd_user中
-- 切换到jiar数据库
use jiradb;

-- 更改密码为123456
update cwd_user
set credential='{PKCS5S2}ms9AdSR9vnOXqnNdEmRG/kxRc22qTnx3Y/nwdyaNEg5/XAANouQ+akxcQbFjJiQ4'
where user_name='XXXX'
;
```



