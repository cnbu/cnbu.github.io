#### 查看程序所有进程pid及其启动目录

```
pidof [程序名] | xargs pwdx

例如：
获取linux服务器所有java进程及名称
pidof java | xargs pwdx
```

#### Jdk环境变量

```
export JAVA_HOME=/usr/java/jdk1.8.0_201-amd64
export JRE_HOME=${JAVA_HOME}/jre 
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib 
export PATH=${JAVA_HOME}/bin:$PATH
```

#### jvm 参考

```java
-server 
-XX:+UseG1GC 
-XX:MaxGCPauseMillis=50 
-Xms1G -Xmx1G
-XX:MetaspaceSize=128m JAVA_MAXMETA_SIZE="512m" 
-XX:LargePageSizeInBytes=128m 
-XX:+ParallelRefProcEnabled 
-XX:+PrintAdaptiveSizePolicy 
-XX:+UseFastAccessorMethods 
-XX:+TieredCompilation 
-XX:+ExplicitGCInvokesConcurrent 
-XX:AutoBoxCacheMax=20000 
-XX:+UnlockExperimentalVMOptions 
-XX:+UseCGroupMemoryLimitForHeap 
-XX:+PerfDisableSharedMem 
-verbosegc  -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:/app/logs/gc.log"
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/app/logs/oom-`date +%Y%m%d%H%M%S`.hprof"
```

jar命令
```
#查看 jar 包中的文件列表，并进行重定向
jar -tvf a.jar > a.txt 


#更新文件到 jar 中，目录需对应
jar -uf a.jar com/a.class
```
