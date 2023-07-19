# 虚拟机操作命令

###### 1.1、查看运行的虚拟机

```
virsh list

查看所有的虚拟机（关闭和运行的，不包括摧毁的）
virsh list --all
```

###### 1.2、启动虚拟机

```
启动虚拟机
virsh start 虚拟机名称

虚拟机随物理机启动而启动
virsh autostart 虚拟机名称

取消虚拟机随物理机启动而启动
virsh autostart --disable 虚拟机名称
```

###### 1.3、连接虚拟机

```
virsh console 虚拟机名称
```

###### 1.4、退出虚拟机

```
快捷键： ctrl+]
```

###### 1.5、关闭虚拟机

```
virsh shutdown 虚拟机名称
\#前提虚拟机需要（安装acpid服务）
yum install -y acpid
/etc/init.d/acpid start
```

###### 1.6、暴力关闭虚拟机(断电)

```
virsh destroy 虚拟机名称
```

###### 1.7、彻底删除虚拟机

```
\#解除标记
virsh undefine 虚拟机名称
然后删除虚拟机存储所在的位置
```

###### 1.8、虚拟机扩容

```
找到要修改的虚拟机
virsh list --all |grep 189

编辑虚拟机
virsh edit qx-prod-ns-172-19-8-189

\---------------------------------------------------------------------------------------------------

 <name>qx-prod-ns-172-19-8-189</name>

 <uuid>688a6a6d-12ea-46b5-ad3c-df798958856b</uuid>

 <memory unit='KiB'>16777216</memory>

 <currentMemory unit='KiB'>16777216</currentMemory>

 <vcpu placement='static'>4</vcpu>

\----------------------------------------------------------------------------------------------------

修改完成后关闭虚拟机
virsh shutdown qx-prod-ns-172-19-8-189

再启动虚拟机
virsh start qx-prod-ns-172-19-8-189
```

###### 1.9、查看已使用ip地址

```
ansible all_hosts --list
```

###### 2.0、关闭虚拟机

```
virsh shutdown wp-xxl-job-ns-dev-172-19-0-182
```

###### 2.1、删除虚拟机

```
virsh undefine today-test-node-22
```

###### 2.2、查看vnc端口

```
virsh destroy 虚拟机名称
```

###### 2.3、kvm修改内存以及cpu

```
  <memory unit='KiB'>8388608</memory>
  <currentMemory unit='KiB'>4194304</currentMemory>
  <vcpu placement='static' current='4'>8</vcpu>



现行的内存为4G，最大可为8G，现行的cpu为4核，最大可行的为8核。
virsh setvcpus $hostname 5 --live 更改现行的cpu为5核
virsh setmem $hostname 8G --live 更改现行的内存为8G
```

###### 2.4 、虚拟机热加载硬盘

```
1.KVM宿主机上创建磁盘
/usr/bin/qemu-img create -f qcow2 /data/images/test-mysql-0-140-data1-500G.qcow2 500G

2.将磁盘关联具体kvm实例
/usr/bin/virsh attach-disk --domain test-mysql-0-140 --source /data/images/test-mysql-0-140-data1-500G.qcow2  --target vdb --targetbus virtio --driver qemu --subdriver qcow2 --sourcetype file --cache none --persistent

3.在实例里执行挂载硬盘
#注意挂载的磁盘名称   mkfs.xfs -f /dev/vdb
#注意目录是否已存在   mkdir -p /data1
/usr/sbin/parted -s /dev/sdb mkpart xfs 0% 100% && sleep 10  && mkfs.xfs -f /dev/sdb ;if [ $? -ne 0 ];then  exit 11;else echo ok;fi && sleep 5 &&  mkdir -p /data && sleep 5 && mount /dev/sdb /data && echo \"/dev/sdb /data xfs defaults 0 0 \" >>  /etc/fstab  

报错Error: /dev/vdb: unrecognised disk label
磁盘打标签
[root@hz-dev-ns-ClickHouse-172-19-0-247 ~]# parted /dev/vdb 
(parted) mklabel 
Partition Table: gpt
```

