#### 创建用户
```
create user 'bd_sync'@'%' identified by 'Q0GkUtn6p4';
```
#### 修改用户密码
```
ALTER USER 'bd_sync'@'%' IDENTIFIED BY '123456';
```
#### 创建库
```
create database bd_ext;
```
#### 授权所有
```
grant all on *.* to 'bd_sync'@'%';
```
#### 查看所有授权
```
#所有用户授权
SHOW ALL GRANTS;

#查看单个用户
SHOW GRANTS FOR jack@'%';

#查看当前用户权限
SHOW GRANTS;
```
