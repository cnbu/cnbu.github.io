# 1 ansible-2.5（pip部署）

#### Ansible安装

```shell
##安装Python 3.6 （略）
pip install virtualenv
yum install git nss curl -y
# 创建用户（日常工作，同Jenkins共用）
useradd deploy
su - deploy
# 初始化ptyhon环境
virtualenv -p /usr/local/python3.6/bin/python3.6 .py3-a2.5-env
# 安装基础包

# 加速配置
mkdir ~/.pip/ && cat >> ~/.pip/pip.conf <<EOF
    [global]
    index-url = https://mirrors.aliyun.com/pypi/simple/

    [install]
    trusted-host=mirrors.aliyun.com
EOF
# 切换环境（每次都要）
source  /home/deploy/.py3-a2.5-env/bin/activate
# 安装ansible
pip install ansible
```

# 2 基于Roles的Playbook

#### 目录框架

```shell
inventory/  # Server清单
    testenv
    productenv
roles/     # 任务列表
    testbox/  # 项目名称orAPP名称
        tasks/
            main.yml # 任务主文件
deploy.yml # 任务入口，调度roles任务
```

#### 目录约定

```shell
# 主配置文件
tasks - contains the main list of tasks to be executed by the role.
# 处理任务
handlers - contains handlers, which may be used by this role or even anywhere outside this role.
# 默认变量，我理解是全局
defaults - default variables for the role (see Variables for more information).
# 自定义变量，我理解是临时
vars - other variables for the role (see Variables for more information).
# 要拷贝的文件（不需要更改）
files - contains files which can be deployed via this role.
# 要使用的模版（需要更改）
templates - contains templates which can be deployed via this role.
# 元数据（待研究）
meta - defines some meta data for this role. See below for more details.
```

#### 基础语法(多行缩进)

```
对齐很重要，不要混用tab和空格
建议用2个空格分级
字符串不一定需要双引号
允许空行，增加可读性
连续项目使用"-"
map结构中的key/value用":"分割，：后面有空格
"---"开头，顶行首写
k/v值大小写敏感
最少需要name:task，一个name只能有一个task
```

#### 在Playbook中设置变量

```yaml
---
- hosts: all
  gather_facts: no
  vars:
      user: "ding1"
  tasks:
    - name: create user
      user: name="{{user}}"

    - name: echo user
      debug: msg="{{user}}"
```

#### 在调用文件中的变量

vars.yml

```yaml
---
- hosts: all
  gather_facts: no
  vars_files:
    - var.yml
  tasks:
    - name: create user
      user: name="{{user}}" 

    - name: echo user
      debug: msg="{{user}}---->{{sex}}"
```

var.yml

```
---
user: ding
sex: man
```

#### 在Roles的vars和defaults中设置变量

使用一个在playbook中注册的变量

```yaml
---
- hosts: all
  gather_facts: no
  tasks:
    - name: register vars
      shell: hostname
      register: info
    - name: display vars
      debug: msg="{{info.stdout}}"
```

# 3 循环语法

#### 循环语法（列表）

一次安装多个软件

```yaml
---
- hosts: all
  tasks:
    - name: debug loops
      debug: msg="{{item}}"
      with_items:
        - one
        - tow
        - three
        - four
```

#### 循环语法（字典）

```yaml
---
- hosts: all
  tasks: 
    - name: debug dict
      debug: msg="name-->{{item.key}} value-->{{item.value}}"
      with_items:
        - {key: "one", value: "v1"}
        - {key: "two", value: "v2"}
```

#### 嵌套循环

注意关键字变化

```yaml
---
- hosts: all
  tasks:
    - name: debug dict
      debug: msg="name->{{item[0]}} value->{{item[1]}}"
      with_nested:
        - ['A', 'B']
        - ['a', 'b', 'c']
```

#### 散列循环（失败）

```yaml
---
- hosts: all
  gather_facts: no
  vars:
    user:
      sheng:
        name: ding
        shell: bash
      ceshi:
        name: zhang
        shell: zsh

  tasks:
    - name: debug shell
      debug: msg="{{item.key}}"
      with_dict: "{{user}}"
```

#### 文件循环

```yaml
---
- hosts: all
  gather_facts: false
  tasks:
    - name: loop file
      debug: msg="{{item}}"
      with_fileglob: 
        - /etc/ansible/playbook/*.yml
```

#### 判断语句

判断文件第一行是不是abc

```yaml
---
- hosts: all
  gather_facts: no
  tasks: 
    - name: debug loop judgement
      shell: cat /tmp/abc
      register: hosts
      until: hosts.stdout.startswith("abc")
      retries: 5
      delay: 5
```

#### 变量转换、

```shell
#变量的定义
age1是一个int类型的变量，例如：age1: 21
age2是一个string类型的变量,例如：age2: '21'
married和married2是一个布尔类型的变量，例如：married: True 或 married2: true
married3是一个string类型的变量，例如：married3: 'true'

#变量类型转换
age1|string 可以变int转换为string，然后进行比较运算
age2|int 可以把string转为为int，然后进行比较运算
married|string 可以把布尔类型转换为string，然后进行比较运算。
married2|string 可以把布尔类型转换为string，然后进行比较运算。
```
