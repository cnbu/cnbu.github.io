过滤nginx日志

```
cat 2021-03-12-all.log | awk '$8 >="500" && $8<="600"' > /tmp/log23.txt
```

### 上传文件限制
```
client_body_buffer_size 配置请求体缓存区大小, 不配的话，

client_body_temp_path 设置临时文件存放路径。只有当上传的请求体超出缓存区大小时，才会写到临时文件中

client_max_body_size 设置上传文件的最大值

所以查出来，问题出现的原因是

1.文件大小超过了client_body_buffer_size

2.client_body_temp_path的临时文件路径居然没有写权限

以上两个原因导致了返回500错误。

如果上传文件大小超过client_max_body_size时，会报413 entity too large的错误。

原因知道了，修正就简单了。

1.client_body_buffer_size 尽量设置的大点，这是基于速度的考虑，如果因为设置的过小，导致上传的文件老要写磁盘，那速度就太慢了。

2.client_body_temp_path 路径要有可写权限，这个是明显的错误了。改正了就好

3.client_max_body_size 设置上传文件的最大值，这个是基于安全的考虑，我们认为正常用户不会或者基本不会上传太大的文件。

可以设置为client_max_body_size 100m;  或者按照自己的业务来设置这个值
```
### 匹配规则
```
localtion / {
 # 所有请求都匹配以下规则
 # 因为所有的地址都以 / 开头，所以这条规则将匹配到所有请求
 # xxx 你的配置写在这里
}
 
location = / {
 # 精确匹配 / ，后面带任何字符串的地址都不符合
}
 
localtion /api {
 # 匹配任何 /api 开头的URL，包括 /api 后面任意的, 比如 /api/getList
 # 匹配符合以后，还要继续往下搜索
 # 只有后面的正则表达式没有匹配到时，这一条才会采用这一条
}
 
localtion ~ /api/abc {
 # 匹配任何 /api/abc 开头的URL，包括 /api/abc 后面任意的, 比如 /api/abc/getList
 # 匹配符合以后，还要继续往下搜索
 # 只有后面的正则表达式没有匹配到时，这一条才会采用这一条
}
以/ 通用匹配, 如果没有其它匹配,任何请求都会匹配到
=开头表示精确匹配
如 A 中只匹配根目录结尾的请求，后面不能带任何字符串。
^~ 开头表示uri以某个常规字符串开头，不是正则匹配
~ 开头表示区分大小写的正则匹配;
~* 开头表示不区分大小写的正则匹配
```

### 日志格式
```
log_format  main  '$clientRealIp - $remote_user $year-$month-$day $hour:$minute:$second $host "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" $upstream_addr $upstream_status $upstream_response_time $request_time $scheme';
    access_log  /var/log/nginx/access.log  main;
```
