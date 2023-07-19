### 下载脚本
```
wget https://hz-package.hzins.com/k8s.sh
```
### 使用说明
运行方法
```
bash k8s.sh -i
# 根据提示进行操作即可，注意，输入时不能携带特殊字符
```
说明
1.输入IP和K8s版本，即可完成K8s二进制高可用安装（含Ingress-Controller和CoreDNS）;
2.证书有效期为100年;
3.最少要求3个节点，即3个Master同时作为Node来部署，要求宿主机系统版本为RHEL 7/CentOS 7;
4. 网络插件为Flannel，后端为VXLAN，暂时不能更换；kube-proxy转发模式为ipvs，即所有service层面的流量都由ipvs模块进行转发;  
