### pod调度到gpu机器

```yaml
通过 `nodeSelector` 调度到含有 `gpu=true` 标签的节点上


apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  # 调度到含有 gpu=true 的节点上
  nodeSelector:
    gpu: "true"
  containers:
  - image: luksa/kubia
    name: kubia
    
    
注意：含有该标签的节点可能有多个，届时将选择其中一个。候选节点最好是一个集合，避免单个节点故障会造成服务不可用
>>>>>>> 8f6ea10f5c4005b46d9815cb98001ae9e810bb77
```

### 系统优化

vi /etc/security/limits.conf

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

echo "vm.swappiness = 0">> /etc/sysctl.conf
swapoff -a && swapon -a
sysctl -p  (执行这个使其生效，不用重启)
```




### 系统前置

```shell
#配置hosts
cat /etc/hosts
10.151.30.57 master
10.151.30.62 node01

#关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

#禁用SELINUX
setenforce 0
cat /etc/selinux/config
SELINUX=disabled

#创建/etc/sysctl.d/k8s.conf文件，添加如下内容
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1

#执行如下命令使修改生效
modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf


#添加ipvs模块
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
 #!/bin/bash 
 modprobe -- ip_vs 
 modprobe -- ip_vs_rr 
 modprobe -- ip_vs_wrr 
 modprobe -- ip_vs_sh 
 modprobe -- nf_conntrack_ipv4 
EOF
```

###  安装docker

```shell
#安装必要软件
yum install -y yum-utils device-mapper-persistent-data lvm2 wget

#更新yum源
wget -O /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/centos/docker-ce.repo
sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo

#查看版本列表
yum list docker-ce.x86_64  --showduplicates | sort -r

#安装docker并设置开机自启
yum -y install docker-ce-19.03.14 docker-ce-cli-19.03.14
systemctl start docker && systemctl enable docker
```
#### 修改docker配置

```
cat > /etc/docker/daemon.json <<EOF
{
    "oom-score-adjust": -1000,
    "data-root": "/data/docker",
    "log-driver": "json-file",
    "log-opts": {
    "max-size": "100m",
    "max-file": "3"
    },
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 10,
    "registry-mirrors": ["https://7bezldxe.mirror.aliyuncs.com"],
    "storage-driver": "overlay2",
    "storage-opts": [
    "overlay2.override_kernel_check=true"
    ]，
    "live-restore": true,
    "exec-opts": ["native.cgroupdriver=systemd"]
}

```

### 安装k8s

```shell
#添加k8s的yum源
vim /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=kubernetes
baseurl=https://mirrors.tuna.tsinghua.edu.cn/kubernetes/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=0

#master安装kubeadm  kubelet kubectl
#node安装kubeadm  kubelet
yum install -y kubeadm-1.18.8 kubelet-1.18.8 kubectl-1.18.8
systemctl enable kubelet.service

#初始化集群
kubeadm init --apiserver-advertise-address=10.0.0.11 --control-plane-endpoint=10.0.0.11 --apiserver-bind-port=6443 --kubernetes-version=v1.18.8 --pod-network-cidr=10.100.0.0/16 --service-cidr=10.200.0.0/16 --service-dns-domain=changcheng.local --image-repository registry.aliyuncs.com/google_containers


kubeadm join 192.168.77.51:6443 --token lf1gfr.3k0trv3uo4obaf9d \
    --discovery-token-ca-cert-hash sha256:b3c33a8eda3f142cc43cda23520b4c767e5a36c4df30695a2231e4ffb30b3d9a \
    --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.77.51:6443 --token lf1gfr.3k0trv3uo4obaf9d \
    --discovery-token-ca-cert-hash sha256:b3c33a8eda3f142cc43cda23520b4c767e5a36c4df30695a2231e4ffb30b3d9a 
    
    
    
#添加--certificate-key  
master01上执行 kubeadm init phase upload-certs --upload-certs

#节点加入集群
kubeadm join 192.168.77.51:6443 --token f78e73.celogie7ty7zcq9p \
    --discovery-token-ca-cert-hash sha256:faaf2a103f2a59c60a4ee92645bdc1bb1e5e09116181b852401ac379eb1f2993 \
    --control-plane --certificate-key 946d00f3947aec0da74e08bdc2ec5dc141c14fcbdcc9e7bbd1a9c2f0651ca11e

[root@k8s-master01 ~]# kubectl get pod -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
net-test1   1/1     Running   0          27s   10.100.1.2   k8s-node01   <none>           <none>
net-test2   1/1     Running   0          16s   10.100.2.2   k8s-node02   <none>           <none>





kubectl get node
报错
the connection to the server localhost:8080 was refused - did you specify the right host or port


具体根据情况，此处记录linux设置该环境变量
方式一：编辑文件设置
	   vim /etc/profile
	   在底部增加新的环境变量 export KUBECONFIG=/etc/kubernetes/admin.conf
