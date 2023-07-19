## 常用命令

```shell
#两种办法
保存镜像（保存镜像载入后获得跟原镜像id相同的镜像）
保存容器（保存容器载入后获得跟原镜像id不同的镜像）
#保存镜像
docker save 镜像id -o /home/mysql.tar
docker save 镜像id > /home/mysql.tar
#载入镜像
docker load -i mysql.tar
#保存容器
docker export 镜像id -o /home/mysql-export.tar
#载入容器
docker import  mysql-export.tar
```

## 容器生命周期管理命令

```shell
### run

#创建一个新的容器。

# 使用docker镜像nginx:latest以后台模式启动一个容器,

# 并将容器命名为mynginx。  

docker run --name mynginx -d nginx:latest  

# 使用镜像 nginx:latest，以后台模式启动一个容器,

# 将容器的 80 端口映射到主机的 80 端口,

# 主机的目录 /data 映射到容器的 /data。  

docker run -p 80:80 -v /data:/data -d nginx:latest  

# 使用镜像nginx:latest以交互模式启动一个容器,

# 在容器内执行/bin/bash命令。  

docker run -it nginx:latest /bin/bash  

### start/stop/restart

- **docker start** : 启动一个或多个已经被停止的容器。
- **docker stop** : 停止一个运行中的容器。
- **docker restart** : 重启容器。

# 启动已被停止的容器mynginx  

docker start mynginx  

# 停止运行中的容器mynginx  

docker stop mynginx  

# 重启容器mynginx  

docker restart mynginx  

### kill

#杀掉一个运行中的容器。可选参数：

- **-s :** 发送什么信号到容器，默认 KILL

# 根据容器名字杀掉容器  

docker kill tomcat7  

# 根据容器ID杀掉容器  

docker kill 65d4a94f7a39  

### rm

删除一个或多个容器。

#强制删除容器 db01、db02：  
docker rm -f db01 db02  

#删除容器 nginx01, 并删除容器挂载的数据卷：  
docker rm -v nginx01  

#删除所有已经停止的容器：  
docker rm $(docker ps -a -q)  

### create

#创建一个新的容器但不启动它。

# 使用docker镜像nginx:latest创建一个容器,并将容器命名为mynginx  

docker create --name mynginx nginx:latest     

### exec

#在运行的容器中执行命令。可选参数：

- **-d :** 分离模式: 在后台运行
- **-i :** 即使没有附加也保持STDIN 打开
- **-t :** 分配一个伪终端

# 在容器 mynginx 中以交互模式执行容器内 /root/nginx.sh 脚本  

docker exec -it mynginx /bin/sh /root/nginx.sh  

# 在容器 mynginx 中开启一个交互模式的终端  

docker exec -i -t  mynginx /bin/bash  

# 也可以通过 docker ps -a 命令查看已经在运行的容器，然后使用容器 ID 进入容器。  

docker ps -a   
docker exec -it 9df70f9a0714 /bin/bash  

### pause/unpause

- **docker pause** :暂停容器中所有的进程。
- **docker unpause** :恢复容器中所有的进程。

# 暂停数据库容器db01提供服务。  

docker pause db01  

# 恢复数据库容器 db01 提供服务  

docker unpause db0
```

## 容器操作命令

