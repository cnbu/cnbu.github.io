# 一、service和iptables的关系
service 的代理是 kube-proxy
kube-proxy 运行在所有节点上，它监听 apiserver 中 service 和 endpoint 的变化情况，创建路由规则以提供服务 IP 和负载均衡功能。简单理解此进程是Service的透明代理兼负载均衡器，其核心功能是将到某个Service的访问请求转发到后端的多个Pod实例上，而kube-proxy底层又是通过iptables和ipvs实现的。
### iptables原理
Kubernetes从1.2版本开始，将iptables作为kube-proxy的默认模式。iptables模式下的kube-proxy不再起到Proxy的作用，其核心功能：通过API Server的Watch接口实时跟踪Service与Endpoint的变更信息，并更新对应的iptables规则，Client的请求流量则通过iptables的NAT机制“直接路由”到目标Pod。
### ipvs原理
IPVS在Kubernetes1.11中升级为GA稳定版。IPVS则专门用于高性能负载均衡，并使用更高效的数据结构（Hash表），允许几乎无限的规模扩张，因此被kube-proxy采纳为最新模式。
在IPVS模式下，使用iptables的扩展ipset，而不是直接调用iptables来生成规则链。iptables规则链是一个线性的数据结构，ipset则引入了带索引的数据结构，因此当规则很多时，也可以很高效地查找和匹配。
可以将ipset简单理解为一个IP（段）的集合，这个集合的内容可以是IP地址、IP网段、端口等，iptables可以直接添加规则对这个“可变的集合”进行操作，这样做的好处在于可以大大减少iptables规则的数量，从而减少性能损耗。
### kube-proxy ipvs和iptables的异同
iptables与IPVS都是基于Netfilter实现的，但因为定位不同，二者有着本质的差别：iptables是为防火墙而设计的；IPVS则专门用于高性能负载均衡，并使用更高效的数据结构（Hash表），允许几乎无限的规模扩张。
与iptables相比，IPVS拥有以下明显优势：

