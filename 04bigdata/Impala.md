### Apache Impala

- impla是个实时的sql查询工具，类似于hive的操作方式，只不过执行的效率极高，号称当下大数据生态圈中执行效率最高的sql类软件
- impala来自于cloudera，后来贡献给了apache
- impala工作底层执行依赖于hive  与hive共用一套元数据存储。在使用impala的时候，必须保证hive服务是正常可靠的，至少metastore开启。
- impala最大的跟hive的不同在于 不在把sql编译成mr程序执行 编译成执行~~计划数~~（勘误：计划树）。
- impala的sql语法几乎兼容hive的sql语句。

---

impala是一个适用于实时交互查询的sql软件 hive适合于批处理查询的sql软件。通常是两个互相配合。

- impala  可以集群部署 
   - Impalad(impala server):可以部署多个不同机器上，通常与datanode部署在同一个节点 方便数据本地计算，负责具体执行本次查询sql的impalad称之为Coordinator。每个impala server都可以对外提供服务。
   - impala state store:主要是保存impalad的状态信息 监视其健康状态
   - impala catalogd :metastore维护的网关 负责跟hive 的metastore进行交互  同步hive的元数据到impala自己的元数据中。
   - CLI:用户操作impala的方式（impala shell、jdbc、hue）
- impala 查询处理流程 
   - impalad分为java前端（接受解析sql编译成执行计划树），c++后端（负责具体的执行计划树操作）
   - impala sql---->impalad（Coordinator）---->调用java前端编译sql成计划树------>以Thrift数据格式返回给C++后端------>根据执行计划树、数据位于路径（libhdfs和hdfs交互）、impalad状态分配执行计划 查询----->汇总查询结果----->返回给java前端---->用户cli
   - 跟hive不同就在于整个执行中已经没有了mapreduce程序的存在

---

-  impala集群安装规划 
   - node-3 ：impalad 、impala state store、impala catalogd、impala-shell
   - node-2：impalad
   - node-1：impalad
-  impala安装 
   - impala没有提供tar包 只有rpm包  这个rpm包只有cloudera公司
   - 要么自己去官网下载impala rpm包和其相关的依赖  要么自己制作本地yum源
   - 特别注意本地yum源的安装 需要Apache server对外提供web服务 使得各个机器都可以访问下载yum源
   - 在指定的每个机器上根据规划 yum安装指定的服务
   - 保证hadoop hive服务正常，开启相关的服务 
      - hive   metastore  hiveserver2
      - hadoop hdfs-site.xml  开启本地读取数据的功能
      - 要把配置文件scp给其他机器 重启
   - 修改impala配置文件
   - 修改bigtop  指定java路径
   - 根据规划分别启动对应的impala进程
   - 如果出错  排查的依据就是去，日志默认都在/var/log/impala
-  impala集群的启动关闭 
   -  主节点  按照顺序启动以下服务 
```
service impala-state-store start
service impala-catalog start
service impala-server start
```
 

   -  从节点 
```
service impala-server start
```
 

   -  如果需要关闭impala  把上述命令中start 改为stop 
   -  通过ps -ef|grep impala 判断启动的进程是否正常 如果出错 日志是你解决问题的唯一依据。 
```
/var/log/impala
```
 