```shell
### ps

列出容器。可选参数：

- **-a :** 显示所有的容器，包括未运行的。
- **-f :** 根据条件过滤显示的内容。
- **–format :** 指定返回值的模板文件。
- **-l :** 显示最近创建的容器。
- **-n :** 列出最近创建的n个容器。
- **–no-trunc :** 不截断输出。
- **-q :** 静默模式，只显示容器编号。
- **-s :** 显示总的文件大小。

# 列出所有在运行的容器信息。  
docker ps  
# 列出最近创建的5个容器信息。  
docker ps -n 5  

# 列出所有创建的容器ID。  

docker ps -a -q  

> 补充说明：
>
> 容器的7种状态：created（已创建）、restarting（重启中）、running（运行中）、removing（迁移中）、paused（暂停）、exited（停止）、dead（死亡）。

### inspect

获取容器/镜像的元数据。可选参数：

- **-f :** 指定返回值的模板文件。
- **-s :** 显示总的文件大小。
- **–type :** 为指定类型返回JSON。

# 获取镜像mysql:5.7的元信息。  

docker inspect mysql:5.7  

# 获取正在运行的容器mymysql的 IP。  

docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mymysql  

### top

查看容器中运行的进程信息，支持 ps 命令参数。

# 查看容器mymysql的进程信息。  

docker top mymysql  

# 查看所有运行容器的进程信息。  

for i in  `docker ps |grep Up|awk '{print $1}'`;do echo \ &&docker top $i; done  

### events

获取实时事件。参数说明：

- **-f ：** 根据条件过滤事件；
- **–since ：** 从指定的时间戳后显示所有事件；
- **–until ：** 流水时间显示到指定的时间为止；

# 显示docker 2016年7月1日后的所有事件。  

docker events  --since="1467302400"  

# 显示docker 镜像为mysql:5.6 2016年7月1日后的相关事件。  

docker events -f "image"="mysql:5.6" --since="1467302400"   

> 说明：如果指定的时间是到秒级的，需要将时间转成时间戳。如果时间为日期的话，可以直接使用，如–since=“2016-07-01”。

### logs

获取容器的日志。参数说明：

- **-f :** 跟踪日志输出
- **–since :** 显示某个开始时间的所有日志
- **-t :** 显示时间戳
- **–tail :** 仅列出最新N条容器日志

# 跟踪查看容器mynginx的日志输出。  

docker logs -f mynginx  

# 查看容器mynginx从2016年7月1日后的最新10条日志。  

docker logs --since="2016-07-01" --tail=10 mynginx  

### export

将文件系统作为一个tar归档文件导出到STDOUT。参数说明：

- **-o :** 将输入内容写到文件。

# 将id为a404c6c174a2的容器按日期保存为tar文件。  

docker export -o mysql-`date +%Y%m%d`.tar a404c6c174a2  

ls mysql-`date +%Y%m%d`.tar  

### port

列出指定的容器的端口映射。

# 查看容器mynginx的端口映射情况。  

docker port mymysql  

## 容器rootfs命令

### commit

从容器创建一个新的镜像。参数说明：

- **-a :** 提交的镜像作者；
- **-c :** 使用Dockerfile指令来创建镜像；
- **-m :** 提交时的说明文字；
- **-p :** 在commit时，将容器暂停。

# 将容器a404c6c174a2 保存为新的镜像,

# 并添加提交人信息和说明信息。  

docker commit -a "guodong" -m "my db" a404c6c174a2  mymysql:v1   

### cp

用于容器与主机之间的数据拷贝。参数说明：

- **-L :** 保持源目标中的链接

# 将主机/www/runoob目录拷贝到容器96f7f14e99ab的/www目录下。  

docker cp /www/runoob 96f7f14e99ab:/www/  

# 将主机/www/runoob目录拷贝到容器96f7f14e99ab中，目录重命名为www。  

docker cp /www/runoob 96f7f14e99ab:/www  

# 将容器96f7f14e99ab的/www目录拷贝到主机的/tmp目录中。  

docker cp  96f7f14e99ab:/www /tmp/  

### diff

#检查容器里文件结构的更改。

# 查看容器mymysql的文件结构更改。  

docker diff mymysql
```

## 镜像仓库命令

