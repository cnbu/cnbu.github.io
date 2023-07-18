# 系统安装

#### 修改网卡为eth0
```
#第一个历程：选择安装Centos7系统

#第二个历程：按tab键

#第三个历程：quiet后面加上net.ifnames=0 biosdevname=0（要有空格）

#第四个历程：按回车键
```
#### VMvare fusion虚拟机许可证

```
YF390-0HF8P-M81RQ-2DXQE-M2UT6
```

```shell
#设置raid条带为64k，读策略设置为已预读，写策略配为回写（这主要是针对数据库服务器）
#CPU模式设为performance，因BIOS设置不能生效，需在开机自启动项里配置脚本
mkdir -p /data/auto
-------------------------------------------------------------------
vim /data/auto/auto-set-performance.sh
#!/bin/bash

sleep 5                                                ##先等待5秒再设置高可用模式
/usr/bin/cpupower frequency-set -g performance          ##设置performance模式

sleep 10                                                 ##等待10秒再查看
module=`/usr/bin/cpupower frequency-info|grep -v governors|grep governor|awk '{print $3}'`
if [[ $module == '"powersave"' ]];then                                                          ##如果还是节能模式就设置performance
   /usr/bin/cpupower frequency-set -g performance
fi
-------------------------------------------------------------------
chmod +x /data/auto/auto-set-performance.sh
echo 'sh /data/auto/auto-set-performance.sh' >> /etc/rc.d/rc.local
chmod +x /etc/rc.d/rc.local



#主机kmv安装系统时选择勾选安装Virtualzation Host

#配置阵列：0，1槽位RAID1；2,3,4,5,6槽位组成RAID5
#文件系统要求：文件系统采用ext4格式， 禁止使用xfs
#大数据机器文件系统采用xfs格式

#磁盘阵列: 
第一组：RAID1（2*300G 15K 2.5寸SAS硬盘）0，1（硬盘位） 
第二组：RAID5 5*2.4T SAS(2,3,4,5,6,7)（其中第7位为热备盘

#3、系统安装：（最小化安装） 
操作系统版本：Centos7.7
镜像下载地址：https://mirrors.aliyun.com/centos-vault/7.7.1908/isos/x86_64/CentOS-7-x86_64-DVD-1908.iso
文件系统全部采用: ext4

#系统引导：UEFI 模式 （配置服务器默认使用UEFI模式引导，开机后F2可进入配置界面修改） 
1）时区：Asia/Shanghai 
2）root密码：!Hzins2016# 
3）分区： 
第一组raid1 
/boot 1G 
/boot/efi 1G 
/swap 128G 
/ 第一组raid1剩余空间 

第二组raid5 
/data 第二组raid5全部空间 

注意事项：之前发系统出现软阵列情况，执行cat /proc/mdstat 
装完系统之后不得出现 /dev/md 设备，出现此设备将导致系统严重故障

#配置内核加载
vim /etc/modprobe.d/bonding.conf

alias bond0 bonding
options bond0 miimon=100 mode=4


#加载内核模块命令
modprobe bonding

#重启网络
systemctl restart network

6、关闭相关的服务 
systemctl disable firewalld.service 
systemctl stop firewalld.service 
systemctl disable NetworkManager.service 
systemctl stop NetworkManager.service 
setenforce 0 
sed -i 's/=enforcing/=disabled/g' /etc/selinux/config


#磁盘性能测试
#!/bin/bash

echo "顺序读写"
fio -direct=1  -iodepth=64  -rw=read -ioengine=psync -bs=16k -size=1G -numjobs=10 -runtime=100 -group_reporting -name=/data/iotest|grep 'read: IOPS='
fio -direct=1  -iodepth=64  -rw=write -ioengine=psync -bs=16k -size=1G -numjobs=10 -runtime=100 -group_reporting -name=/data/iotest|egrep 'write: IOPS'

echo "随机读写"
fio -direct=1  -iodepth=64  -rw=randread -ioengine=psync -bs=16k -size=1G -numjobs=10 -runtime=100 -group_reporting -name=/data/iotest |grep 'read: IOPS='
fio -direct=1  -iodepth=64  -rw=randwrite -ioengine=psync -bs=16k -size=1G -numjobs=10 -runtime=100 -group_reporting -name=/data/iotest|egrep 'write: IOPS'

#网卡性能测试
服务端执行:
iperf3 -s 

客户端执行：
iperf3 -c 这里填服务端的ip -t90


#k8s资源管理
1、320G配置节点k8s-node: kubelet参数max-pod为110
2、512G配置节点k8s-node: kubelet参数max-pod为200
3、每容器requests-memory：180m
4、每容器requests-memory：2000Mi

#文件系统
1、kvm虚拟化相关用途的宿主服务器和虚拟机采用ext4文件系统，其它用途配置为xfs文件系统;
2、磁盘分区名数据盘默认为/data, 多块盘依此递增/data1 /data2;
3、物理机磁盘根分区设置为150G， swap分为128G，数据盘设全部磁盘空间;
4、虚拟机磁盘根分区设置为250G,  swap为空，禁止设置swap空间，数据盘设全部磁盘空间。
```

