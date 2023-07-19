
###### 下载node.js 安装包

```
wget https://nodejs.org/dist/v10.13.0/node-v10.13.0-linux-x64.tar.gz
```

###### 解压node.js 安装包

```
tar xf node-v10.13.0-linux-x64.tar.gz
```

###### 创建链接

```
ln -s ~/node-v10.13.0-linux-x64/bin/node /usr/bin/node
ln -s ~/node-v10.13.0-linux-x64/bin/npm /usr/bin/npm
```

###### 查看版本号

```
node -v
npm -v
```

使用npm安装elasticdump，执行如下命令。

```
npm install elasticdump
```

#### 进入elasticdump脚本目录

```
cd /root/node-v10.13.0-linux-x64/lib/node_modules/elasticdump/bin
```

#### 方法一：索引数据导出为文件

```shell
# 导出索引Mapping数据
./bin/elasticdump \
  --input=http://es实例IP:9200/index_name/index_type \
  --output=/data/my_index_mapping.json \    # 存放目录
  --type=mapping 
# 导出索引数据
./bin/elasticdump \
  --input=http://es实例IP:9200/index_name/index_type \
  --output=/data/my_index.json \
  --type=data
```

#### 方法二：索引数据文件导入至索引

```shell
# Mapping 数据导入至索引
./bin/elasticdump \
  --output=http://es实例IP:9200/index_name \
  --input=/home/indexdata/roll_vote_mapping.json \ # 导入数据目录
  --type=mapping
# ES文档数据导入至索引
./bin/elasticdump \
  --output=http:///es实例IP:9200/index_name \
  --input=/home/indexdata/roll_vote.json \ 
  --type=data
```