```shell
### login/logout

#**docker login :** 登陆到一个Docker镜像仓库，如果未指定镜像仓库地址，默认为官方仓库 Docker Hub**docker logout :**登出一个Docker镜像仓库，如果未指定镜像仓库地址，默认为官方仓库 Docker Hub参数说明：

- **-u :** 登陆的用户名
- **-p :** 登陆的密码

# 登陆到Docker Hub  
docker login -u 用户名 -p 密码  

# 登出Docker Hub  
docker logout  

### pull

从镜像仓库中拉取或者更新指定镜像。参数说明：

- **-a :** 拉取所有 tagged 镜像
- **–disable-content-trust :** 忽略镜像的校验,默认开启


# 从Docker Hub下载java最新版镜像。  
docker pull java  

# 从Docker Hub下载REPOSITORY为java的所有镜像。  
docker pull -a java  


### push
#将本地的镜像上传到镜像仓库,要先登陆到镜像仓库。参数说明：

- **–disable-content-trust :** 忽略镜像的校验,默认开启

# 上传本地镜像myapache:v1到镜像仓库中。  
docker push myapache:v1  

### search
#从Docker Hub查找镜像。参数说明：

- **–automated :** 只列出 automated build类型的镜像；
- **–no-trunc :** 显示完整的镜像描述；
- **-f \<过滤条件>:** 列出指定条件的镜像。


# 从 Docker Hub 查找所有镜像名包含 java，并且收藏数大于 10 的镜像  
docker search -f stars=10 java  

NAME                  DESCRIPTION                           STARS   OFFICIAL   AUTOMATED  
java                  Java is a concurrent, class-based...   1037    [OK]         
anapsix/alpine-java   Oracle Java 8 (and 7) with GLIBC ...   115                [OK]  
develar/java                                                 46                 [OK]  
```

#每列参数说明：

- **NAME:** 镜像仓库源的名称
- **DESCRIPTION:** 镜像的描述
- **OFFICIAL:** 是否 docker 官方发布
- **stars:** 类似 Github 里面的 star，表示点赞、喜欢的意思
- **AUTOMATED:** 自动构建

## 本地镜像管理命令

### images

#列出本地镜像。参数说明：

- **-a :** 列出本地所有的镜像（含中间映像层，默认情况下，过滤掉中间映像层）；
- **–digests :** 显示镜像的摘要信息；
- **-f :** 显示满足条件的镜像；
- **–format :** 指定返回值的模板文件；
- **–no-trunc :** 显示完整的镜像信息；
- **-q :** 只显示镜像ID。


# 查看本地镜像列表。  
docker images  

# 列出本地镜像中REPOSITORY为ubuntu的镜像列表。  
docker images  ubuntu  

# rmi
#删除本地一个或多个镜像。参数说明：

- **-f :** 强制删除；
- **–no-prune :** 不移除该镜像的过程镜像，默认移除；


# 强制删除本地镜像 guodong/ubuntu:v4。  
docker rmi -f guodong/ubuntu:v4  
```

### tag

标记本地镜像，将其归入某一仓库。


# 将镜像ubuntu:15.10标记为 runoob/ubuntu:v3 镜像。  
docker tag ubuntu:15.10 runoob/ubuntu:v3  


### build

#用于使用 Dockerfile 创建镜像。参数说明：

- **–build-arg=[] :** 设置镜像创建时的变量；
- **–cpu-shares :** 设置 cpu 使用权重；
- **–cpu-period :** 限制 CPU CFS周期；
- **–cpu-quota :** 限制 CPU CFS配额；
- **–cpuset-cpus :** 指定使用的CPU id；
- **–cpuset-mems :** 指定使用的内存 id；
- **–disable-content-trust :** 忽略校验，默认开启；
- **-f :** 指定要使用的Dockerfile路径；
- **–force-rm :** 设置镜像过程中删除中间容器；
- **–isolation :** 使用容器隔离技术；
- **–label=[] :** 设置镜像使用的元数据；
- **-m :** 设置内存最大值；
- **–memory-swap :** 设置Swap的最大值为内存+swap，"-1"表示不限swap；
- **–no-cache :** 创建镜像的过程不使用缓存；
- **–pull :** 尝试去更新镜像的新版本；
- **–quiet, -q :** 安静模式，成功后只输出镜像 ID；
- **–rm :** 设置镜像成功后删除中间容器；
- **–shm-size :** 设置/dev/shm的大小，默认值是64M；
- **–ulimit :** Ulimit配置。
- **–squash :** 将 Dockerfile 中所有的操作压缩为一层。
- **–tag, -t:** 镜像的名字及标签，通常 name:tag 或者 name 格式；可以在一次构建中为一个镜像设置多个标签。
- **–network:** 默认 default。在构建期间设置RUN指令的网络模式

shell
# 使用当前目录的 Dockerfile 创建镜像，标签为 runoob/ubuntu:v1  
docker build -t runoob/ubuntu:v1 .   

# 使用URL github.com/creack/docker-firefox 的 Dockerfile 创建镜像  
docker build github.com/creack/docker-firefox  

# 通过 -f Dockerfile文件的位置 创建镜像  
docker build -f /path/to/a/Dockerfile .  


### history

查看指定镜像的创建历史。参数说明：

- **-H :** 以可读的格式打印镜像大小和日期，默认为true；
- **–no-trunc :** 显示完整的提交记录；
- **-q :** 仅列出提交记录ID。

```shell
# 查看本地镜像 guodong/ubuntu:v3 的创建历史。  
docker history guodong/ubuntu:v3  
```

### save

#将指定镜像保存成 tar 归档文件。参数说明：

- **-o :** 输出到的文件。

```shell
# 将镜像 runoob/ubuntu:v3 生成 my_ubuntu_v3.tar 文档  
docker save -o my_ubuntu_v3.tar runoob/ubuntu:v3  
```

### load

#导入使用 `docker save` 命令导出的镜像。参数说明：

- **–input , -i :** 指定导入的文件，代替 STDIN。
- **–quiet , -q :** 精简输出信息。

```
# 导入镜像  
docker load --input fedora.tar  
```

### import

从归档文件中创建镜像。参数说明：

- **-c :** 应用docker 指令创建镜像；
- **-m :** 提交时的说明文字；

```
# 从镜像归档文件my_ubuntu_v3.tar创建镜像，命名为runoob/ubuntu:v4  
docker import  my_ubuntu_v3.tar runoob/ubuntu:v4    
```
```