- 为大型集群提供了更好的可扩展性和性能；
- 支持比iptables更复杂的复制均衡算法（最小负载、最少连接、加权等）；
- 支持服务器健康检查和连接重试等功能；
- 可以动态修改ipset的集合，即使iptables的规则正在使用这个集合
```
iptables：

灵活 功能强大
规则便利匹配和更新，呈线性时延

ipvs：

工作中在内核态，有更好的性能
调度算法丰富：rr，wrr, lc,wlc,ip hash
```
# 二、k8s集群中分析service和kube-proxy
![image.png](https://cdn.nlark.com/yuque/0/2022/png/700245/1659892851015-9e927e8d-b27e-4623-b243-e3935c7b78e3.png#clientId=u98d18639-4268-4&from=paste&height=593&id=u9648cf3f&originHeight=593&originWidth=1129&originalType=binary&ratio=1&rotation=0&showTitle=false&size=75258&status=done&style=none&taskId=u163de3de-37a9-475b-8123-09fc1495d81&title=&width=1129)
**访问Service的请求，不论是Cluster IP+TargetPort的方式；还是用Node节点IP+NodePort的方式，都被Node节点的Iptables规则重定向到Kube-proxy监听Service服务代理端口。kube-proxy接收到Service的访问请求后，根据负载策略，转发到后端的Pod。**

# 三、kube-proxy 工作原理

kube-proxy当前实现了三种代理模式：userspace, iptables, ipvs
（1）userspace mode:  userspace是在用户空间，通过kube-proxy来实现service的代理服务
（2）iptables mode, 该模式完全利用内核iptables来实现service的代理和LB, 这是K8s在v1.2及之后版本默认模式

![image.png](https://cdn.nlark.com/yuque/0/2022/png/700245/1659893148030-50a6eb3b-c05f-4cf9-bcc1-ee6d5e4cce36.png#clientId=u98d18639-4268-4&from=paste&height=509&id=u7c4aa633&originHeight=509&originWidth=806&originalType=binary&ratio=1&rotation=0&showTitle=false&size=72389&status=done&style=none&taskId=u47214cf5-9952-4182-8b85-ee3d95b9244&title=&width=806)
API Server 对内（集群中的其他组件）和对外（用户）提供统一的 REST API，其他组件均通过 API Server 进行通信
Controller Manager、Scheduler、Kube-proxy 和 Kubelet 等均通过 API Server watch API 监测资源变化情况，并对资源作相应的操作
（3）ipvs mode.  在kubernetes 1.8以上的版本中，对于kube-proxy组件增加了除iptables模式和用户模式之外还支持ipvs模式。
kube-proxy ipvs 是基于 NAT 实现的，通过ipvs的NAT模式，对访问k8s service的请求进行虚IP到POD IP的转发。当创建一个 service 后，kubernetes 会在每个节点上创建一个网卡，同时帮你将 Service IP(VIP) 绑定上，此时相当于每个 Node 都是一个 ds，而其他任何 Node 上的 Pod，甚至是宿主机服务(比如 kube-apiserver 的 6443)都可能成为 rs；
与iptables、userspace 模式一样，kube-proxy 依然监听Service以及Endpoints对象的变化, 不过它并不创建反向代理, 也不创建大量的 iptables 规则, 而是通过netlink 创建ipvs规则，并使用k8s Service与Endpoints信息，对所在节点的ipvs规则进行定期同步; netlink 与 iptables 底层都是基于 netfilter 钩子，但是 netlink 由于采用了 hash table 而且直接工作在内核态，在性能上比 iptables 更优。其工作流程大体如下:
![image.png](https://cdn.nlark.com/yuque/0/2022/png/700245/1659893229839-4385b2f3-0760-43b4-8e3d-0bdc6069d645.png#clientId=u98d18639-4268-4&from=paste&height=351&id=u97157638&originHeight=351&originWidth=796&originalType=binary&ratio=1&rotation=0&showTitle=false&size=101585&status=done&style=none&taskId=uddc284d9-b5a5-4ef2-ab6b-0c761ec4f6d&title=&width=796)
**k8s的service和endpoine是如何关联和相互影响的？**
1、 api-server创建service对象，与service绑定的pod地址：称之为endpoints（kubectl get ep可以查看）
2、服务发现方面：kube-proxy监控service后端endpoint的动态变化，并且维护service和endpoint的映射关系
#### kubernetes服务注册（dns），服务发现是service
http://www.dockone.io/article/9936   有详细讲解
Kubernetes提供了两种方式进行服务发现, 即环境变量和DNS
（1）**环境变量**： 当你创建一个Pod的时候，kubelet会在该Pod中注入集群内所有Service的相关环境变量。**需要注意: **要想一个Pod中注入某个Service的环境变量，则必须Service要先比该Pod创建。这一点，几乎使得这种方式进行服务发现不可用。比如，一个ServiceName为redis-master的Service，对应的ClusterIP:Port为172.16.50.11:6379，则其对应的环境变量为:
REDIS_MASTER_SERVICE_HOST=172.16.50.11 REDIS_MASTER_SERVICE_PORT=6379 REDIS_MASTER_PORT=tcp://172.16.50.11:6379 REDIS_MASTER_PORT_6379_TCP=tcp://172.16.50.11:6379 REDIS_MASTER_PORT_6379_TCP_PROTO=tcp REDIS_MASTER_PORT_6379_TCP_PORT=6379 REDIS_MASTER_PORT_6379_TCP_ADDR=172.16.50.11 
（2）**DNS**：这是k8s官方强烈推荐的方式!!!  可以通过cluster add-on方式轻松的创建KubeDNS来对集群内的Service进行服务发现。

# 四、修改kube-proxy模式为ipvs（默认为iptables）

#### 1、加载内核模块
#lsmod | grep  ip_vs
ip_vs_sh               12688  0
ip_vs_wrr              12697  0
ip_vs_rr               12600  0
ip_vs                 145497  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          139264  7 ip_vs,nf_nat,nf_nat_ipv4,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_ipv4
libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack
#### 2、升级内核模块，ipvs对内核版本有要求
yum update -y
#### 
3、查看kube-proxy组件

kubectl get pod -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
kube-proxy-5clwf                           1/1     Running   29         78d
kube-proxy-ht25x                           1/1     Running   30         78d
kube-proxy-x9ml8                           1/1     Running   30         78d
kube-scheduler-master                      1/1     Running   49         78d
metrics-server-584b5f4754-7vlhm            1/1     Running   56         78d

#### 4、产看kube-proxy的配置文件configmaps

kubectl get configmaps  -n kube-system

NAME                                 DATA   AGE
calico-config                        4      78d
coredns                              1      78d
extension-apiserver-authentication   6      78d
kube-proxy                           2      78d
kubeadm-config                       2      78d
kubelet-config-1.18                  1      78d

#### 5、编辑kube-proxy的configmaps

kubectl edit  configmaps  kube-proxy -n kube-system
mode "ipvs"

删除旧的kube-proxy,kubelet自动重新拉起，应用ipvs
kubectl delete pod   kube-proxy-x9ml8   -n kube-system
pod "kube-proxy-x9ml8" deleted
[root[@master ](/master ) ~]# kubectl delete pod   kube-proxy-ht25x   -n kube-system 
pod "kube-proxy-ht25x" deleted

kubectl logs kube-proxy-4cnbn  -n kube-system

I1015 14:03:08.402147       1 node.go:136] Successfully retrieved node IP: 192.168.40.132
I1015 14:03:08.402259       1 server_others.go:259] Using ipvs Proxier.
W1015 14:03:08.402618       1 proxier.go:429] IPVS scheduler not specified, use rr by default

#### 6、安装ipvsadm
yum install -y ipvsadm.x86_64

ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
-> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  127.0.0.1:30564 rr  （轮询规则）
-> 10.244.104.36:8080           Masq    1      0          0
-> 10.244.166.147:8080          Masq    1      0          0
TCP  127.0.0.1:32618 rr
-> 10.244.104.20:80             Masq    1      0          0
TCP  172.17.0.1:30564 rr
-> 10.244.104.36:8080           Masq    1      0          0

#### 7、系统生成虚拟网卡kube-ipvs0
ip a

6: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
link/ether a2:09:a4:9d:fb:f8 brd ff:ff:ff:ff:ff:ff
inet 10.110.180.143/32 brd 10.110.180.143 scope global kube-ipvs0
valid_lft forever preferred_lft forever
inet 10.110.180.143/32专门用于接收svc web1的请求

#### 8、svc的流量被ipvs分发
kubectl get svc
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
web1          NodePort    10.110.180.143           80:30746/TCP   23h

curl  10.110.180.143

kubectl get ep
NAME          ENDPOINTS                                              AGE
kubernetes    192.168.40.132:6443                                    78d
web1          10.244.104.15:80,10.244.166.143:80,10.244.166.148:80   23h
ipvs捕捉到svc的流量

ipvsadm -L -n

TCP  10.110.180.143:80 rr
-> 10.244.104.15:80             Masq    1      0          0
-> 10.244.166.143:80            Masq    1      0          0
-> 10.244.166.148:80            Masq    1      0          1