# 初始化系统

```
yum install  bash-completion lrzsz  nmap  nc  tree  htop iftop  net-tools ntpdate  lsof screen tcpdump conntrack ntp ipvsadm ipset jq sysstat libseccomp nmon iptraf mlocate strace nethogs iptraf iftop bridge-utils bind-utils nc nfs-tuils rpcbind dnsmasq python python-devel tree telnet git sshpass bind-utils -y

cd 
vim .bashrc
alias vimn="vim /etc/sysconfig/network-scripts/ifcfg-ens33"
```

##### 关闭防火墙

```
[root@moban ~]# systemctl disable firewalld
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
[root@moban ~]# iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
[root@moban ~]# iptables -P FORWARD ACCEPT
[root@moban ~]# firewall-cmd --state
not running
[root@moban ~]# setenforce 0
[root@moban ~]# sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

```
cat >> /etc/security/limits.conf <<EOF
# End of file
* soft nofile 65525
* hard nofile 65525
* soft nproc 65525
* hard nproc 65525
EOF
```

##### sysctl.conf

```shell
# Controls source route verification
net.ipv4.conf.default.rp_filter = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1

# Do not accept source routing
net.ipv4.conf.default.accept_source_route = 0

# Controls the System Request debugging functionality of the kernel
kernel.sysrq = 0

# Controls whether core dumps will append the PID to the core filename.
# Useful for debugging multi-threaded applications.
kernel.core_uses_pid = 1

# Controls the use of TCP syncookies
net.ipv4.tcp_syncookies = 1

# Disable netfilter on bridges.
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 0

# Controls the default maxmimum size of a mesage queue
kernel.msgmnb = 65536

# # Controls the maximum size of a message, in bytes
kernel.msgmax = 65536

# Controls the maximum shared segment size, in bytes
kernel.shmmax = 68719476736

# # Controls the maximum number of shared memory segments, in pages
kernel.shmall = 4294967296




# TCP kernel paramater
net.ipv4.tcp_mem = 786432 1048576 1572864
net.ipv4.tcp_rmem = 4096        87380   4194304
net.ipv4.tcp_wmem = 4096        16384   4194304
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_sack = 1

# socket buffer
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 262144
net.core.somaxconn = 20480
net.core.optmem_max = 81920


# TCP conn
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_syn_retries = 3
net.ipv4.tcp_retries1 = 3
net.ipv4.tcp_retries2 = 15

# tcp conn reuse
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 1


net.ipv4.tcp_max_tw_buckets = 20000
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_timestamps = 1 
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syncookies = 1

# keepalive conn
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.ip_local_port_range = 10001    65000

# swap
vm.overcommit_memory = 0
vm.swappiness = 10