## 基础版本信息命令

```shell
### info
显示 Docker 系统信息，包括镜像和容器数。

# 查看docker系统信息。  
docker info  


#version
#显示 Docker 版本信息。
docker version
```

## 
## 1.2 常用命令

#### 启动容器

```shell
# 启动
docker run dinghe/flask-hello
# 设定名字启动

# 后台启动
docker run -d dinghe/flask-hello
# 映射端口
docker run --name ding-flask -p 9501:9501 dinghe/flask-hello 


#列出所有容器（包括已停止的容器）
docker ps -a

#一次删除所有停止的容器
docker rm $(docker ps -a -q)
```

#### 备注：启动容器可以限制使用资源，如内存，CPU

```
--cpu-share 3 (相对的，不同容器根据该值进行比例分配)
--memory-swap byts
--memory  byts
```

#### 停止启动容器

```
docker stop ding-flask
docker start ding-flask
```

#### 查看image

```
docker images
```

#### 删除iamge

```shell
docker rmi xxxxxx(container id)
#删除前需停止基于该镜像启动的容器

#批量删除镜像        
docker rmi  $(docker image ls -a -q)
```

#### 查看当前启动的容器

```
docker ps -a
docker container ls
```

#### 删除当前启动的容器

```shell
docker rm xxxxxx(container id)

# 批量删除
docker rm ${docker ps -qa}
docker rm ${docker ps -qa|grep xxx}

#批量删除容器        
docker rm  $(docker container ls -a -q)
```

#### 基于dockerfile编译一个容器

```
docker build -t ding/centos-vim .
# . 代表当前目录下的dockerfile
```

#### 基于dockerfile编译一个容器

```
docker build -t ding/centos-vim .
# . 代表当前目录下的dockerfile
```

#### 查看当前镜像详细内容

```
docker history e0985896a946
```

#### 批量删除容器

```
docker rm $(docker ps -aq)
```

#### 批量删除退出的容器

```
docker rm $(docker ps -f "status=exited" -qa)
```

#### 将容器中的修改更新到镜像中(不推荐，不知道做了什么)

```
docker commit crazy_johnson dinghe/centos-vim
```

#### 推送镜像到docker hub（不推荐）

```
# 先注册登录，注意使用用户名/镜像名称
docker  login
docker push dinghe/hello-work
```

#### 使用dockerfile自动进行构建（推荐）

```
# Docker hub -> Create -> create automated build
-> Link Github -> 同步dockerfile -> docker hub自动构建
```

#### 进入容器内部

