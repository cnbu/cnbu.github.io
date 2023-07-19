## 查看

```shell
#查看集群信息 
kubectl cluster-info
#查看各组件信息 
kubectl get componentstatuses
#api接口查看
kubectl api-resources
#查看pod 日志 （如果pod有多个容器需要加-c 容器名） 
kubectl logs xxx -n kube-system
#查看所有namespace的pods运行情况
kubectl get pods --all-namespaces 
#查看具体pods，记得后边跟namespace名字哦
kubectl get pods  kubernetes-dashboard-76479d66bb-nj8wr --namespace=kube-system
#查看pods具体信息
kubectl get pods -o wide kubernetes-dashboard-76479d66bb-nj8wr --namespace=kube-system
# 查看集群健康状态
kubectl get cs
# 获取所有deployment
kubectl get deployment --all-namespaces
# 查看kube-system namespace下面的pod/svc/deployment 等等（-o wide 选项可以查看存在哪个对应的节点）
kubectl get pod /svc/deployment -n kube-system
# 列出该 namespace 中的所有 pod 包括未初始化的
kubectl get pods --include-uninitialized
# 查看deployment()
kubectl get deployment nginx-app
# 查看rc和servers
kubectl get rc,services
# 查看pods结构信息（重点，通过这个看日志分析错误）
# 对控制器和服务，node同样有效
kubectl describe pods xxxxpodsname --namespace=xxxnamespace
# 其他控制器类似吧，就是kubectl get 控制器 控制器具体名称
# 查看pod日志
kubectl logs $POD_NAME
# 查看pod变量
kubectl exec my-nginx-5j8ok -- printenv | grep SERVICE
```

## 集群

```shell

# 集群健康情况
kubectl get cs
# 集群核心组件运行情况
kubectl cluster-info
# 表空间名
kubectl get namespaces
# 版本
kubectl version 
# API
kubectl api-versions
# 查看事件
kubectl get events
 #获取全部节点
kubectl get nodes  
#//删除节点
kubectl delete node k8s2
#查看deployment状态
kubectl rollout status deploy nginx-test
kubectl get deployment --all-namespaces
kubectl get svc --all-namespaces
```

## kubectl启动命令补全

```shell
#安装 bash-completion
apt-get install bash-completion或yum install bash-completion

#启用 kubectl 自动补全
echo 'source <(kubectl completion bash)' >>~/.bashrc
kubectl completion bash >/etc/bash_completion.d/kubectl
#如果上一句执行错误的话，就执行下面一句
#kubectl completion bash |sudo tee /etc/bash_completion.d/kubectl

#别名设置（可选）
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
```

## 创建

```shell
kubectl create -f ./nginx.yaml           # 创建资源
kubectl apply -f xxx.yaml （创建+更新，可以重复使用）
kubectl create -f .                            # 创建当前目录下的所有yaml资源
kubectl create -f ./nginx1.yaml -f ./mysql2.yaml     # 使用多个文件创建资源
kubectl create -f ./dir                        # 使用目录下的所有清单文件来创建资源
kubectl create -f https://git.io/vPieo         # 使用 url 来创建资源
kubectl run -i --tty busybox --image=busybox    ----创建带有终端的pod
kubectl run nginx --image=nginx                # 启动一个 nginx 实例
kubectl run mybusybox --image=busybox --replicas=5    ----启动多个pod
kubectl explain pods,svc                       # 获取 pod 和 svc 的文档
```

## 滚动更新

```shell
kubectl rollout history deployment nginx-deploy                #查看升级的历史记录
kubectl rolling-update python-v1 -f python-v2.json             # 滚动更新 pod frontend-v1
kubectl rolling-update python-v1 python-v2 --image=image:v2    # 更新资源名称并更新镜像
kubectl rolling-update python --image=image:v2                 # 更新 frontend pod 中的镜像
kubectl rolling-update python-v1 python-v2 --rollback          # 退出已存在的进行中的滚动更新
cat pod.json | kubectl replace -f -                            # 基于 stdin 输入的 JSON 替换 pod
```

## 编辑资源

```shell
kubectl edit svc/docker-registry                          # 编辑名为 docker-registry 的 service
KUBE_EDITOR="nano" kubectl edit svc/docker-registry       # 使用其它编辑器
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf #修改启动参数
```

## 动态伸缩pod

```shell
kubectl scale --replicas=3 rs/foo                                 # 将foo副本集变成3个
kubectl scale --replicas=3 -f foo.yaml                            # 缩放“foo”中指定的资源。
kubectl scale --current-replicas=2 --replicas=3 deployment/mysql  # 将deployment/mysql从2个变成3个
kubectl scale --replicas=5 rc/foo rc/bar rc/baz                   # 变更多个控制器的数量
kubectl rollout status deploy deployment/mysql                         # 查看变更进度

#label 操作
kubectl label：添加label值 kubectl label nodes node1 zone=north #增加节点lable值 spec.nodeSelector: zone: north #指定pod在哪个节点
kubectl label pod redis-master-1033017107-q47hh role=master #增加lable值 [key]=[value]
kubectl label pod redis-master-1033017107-q47hh role- #删除lable值
kubectl label pod redis-master-1033017107-q47hh role=backend --overwrite #修改lable值
```

