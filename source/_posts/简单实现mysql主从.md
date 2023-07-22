---
title: 简单实现mysql主从
date: 2023-05-01 22:18:44
tags: [Mysql,Databases]
categories: 实用教程
cover: https://pan.alybaba.top/images/mysql.jpg
---
原理

```bash
两个日志，三个线程
master节点上会为每一个slave的节点开启一个dump线程，用来提供master本机的二进制日志
slave节点上的i/o线程会请求master节点dump线程传输的二进制事件，并将得到的内容写入replay日志中
slave节点的SQL线程实时检测replay log内容，有更新则解析成sql语句还原到slave数据库中，这样保证主从数据同步
```
master节点配置
```bash
vim /etc/my.cnf
[mysqld]                
log_bin=/mysql/log/binlog 
#启动并修改二进制日志目录,可选，不添加则为默认目录，这个binlog指的是日志命名
server-id=128  
#为当前节点设置全局唯一ID，一般都是以ip地址结尾数字命名
:wq
mkdir -p /mysql/log  &&  chown -R mysql.mysql  /mysql
#对应配置文件log-bin日志路径
create user backuser@'172.18.0.%' identified by '123456'
#这里是我的内网，如果是公网，应具体到某一个的32位ip（如果没有vpn的话），同时注意生产环境请不要设这么弱鸡的密码
grant replication slave on *.* to backuser@'172.18.0.%'
授权，因为是要复制所有库表信息，这里给*.*，更为详细的权限分配这里暂且就不提
systemctl restart mysqld.service
```
slave节点配置
```bash
vim /etc/my.cnf
[mysqld]
server-id=43
#为当前节点设置一个全局惟的ID号
log-bin=/mysql/log/binlog
#开启并修改slave节点二进制日志
read_only=1
#设置数据库只读，针对超级用户无效
:wq
systemctl restart mysqld.service
mkdir -p /mysql/log && chown -R mysql.mysql /mysql
```

执行一下sql语句，这里注意注释不要，如果懒的删除，可以复制到文件里，再mysql命令行界面source /you/path/file即可,也可以再bash下执行mysql -u user -p < /you/path/file
```bash
mysql -uroot -p 'enter you password'
CHANGE MASTER TO MASTER_HOST='172.18.0.127', 
#指定master节点
MASTER_USER='backuser', 
#连接用户
MASTER_PASSWORD='123456', 
#连接密码
#下面两条语句请根据master节点执行show master status给出的信息填写
MASTER_LOG_FILE='binlog.xxxxxx', 
#从哪个二进制文件开始复制
MASTER_LOG_POS=123;
#指定同步开始的位置

启动服务并查看状态
start slave;
show slave status;

删除配置
reset master all;
reset slave all;

查看端口是否监听
ss -tun
netstat -an | grep EST
```

总结：总体配置完成后，即可在master上进行写入操作,随后可查看slave是否同步，这里值得深思的是，如果mysql版本不同，请让低版本做master，高版本做slave，别问为什么，问就是百度。其次有一点要注意的是关于binlog的选择上，如果是binlog被清理过，那就需要执行mysqldump(ps不止这一种备份访问，看个人爱好)全量备份，还原到slave上，才能实现主从一致
