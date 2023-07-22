---
title: MySQL半同步复制简单实现
date: 2023-05-05 20:36:00
tags: [数据库,Database]
categories: 实用教程
cover: https://pan.alybaba.top/images/mysql.jpg
---
在mysql8.0中实现半同步复制
```bash
主机IP 	         角色    MySQL版本
172.18.0.127	master     8以上
172.18.0.48	slave-1    8以上
172.18.0.43 	slave-2    8以上
```
master配置
```bash
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
--安装半同步插件
show master logs; 
--查看二进制文件,主从同步用
create user  backuser@'172.18.0.%' identified by '123456' 
 --创建主从账号
grant replication slave on *.* to backuser@'172.18.0.%'   
 --给与相应权限

vim /etc/my.cnf
[mysqld]
server-id=123
rpl_semi_sync_master_enabled
log-bin=/mysql/log/binlog

mkdir -p /mysql/log && chown -R mysql.mysql /mysql
systemctl restart mysqld.service
select @@rpl_semi_sync_master_enabled;  
--查看是否开启
```


slave节点48配置
```bash
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
--安装slave半同步插件

vim /etc/my.cnf
[mysqld]
server-id=48
rpl_semi_sync_master_enabled
log-bin=/mysql/log/binlog
mkdir -p /mysql/log && chown -R mysql.mysql /mysql
systemctl restart mysqld.service
select @@rpl_semi_sync_slave_enabled;  

CHANGE MASTER TO MASTER_HOST='172.18.0.127',
MASTER_USER='backuser',
MASTER_PASSWORD='123456',
MASTER_LOG_FILE='binlog.000001',
MASTER_LOG_POS=157;

strat slave;
```
slave节点43配置
```bash
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
--安装slave半同步插件

vim /etc/my.cnf
[mysqld]
server-id=43
rpl_semi_sync_slave_enabled
log-bin=/mysql/log/binlog
mkdir -p /mysql/log && chown -R mysql.mysql /mysql
systemctl restart mysqld.service
select @@rpl_semi_sync_slave_enabled;  

CHANGE MASTER TO MASTER_HOST='172.18.0.127',
MASTER_USER='backuser',
MASTER_PASSWORD='123456',
MASTER_LOG_FILE='binlog.000001',
MASTER_LOG_POS=157;

start slave
```
到这里我们的半同步复制就基本上完成了，由于这里是基于全新机器，所以比较快，当然如果是现有数据，直接备份主机还原的从鸡上就行了。
```bash
查看半同步策略
show global variables like '%semi%';
--查看有几个节点
show global status like '%semi%';
```
这里我们来测试下半同步的原理（至少有一个节点同步成功才会给客户端返回成功，当然也有可能是超时，超时时间可自定义）
```bash
#首先我们测试从机是否能够同步
Master
mysql> create database testdb1;
Query OK, 1 row affected (0.01 sec)

Slave-1
mysql> show databases;
+-----------------------+
| Database              |
+-----------------------+
| db1                   |
| information_schema    |
| mysql                 |
| performance_schema    |
| sys                   |
| testdb1               |
+-----------------------+
6 rows in set (0.00 sec)

Slave-2
mysql> show databases;
+-----------------------+
| Database              |
+-----------------------+
| db1                   |
| information_schema    |
| mysql                 |
| performance_schema    |
| sys                   |
| testdb1               |
+-----------------------+
6 rows in set (0.00 sec)
```
可以看到都没问题，那我们停掉一台从机，再写入数据试试
```bash
systemctl stop mysqld.service

create database testdb2;
Query OK, 1 row affected (0.01 sec)
```
还是没问题，我我们停掉第二台主机试试
```bash
systemctl stop mysqld.service
create database testdb3;
Query OK, 1 row affected (10.00 sec)
```
现在可以看到，当我们把两台从机停了之后，主机不知道找谁写入了，虽然如此，但还是有一个超时机制，默认10s。10s后不管有没有其他从机连上，都会提交本次写入。
```bash
--修改超时配置my.cnf
rpl_semi_sync_master_timeout=3000
```

总结：半同步复制是 MySQL 的一种数据复制方式，相对于异步复制，它提高了数据复制的安全性，同时保持了较高的性能。半同步复制的基本思路是：当主库执行一个事务后，只有等到至少一个从库接收并写入了该事务才会提交成功。这样可以保证从库的数据和主库的数据是同步的。（PS：如果从机全部掉线也不是不可能）
半同步复制的好处是：提高数据复制的安全性，防止从库数据丢失。相对于同步复制，半同步复制的性能有所提高，因为主库不需要等待所有从库都接收并写入数据。半同步复制的缺点是：相对于异步复制，半同步复制会增加主库的负载，因为主库需要等待至少一个从库接收数据。但总的来说，半同步复制是一种较好的数据复制方式，它提高了数据复制的安全性，同时也能保持较高的性能。
同时我们还可以使用复制过滤器让从节点仅复制指定的数据库，或指定数据库的指定表，有兴趣的可以自行尝试。其实现方法就是往mysql配置文件写黑白名单。
