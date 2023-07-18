## **k8s 1.1410二进制高可用集群布署**

### **节点构造如下 :**
| 节点ip | 节点角色 | hostname |
| --- | --- | --- |
| 172.20.0.101 | node | k8s-test-master-1 |
| 172.20.0.102 | node | k8s-test-master-2 |
| 172.20.0.103 | node | k8s-test-master-3 |


##### 三个节点既是master也是node,etcd布署在master之上，节点配置主机名host ip 映射，ssh免密。

##### k8s客户端通过vip和k8s-api通讯，vip对接haproxy，haproxy负载三台api master:6443

### **集群网络结构：**
| 网络名称 | 网络范围 |
| --- | --- |
| 集群pod网络 | 10.12.0.0/14 |
| svc网络 | 10.16.0.0/14 |
| 物理网络 | 172.20.0.0/16 |
| HA-vip | 172.20.0.105/16 |


### **组件配置：**
| 系统 | 参数 |
| --- | --- |
| 系统 | centos7.7或7.9以上系统均可 |
| 内核版本 | 3.10.0-1062 以上 |
| docker-data数据盘 | ext4 |
| docker | 18-ce09-2 |
| flannel | v0.10.0 |
| etcd | v3.3.11 |
| Storage | Driver: overlay2 |
| Backing | Filesystem: extfs |
| Logging | Driver: journald |
| Cgroup | Driver: systemd |


### 一、所有节点升级内核，安装docker-ce-19.03.8-3.el7.x86_64

1.1    能连通外网可优先升级内核，如不能连接yum可不升级内核

```
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm ;yum --enablerepo=elrepo-kernel install  kernel-lt-devel kernel-lt -y

#查看默认启动顺序
awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg  

CentOS Linux (4.4.4-1.el7.elrepo.x86_64) 7 (Core)  
CentOS Linux (3.10.0-1062.el7.x86_64) 7 (Core)  
CentOS Linux (0-rescue-c52097a1078c403da03b8eddeac5080b) 7 (Core)

#默认启动的顺序是从0开始，新内核是从头插入（目前位置在0，而4.4.4的是在1），所以需要选择0。

grub2-set-default 0  

#重启
reboot

#检查内核，成功升级到4.4
uname -a
Linux bigdata5 4.4.104-1.el7.elrepo.x86_64 #1 SMP Tue Dec 5 12:46:32 EST 2017 x86_64 x86_64 x86_64 GNU/Linux
```

1.2 所有节点安装docker和相关软件

```
#安装yum工具
yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
  #安装yum 源
 yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

#安装docker版本以及依赖
yum install -y  ipvsadm ipset conntrack   nfs-utils socat ipset  ntpdate  docker-ce-19.03.8-3.el7.x86_64

#所有节点关闭SWAP
swapoff -a
```

##### 清除iptables

```
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

##### 开启sysctl.conf网络转发

```
net.ipv4.ip_forward = 1
```

##### 关闭sysctl.conf swap

```
vm.swappiness = 0
```

1.3 所有节点ntp同步时间

```bash
ntpdate ntp1.aliyun.com
```

### 二、准备 k8s-node、master、etcd、flanneld二进制文件

```
#### 注意所有的文件由master ingest01这台机下发，配置ssh信任所有机器
#### 下载目录为/root/
[root@ingest01 ~]# pwd
cd /root

wget https://dl.k8s.io/v1.14.1/kubernetes-server-linux-amd64.tar.gz

 wget https://storage.googleapis.com/etcd/v3.3.11/etcd-v3.3.11-linux-amd64.tar.gz

wget https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz
```

### 三、下发所有二进制文件

3.1 解压

```

tar xvf kubernetes-server-linux-amd64.tar.gz && tar xvf etcd-v3.3.11-linux-amd64.tar.gz && tar xvf flannel-v0.10.0-linux-amd64.tar.gz
```

3.2 创建node，master ,etcd所需的二进制目录并进行归类

```
mkdir -p  /root/kubernetes/server/bin/master
\cp  -f /root/kubernetes/server/bin/kubelet /root/kubernetes/server/bin/master/
\cp -f  /root/mk-docker-opts.sh /root/kubernetes/server/bin/master/
\cp -f /root/flanneld /root/kubernetes/server/bin/master/