方式二:直接追加文件内容
	echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /etc/profile

	source /etc/profile
```

#### kube-haproxy开启ipvs
```
#修改ConfigMap的kube-system/kube-proxy中的config.conf，mode: "ipvs"：

[root@k8s-master01 ~]# kubectl edit cm kube-proxy -n kube-system
 
#重启kube-proxy pod
[root@k8s-master01 ~]#  kubectl get pod -n kube-system | grep kube-proxy | awk '{system("kubectl delete pod "$1" -n kube-system")}'
pod "kube-proxy-4skt5" deleted
pod "kube-proxy-fxjl5" deleted

#查看Kube-proxy pod状态
[root@k8s-master01 ~]# kubectl get pod -n kube-system | grep kube-proxy
kube-proxy-7vg6s                           1/1     Running   0          82s
kube-proxy-dtpvd                           1/1     Running   0          2m2s

 
#查看是否开启了ipvs
[root@k8s-master01 ~]# kubectl logs kube-proxy-ssv94 -n kube-system
I0727 02:23:52.411755       1 server_others.go:170] Using ipvs Proxier.
W0727 02:23:52.412270       1 proxier.go:395] clusterCIDR not specified, unable to distinguish between internal and external traffic
W0727 02:23:52.412302       1 proxier.go:401] IPVS scheduler not specified, use rr by default
I0727 02:23:52.412480       1 server.go:534] Version: v1.15.1
I0727 02:23:52.427788       1 conntrack.go:52] Setting nf_conntrack_max to 131072
I0727 02:23:52.428163       1 config.go:187] Starting service config controller
I0727 02:23:52.428199       1 config.go:96] Starting endpoints config controller
I0727 02:23:52.428221       1 controller_utils.go:1029] Waiting for caches to sync for endpoints config controller
I0727 02:23:52.428233       1 controller_utils.go:1029] Waiting for caches to sync for service config controller
I0727 02:23:52.628536       1 controller_utils.go:1036] Caches are synced for service config controller
I0727 02:23:52.628636       1 controller_utils.go:1036] Caches are synced for endpoints config controller



[root@k8s-master01 ~]# kubectl logs kube-proxy-ssv94 -n kube-system  | grep "ipvs"
I0727 02:23:52.411755       1 server_others.go:170] Using ipvs Proxier.
```
#### 查看ipvs状态
```
[root@k8s-master data]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.200.0.1:443 rr
  -> 10.0.0.11:6443               Masq    1      0          0         
TCP  10.200.0.10:53 rr
  -> 10.100.58.193:53             Masq    1      0          0         
  -> 10.100.235.193:53            Masq    1      0          0         
TCP  10.200.0.10:9153 rr
  -> 10.100.58.193:9153           Masq    1      0          0         
  -> 10.100.235.193:9153          Masq    1      0          0         
UDP  10.200.0.10:53 rr
  -> 10.100.58.193:53             Masq    1      0          0         
  -> 10.100.235.193:53            Masq    1      0          0  
```
#### 查看集群状态
```
[root@k8s-master data]# kubectl get all -A
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
kube-system   pod/calico-kube-controllers-5f579f6866-v678f   1/1     Running   0          14m
kube-system   pod/calico-node-5wzjs                          1/1     Running   0          14m
kube-system   pod/calico-node-97fkt                          1/1     Running   0          14m
kube-system   pod/calico-node-lwhwg                          1/1     Running   0          14m
kube-system   pod/coredns-7ff77c879f-78wgk                   1/1     Running   0          30m
kube-system   pod/coredns-7ff77c879f-jjgl2                   1/1     Running   0          30m
kube-system   pod/etcd-k8s-master                            1/1     Running   1          30m
kube-system   pod/kube-apiserver-k8s-master                  1/1     Running   1          30m
kube-system   pod/kube-controller-manager-k8s-master         1/1     Running   1          30m
kube-system   pod/kube-proxy-b2j48                           1/1     Running   0          5m16s
kube-system   pod/kube-proxy-cd6m8                           1/1     Running   0          5m32s
kube-system   pod/kube-proxy-jbwcl                           1/1     Running   0          5m22s
kube-system   pod/kube-scheduler-k8s-master                  1/1     Running   1          30m

NAMESPACE     NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.200.0.1    <none>        443/TCP                  30m
kube-system   service/kube-dns     ClusterIP   10.200.0.10   <none>        53/UDP,53/TCP,9153/TCP   30m

NAMESPACE     NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/calico-node   3         3         3       3            3           kubernetes.io/os=linux   14m
kube-system   daemonset.apps/kube-proxy    3         3         3       3            3           kubernetes.io/os=linux   30m

NAMESPACE     NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/calico-kube-controllers   1/1     1            1           14m
kube-system   deployment.apps/coredns                   2/2     2            2           30m

NAMESPACE     NAME                                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/calico-kube-controllers-5f579f6866   1         1         1       14m
kube-system   replicaset.apps/coredns-7ff77c879f                   2         2         2       30m
```

### flannel组件

```yaml
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.flannel.unprivileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
    apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
