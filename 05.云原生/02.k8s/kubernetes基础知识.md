# 一：理解
```
#容器：
1.隔离边界实现手段
一个程序运行起来后的计算机执行环境的总和，就是我们今天的主角：进程。
进程来说，它的静态表现就是程序，平常都安安静静地待在磁盘上；而一旦运行起来，它就变成了计算机里的数据和状态的总和，这就是它的动态表现。
而容器技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个“边界”。
Cgroups 技术是用来制造约束的主要手段，而Namespace 技术则是用来修改进程视图的主要方法。
容器，其实是一种特殊的进程而已。

Namespace 技术实际上修改了应用进程看待整个计算机“视图”，即它的“视线”被操作系统做了限制，只能“看到”某些指定的内容
Linux Cgroups 就是 Linux 内核中用来为进程设置资源限制的一个重要功能
Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等

#Docker 项目来说，它最核心的原理实际上就是为待创建的用户进程：
1.启用 Linux Namespace 配置；
2.设置指定的 Cgroups 参数；
3.切换进程的根目录（Change Root）

Dockerfile 的设计思想，是使用一些标准的原语（即大写高亮的词语），描述我们所要构建的 Docker 镜像。并且这些原语，都是按顺序处理的。
Dockerfile 中的每个原语执行后，都会生成一个对应的镜像层

Kubernetes 项目所擅长的，是按照用户的意愿和整个系统的规则，完全自动化地处理好容器之间的各种关系。这种功能，就是我们经常听到的一个概念：编排。

告诉 Kuberetes 你希望的状态， 然后 Kubemetes 会采取相关的必要措施来将集群的状态 切换到你期望的样子


#K8s中存在两种类型的探针：liveness probe和readiness probe。
#liveness probe（存活探针）
用于判断容器是否存活，即Pod是否为running状态，如果LivenessProbe探针探测到容器不健康，则kubelet将kill掉容器，并根据容器的重启策略是否重启。如果一个容器不包含LivenessProbe探针，则Kubelet认为容器的LivenessProbe探针的返回值永远成功。
有时应用程序可能因为某些原因（后端服务故障等）导致暂时无法对外提供服务，但应用软件没有终止，导致K8S无法隔离有故障的pod，调用者可能会访问到有故障的pod，导致业务不稳定。K8S提供livenessProbe来检测应用程序是否正常运行，并且对相应状况进行相应的补救措施。

注意，liveness探测失败并一定不会重启pod，pod是否会重启由你的restart policy 控制。

#readiness probe（就绪探针）
用于判断容器是否启动完成，即容器的Ready是否为True，可以接收请求，如果ReadinessProbe探测失败，则容器的Ready将为False，控制器将此Pod的Endpoint从对应的service的Endpoint列表中移除，从此不再将任何请求调度此Pod上，直到下次探测成功。通过使用Readiness探针，Kubernetes能够等待应用程序完全启动，然后才允许服务将流量发送到新副本。
比如使用tomcat的应用程序来说，并不是简单地说tomcat启动成功就可以对外提供服务的，还需要等待spring容器初始化，数据库连接没连上等等。对于spring boot应用，默认的actuator带有/health接口，可以用来进行启动成功的判断。
#每类探针都支持三种探测方法：

exec：通过执行命令来检查服务是否正常，针对复杂检测或无HTTP接口的服务，命令返回值为0则表示容器健康。
httpGet：通过发送http请求检查服务是否正常，返回200-399状态码则表明容器健康。
tcpSocket：通过容器的IP和Port执行TCP检查，如果能够建立TCP连接，则表明容器健康。


#探针探测的结果有以下三者之一：

Success：Container通过了检查。
Failure：Container未通过检查。
Unknown：未能执行检查，因此不采取任何措施。


#Pod重启策略：

Always: 总是重启
OnFailure: 如果失败就重启
Never: 永远不重启
```
# 二：架构

### k8s 集群分两部分：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/700245/1649818017405-3425b5bf-b883-4d5e-819c-660e527414c5.png#clientId=ub7a4aab7-8109-4&from=paste&id=uac2a338a&originHeight=874&originWidth=1126&originalType=url&ratio=1&rotation=0&showTitle=false&size=201889&status=done&style=none&taskId=u14fc42ad-fbba-44ec-b4b3-5c93e508bb1&title=)
#### master node

```
 (The k8s Control Plane, 控制面板)：存储和管理集群的状态
```

###### etcd 分布式持久化存储

```
一个分布式的一个存储系统，API Server 中所需要的这些原
信息都被放置在 etcd 中，etcd 本身是一个高可用系统，通过 etcd
保证整个 Kubernetes 的 Master 组件的高可用性
```

###### API 服务器(API Serve)

```
用来处理 API 操作的，Kubernetes 中所有
的组件都会和 API Server 进行连接，组件与组件之间一般不进行独
立的连接，都依赖于 API Server 进行消息的传送
```

###### 调度器(Controller)

```
“调度器”顾名思义就是完成调度的操作，就是
我们刚才介绍的第一个例子中，把一个用户提交的 Container，依据
它对 CPU、对 memory 请求大小，找一台合适的节点，进行放置
```

#### work node

```
- - Kubelet
  - Kubelet 服务代理( kube-proxy)
  - 容器运行时(Docker、rkt 或者其他)

- 附加组件

- - KubemetesDNS 服务器：通过 IP 对外暴露 HTTP 服务；
  - 仪表板
  - Ingress 控制器：对客户端 ip 保存，后续多次连接路由到 同一个 pod
  - Heapster(容器集群监控）
  - 容器网络接口插件
```

![](https://cdn.nlark.com/yuque/0/2022/webp/700245/1648695880318-cde14a37-9657-462f-8fd6-351dfa52a9f5.webp#clientId=ua89518c3-10c1-4&id=u7OjJ&originHeight=330&originWidth=509&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=ue0ac3e75-63ea-444b-822a-707f37143a7&title=)

```
kubectl get componentstatus ：显示控制面板各个组件的状态
```

### pod 的定义

```yaml
包含三个部分：

- metadata 包括名称、命名空间、标签和关于该容器的其他信息 。
- spec 包含 pod 内容的实际说明 ， 例如 pod 的容器、卷和其他数据 。
- status 包含运行中的 pod 的当前信息，例如 pod 所处的条件 、 每个容器的描述和状态，以及内部 IP 和其他基本信息 。


apiVersion: v1  # 分组和版本
kind: Pod       # 资源类型
metadata:
   name: kubia-manual  # Pod 名称
spec:
  containers:
  - image: luksa/kubia  # 容器使用的镜像
    name: kubia
    ports:
    - containerPort: 8080 # 应用监听的端口
      protocol: TCP
```

### kubernetes service (服务)概述

```
Kubernetes 中的 Service（服务） 提供了这样的一个抽象层，它选择具备某些特征的 Pod（容器组）并为它们定义一个访问方式。

Service（服务）使 Pod（容器组）之间的相互依赖解耦（原本从一个 Pod 中访问另外一个 Pod，需要知道对方的 IP 地址）。

一个 Service（服务）选定哪些 Pod（容器组） 通常由 LabelSelector(标签选择器) 来决定。
```

