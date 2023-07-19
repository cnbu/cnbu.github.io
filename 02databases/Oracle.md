# 常用命令

```shell
#查看版本
select * from v$version;

#oracle查看所有的表空间
select name from v$tablespace;

#
select * from Dba_Tablespaces
#登录创建
create user szymou identified by 123;   --创建用户密码为123

#可以通过此命令查看当前的表空间和位置
select tablespace_name,file_id,bytes/1024/1024,file_name from dba_data_files order by file_id;


#切换用户
conn szymou/123;

#授权命令
grant dba to admin2;                      --赋予管理员的权限
grant create session to admin3;           --赋予登陆的权限
grant create any table to admin3;         --给admin3创建表的权限
 
grant create table to qinfudian;           --授予创建表的权限
grant drop any table to qinfudian;         --授予删除表的权限
grant insert any table to qinfudian;       --插入表的权限
grant update any table to qinfudian;       --修改表的权限
grant select any table to qinfudian;       --查询表的权限
 
revoke create table from admin3;           --去除建表的权限
grant unlimited tablespace to admin3;      --赋予表空间权限

#查看权限
select * from user_sys_privs;    //查看当前用户所有权限
select * from user_tab_privs;    //查看所用用户对表的权限



#刷盘日志
alter system switch logfile;

#查看所有用户：
select * from dba_users; 
select * from all_users; 
select * from user_users;

#查看用户或角色系统权限(直接赋值给用户或角色的系统权限)：
select * from dba_sys_privs; 
select * from user_sys_privs; (查看当前用户所拥有的权限)

#查看角色(只能查看登陆用户拥有的角色)所包含的权限
select * from role_sys_privs;

#查看用户对象权限：
select * from dba_tab_privs; 
select * from all_tab_privs; 
select * from user_tab_privs;

#查看当前用户名
show user
select user from dual


#查看所有角色： 
select * from dba_roles;

#查看用户或角色所拥有的角色：
select * from dba_role_privs; 
select * from user_role_privs;

#查看哪些用户有sysdba或sysoper系统权限(查询时需要相应权限)
select * from V$PWFILE_USERS

#SqlPlus中查看一个用户所拥有权限
SQL>select * from dba_sys_privs where grantee='username'; 其中的username即用户名要大写才行。
比如： SQL>select * from dba_sys_privs where grantee='TOM';

#Oracle删除指定用户所有表的方法
select 'Drop table '||table_name||';' from all_tables where owner='要删除的用户名(注意要大写)';

#删除用户
drop user user_name cascade; 如：drop user SMCHANNEL CASCADE

#获取当前用户下所有的表：
select table_name from user_tables;

#删除某用户下所有的表数据:
select 'truncate table ' || table_name from user_tables;


#解锁scott用户，并修改密码为 password
alter user scott account unlock;
alter user scott identified by password;
```


#### 启动oracle
```shell
1.连接到liunx

2.切换到oracle
su - oracle

3.启动监听
lsnrctl start

4.login到sqlplus
sqlplus /nolog

5.连接
conn /as sysdba

6.启动数据库
startup

进入到sqlplus
关闭数据库
shutdown immediate

退出sqlplus
exit

关闭监听
lsnrctl stop
```
# ogg

```
要重启，su ogg 
./ggsci
stop mgr
start mgr
```
