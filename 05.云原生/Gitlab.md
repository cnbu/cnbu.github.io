# 1 常用命令

##### 相关目录

```shell
#配置文件目录
/etc/gitlab
#运行 pid 目录
/run/gitlab
#安装目录
/opt/gitlab
#数据目录
/var/opt/gitlab
#日志目录
/var/log/gitlab
```

```shell
#用于启动控制台进行特殊操作，比如修改管理员密码、打开数据库控制台
gitlab-rails dbconsole
#数据库命令行
gitlab-psql
#数据备份恢复等数据操作
gitlab-rake
#客户端命令行操作行
gitlab-ctl
#停止 gitlab
gitlab-ctl stop
#启动 gitlab
gitlab-ctl start
#重启 gitlab
gitlab-ctl restar 
#查看组件运行状态 
gitlab-ctl status
#查看某个组件的日志
gitlab-ctl tail nginx
#修改完配置文件要执行此操作
gitlab-ctl reconfigure
```

# 2 Gitlab-CE-13.3.4（单节点）

##### 脚本安装

```shell
#!/bin/bash
# 访问设置
external_url="http://192.168.10.10:9000"
# 邮箱设置
smtp_user_name="xxxxxxxxxxx@163.com"
smtp_password="xxxxxxxx"
smtp_address="smtp.163.com"
smtp_port=465
smtp_domain="163.com"
# 发件人显示名称,字符串中间不能有空格
gitlab_email_display_name="Admin_Gitlab"
# 软件版本
software="gitlab-ce-14.4.4-ce.0.el7.x86_64.rpm"
soft_url="https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/"
 
# 如下自动执行
softname="gitlab-ce"
## 安装依赖
echo "安装依赖"
yum install -y curl policycoreutils-python openssh-server perl postfix wget > /dev/null 2>&1
systemctl enable postfix > /dev/null 2>&1
systemctl start postfix > /dev/null 2>&1
echo "安装依赖成功"
# 下载软件
if [ ! -f ${software} ];then
	echo "正在下载安装包：${software}"
    wget --no-check-certificate ${soft_url}${software} > /dev/null 2>&1
    [ ! -f ${software} ] && echo "${software} download failure,exit" && exit
else
	echo "安装包已存在：${software}"
fi
 
# 检查并安装软件
rpm -q ${softname} > /dev/null 2>&1
if [ "$?" -ge 1 ];thenecho "${software} 正在安装中..."
    yum -y install ${software} > /dev/null 2>&1
    rpm -q ${softname} > /dev/null 2>&1
    [ $? -ge 1 ] && echo "${software} 安装失败,退出整个程序" && exitecho "${software} 安装成功"
else
	echo "${softname} 已经安装"
fi
 
# 设置配置文件
sed -i 's#^external_url.*$#external_url '\'''${external_url}''\''#g' /etc/gitlab/gitlab.rb
 
# 设置邮件
# gitlab_email_enabled
sed -i 's/^.*gitlab_rails\['\''gitlab_email_enabled'\''\].*$/gitlab_rails['\''gitlab_email_enabled'\''] = true/g' /etc/gitlab/gitlab.rb
 
# gitlab_email_from
sed -i 's/^.*gitlab_rails\['\''gitlab_email_from'\''\].*$/gitlab_rails['\''gitlab_email_from'\''] = '\'''${smtp_user_name}''\''/g' /etc/gitlab/gitlab.rb
 
# gitlab_email_display_name
sed -i 's/^.*gitlab_rails\['\''gitlab_email_display_name'\''\].*$/gitlab_rails['\''gitlab_email_display_name'\''] = '\'''${gitlab_email_display_name}''\''/g' /etc/gitlab/gitlab.rb
 
# smtp_enable
sed -i 's/^.*gitlab_rails\['\''smtp_enable'\''\].*$/gitlab_rails['\''smtp_enable'\''] = true/g' /etc/gitlab/gitlab.rb
 
# smtp_address
sed -i 's/^.*gitlab_rails\['\''smtp_address'\''\].*$/gitlab_rails['\''smtp_address'\''] = "'${smtp_address}'"/g' /etc/gitlab/gitlab.rb
 
# smtp_port
sed -i 's/^.*gitlab_rails\['\''smtp_port'\''\].*$/gitlab_rails['\''smtp_port'\''] = '${smtp_port}'/g' /etc/gitlab/gitlab.rb
 
# smtp_user_name
sed -i 's/^.*gitlab_rails\['\''smtp_user_name'\''\].*$/gitlab_rails['\''smtp_user_name'\''] = "'${smtp_user_name}'"/g' /etc/gitlab/gitlab.rb
 
# smtp_password
sed -i 's/^.*gitlab_rails\['\''smtp_password'\''\].*$/gitlab_rails['\''smtp_password'\''] = "'${smtp_password}'"/g' /etc/gitlab/gitlab.rb
 
# smtp_domain
sed -i 's/^.*gitlab_rails\['\''smtp_domain'\''\].*$/gitlab_rails['\''smtp_domain'\''] = "'${smtp_domain}'"/g' /etc/gitlab/gitlab.rb
 
# smtp_authentication
sed -i 's/^.*gitlab_rails\['\''smtp_authentication'\''\].*$/gitlab_rails['\''smtp_authentication'\''] = "login"/g' /etc/gitlab/gitlab.rb
 
# smtp_enable_starttls_auto
sed -i 's/^.*gitlab_rails\['\''smtp_enable_starttls_auto'\''\].*$/gitlab_rails['\''smtp_enable_starttls_auto'\''] = true/g' /etc/gitlab/gitlab.rb
 
# smtp_tls
sed -i 's/^.*gitlab_rails\['\''smtp_tls'\''\].*$/gitlab_rails['\''smtp_tls'\''] = true/g' /etc/gitlab/gitlab.rb
 
# 显示结果
echo "修改配置后的结果"
cat /etc/gitlab/gitlab.rb |grep -E "${external_url}|gitlab_email_enabled|gitlab_email_from|gitlab_email_display_name|smtp_enable|smtp_address|smtp_port|smtp_user_name|smtp_password|smtp_domain|smtp_authentication|smtp_enable_starttls_auto|smtp_tls"
 
# 启动gitlab
echo "配置gitlab：gitlab-ctl reconfigure"
gitlab-ctl reconfigure > /dev/null 2>&1
 
# 密码输出
echo "URL: ${external_url}"
echo "账号：root"
cat /etc/gitlab/initial_root_password |grep ^Password
```