```
# 这里要区别于启动容器直接进入，这个是容器运行了，想随时进入
docker exec -it 43236b2f5b7e /bin/bash
```

#### 如果容器启动失败，怎么进入

```
1. 修改Dockerfile，去掉最后的启动命令
2. docker run -d --name t1 -it image-t1 /bin/bash
3. docker exec -it xxxx进入后模拟启动
```

#### 查看容器基本信息

```
docker insepct 43236b2f5b7e
```

#### 查看docker运行日志

```
docker logs 43236b2f5b7e
```

#### 备份和恢复镜像

```
docker save -o mysql.tar daocloud.io/mysql
docker load -i mysql.tar
```

##### 资源限制

```
docker run -d --name mysql --memory="500m" --memory-swap="600M" --oom-kill-disabel mysql01
# 限制CPU
docker run -d --name mysql --cpus=".5"
docker run -d --name mysql --cpus="1.5
```

## 1.3 Dockerfile编写

#### 制作容器基本步骤

```
1. 确保程序运行环境（版本，插件，依赖）
2. 测试代码和拷贝
3. 运行程序
```

#### 制作一个包含程序的镜像

```
FROM python:3.6
LABEL maintainer="Ptyhon3.6 dinghe@outlook.com"
RUN pip3 install flask
COPY app.py /app/
WORKDIR /app
EXPOSE 5000
CMD ["python3","app.py"]
```

#### 制作并运行

```
cd /opt/dinghe/flask-hello/
docker build -t dinghe/flask-hello .
# 前台执行  docker run dinghe/flask-hello
# 后台执行  docker run -d dinghe/flask-hello
```

#### FROM确定引用还是制作

```
# 尽量使用官方Image
FROM scratch
FROM centos:7
```

#### LABEL定义镜像元信息

```
# 必须要有
LABEL maintainer="dinghe_1985@126.com"
LABEL version=0.1
LABEL deacription="Dev Basic PHP 5.3/5.6/7.2"
```

#### Shell

```
# 默认在bash环境中执行所有命令
RUN yum install -y vim
CMD echo "hello Ding"
ENTRYPOINT echo "hello Ding"
```

#### Exec

```
# 第一个参数是命令，后面的全是值
RUN ["yum", "install", "-y", "vim"]
CMD ["/bin/echo", "hello Ding"]
ENTRYPOINT ["/bin/echo", "hello Ding"]

# 如果要在bash环境执行，需要调整
# 注意，最后要执行的命令放在一对""里面
ENV name Ding
ENTRYPOINT ["/bin/sh/", "-c", "/bin/echo hello $name" ]
```

#### RUN

```
# 每运行一次RUN，就会生成一层
# 多条命令合并为一行，回行使用 \
# 注意清理Cache文件，避免镜像过大

RUN yum update && yum install vim \
    python -dev
```

#### CMD

```
# 容器启动时执行的命令
# 多个CMD只有最后一个会执行
# docker run image默认执行最后一个CMD
# docker run -it image /bin/bash 忽略CMD
```

#### ENTRYPOINT

```
# 让容器以一个应用程序或者服务的形式运行
# 不会忽略，一定执行
# shell脚本使用
```

#### WORKDIR设置工作目录

```
# 如果没有会创建并进入
# 不要用RUN cd，尽量使用绝对路径
推荐：WORKDIR /opt/lnmp

WORKDIR /opt/
WORKDIR lnmp
# pwd /opt/lnmp/
```

#### COPY和ADD的区别

```
# COPY仅支持复制(推荐使用)
# ADD 支持解压，wget，会根据目标类型判断覆盖还是放入

WORKDIR /opt
ADD hello.sh lnmp/
# pwd  /opt/lnmp/hello.sh
```

#### 声明变量

```
ENV MYSQL_VERSION 5.6
yum install "${MYSQL_VERSION}"
```

## 1.4 容器网络

#### docker4种网络模式
```
none模式	    –net=none	 容器有独立的Network namespace，但并没有对其进行任何网络设置，如分配veth pair 和网桥连接，配置IP等。

container模式	–net=container:NAME_or_ID	容器和另外一个容器共享Network namespace。 kubernetes中的pod就是多个容器共享一个Network namespace。

host模式	    –net=host	容器和宿主机共享Network namespace。

bridge模式	  –net=bridge	（默认为该模式）
```