#net.ipv4.conf.eth1.rp_filter = 0
#net.ipv4.conf.lo.arp_ignore = 1
#net.ipv4.conf.lo.arp_announce = 2
#net.ipv4.conf.all.arp_ignore = 1
#net.ipv4.conf.all.arp_announce = 2
```

##### limits.conf

```shell
*                soft    core            unlimited
*                hard    core            unlimited
*                soft    nproc           1000000
*                hard    nproc           1000000
*                soft    nofile          1000000
*                hard    nofile          1000000
*                soft    memlock         32000
*                hard    memlock         32000
*                soft    msgqueue        8192000
*                hard    msgqueue        8192000
```

##### vim配置

```shell
vim .vimrc


set ts=4
set expandtab
set ignorecase
set cursorline
set autoindent
autocmd BufNewFile *.sh exec ":call SetTitle()"
func SetTitle()
	if expand("%:e") == 'sh'
	call setline(1,"#!/bin/bash") 
	endif
endfunc
autocmd BufNewFile * normal G
```

##### 设置语言

英文语言，中文支持

```
#vi /etc/sysconfig/i18n``LANG="en_US.UTF-8"``SUPPORTED="zh_CN.UTF-8:zh_CN:zh"``SYSFONT="latarcyrheb-sun16"
```

# 各种记录

##### raid

```
RAID 0：通过把多块硬盘粘合成一个容量更大的硬盘组，提高了磁盘的性能和吞吐量。这种方式成本低，要求至少两个磁盘，但是没有容错和数据修复功能，因而只能用在对数据安全性要求不高的环境中。

RAID 1：也就是磁盘镜像，通过把一个磁盘的数据镜像到另一个磁盘上，最大限度地保证磁盘数据的可靠性和可修复性，具有很高的数据冗余能力，但磁盘利用率只有50%，因而，成本最高，多用在保存重要数据的场合。

RAID5：采用了磁盘分段加奇偶校验技术，从而提高了系统可靠性，RAID5读出效率很高，写入效率一般，至少需要3块盘。允许一块磁盘故障，而不影响数据的可用性。

RAID0+1：把RAID0和RAID1技术结合起来就成了RAID0+1，至少需要4个硬盘。此种方式的数据除分布在多个盘上外，每个盘都有其镜像盘，提供全冗余能力，同时允许一个磁盘故障，而不影响数据可用性，并具有快速读/写能力。

对于对写操作频繁而对数据安全性要求不高的应用，可以把磁盘做成RAID 0；
而对于对数据安全性较高，对读写没有特别要求的应用，可以把磁盘做成RAID 1；
对于对读操作要求较高，而对写操作无特殊要求，并要保证数据安全性的应用，可以选择RAID 5；
对于对读写要求都很高，并且对数据安全性要求也很高的应用，可以选择RAID10/01。
```

##### inodes耗尽处理

```
df -i

#查看文件最多的目录
for i in /*; do echo $i; find $i | wc -l; done

#删除大量文件
ls | xargs -n 1000 rm -rf 需要使用xargs命令，不然会删除失败


#定时任务
*/10 * * * * /tmp/test.sh >/dev/null 2>&1

find 目录 -type f -mtime +30 | xargs -n 1000 rm -f**
```


### **请问多线程与多进程的区别，在什么时候用线程或进程更合适？**
```
解答：
进程：
优点：多进程可以同时利用多个 CPU，能够同时进行多个操作。
缺点：耗费资源（创建一个进程重新开辟内存空间）。
进程不是越多越好，一般进程个数等于 cpu 个数。
线程：
优点：共享内存，尤其是进行 IO 操作（网络、磁盘）的时候（IO 操作很少用 cpu），可以使用多线
程执行并发操作。
缺点：抢占资源。
线程也不是越多越好，具体案例具体分析，切换线程关系到请求上下文切换耗时。
计算机中执行任务的最小单元：线程。


IO 密集型（不用 cpu）：多线程 (磁盘读写多)
计算密集型（用 cpu）：多进程 (计算多)
```
### **同步和异步**
```
异步 ： 写入 >内存(数据全部写入完成) >再从内存写入到硬盘   数据先全部写入内存，再写入硬盘


同步：数据写入>内存>硬盘    同时进行
```