##### GitLab部署（yum）

```shell
# 清华源下载
https://mirror.tuna.tsinghua.edu.cn/help/gitlab-ce/

# 创建yum源
cat >/etc/yum.repos.d/gitlab.repo << 'EOF'
[gitlab-ce]
name=Gitlab CE Repository
baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el$releasever/
gpgcheck=0
enabled=1
EOF
# 安装
yum list gitlab-ce --showduplicates
yum install gitlab-ce-13.3.4-ce.0.el7
```

##### 配置初始化

```shell
# 使用证书（直接配置）
阿里云免费证书，下载nginx，按照文档，找到对应配置开启即可
# 修改nginx域名(使用https)
external_url 'https://gitlab.noteshare.cn'
nginx['enable'] = true
nginx['client_max_body_size'] = '250m'
nginx['redirect_http_to_https'] = true
nginx['redirect_http_to_https_port'] = 80
nginx['ssl_certificate'] = "/etc/gitlab/ssl/17kb.com.pem"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/17kb.com.key"
nginx['ssl_ciphers'] = "ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;"
nginx['ssl_prefer_server_ciphers'] = "on"
nginx['ssl_protocols'] = "TLSv1 TLSv1.1 TLSv1.2 TLSv1.3"
nginx['ssl_session_timeout'] = "5m"

# 使用证书（代理配置）
代理配置证书，gitlab上配置端口跳转
# 修改nginx域名(使用http)
external_url 'http://gitlab.noteshare.cn'
nginx['enable'] = true
nginx['client_max_body_size'] = '250m'
nginx['redirect_http_to_https'] = true
nginx['redirect_http_to_https_port'] = 80

#关闭用户注册
gitlab-cli reconfigure
gitlab-ctl restart
启动需要一段时间，这段时间会是502页面，等1分钟左右再刷新

#修改root密码
管理中心->用户->编辑administratur
```

