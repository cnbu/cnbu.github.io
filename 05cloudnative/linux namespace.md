## 隔离性 - linux Namespace
Linux Namespace 是一种 Linux Kernel 提供的资源隔离方案：						 							 

-  系统可以为进程分配不同的 Namespace; 						
-  并保证不同的 Namespace 资源独立分配、进程彼此隔离，即不同的 Namespace 下的进程互不干扰 
#### pid namespace
```
不同用户的进程就是通过pid namespace 隔离开的，且不同namespace中可以有相同的pid
有了pid namespace，每个namespace中的pid 能够相互隔离
```
#### net namespace
```
网络隔离是通过net namespace实现，每个net namespace有独立的network derives，ip address，ip routing tables，/proc/net目录

docker 默认采用veth的方式将container中的虚拟网卡同host上的一个docker bridge：docker0链接中一起
```
#### ipc namespace
```
Container 中进程交互还是采用 linux 常见的进程间交互方法 (interprocess communication – IPC), 包 括常见的信号量、消息队列和共享内存。

container 的进程间交互实际上还是 host上 具有相同 Pid namespace 中的进程间交互，因此需要在 IPC 资源申请时加入 namespace 信息 - 每个 IPC 资源有一个唯一的 32 位 ID
```
#### mnt namespace
```
mnt namespace 允许不同 namespace 的进程看到的文件结构不同，这样每个 namespace 中的进程所看 到的文件目录就被隔离开了。
```
#### uts namespace
```
UTS(“UNIX Time-sharing System”) namespace允许每个 container 拥有独立的 hostname 和 domain name, 使其在网络上可以被视作一个独立的节点而非 Host 上的一个进程。
```
#### user namespace
```
每个 container 可以有不同的 user 和 group id, 也就是说可以在 container 内部用 container 内部的用户 执行程序而非 Host 上的用户。
```
### 常用命令
```
查看当前系统的 namespace: 
lsns –t <type>

查看某进程的 namespace: 
ls -la /proc/<pid>/ns/

进入某 namespace 运行命令: 
nsenter -t <pid> -n ip addr
```
## Cgroups
```
• Cgroups (Control Groups)是 Linux 下用于对一个或一组进程进行资源控制和监控的机制;
• 可以对诸如 CPU 使用时间、内存、磁盘 I/O 等进程所需的资源进行限制;
• 不同资源的具体管理工作由相应的 Cgroup 子系统(Subsystem)来实现 ;
• 针对不同类型的资源限制，只要将限制策略在不同的的子系统上进行关联即可 ;
• Cgroups 在不同的系统资源管理子系统中以层级树(Hierarchy)的方式来组织管理:每个 Cgroup 都可以 包含其他的子 Cgroup，因此子 Cgroup 能使用的资源除了受本 Cgroup 配置的资源参数限制，还受到父 Cgroup 设置的资源限制 。
```