#### 网络基础

```
# 网络模型
TCP/IP 5层模型
ISO/OSI 7层模型
```

#### 模拟Docker建立命名空间

```shell
# 查看当前命令空间
ip netns list
# 创建命名空间
ip netns add test1
ip netns add test2
# 创建网卡并绑定
ip link add veth-test1 type veth peer name veth-test2
# 将网卡绑定到命名空间
ip link set veth-test1 netns test1
ip link set veth-test2 netns test2
# 启动网卡
ip netns exec test1 ip link set dev veth-test1 up 
ip netns exec test2 ip link set dev veth-test2 up 
# 设置IP地址
ip netns exec test2 ip addr ad 192.168.1.2/24 dev veth-test2
ip netns exec test1 ip addr add 192.168.1.1/24 dev veth-test1
# 查看IP地址
ip netns exec test2 ip a
ip netns exec test1 ip a
# 测试联通
ip netns exec test2 ping 192.168.1.1
ip netns exec test1 ping 192.168.1.2
```

#### 容器网络

容器间通过将网络空间都连接到docker0进行通信
容器访问外网通过NAT方式，由iptables的默认网关实现

```
# 命名空间添加默认网关
ip netns exec 27383 route add default gw 192.168.1.1
# 宿主机添加默认路由
sudo iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o em1 -j MASQUERADE
```

#### docker --link 容器间通过名字通信

```
docker run -d --name test2 --link test1 centos:7 /bin/sh -c "while true;do sleep 3600; done"
```

#### docker --link原理

```shell
# 增加
docker network create -d bridge my-bridge
# 查看
docker network ls
# 创建网络时指定
docker network create -d bridge my-network
# 让容器连接到这个网络
docker network connect my-network test2
docker network connect my-network test3
```

#### 端口映射

```shell
# 第一个80是docker端口，第二个80是宿主机端口
docker run -d --name web -p 80:80 nginx
```

#### Overlay网络（多主机间容器互联）

```
# 需结合Etcd服务，实现数据通信，启动服务时进行注册
# 后续集成K8s后，自动实现
```

## 1.5 数据持久化

#### 场景

```
# 容器释放后，数据保存
# MySQL
```

#### Data Volume
```yaml

#譬如我要启动一个centos容器，宿主机的/test目录挂载到容器的/soft目录，可通过以下方式指定：

docker run -it -v /test:/soft centos /bin/bash

冒号":"前面的目录是宿主机目录，后面的目录是容器内目录。
```
```shell
# 使用宿主机的数据卷，并在后续工作中继续使用
docker run -d --name mysql2 -v mysql:/var/lib/mysql -e MYSQL_ALLOW_EMPTY_PASSWORD=true daocloud.io/mysql:5.7

mysql:/var/lib/mysql
mysql是卷名字，便于后续调用，后面是容器中的位置

# Dockerfile调用
VOLUME["/var/lib/mysql"]
```

#### Bind Mouting

```
# 定义与宿主机的目录关系（软连接？）
# 适合本地开发，自动发布
docker run -d --name mysql2 -v /data/mysql3306:/var/lib/mysql -e MYSQL_ALLOW_EMPTY_PASSWORD=true daocloud.io/mysql:5.7
```


#### docker-compose

#### 部署（Centos）

```shell
# Windows和Mac默认就会安装
curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

#### 常用命令

```shell
# 启动命令
docker-compose -f /opt/docker-compose.yml up
# 启动容器（不看日志）
docker-compose -f /opt/docker-compose.yml up -d

# 查看当前启动容器
docker-compose ps
# 查看docker-compose中的容器和使用镜像
docker-compose images
# 进入docker-compose中的容器
docker-compose exec mysql bash
# 销毁所有容器和网络
docker-compose down
# 停止和启动容器
docker-compose start/stop
```

#### 问题

Cache有时候会耽误事，建议关掉

```
docker-compose build --no-cache
```

有时候需要停了再开，而不是up，如mysql初始化问题