##### 项目初始化

```
创建group（通常按产品线或语言来创建）
创建user，设置密码
到group中增加user，设置权限
```

##### git基本流程

```
添加提交文件-->添加注释信息-->commit（提交到暂存区）-->推送远程服务器（需要有权限，否则需要推送指定分支，平台提交合并申请）-->开发新项目->拉取-->从服务器更新到本地仓库
```

管理工作有哪些？

```
1. 系统资源监察
2. 权限管控
3. 备份
```

开发人员在公司办公，用svn和git有区别么？

```
1.分布式基本用不上（异地vpn解决）
2.权限管理限制svn更严格（在一个项目里进行控制）
3.备份git自带工具，svn使用脚本异地备份
```

# 3. git基础

#### git基本流程

```
添加提交文件-->添加注释信息-->commit（提交到本地仓库）-->推送（推送到服务器）-->拉取-->从服务器更新到本地仓库
```

#### 全局初始化配置

```
git config --global user.name 'dinghe'
git config --global user.email 'dinghe_1985@126.com'
# 保证邮箱可以正常收发邮件
```

#### 单个项目

```
git config --local user.name '丁贺'
git config --local user.mail 'ding.he@onlylady.com'
```

#### 查看变量

```
git config --list --global
git config --list --local
```

#### 创建第一个Git仓库

#### 获取完整代码

```
git clone
```

#### 再次拉取最新代码

```
git pull
```

#### Git提交代码流程

本地文件->本地缓存->本地仓库-> 远程仓库

```shell
#本地文件添加到缓存
git add * 或 git add file1.py
#提交到本地仓库，并填写注释
git commit -m "first commit"
#推送到远程仓库
git push
```

# 4 gitlb数据备份恢复

#### 停止 gitlab  数据服务

```
root@s1:~# gitlab-ctl stop unicorn
ok: down: unicorn: 1s, normally up
root@s1:~# gitlab-ctl stop sidekiq
ok: down: sidekiq: 0s, normally up
```

#### 手动备份数据

```shell
#在任意目录即可备份当前 gitlab 数据
gitlab-rake gitlab:backup:create
#备份完成后启动 gitlab
gitlab-ctl start
```

#### 查看要 恢复的文件

```shell
# Gitlab 数据备份目录，需要使用命令备份的
/var/opt/gitlab/backups/
#nginx 配置文件
/var/opt/gitlab/nginx/conf
#gitlab 配置文件
/etc/gitlab/gitlab.rb
#key 文件
/etc/gitlab/gitlab-secrets.json
```

#### 执行恢复

```shell
#恢复数据之前停止服务
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq
#恢复
gitlab-rake gitlab:backup:restore BACKUP=备份文件名
```

#### 启动服务

```
gitlab-ctl start sidekiq
ok: run: sidekiq: (pid 16859) 0s

gitlab-ctl start unicorn
ok: run: unicorn: (pid 16882) 1s
```

```shell
# 手动备份
# 自动备份到：/var/opt/gitlab/backups/ :配置文件：gitlab_rails['backup_path']
# 默认保留7天：604800	:配置文件：gitlab_rails[‘backup_keep_time’]
# 如下手动备份命令
gitlab-rake gitlab:backup:create
# 输出：1639315073_2021_12_12_14.4.4_gitlab_backup.tar
# 或者查看目录：/var/opt/gitlab/backups/
 
# 自动备份，使用系统自带的crontab
crontab -e
0 3 * * * /opt/gitlab/bin/gitlab-rake gitlab:backup:create CRON=1
```