\cp -f /root/kubernetes/server/bin/kube-* /root/kubernetes/server/bin/master/
\cp -f /root/kubernetes/server/bin/kubectl /root/kubernetes/server/bin/master/
\cp -f /root/etcd-v3.3.11-linux-amd64/etcd* /root/kubernetes/server/bin/master/
```

3.3 下发master 二进制文件

```
#ssh 免密
 ssh-keygen -t rsa

ssh-copy-id root@k8s-test-master-1
ssh-copy-id root@k8s-test-master-2
ssh-copy-id root@k8s-test-master-3
```

```
for master in k8s-test-master-1 k8s-test-master-2 k8s-test-master-3;do
	rsync  -avzP   /root/kubernetes/server/bin/master/ ${master}:/usr/local/bin/
done
```

3.4 每台机加载ipvs模块

```
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF



# 授权
chmod 755 /etc/sysconfig/modules/ipvs.modules 


# 加载模块
bash /etc/sysconfig/modules/ipvs.modules


# 查看加载
lsmod | grep -e ip_vs -e nf_conntrack_ipv4

# 输出如下:
-----------------------------------------------------------------------
nf_conntrack_ipv4      20480  0 
nf_defrag_ipv4         16384  1 nf_conntrack_ipv4
ip_vs_sh               16384  0 
ip_vs_wrr              16384  0 
ip_vs_rr               16384  0 
ip_vs                 147456  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          110592  2 ip_vs,nf_conntrack_ipv4
libcrc32c              16384  2 xfs,ip_vs
-----------------------------------------------------------------------
```

### 四、创建集群systemctl 启动服务service文件

4.1 创建服务归类文件夹

```
mkdir -p  /root/kubernetes/server/bin/{master-service,etcd-service,docker-service,ssl}
```

4.2 创建二进制服务所需的文件

```
#etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd
#User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/usr/local/bin/etcd \
  --name=etcd1 \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls=https://172.20.0.101:2380 \
  --listen-peer-urls=https://172.20.0.101:2380 \
  --listen-client-urls=https://172.20.0.101:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://172.20.0.101:2379 \
  --initial-cluster-token=k8s-etcd-cluster \
  --initial-cluster=etcd1=https://172.20.0.101:2380,etcd2=https://172.20.0.102:2380,etcd3=https://172.20.0.103:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target



#docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target docker-storage-setup.service
Wants=docker-storage-setup.service

[Service]
Type=notify
EnvironmentFile=/run/flannel/docker
Environment=GOTRACEBACK=crash
ExecReload=/bin/kill -s HUP $MAINPID
Delegate=yes
KillMode=process
ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT
ExecStart=/usr/bin/dockerd \
          $DOCKER_OPTS \
          $DOCKER_STORAGE_OPTIONS \
          --exec-opt native.cgroupdriver=systemd \
          --insecure-registry registry.hz.local \
          $DOCKER_NETWORK_OPTIONS \
          $DOCKER_DNS_OPTIONS \
          $INSECURE_REGISTRY
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=1min
Restart=on-abnormal