###### 2.5：关机后无法开机

```
 Could not connect: No such file or directory (g-io-error-quark, 1)
centos7的系统重启网卡遇到这个问题。
解决办法：
先 vim /etc/fstab
注销掉 /dev/vdb /dat1 xfs defaults 0 0类似这种的。
然后reboot
之后开始重新挂载
blkid  /dev/vdb
vim /etc/fstab
blkid  /dev/vdb>> /etc/fstab
vim /etc/fstab
mount -a

vim /etc/fstab
配置：
UUID="9fd363af-2367-40c6-b469-0c9aa35d30c2"  /data1               xfs defaults 1 1
在mount -a前改成像这样的。
重启机器
```

# KVM新建虚拟机

##### 1、手动创建虚拟机

###### 1.1：新建一块虚拟磁盘

新建一块虚拟机磁盘文件,并使用qcow2格式，能最大化的节省宿主机磁盘空间。

```shell
qemu-img create -f qcow2 /data/images/192.168.1.1.qcow2 150G
# -f 后的三个参数分别为：虚拟硬盘格式、虚拟硬盘存档路径及文件名、虚拟硬盘容量。因硬盘文件格式为qcow2，所以该文件创建之后实际大小只有17K，只会随着系统的安装和使用增加，最多可增大到150G。
```

###### 1.2：新建虚拟机，以CentOS7为例

首先需要将centos7镜像上传至/data/iso，然后执行以下命令

```shell
virt-install --name kvm-test --ram=8192 --arch=x86_64 --vcpus=4 --check-cpu --os-type=linux --os-variant='rhel7' -c /data/iso/CentOS-7-x86_64-DVD-1511.iso --disk path=/data/images/192.168.1.1.qcow2,device=disk,bus=virtio,format=qcow2 --bridge=br48 --vnc --noautoconsole --vnclisten=0.0.0.0
# --name	虚拟机名
# --ram		内存大小（MB）
# --vcpu	虚拟CPU个数
# --os-type	操作系统类型
# --os-variant	操作系统发行版，kvm中都–os-variant参数不支持 win10 所以在安装win10 系统都时候这个参数需要去掉
# -c		安装系统的镜像存放的目录
# --disk	虚拟硬盘文件存放目录和硬盘的相关参数
# --bridge	网卡模式，默认只有一块网卡
```

**若要安装centos6.5，则命令如下:**

```shell
virt-install --name 192.168.1.1 --ram=8192 --arch=x86_64 --vcpus=4 --check-cpu --os-type=linux --os-variant='rhel6' -c /data/iso/CentOS-6.5-x86_64-bin-DVD1.iso --disk path=/data/images/192.168.1.1.qcow2,device=disk,bus=virtio,size=150,format=qcow2 --bridge=br2 --vnc --noautoconsole --vnclisten=0.0.0.0
```

**若要安装ubuntu-14.04，则命令如下:**

```shell
virt-install --name 192.168.1.1 --ram=8192 --arch=x86_64 --vcpus=4 --check-cpu --os-type=linux --os-variant='ubuntuPrecise' -c /data/iso/ubuntu-14.04.3-server-amd64.iso --disk path=/data/images/192.168.1.1.qcow2,device=disk,bus=virtio,size=150,format=qcow2 --bridge=br2 --vnc --noautoconsole --vnclisten=0.0.0.0
```

**若要安装win10系统，则命令如下：**

```shell
virt-install --name win10_64_standard --ram=4096 --arch=x86_64 --vcpus=2 --check-cpu --cpu host --disk path=/data/images/win10_64_standard.qcow2,format=qcow2, --network bridge=br2 --os-type=windows -c /data/iso/cn_windows_10_enterprise_version_1607_updated_jul_2016_x64_dvd_9057083.iso --vnc --noautoconsole --vnclisten=0.0.0.0 --disk path=/data/iso/virtio-win-0.1.141_amd64.vfd
```

