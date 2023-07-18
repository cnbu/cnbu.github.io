#### Go 有四类数据类型：
- 基本类型：数字、字符串和布尔值
- 聚合类型：数组和结构
- 引用类型：指针、切片、映射、函数和通道
- 接口类型：接口



#### 常用第三方包
```
gin: web框架库
gorm: 开发人员友好的ORM库
gin-swagger: 接口文档
logrus：日志库
cobra：编写命令行
viper：处理配置信息
path/filepath：兼容操作系统的文件路径操作
io：提供了 I/O 原语的基本接口
os：为操作系统功能提供了一个独立于平台的接口
strings: 实现了简单的函数来操作 UTF-8 编码的字符串
net/http: 提供 HTTP 客户端和服务器实现
time：提供测量和显示时间的功能
mime/multipart: 实现 MIME 多部分解析，如 RFC 2046 中所定义
strconv: 实现与基本数据类型的字符串表示之间的转换
reflect: 实现运行时反射，允许程序操作任意类型的对象
regexp: 正则表达式搜索
github.com/robfig/cron: 定时任务
github.com/gin-contrib/cors: 启用 CORS 支持的 Gin 中间件/处理程序
encoding/json: 实现了RFC 7159 中定义的 JSON 编码和解码
sort: 提供了用于对切片和用户定义的集合进行排序的原语
```
#### 
#### vscode安装插件

```shell
#尝试下载了镜像网站 github.com/golang 里面的 tools 也不靠谱
#因为安装时总会缺少非常多的插件，导致无法简单地执行 go install golang.org/x/tools/gopls

#最终解决方案是修改代理，然后在 cmd 下面输入：
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct

#修改代理到国内的go，然后在 cmd 重新获取即可成功：
go get -v golang.org/x/tools/gopls

此时，顺便可以将 vocode 的其他必要插件都安装一下，因为改了代理所以可以非常顺利地完成安装。

#最后需要关掉 GO111MODULE，否则运行任何代码都会提示缺少 main.go：
go env -w GO111MODULE=off

go env -w GO111MODULE=auto
```
#### 项目结构
```
> cmd:  程序初始化
> config: 配置相关
> controller: 服务入口，负责处理路由，参数校验，请求转发。
> db: 数据库配置相关
> docs: swagger接口文档
> middleware: 第三方调用,获取数据
> model: 数据结构
> pkg: 公共组件
> router: 路由
> scripts: 脚本
> service：逻辑（服务）层，处理业务逻辑
> go.mod: 依赖
> Dockerfile: 部署
> main.go: 程序主入口
> README.md: 项目浅析
> test: 测试
```


