```
今天我们的一台数据库服务器，业务研发反馈tp999会不时的彪高，我们查询了各种指标，发现网络重传比较高，同事cpu的load比较高，但是统一宿主机上其他的docker没有重传，因此不是网卡的问题，通过dmesg，发现有cpu降频的相关日志。发现是cpu降频引起的。 查看，系统设置的是非高性能模式。需要设置成高性能模式。

相关日志如下：perf: interrupt took too long (166702 > 165147), lowering kernel.perf_event_max_sample_rate to 1000

一般服务器的CPU都支持自动睿频，而服务器的CPU一般默认运行于ondemand模式，会有中断开销，睿频的时候提升下降也是有额外的开销，特别是对于一些低端cpu比如C2350,C2338,N2800这些低价独服的CPU，影响更大。

模式说明：
performance          运行于最大频率
powersave            运行于最小频率
userspace            运行于用户指定的频率
ondemand             按需快速动态调整CPU频率， 一有cpu计算量的任务，就会立即达到最大频率运行，空闲时间增加就降低频率
conservative         按需快速动态调整CPU频率， 比 ondemand 的调整更保守
schedutil            基于调度程序调整 CPU 频率


Centos7 设置方法：

# yum install -y cpupowerutils
# cpupower frequency-info
# cat /proc/cpuinfo
# cpupower frequency-set -g performance

查看方式，可以比较前后的设置
# cat /proc/cpuinfo | grep MHz

Debian设置方法
安装工具
apt install cpufrequtils

编辑 /etc/default/cpufrequtils 如不存在则创建，添加条目
GOVERNOR=”performance”

重启生效
systemctl restart cpufrequtils
```