spec:
  privileged: false
  volumes:
  - configMap
  - secret
  - emptyDir
  - hostPath
  allowedHostPaths:
  - pathPrefix: "/etc/cni/net.d"
  - pathPrefix: "/etc/kube-flannel"
  - pathPrefix: "/run/flannel"
  readOnlyRootFilesystem: false
  # Users and groups
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  # Privilege Escalation
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  # Capabilities
  allowedCapabilities: ['NET_ADMIN', 'NET_RAW']
  defaultAddCapabilities: []
  requiredDropCapabilities: []
  # Host namespaces
  hostPID: false
  hostIPC: false
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  # SELinux
  seLinux:
    # SELinux is unused in CaaSP
    rule: 'RunAsAny'
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups: ['extensions']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames: ['psp.flannel.unprivileged']
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.100.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: easzlab/flannel:v0.13.0-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: easzlab/flannel:v0.13.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
```

### 拷贝密钥脚本

```shell
#前置
yum install -y sshpass


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

# yaml 编写

### 获取yaml资源帮助

```shell
#获取yaml文件编写需要的内容
kubectl  explain  [资源名字]

#查看创建pod需要的信息
kubectl explain pods

#查看pod中spec需要的信息
kubectl explain pods.spec
```

### yaml例子

```yaml
# yaml格式的pod定义文件完整内容：
apiVersion: v1       #必选，版本号，例如v1
kind: Pod       #必选，Pod
metadata:       #必选，元数据
  name: string       #必选，Pod名称
  namespace: string    #必选，Pod所属的命名空间
  labels:      #自定义标签
    - name: string     #自定义标签名字
  annotations:       #自定义注释列表
    - name: string
spec:         #必选，Pod中容器的详细定义
  containers:      #必选，Pod中容器列表
  - name: string     #必选，容器名称
    image: string    #必选，容器的镜像名称
    imagePullPolicy: [Always | Never | IfNotPresent] #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像，否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]    #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]     #容器的启动命令参数列表
    workingDir: string     #容器的工作目录
    volumeMounts:    #挂载到容器内部的存储卷配置
    - name: string     #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string    #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean    #是否为只读模式
    ports:       #需要暴露的端口库号列表
    - name: string     #端口号名称
      containerPort: int   #容器需要监听的端口号
      hostPort: int    #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string     #端口协议，支持TCP和UDP，默认TCP
    env:       #容器运行前需设置的环境变量列表
    - name: string     #环境变量名称
      value: string    #环境变量的值
    resources:       #资源限制和请求的设置
      limits:      #资源限制的设置
        cpu: string    #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string     #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:      #资源请求的设置
        cpu: string    #Cpu请求，容器启动的初始可用数量
        memory: string     #内存清楚，容器启动的初始可用数量
    livenessProbe:     #对Pod内个容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:      #对Pod容器内检查方式设置为exec方式
        command: [string]  #exec方式需要制定的命令或脚本
      httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0  #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0   #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged:false
    restartPolicy: [Always | Never | OnFailure]#Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject  #设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定
    imagePullSecrets:    #Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
    hostNetwork:false      #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:       #在该pod上定义共享存储卷列表
    - name: string     #共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}     #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string     #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string     #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:      #类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:     #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
          path: string
```

### 例2，nginx

```yaml
apiVersion: apps/v1	#与k8s集群版本有关，使用 kubectl api-versions 即可查看当前集群支持的版本
kind: Deployment	#该配置的类型，我们使用的是 Deployment
metadata:	        #译名为元数据，即 Deployment 的一些基本属性和信息
  name: nginx-deployment	#Deployment 的名称
  labels:	    #标签，可以灵活定位一个或多个资源，其中key和value均可自定义，可以定义多组，目前不需要理解
    app: nginx	#为该Deployment设置key为app，value为nginx的标签
spec:	        #这是关于该Deployment的描述，可以理解为你期待该Deployment在k8s中如何使用
  replicas: 1	#使用该Deployment创建一个应用程序实例
  selector:	    #标签选择器，与上面的标签共同作用，目前不需要理解
    matchLabels: #选择包含标签app:nginx的资源
      app: nginx
  template:	    #这是选择或创建的Pod的模板
    metadata:	#Pod的元数据
      labels:	#Pod的标签，上面的selector即选择包含标签app:nginx的Pod
        app: nginx
    spec:	    #期望Pod实现的功能（即在pod中部署）
      containers:	#生成container，与docker中的container是同一种
      - name: nginx	#container的名称
        image: nginx:1.7.9	#使用镜像nginx:1.7.9创建container，该container默认80端口可访问
```

# helm

#### 安装helm

```
tar -xf helm-v3.0.1-linux-amd64.tar.gz 
mv linux-amd64 helm
cd helm
ls
cp helm /usr/local/bin
sudo cp helm /usr/local/bin
```