## 滚动升级

```shell
#kubectl rolling-update：
kubectl rolling-update redis-master -f redis-master-controller-v2.yaml  #配置文件滚动升级
kubectl rolling-update redis-master --image=redis-master:2.0            #命令升级
kubectl rolling-update redis-master --image=redis-master:1.0 --rollback #pod版本回滚
```

## etcdctl 常用操作

```shell
etcdctl cluster-health                                         #检查网络集群健康状态
etcdctl --endpoints=https://192.168.71.221:2379 cluster-health #带有安全认证检查网络集群健康状态
etcdctl member list
etcdctl set /k8s/network/config ‘{ “Network”: “10.1.0.0/16” }’
etcdctl get /k8s/network/config
```

## 删除

```shell
kubectl delete pod -l app=flannel -n kube-system                          # 根据label删除：
kubectl delete -f ./pod.json                                              # 删除 pod.json 文件中定义的类型和名称的 pod
kubectl delete pod,service baz foo                                        # 删除名为“baz”的 pod 和名为“foo”的 service
kubectl delete pods,services -l name=myLabel                              # 删除具有 name=myLabel 标签的 pod 和 serivce
kubectl delete pods,services -l name=myLabel --include-uninitialized      # 删除具有 name=myLabel 标签的 pod 和 service，包括尚未初始化的
kubectl -n my-ns delete po,svc --all                                      # 删除 my-ns namespace下的所有 pod 和 serivce，包括尚未初始化的
kubectl delete pods prometheus-7fcfcb9f89-qkkf7 --grace-period=0 --force  #强制删除
kubectl delete deployment kubernetes-dashboard --namespace=kube-system
kubectl delete svc kubernetes-dashboard --namespace=kube-system
kubectl delete -f kubernetes-dashboard.yaml
kubectl replace --force -f ./pod.json                                     # 强制替换，删除后重新创建资源。会导致服务中断。
```

## 清理

```shell
#Kubernetes 基础对象清理
#清理 Evicted 状态的 Pod
$ kubectl get pods --all-namespaces -o wide | grep Evicted | awk '{print $1,$2}' | xargs -L1 kubectl delete pod -n
#清理 Error 状态的 Pod
$ kubectl get pods --all-namespaces -o wide | grep Error | awk '{print $1,$2}' | xargs -L1 kubectl delete pod -n
#清理 Completed 状态的 Pod
$ kubectl get pods --all-namespaces -o wide | grep Completed | awk '{print $1,$2}' | xargs -L1 kubectl delete pod -n
#清理没有被使用的 PV
$ kubectl describe -A pvc | grep -E "^Name:.*$|^Namespace:.*$|^Used By:.*$" | grep -B 2 "<none>" | grep -E "^Name:.*$|^Namespace:.*$" | cut -f2 -d: | paste -d " " - - | xargs -n2 bash -c 'kubectl -n ${1} delete pvc ${0}'
#清理没有被绑定的 PVC
$ kubectl get pvc --all-namespaces | tail -n +2 | grep -v Bound | awk '{print $1,$2}' | xargs -L1 kubectl delete pvc -n
#清理没有被绑定的 PV
$ kubectl get pv | tail -n +2 | grep -v Bound | awk '{print $1}' | xargs -L1 kubectl delete pv

#Linux 清理
查看磁盘全部空间
$ df -hl /

Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       100G   47G   54G  47% /
查看指定目录占用
$ du -sh .

24G .
#删除指定前缀的文件夹
$ cd /nfsdata
$ ls | grep archived- |xargs -L1 rm -r
#清理僵尸进程
$ ps -A -ostat,ppid | grep -e '^[Zz]' | awk '{print }' | xargs kill -HUP > /dev/null 2>&1
3Docker 清理
#查看磁盘使用情况
$ docker system df

TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              361                 23                  178.5GB             173.8GB (97%)
Containers          29                  9                   6.682GB             6.212GB (92%)
Local Volumes       4                   0                   3.139MB             3.139MB (100%)
Build Cache         0                   0                   0B                  0B
#清理 none 镜像
$ docker images | grep none | awk '{print $3}' | xargs docker rmi
#清理不再使用的数据卷
$ docker volume rm $(docker volume ls -q)
或者

$ docker volume prune
#清理缓存
$ docker builder prune
#全面清理
#删除关闭的容器、无用的存储卷、无用的网络、dangling 镜像（无 tag 镜像）

$ docker system prune -f 
#清理正则匹配上的镜像
这里清理的是 master-8bcf8d7-20211206-111155163 格式的镜像。

$ docker images |grep -E "([0-9a-z]*[-]){3,}[0-9]{9}" |awk '{print $3}' | xargs docker rmi
#设置定时
#查看定时任务
$ crontab -l
#设置定时任务
$ crontab -e
#文本新增定时任务

*/35 */6 * * *  docker images | grep none | awk '{print $3}' | xargs docker rmi
45 1 * * * docker system prune -f
#这里第一个任务是每隔六个小时的第 35 分钟执行，第二个任务每天的 1 时 45 分执行。

#定时任务的格式
设置定时格式: * * * * * shell

第一个星号，minute，分钟，值为 0-59
第二个星号，hour，小时，值从 0-23
第三个星号，day，天，值为从 1-31
第四个星号，month，月，值为从 1-12 月，或者简写的英文，比如 Nov、Feb 等
第五个星号，week 周，值为从 0-6 或者简写的英文，Wen、Tur 等，代表周几，其中 0 代表周末
```

