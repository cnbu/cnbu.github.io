```shell
[root@docker ~]# docker --help
Usage:    docker [OPTIONS] COMMAND
Commands:
    attach    Attach to a running container                 # 当前 shell 下 attach 连接指定运行镜像
    build     Build an image from a Dockerfile              # 通过 Dockerfile 定制镜像
    commit    Create a new image from a container's changes # 提交当前容器为新的镜像
    cp        Copy files/folders from the containers filesystem to the host path  # 从容器中拷贝指定文件或者目录到宿主机中
    create    Create a new container                        # 创建一个新的容器，同 run，但不启动容器
    diff      Inspect changes on a container's filesystem   # 查看 docker 容器变化
    events    Get real time events from the server          # 从 docker 服务获取容器实时事件
    exec      Run a command in an existing container        # 在已存在的容器上运行命令
    export    Stream the contents of a container as a tar archive    # 导出容器的内容流作为一个 tar 归档文件[对应 import ]
    history   Show the history of an image                  # 展示一个镜像形成历史
    images    List images                                   # 列出系统当前镜像
    import    Create a new filesystem image from the contents of a tarball  # 从tar包中的内容创建一个新的文件系统映像[对应 export]
    info      Display system-wide information               # 显示系统相关信息
    inspect   Return low-level information on a container   # 查看容器详细信息
    kill      Kill a running container                      # kill 指定 docker 容器
    load      Load an image from a tar archive              # 从一个 tar 包中加载一个镜像[对应 save]
    login     Register or Login to the docker registry server   # 注册或者登陆一个 docker 源服务器
    logout    Log out from a Docker registry server         # 从当前 Docker registry 退出
    logs      Fetch the logs of a container                 # 输出当前容器日志信息
    port      Lookup the public-facing port which is NAT-ed to PRIVATE_PORT # 查看映射端口对应的容器内部源端口
    pause     Pause all processes within a container        # 暂停容器
    ps        List containers                               # 列出容器列表
    pull      Pull an image or a repository from the docker registry server # 从docker镜像源服务器拉取指定镜像或者库镜像
    push      Push an image or a repository to the docker registry server # 推送指定镜像或者库镜像至docker源服务器
    restart   Restart a running container                   # 重启运行的容器
    rm        Remove one or more containers                 # 移除一个或者多个容器
    rmi       Remove one or more images       # 移除一个或多个镜像[无容器使用该镜像才可删除，否则需删除相关容器才可继续或 -f 强制删除]
    run       Run a command in a new container  # 创建一个新的容器并运行一个命令
    save      Save an image to a tar archive                # 保存一个镜像为一个 tar 包[对应 load]
    search    Search for an image on the Docker Hub         # 在 docker hub 中搜索镜像
    start     Start a stopped containers                    # 启动容器
    stop      Stop a running containers                     # 停止容器
    tag       Tag an image into a repository                # 给源中镜像打标签
    top       Lookup the running processes of a container   # 查看容器中运行的进程信息
    unpause   Unpause a paused container                    # 取消暂停容器
    version   Show the docker version information           # 查看 docker 版本号
    wait      Block until a container stops, then print its exit code    # 截取容器停止时的退出状态值
```

