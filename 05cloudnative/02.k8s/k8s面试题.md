#### 简述Kubernetes创建一个Pod的主要流程？
```
Kubernetes中创建一个Pod涉及多个组件之间联动，主要流程如下：

1、客户端提交Pod的配置信息（可以是yaml文件定义的信息）到kube-apiserver。

2、Apiserver收到指令后，通知给controller-manager创建一个资源对象。

3、Controller-manager通过api-server将pod的配置信息存储到ETCD数据中心中。

4、Kube-scheduler检测到pod信息会开始调度预选，会先过滤掉不符合Pod资源配置要求的节点，然后开始调度调优，主要是挑选出更适合运行pod的节点，然后将pod的资源配置单发送到node节点上的kubelet组件上。

5、Kubelet根据scheduler发来的资源配置单运行pod，运行成功后，将pod的运行信息返回给scheduler，scheduler将返回的pod运行状况的信息存储到etcd数据中心
```