## 交互

```shell
kubectl logs nginx-pod                                 # dump 输出 pod 的日志（stdout）
kubectl logs nginx-pod -c my-container                 # dump 输出 pod 中容器的日志（stdout，pod 中有多个容器的情况下使用）
kubectl logs -f nginx-pod                              # 流式输出 pod 的日志（stdout）
kubectl logs -f nginx-pod -c my-container              # 流式输出 pod 中容器的日志（stdout，pod 中有多个容器的情况下使用）
kubectl run -i --tty busybox --image=busybox -- sh     # 交互式 shell 的方式运行 pod
kubectl attach nginx-pod -i                            # 连接到运行中的容器
kubectl port-forward nginx-pod 5000:6000               # 转发 pod 中的 6000 端口到本地的 5000 端口
kubectl exec nginx-pod -- ls /                         # 在已存在的容器中执行命令（只有一个容器的情况下）
kubectl exec nginx-pod -c my-container -- ls /         # 在已存在的容器中执行命令（pod 中有多个容器的情况下）
kubectl top pod POD_NAME --containers                  # 显示指定 pod和容器的指标度量
kubectl exec -ti podName /bin/bash                     # 进入pod
```

## 调度配置

```shell
#标记 my-node 不可调度
kubectl cordon k8s-node
#标记 my-node 可调度
kubectl uncordon k8s-node
#清空my-node 以待维护,强制驱逐节点上的pod
kubectl drain nodename --delete-local-data --ignore-daemonsets --force
#显示 my-node 的指标度量
kubectl top node k8s-node
#将当前集群状态输出到 stdout
kubectl cluster-info dump      
#将当前集群状态输出到 /path/to/cluster-state
kubectl cluster-info dump --output-directory=/path/to/cluster-state
#如果该键和影响的污点（taint）已存在，则使用指定的值替换
kubectl taint nodes foo dedicated=special-user:NoSchedule
#查看kubelet进程启动参数
ps -ef | grep kubelet

#设为不可调度状态： 
kubectl cordon node1
#将pod赶到其他节点： 
kubectl drain node1
#解除不可调度状态 
kubectl uncordon node1
#master运行pod 
kubectl taint nodes master.k8s node-role.kubernetes.io/master-
#master不运行pod 
kubectl taint nodes master.k8s node-role.kubernetes.io/master=:NoSchedule
```

## 查看日志以及下载日志

```shell
journalctl -u kubelet -f

导出配置文件：
　　导出proxy
　　kubectl get ds -n kube-system -l k8s-app=kube-proxy -o yaml>kube-proxy-ds.yaml
　　导出kube-dns
　　kubectl get deployment -n kube-system -l k8s-app=kube-dns -o yaml >kube-dns-dp.yaml
　　kubectl get services -n kube-system -l k8s-app=kube-dns -o yaml >kube-dns-services.yaml
　　导出所有 configmap
　　kubectl get configmap -n kube-system -o wide -o yaml > configmap.yaml

复杂操作命令：
　删除kube-system 下Evicted状态的所有pod：
kubectl get pods -n kube-system |grep Evicted| awk ‘{print $1}’|xargs kubectl delete pod -n kube-system
以下为维护环境相关命令：
重启kubelet服务
systemctl daemon-reload
systemctl restart kubelet




#获取容器日志
找到对应的namespace
kubectl get pod -A | grep travel-canal-server
#进到容器内
kubectl exec -it portal-operation-server-0b5nc-66dfb7df58-b58bn -n  hz-prod   /bin/bash
#多容器进入别的容器方法
kubectl exec -it portal-operation-server-0b5nc-66dfb7df58-b58bn -n  hz-prod   /bin/bash -c 容器名称
#拷贝到node本地
kubectl cp titan-cc-prod/titan-cc-core-657885dc68-gwq7k:/home/deploy/serviceDump.dat /tmp/

#通过python获取容器日志
1.查看ip  
ip a

2.进入容器后执行命令
python -m SimpleHTTPServer

3.dba服务器访问容器ip:8000端口


## 为 nginx RC 创建服务，启用本地 80 端口连接到容器上的 8000 端口
kubectl expose rc nginx --port=80 --target-port=8000
## 更新单容器 pod 的镜像版本（tag）到 v4
kubectl get pod nginx-pod -o yaml | sed 's/(image: myimage):.*$/1:v4/' | kubectl replace -f -
kubectl label pods nginx-pod new-label=awesome                      # 添加标签
kubectl annotate pods nginx-pod icon-url=http://goo.gl/XXBTWq       # 添加注解
kubectl autoscale deployment foo --min=2 --max=10                # 自动扩展 deployment “foo”
```