>  
>  

```
需要注意一点：
1.kvm中都–os-variant参数不支持 win10 所以在安装win10 系统都时候这个参数需要去掉

2.`--cpu host`这个选项是安装 windows10 专用的 , 因为 windows 不支持很多老的虚拟化的 CPU,

3.如果安装 windows10 使用了这个选项依然安装的时候蓝屏 , 就把这个选项改成 `--cpu core2duo` 就是使用 core2duo 的 cpu 配置 . 如果是老的 win2008,win2012 或者 linux, 可以不需要这个选项 .
```

##### 2、使用模板创建安装

###### 2.1：复制模板生成镜像文件，镜像文件名即虚拟机名

cd /data/images/

```shell
cp templates-centos7.5.qcow2  kvm-test.qcow2
```

###### 2.2：创建虚拟机

```shell
virt-install --name kvm-test --ram=8192 --arch=x86_64 --vcpus=4 --check-cpu --os-type=linux --os-variant='rhel7' --import --disk path=/data/images/kvm-test.qcow2,format=qcow2 --bridge=br0 --vnc --noautoconsole --vnclisten=0.0.0.0 --autostart

# --name  虚拟机名
# --ram  内存大小（MB）

# --vcpu  虚拟CPU个数
# --os-type	操作系统类型
# --os-type=linux
# --os-type=windows
# --os-variant	操作系统发行版
	ubuntu
		--os-variant='ubuntuPrecise'
	redhat/centos6
		--os-variant='rhel6'
	redhat/centos7
		--os-variant='rhel7'
	windows2008R2
		--os-variant='win2k8'
```

>  
>  

```
需要注意一点：
1.kvm中都–os-variant参数不支持 win10 所以在安装win10 系统都时候这个参数需要去掉

2.`--cpu host`这个选项是安装 windows10 专用的 , 因为 windows 不支持很多老的虚拟化的 CPU,

3.如果安装 windows10 使用了这个选项依然安装的时候蓝屏 , 就把这个选项改成 `--cpu core2duo` 就是使用 core2duo 的 cpu 配置 . 如果是老的 win2008,win2012 或者 linux, 可以不需要这个选项 .
```

##### 3、vnc连接虚拟机

###### 3.1：脚本查询运行的虚拟机所对应的vnc端口号

```shell
vim /root/show-vncport.sh 
#!/bin/bash 
domain=`virsh list | awk '{print $2}' | sed '1,2d'` 
for i in $domain 
do 
echo $i `virsh vncdisplay $i` 
done
```

###### 3.2：给与执行权限

```shell
chomd +x /root/show-vncport.sh
```

###### 3.3：然后执行，结果如下：

