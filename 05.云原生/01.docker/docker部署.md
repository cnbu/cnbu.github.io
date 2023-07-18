
#### Ubuntu 1604安装docker

```
sudo apt-get remove docker docker-engine docker-ce docker.io

sudo apt-get update

sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update

sudo apt-get install -y docker-ce

systemctl start docker
```

##### 清理旧的Docker

```shell
yum remove docker docker-client  docker-client-latest  docker-common  docker-latest  docker-latest-logrotate  docker-logrotate  docker-selinux  docker-engine-selinux  docker-engine -y
```

##### 安装依赖

```
yum install -y yum-utils  device-mapper-persistent-data   lvm2
```

##### 配置阿里云Docker源

```
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

##### 查看源中的版本

```shell
#默认Docker版本1.13比较老
yum list docker-ce.x86_64 --showduplicates | sort -r
```

##### 安装Docker

```
yum -y install docker-ce-19.03.5-3.el7
```

##### 设置开机启动并启动服务

```
systemctl enable docker
systemctl start docker
```

##### 镜像加速，修改源，并修改docker的驱动方式，为k8s管理做准备

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

sudo systemctl daemon-reload
sudo systemctl restart docker
```

##### 如果加速地址有变，请到官网进行更新

```shell
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
```

##### 备注：启动服务会失败，因为配置文件多了逗号

```
将/etc/docker/daemon.json最后的逗号去掉
```

##### 测试拉取和运行

```
docker pull library/hello-world
docker run hello-world
```

##### 设置网络

```
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

想拉取Docker Hub，先要注册，或者直接从Daocloud拉取

```
docker pull daocloud.io/centos:7
或
docker login
docker pull centos:7
```