[Install]
WantedBy=multi-user.target

 
 #flanneld.service 
 
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service
[Service]
Type=notify
ExecStart=/usr/local/bin/flanneld \
-etcd-cafile=/etc/kubernetes/ssl/ca.pem \
-etcd-certfile=/etc/kubernetes/ssl/flanneld.pem \
-etcd-keyfile=/etc/kubernetes/ssl/flanneld-key.pem \
-etcd-endpoints=https://172.20.0.101:2379,https://172.20.0.102:2379,https://172.20.0.103:2379 \
-etcd-prefix=/kubernetes/network \
-iface=eth0
ExecStartPost=/usr/local/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure
[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
 
 #kubelet.service 

[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service
[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
--address=172.20.0.101 \
--hostname-override=k8s-test-master-1 \
--pod-infra-container-image=registry.hz.local/public/pause-amd64:3.1 \
--experimental-bootstrap-kubeconfig=/etc/kubernetes/ssl/bootstrap.kubeconfig \
--kubeconfig=/etc/kubernetes/ssl/kubelet.kubeconfig \
--cert-dir=/etc/kubernetes/ssl \
--hairpin-mode promiscuous-bridge \
--allow-privileged=true \
--serialize-image-pulls=false \
--logtostderr=true \
--housekeeping-interval=1m \
--cgroup-driver=systemd \
--runtime-cgroups=/systemd/system.slice \
--kubelet-cgroups=/systemd/system.slice \
--cluster_dns=10.16.0.2 \
--max-pods=180 \
--cluster_domain=cluster.local. \
--v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target

#--eviction-hard=memory.available<50m,nodefs.available<10% \
#--cgroup-driver=cgroupfs \
#--eviction-soft-grace-period=memory.available=60s \

 #kube-proxy.service 

[Unit]
Description=Kubernetes Kube-Proxy Server
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --bind-address=172.20.0.101 \
  --hostname-override=k8s-test-master-1 \
  --kubeconfig=/etc/kubernetes/ssl/kube-proxy.kubeconfig \
  --masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes

Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

 #kube-apiserver.service 

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/kube-apiserver \
--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
--advertise-address=0.0.0.0 \
--allow-privileged=true \
--apiserver-count=3 \
--audit-policy-file=/etc/kubernetes/ssl/audit-policy.yaml \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/var/log/kubernetes/audit.log \
--authorization-mode=Node,RBAC \
--bind-address=0.0.0.0 \
--secure-port=6443 \
--client-ca-file=/etc/kubernetes/ssl/ca.pem \
--enable-swagger-ui=true \
--event-ttl=1h \
--kubelet-https=true \
--insecure-bind-address=127.0.0.1 \
--insecure-port=8080 \
--service-cluster-ip-range=10.16.0.0/14 \
--service-node-port-range=300-50000 \
--enable-bootstrap-token-auth \
--token-auth-file=/etc/kubernetes/ssl/token.csv \
--tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
--tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
--client-ca-file=/etc/kubernetes/ssl/ca.pem \
--service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
--etcd-cafile=/etc/kubernetes/ssl/ca.pem \
--etcd-certfile=/etc/kubernetes/ssl/etcd.pem \
--etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \
--etcd-servers=https://172.20.0.101:2379,https://172.20.0.102:2379,https://172.20.0.103:2379 \
--v=1
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

#--ReadOnlyAPIDataVolumes=true\
[Install]
WantedBy=multi-user.target

 #kube-controller-manager.service 

[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
--address=127.0.0.1 \
--master=http://127.0.0.1:8080 \
--allocate-node-cidrs=true \
--service-cluster-ip-range=10.16.0.0/14 \
--cluster-cidr=10.12.0.0/14 \
--cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
--cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
--service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
--root-ca-file=/etc/kubernetes/ssl/ca.pem \
--cluster-name=kubernetes \
--leader-elect=true \
--v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target


 #kube-scheduler.service 

[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-scheduler \
--address=127.0.0.1 \
--master=http://127.0.0.1:8080 \
--leader-elect=true \
--v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
```

### 五、下发service文件

5.1 下发master所需的service文件

```
#注意更改service文件中的主机名和ip,每个节点不一样
for master in k8s-test-master-1 k8s-test-master-2  k8s-test-master-3;do
	rsync  -avzP   /root/kubernetes/server/bin/master-service/ ${master}:/lib/systemd/system/
done
```

### 六、创建集群认证证书文件,下发文件

6.1 生成文件

```
#安装 CFSSL

#直接使用二进制源码包安装

wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson_linux-amd64
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

export PATH=/usr/local/bin:$PATH


----------


**#创建CA 证书配置100年有效期*
cat >/root/kubernetes/server/bin/ssl/config.json  <<'HERE'
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
HERE

----------
#csr.json
cat >/root/kubernetes/server/bin/ssl/csr.json  <<'HERE'
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ],
        "ca": {
           "expiry": "876000h"
        }
}
HERE
#生成 CA 证书和私钥
cfssl gencert -initca csr.json | cfssljson -bare ca
2019/05/25 21:28:29 [INFO] signed certificate with serial number 411888594564637899082069619714763433002344045248
[root@k8s-test-master-1 ssl]# ls
ca.csr  ca-key.pem  ca.pem  config.json  csr.json

----------

#kube-proxy-csr.json
cat >/root/kubernetes/server/bin/ssl/kube-proxy-csr.json  <<'HERE'
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shenzhen",
      "L": "Shenzhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
HERE
--------------------- 

#创建admin 证书
cat >/root/kubernetes/server/bin/ssl/admin-csr.json  <<'HERE'
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
HERE
------

--
#生成 admin 证书和私钥

cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=config.json \
  -profile=kubernetes admin-csr.json | cfssljson -bare admin
#查看证书
[root@k8s-test-master-1 ssl]# ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem
[root@k8s-test-master-1 ssl]# 

#生成 kubernetes 配置文件
export KUBE_APISERVER="https://172.20.0.105:9443"

# 配置 kubernetes 集群
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER}
  
#配置客户端认证  
 kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --embed-certs=true \
  --client-key=admin-key.pem
  
  ----
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin

kubectl config use-context kubernetes

#创建k8s证书
#注意，此处需要将dns首ip、vip、代理负载均衡器ip、k8s-master节点的ip都填上
其中 172.20.0.105 为vip 即虚拟ip，在三台master之间进行飘移,通过haproxy负载三台apiserver
cat >/root/kubernetes/server/bin/ssl/kubernetes-csr.json  <<'HERE'

{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "172.20.0.101",
    "172.20.0.102",
    "172.20.0.103",
    "172.20.0.104",
    "172.20.0.105",
    "172.20.0.106",
    "10.16.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
HERE

#生成 kubernetes 证书和私钥
cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=config.json \
  -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
#kubelet 首次启动时向 kube-apiserver 发送 TLS Bootstrapping 请求，kube-apiserver 验证 kubelet 请求中的 token 是否与它配置的 token 一致，如果一致则自动为 kubelet生成证书和秘钥。

export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
echo "Tokne: ${BOOTSTRAP_TOKEN}"

# 创建 encryption-config.yaml 配置

cat > encryption-config.yaml <<EOF
kind: EncryptionConfiguration
apiVersion: apiserver.config.k8s.io/v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${BOOTSTRAP_TOKEN}
      - identity: {}
EOF


# 官方说明 https://kubernetes.io/docs/tasks/debug-application-cluster/audit/


[root@k8s-test-master-1 ssl]# vim  token.csv 
1ebb0f58c2f03b8eaf7869a9bf04177e,kubelet-bootstrap,10001,"system:kubelet-bootstrap"


# 生成高级审核配置文件

cat >> audit-policy.yaml <<EOF
# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
EOF

#创建flanneld证书
cat >/root/kubernetes/server/bin/ssl/flanneld-csr.json  <<'HERE'
{
  "CN": "flanneld",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN", 
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
HERE

#生成flannel密钥
cfssl gencert -ca=ca.pem \
   -ca-key=ca-key.pem \
   -config=config.json \
   -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld

#创建etcd证书
cat >/root/kubernetes/server/bin/ssl/etcd-csr.json  <<'HERE'
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "172.20.0.101",
    "172.20.0.102",
    "172.20.0.103"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

HERE




#生成etcd密钥
cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=config.json \
  -profile=kubernetes etcd-csr.json | \
  cfssljson -bare etcd
  
ls etcd*
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem

#查看证书
cfssl-certinfo -cert etcd.pem
---------

----------
```

6.2  生成通用证书以及kubeconfig

```
#进入ssl目录
cd /root/kubernetes/server/bin/ssl/

cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF

----------

echo "Create kubelet bootstrapping kubeconfig..."
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig


----------

kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
 
----------


kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig


----------

kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

----------


echo "Create kube-proxy kubeconfig..."
cfssl gencert -ca=ca.pem \
   -ca-key=ca-key.pem \
   -config=config.json \
   -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
   
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER}  \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

----------


kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

----------

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
--------------------- 

----------


# 生成集群管理员admin kubeconfig配置文件供kubectl调用
# admin set-cluster
 kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem\
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=./kubeconfig


----------


# admin set-credentials
 kubectl config set-credentials kubernetes-admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=./kubeconfig


----------


# admin set-cluster
 kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=./kubeconfig


----------


# admin set-credentials
 kubectl config set-credentials kubernetes-admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=./kubeconfig


----------


# admin set-context
 kubectl config set-context kubernetes-admin@kubernetes \
    --cluster=kubernetes \
    --user=kubernetes-admin \
    --kubeconfig=./kubeconfig


----------


# admin set default context
 kubectl config use-context kubernetes-admin@kubernetes \
    --kubeconfig=./kubeconfig



#复制生成kubelet.kubeconfig 

cp kubeconfig   kubelet.kubeconfig
---------------------
```

6.3 下发证书文件至所有节点

```
#创建ssl 以及工作目录文件夹
for node in  k8s-test-master-1 k8s-test-master-2 k8s-test-master-3;do
    ssh ${node} "mkdir -p /etc/kubernetes/ssl/ ; mkdir -p /var/lib/etcd ;mkdir -p /var/lib/kubelet  ;mkdir -p /var/lib/kube-proxy "
done

----------

#下发文件
for ssl in k8s-test-master-1 k8s-test-master-2 k8s-test-master-3;do
    rsync  -avzP   /root/kubernetes/server/bin/ssl/  ${ssl}:/etc/kubernetes/ssl/
done

----------


----------
```

#### 七、布署haproxy/keepalived

haproxy和keepalived布署在三个master上，复用节点。

keepalived 用于生成vip在三台master节点之间飘移。

haproxy代理三台master节点的6443端口，haproxy端口为9443。

7.1 三个节点安装haproxy和keepalived

```bash
yum install haproxy  keepalived -y
```

7.2 准备haproxy配置文件

```bash
grep -v ^# /etc/haproxy/haproxy.cfg
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     65535
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 65535




frontend k8s-api  *:9443
	mode tcp
	default_backend	k8s-api
backend k8s-api
        fullconn 65535
	mode tcp
	balance	roundrobin
	server  k8s-test-master-1 172.20.0.101:6443 check
	server  k8s-test-master-2 172.20.0.102:6443 check
	server  k8s-test-master-3 172.20.0.103:6443 check

listen admin_stats
bind *:9444
stats enable
mode http 
log global
stats uri /stats
stats realm Haproxy\ Statistics
stats auth admin:admin
stats admin if TRUE
stats refresh 30s
```

7.3 准备keepalived配置文件

其中 state MASTER   如果配置主从，从服务器改为BACKUP即可

virtual_ipaddress 定义 vip地址

virtual_router_id 全局唯 一

主服务器设为priority  100

从服务器设置小于100

数值越大优先级越高

```bash
[root@k8s-test-master-1 tmp]# cat /etc/keepalived/keepalived.conf 
! Configuration File for keepalived  
global_defs {  
    notification_email {   
        hz19033928@qq.com   
    }   
    notification_email_from admin@test.com  
    smtp_server 127.0.0.1  
    smtp_connect_timeout 30  
    router_id LVS_MASTER  
}  

vrrp_script check_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 3
}  

vrrp_instance VI_1 {  
    state MASTER  # 如果配置主从，从服务器改为BACKUP即可
    interface eth0
    virtual_router_id 60  
    priority 100  # 从服务器设置小于100的数即可
    advert_int 1  
    authentication {  
        auth_type PASS  
        auth_pass 1111  
    }  
    virtual_ipaddress {  
        172.20.0.105/16
    }

    track_script {   
        check_haproxy
    }
}
```

7.4 准备keepalived主备切换脚本

```bash
[root@k8s-test-master-1 tmp]# cat /etc/keepalived/check_haproxy.sh 
#!/bin/bash

flag=$(systemctl status haproxy &> /dev/null;echo $?)

if [[ $flag != 0 ]];then
        echo "haproxy is down,close the keepalived"
        systemctl stop keepalived
fi
```

7.5 下发 keepalived 和haproxy文件至另外两台master

```bash
rsync -avz /etc/keepalived/ 172.20.0.102:/etc/keepalived/
rsync -avz /etc/haproxy/ 172.20.0.102:/etc/haproxy/

rsync -avz /etc/keepalived/ 172.20.0.103:/etc/keepalived/
rsync -avz /etc/haproxy/ 172.20.0.103:/etc/haproxy/

#注意修改 keepalived.config里面的
state MASTER  # 如果配置主从，从服务器改为BACKUP
```

7.6 修改三台机器keepalived.service文件，改为依赖haproxy启动之后启动。

添加Requires=haproxy.service

```bash
[root@k8s-test-master-2 ~]# cat /lib/systemd/system/keepalived.service 
[Unit]
Description=LVS and VRRP High Availability Monitor
After=syslog.target network-online.target
Requires=haproxy.service

[Service]
Type=forking
PIDFile=/var/run/keepalived.pid
KillMode=process
EnvironmentFile=-/etc/sysconfig/keepalived
ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

7.7 启动keepalived 和haproxy服务

```
systemctl enable haproxy keepalived
systemctl daemon-reload
systemctl start haproxy keepalived
```

7.7 验证服务

访问任意一台机

[http://172.20.0.101:9444/stats](http://172.20.0.101:9444/stats)

[http://172.20.0.102:9444/stats](http://172.20.0.102:9444/stats)

[http://172.20.0.103:9444/stats](http://172.20.0.103:9444/stats)

admin admin

#### 八、启动k8s服务，验证服务

###### 注意启动之前确认配置文件修改无误

8.1 启动 etcd 节点服务

```
#启动etcd集群

for node in k8s-test-master-1 k8s-test-master-2 k8s-test-master-3;do
    ssh ${node} "systemctl daemon-reload && systemctl start etcd && systemctl enable etcd"
done


----------


#检查集群健康
etcdctl --endpoints=https://172.20.0.101:2379,https://172.20.0.102:2379,https://172.20.0.103:2379 \
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        cluster-health
member 89146aff54c40243 is healthy: got healthy result from https://172.20.0.102:2379
member a04ec76ba9fd22b3 is healthy: got healthy result from https://172.20.0.103:2379
member bb09864fc5c03bd3 is healthy: got healthy result from https://172.20.0.101:2379
cluster is healthy


---------
#设置集群网络范围

  etcdctl --endpoints=https://172.20.0.101:2379,https://172.20.0.102:2379,https://172.20.0.103:2379 \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  mkdir /kubernetes/network


----------


etcdctl --endpoints=https://172.20.0.101:2379,https://172.20.0.102:2379,https://172.20.0.103:2379 \
  --ca-file=/etc/kubernetes/ssl/ca.pem\
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  mk /kubernetes/network/config '{ "Network": "10.12.0.0/14", "Backend": { "Type": "host-gw" }}'
  
----------
```

8.2 启动master节点服务

```

for node in  k8s-test-master-1 k8s-test-master-2 k8s-test-master-3;do
    ssh ${node} "systemctl daemon-reload && systemctl start flanneld docker kube-apiserver kube-controller-manager  kube-scheduler   "
done
```

8.3 验证集群

```bash
[root@k8s-test-master-1 ssl]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
[root@k8s-test-master-1 ssl]#
```

8.31 验证kube-controller-manager集群高可用

kube-controller-manager leader位于k8s-test-master-2 节点

```bash
[root@k8s-test-master-1 tmp]# kubectl -n kube-system get ep kube-controller-manager -o yaml
apiVersion: v1
kind: Endpoints
metadata:
 annotations:
   control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"k8s-test-master-2_abb0a9c0-9e1e-11e9-9d3e-525400e3801e","leaseDurationSeconds":15,"acquireTime":"2019-07-04T05:43:59Z","renewTime":"2019-07-04T07:25:45Z","leaderTransitions":11}'
 creationTimestamp: "2019-05-26T07:21:11Z"
 name: kube-controller-manager
 namespace: kube-system
 resourceVersion: "6271423"
 selfLink: /api/v1/namespaces/kube-system/endpoints/kube-controller-manager
 uid: d69f6d3b-7f86-11e9-9353-5254002eed3a
```

8.32 验证keeplived vip飘移情况

查看vip 172.20.0.105位于k8s-test-master-1 上，关闭systemctl stop haproxy

```bash
[root@k8s-test-master-1 tmp]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:2e:ed:3a brd ff:ff:ff:ff:ff:ff
    inet 172.20.0.101/16 brd 172.20.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 172.20.0.105/16 scope global secondary eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe2e:ed3a/64 scope link 
       valid_lft forever preferred_lft forever
```

查看vip已切换到k8s-test-master-2

```
[root@k8s-test-master-2 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:e3:80:1e brd ff:ff:ff:ff:ff:ff
    inet 172.20.0.102/16 brd 172.20.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 172.20.0.105/16 scope global secondary eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fee3:801e/64 scope link 
       valid_lft forever preferred_lft forever
```

当master1上haproxy和keepalived启动后，vip会优先绑定在master1上

8.4 通过认证

```
# 在master机器上执行，授权kubelet-bootstrap角色
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap

8.5 启动node组件服务
for node in  k8s-test-master-1 k8s-test-master-2 k8s-test-master-3;do
    ssh ${node} "systemctl daemon-reload && systemctl start kubelet kube-proxy  "
done


#通过所有集群认证
[root@k8s-test-master-1 ssl]# kubectl get csr
NAME                                                   AGE   REQUESTOR           CONDITION
node-csr-F6AAn8zHrJMrjaYL26PqbhO9URc5njIn1FQUhIWNwi8   18m   kubelet-bootstrap   Approved,Issued


kubectl get csr | awk '/Pending/ {print $1}' | xargs kubectl certificate approve

#检查node Ready
[root@k8s-test-master-1 ssl]# kubectl get nodes
NAME                STATUS   ROLES    AGE   VERSION
k8s-test-master-1   Ready    <none>   14m   v1.14.1
```

九、布署core-dns

vim   coredns.yaml

把里面$DNS_SERVER_IP   改为你SVC 第二位网段IP 10.16.0.2

替换 kubernetes $DNS_DOMAIN  为kubernetes cluster.local.

create -f  coredns.yaml

```
# Warning: This is a file generated from the base underscore template file: coredns.yaml.base

apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes $DNS_DOMAIN in-addr.arpa ip6.arpa {
            pods insecure
            upstream
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  # replicas: not specified here:
  # 1. In order to make Addon Manager do not reconcile this replicas parameter.
  # 2. Default is 1.
  # 3. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      nodeSelector:
        beta.kubernetes.io/os: linux
      containers:
      - name: coredns
        image: registry.hz.local/public/coredns:1.3.1
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.16.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
```

#### 十、布署dashboard

下载dashboard的yaml

```
wget https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml
```

vim kubernetes-dashboard.yaml

替换里面的镜像为你自己的地址

```
image: registry.hz.local/public/kubernetes-dashboard-amd64:v1.10.1
#没有镜像就用这个： registry.cn-beijing.aliyuncs.com/minminmsn/kubernetes-dashboard:v1.10.1
```

修改service部份为 nodeport

```
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  ports:
  - nodePort: 6357
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort
```

创建密钥，默认创建有的有问题

```
mkdir -p certs
```

生成证书

```
openssl req -nodes -newkey rsa:2048 -keyout certs/dashboard.key -out certs/dashboard.csr -subj "/C=/ST=/L=/O=/OU=/CN=kubernetes-dashboard"
```

有效期

```
openssl x509 -req -sha256 -days 36500 -in certs/dashboard.csr -signkey certs/dashboard.key -out certs/dashboard.crt
```

创建secrets密钥

```
kubectl create secret generic kubernetes-dashboard-certs --from-file=certs -n kube-system
```

创建admin token

```
kubectl create -f admin-token.yaml

cat admin-token.yaml

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
    roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
```

查看token

```
kubectl describe secret/$(kubectl get secret -nkube-system |grep admin|awk '{print $1}') -n kube-system
```

创建服务

```
kubectl create -f kubernetes-dashboard.yaml
```

登陆dashboard [https://172.20.0.101:6357](https://172.20.0.101:6357)

输入令排：token