#### docker run
```shell
[root@docker ~]# docker run --help
-d, --detach=false 指定容器运行于前台还是后台，默认为false 
-i, --interactive=false 打开STDIN，用于控制台交互 
-t, --tty=false 分配tty设备，该可以支持终端登录，默认为false 
-u, --user="" 指定容器的用户 
-a, --attach=[] 登录容器（必须是以docker run -d启动的容器）
-w, --workdir="" 指定容器的工作目录 
-c, --cpu-shares=0 设置容器CPU权重，在CPU共享场景使用 
-e, --env=[] 指定环境变量，容器中可以使用该环境变量 
-m, --memory="" 指定容器的内存上限 
-P, --publish-all=false 指定容器暴露的端口 
-p, --publish=[] 指定容器暴露的端口 
-h, --hostname="" 指定容器的主机名 
-v, --volume=[] 给容器挂载存储卷，挂载到容器的某个目录 
--volumes-from=[] 给容器挂载其他容器上的卷，挂载到容器的某个目录
--cap-add=[] 添加权限，权限清单详见：http://linux.die.net/man/7/capabilities 
--cap-drop=[] 删除权限，权限清单详见：http://linux.die.net/man/7/capabilities 
--cidfile="" 运行容器后，在指定文件中写入容器PID值，一种典型的监控系统用法 
--cpuset="" 设置容器可以使用哪些CPU，此参数可以用来容器独占CPU 
--device=[] 添加主机设备给容器，相当于设备直通 
--dns=[] 指定容器的dns服务器 
--dns-search=[] 指定容器的dns搜索域名，写入到容器的/etc/resolv.conf文件 
--entrypoint="" 覆盖image的入口点 
--env-file=[] 指定环境变量文件，文件格式为每行一个环境变量 
--expose=[] 指定容器暴露的端口，即修改镜像的暴露端口 
--link=[] 指定容器间的关联，使用其他容器的IP、env等信息 
--lxc-conf=[] 指定容器的配置文件，只有在指定--exec-driver=lxc时使用 
--name="" 指定容器名字，后续可以通过名字进行容器管理，links特性需要使用名字 
--net="bridge" 容器网络设置:
bridge 使用docker daemon指定的网桥 
host //容器使用主机的网络 
container:NAME_or_ID >//使用其他容器的网路，共享IP和PORT等网络资源 
none 容器使用自己的网络（类似--net=bridge），但是不进行配置 
--privileged=false 指定容器是否为特权容器，特权容器拥有所有的capabilities 
--restart="no" 指定容器停止后的重启策略:
no：容器退出时不重启 
on-failure：容器故障退出（返回值非零）时重启 
always：容器退出时总是重启 
--rm=false 指定容器停止后自动删除容器(不支持以docker run -d启动的容器) 
--sig-proxy=true 设置由代理接受并处理信号，但是SIGCHLD、SIGSTOP和SIGKILL不能被代理

#从镜像启动一个容器
会直接进入到容器，并随机生成容器ID和名称
[root@docker ~]# docker run --name b1 -it busybox:latest  ---> --name 自定义容器名称
Unable to find image 'busybox:latest' locally   ---> 本地没有 就从默认的公有仓库中拉取
latest: Pulling from library/busybox
ee153a04d683: Pull complete 
Digest: sha256:9f1003c480699be56815db0f8146ad2e22efea85129b5b5983d0e0fb52d9ab70
Status: Downloaded newer image for busybox:latest
/ #                            ---> 直接进入到容器
/ # ps    ---> 默认情况下进入的是sh
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    8 root      0:00 ps
/ # ls /
bin   dev   etc   home  proc  root  sys   tmp   usr   var    ---> 都是busybox的别名
/ # hostname     ---> 随机生成的容器ID
02379c036216

#退出容器不注销
/ # exit     ---> exit或者ctrl + d 退出容器  容器是退出的  
[root@docker ~]# docker ps   ---> 显示正在运行的容器
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[root@docker ~]# docker container ls  --->列出正在运行的容器
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[root@docker ~]# docker ps -a    ---> 显示所有容器，包括当前正在运行以及已经关闭的所有容器
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                      PORTS               NAMES
02379c036216        busybox:latest      "sh"                4 minutes ago       Exited (0) 14 seconds ago                       b1
[root@docker ~]# docker container ls -a   ---> 显示所有容器，包括当前正在运行以及已经关闭的所有容器
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                          PORTS               NAMES
02379c036216        busybox:latest      "sh"                38 minutes ago      Exited (1) About a minute ago                       b1
[root@docker ~]# docker exec -it b1 sh   ---> 进入容器
Error response from daemon: Container 02379c036216092f07c46c58361c2da6b5df7d15a733b3c39cfc984cd2dcf314 is not running
[root@docker ~]# docker start b1    ---> 启动暂定的容器
b1
[root@docker ~]# docker exec -it b1 sh
/ # hostname 
02379c036216
#容器暂停不删除是对容器原有资源信息无影响的
ctrl +p +q 操作，退出容器，容器依然运行
[root@docker ~]# docker exec -it b1 sh
/ # read escape sequence  
[root@docker ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
02379c036216        busybox:latest      "sh"                17 minutes ago      Up 4 minutes                            b1
```

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

#### 基础版本信息命令

```shell
### info
显示 Docker 系统信息，包括镜像和容器数。

# 查看docker系统信息。  
docker info  


#version
#显示 Docker 版本信息。
docker version
```

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