![](http://ops.oa.com/wp-content/uploads/2018/01/%E5%9B%BE%E7%89%871.png#id=XK6Jg&originHeight=90&originWidth=330&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

192.168.48.55对应的vnc端口是“2”，即5902，然后使用vnc客户端软件来连接。如下图，输入宿主机ip和vnc端口号59默认不用写，0也可省略。即脚本查询出的结果是多少就写多少。

![](http://ops.oa.com/wp-content/uploads/2018/01/%E5%9B%BE%E7%89%872.png#id=aWKMr&originHeight=195&originWidth=350&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

**连接进去之后就可以配置操作系统了。**

##### 4、增加硬盘

如果需要挂载磁盘则创建一块磁盘挂载到虚拟机内部，一般虚拟机无需创建磁盘

###### 4.1：先创建一块数据磁盘

```shell
qemu-img create -f qcow2 /data/images/kvm-test-data-500g.qcow2 500G

# -f 后的三个参数分别为：虚拟硬盘格式、虚拟硬盘存档路径及文件名、虚拟硬盘容量。因硬盘文件格式为qcow2，所以该文件创建之后实际大小只有17K，只会随着系统的安装和使用增加，最多可增大到150G。
```

**Windows：**

virsh edit kvm-test 找到下面这6行：kvm-test为虚拟机名称，具体的根据自己创建的虚拟机

```xml
<disk type='file' device='disk'>
    <driver name='qemu' type='qcow2'/>
    <source file='/data/images/kvm-test.qcow2'/>
    <target dev='hda' bus='ide'/>
    <address type='drive' controller='0' bus='0' target='0' unit='0'/>
</disk>
```

复制这6行修改如下，并粘贴在这6行下方，如下：

```xml
<disk type='file' device='disk'>
    <driver name='qemu' type='qcow2'/>
    <source file='/data/images/kvm-test.qcow2'/>
    <target dev='hda' bus='ide'/>
    <address type='drive' controller='0' bus='0' target='0' unit='0'/>
</disk>
    
<disk type='file' device='disk'>
    <driver name='qemu' type='qcow2' cache='none'/>
    <source file='/data/images/kvm-test-data-500g.qcow2'/>
    <target dev='hdb' bus='ide'/>
    <address type='drive' controller='0' bus='0' target='0' unit='1'/>
</disk>
```

>  
>  

```
注意：
1. dev='hda'  是硬盘盘符，新加的硬盘可以顺着顺序写 hdb hdc hdd之类的
2. unit='0' 是驱动类型的集线设备号码，新加的硬盘这个地方不能和系统盘冲突
```

修改完成后保存、退出，关机：virsh destroy kvm-test，然后virsh start kvm-test，启动后虚拟机中就可以识别到第二块硬盘了。

**Linux主机qcow2格式：**

virsh edit kvm-test 找到下面这6行

```shell
<disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none'/>
      <source file='/data/images/kvm-test.qcow2'/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </disk>
<disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none'/>
      <source file='/data/images/kvm-test-data-500g.qcow2'/>
      <target dev='vdb' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x09' function='0x0'/>
    </disk>
```

>  
>  

```
注意：
1. dev='vda'  是硬盘盘符，新加的硬盘可以顺着顺序写 hdb hdc hdd之类的
2. salt='0' 是驱动类型的集线设备号码，新加的硬盘这个地方不能和系统盘冲突
```

修改完成后保存、退出，关机：virsh destroy kvm-test，然后virsh start kvm-test，启动后虚拟机中就可以识别到第二块硬盘了。

##### 5、分区扩容

如果不是要进行分区扩容，那么将动态扩容增加的200G磁盘空间或者添加第二块硬盘新增的空间进行格式化，挂在分区即可。如果要进行分区扩容（默认只支持win2k8centos7.2），那么就需要进行如下操作：

###### **5.1：win2k8分区扩容**

之前说过只能扩容最后一个分区。这一步需要用到一个分区工具，地址：\hzfile\software\OS\Win7\Win7分区工具\PAServer.zip，在服务器上安装这个分区工具后就可以对最后一个目录进行动态分区。操作很简单，都是图形界面，这里就不再详述了。

只针对分区类型为LVM的系统。我司大部分虚拟机 分区类型均为ext4文件系统。

###### 5.2：CentOS7.2分区扩容

1.首先用fdisk来格式化新加的磁盘空间。格式化完成之后会生成一个 /dev/vda5或者/dev/vdb1，这时要确定需要扩容的分区是挂在在哪个目录。如果要扩容 / 分区，那么必须进入救援模式扩容，本次以扩容 /data 分区为列。

2.首先查看现有的pv卷，CentOS7.2默认只有一个pv卷和一个vg卷

###### ![](https://cdn.nlark.com/yuque/0/2020/png/695221/1580882857977-d67f7e13-1067-4a16-b191-9139657ae71a.png#id=OWign&originHeight=185&originWidth=457&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

###### 5.3：用/dev/vdb1创建pv卷并加入到vg卷的centos组

```
pvcreate /dev/vdb1
vgextend centos /dev/vdb1
```

###### 5.4：查看 lv卷，

总共有3组，分别对应的是 根分区，swap，data分区，用下面的命令把所有剩余的磁盘空间追加到 data分区所在的卷组下

```
lvextend -l +100%FREE /dev/centos/data
```

###### 5.5：执行成功后，重新挂在一下data目录

```
mount -a
```

###### 5.6：这是df -h 发现现实的 data分区的大小没有变，还要执行下面这个命令

```
xfs_growfs /dev/centos/data
```

再次执行df -h 的结果就正常了，到此磁盘扩容的操作就完成了。

###### ![](https://cdn.nlark.com/yuque/0/2020/png/695221/1580882932445-b91d7486-96f2-4fe5-823a-db05c55a6a0e.png#id=vmAsX&originHeight=785&originWidth=507&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

###### 3.CentOS7.4分区

1.首先用fdisk来格式化新加的磁盘空间。格式化完成之后会生成一个 /dev/vda5或者/dev/vdb1

```shell
# 查看分区
fdisk -l
# 对 vdb 进行分区
fdisk /dev/vdb
# 分区加载到分区表
partprobe
# 格式化为xfs
mkfs.xfs /dev/vdb1
```

2.创建挂载目录，并设置开机自动挂载

```shell
# 创建挂载目录
mkdir /data
# 挂载
echo "/dev/vdb1 /data xfs defaults 0 0" >> /etc/fstab
# 测试挂载是否成功
mount -a
```

##### 6、增加配置

###### 6.1：增加CPU和内存

步骤比较简单，也是修改虚拟机的配置文件，编辑虚拟机配置文件，可以从文件开头找到如下内容：

```xml
<memory unit='KiB'>16777216</memory>
  <currentMemory unit='KiB'>16777216</currentMemory>
  <vcpu placement='static'>8</vcpu>
```

修改这3行的参数即可，内存的单位是KB。

###### 6.2：增加网卡

同样是修改虚拟机配置文件，编辑虚拟机配置文件，找到如下行：

```xml
<interface type='bridge'>
      <mac address='52:54:00:60:ff:74'/>
      <source bridge='br1'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
```

复制6行，并在其下粘贴，修改

```xml
<interface type='bridge'>
 <mac address='52:54:00:60:ff:74'/>
 <source bridge='br1'/>
 <model type='virtio'/>
 <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
 </interface>
 
 <interface type='bridge'>
 <mac address='52:54:00:70:ff:74'/>
 <source bridge='br2'/>
 <model type='virtio'/>
 <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
 </interface>
```

```
注意：
1. address='52:54:00:60:ff:74'  是网卡的mac地址，新加网卡的mac地址可以自修改，但不能和原始的冲突
2. salt='0' 是驱动类型的集线设备号码，新加的硬盘这个地方不能和系统盘冲突 
3. 若改成slot='0x07'系统审核不通过，则修改为slot='0x09'

修改完成后保存、退出，关机：virsh destroy kvm-test，然后virsh start kvm-test，启动后虚拟机中就可以识别到第二块网卡了
```

###### 6.3：虚拟机设置千兆网卡

同样是修改虚拟机配置文件，编辑虚拟机配置文件，找到如下行：

```xml
<interface type='bridge'>
      <mac address='52:54:00:60:ff:74'/>
      <source bridge='br1'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>
```

添加如下红色标记行，改为 e1000，千兆以太网卡：

修改之后，网卡配置如下所示：

```xml
<interface type='bridge'>
      <mac address='52:54:00:60:ff:74'/>
      <source bridge='br1'/>
      <model type='virtio'/><!--去掉-->
      <model type='e1000'/><!--替换为这个-->
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>
```

保存，退出。修改完成后保存、退出，关机：virsh destroy kvm-test，然后virsh start kvm-test

```
注意事项：
检查虚拟机是否运行正常，running即正常

[root@test images]# virsh list --all
 Id   Name                                                        State
---------------------------------------------------------------------------
 1    kvm-test													  running
```
