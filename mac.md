```shell
取消自动开机：
sudo nvram AutoBoot=%00

重新启用自动开机：
sudo nvram AutoBoot=%03

开启开机音效：
sudo nvram BootAudio=%01

取消开机音效：
sudo nvram BootAudio=%00
sudo pmset restoredefaults

先关闭网络唤醒等，10分钟关闭显示器，开启深睡眠模式
sudo pmset -a womp 0 darkwakes 0 lessbright 0 halfdim 0 displaysleep 10 disksleep 10 autopoweroff 1 standby 1

无操作2小时sleep，sleep后50小时standby，sleep后51小时poweroff，电池模式下为21分钟sleep
sudo pmset -a sleep 110 autopoweroffdelay 183600 standbydelay 180000
sudo pmset -b sleep 11
```
# 国内快速安装Homebrew
```shell
#苹果电脑 常规安装脚本（推荐 完全体 几分钟安装完成）
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
#极速安装脚本
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)" speed
#卸载脚本
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/HomebrewUninstall.sh)"
```
