---
title: 开源监控Zabbix
date: 2023-06-01 14:06:42
tags: [Zabbix,监控]
categories: 实用教程
cover: https://pan.alybaba.top/images/zabbix_logo.png
---

### Zabbix 介绍
- Zabbix 是一个企业级分布式开源监控解决方案，支持实时监控数干台服务器，虚拟机和网络设备，采集百万级监控指标，适用于任何IT基础架构、服务、应用程序和资源的解决方案
- Zabbix 软件能够监控众多网络参数和服务器的健康度、完整性。Zabbix 使用灵活的告警机制，允许用户为几乎任何事件配置基于邮件的告警。这样用户可以快速响应服务器问题。Zabbix 基于存储的数提供出色的报表和数据可视化功能。这些功能使得 Zabbix 成为容量规划的理想选择。
### 安装配置zabbix
### 环境准备
##### 准备3台主机
1. zabbix、nginx、php。我这里是ubuntu22.04，ip 10.0.0.100
2. mysql ubutnu22.04。ip 10.0.0.130
3. client 随意 ip 10.0.0.131
### 部署步骤
#### zabbix,nginx,php部署
##### 下载安装包
```bash
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-4+ubuntu22.04_all.deb
```
##### 安装源
```bash
dpkg -i zabbix-release_6.0-4+ubuntu22.04_all.deb
apt update
apt install zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf zabbix-sql-scripts zabbix-agent zabbix-get
```
##### 安装中文
```bash
apt -y install language-pack-zh-hans
#英文水平高略过
```
##### zabbix 配置文件mysql配置项目
```bash
vim /etc/zabbix/zabbix_server.conf
DBHost=10.0.0.130
#这里mysql我装在另一台主机了
DBName=zabbix
#数据库名
DBUser=zabbix
#数据库用户
DBPassword=123456
#数据库密码
scp /usr/share/zabbix-sql-scripts/mysql/server.sql.gz  root@10.0.0.130:
#把脚本复制到mysql主机上
```
#### mysql配置
```bash
apt install mysql-server
sed -i 's/^bind-address/#&/' /etc/mysql/mysql.conf.d/mysqld.cnf
systemctl start mysql.service
mysql
mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
mysql> create user zabbix@'10.0.0.%' identified by '123456';
mysql> grant all privileges on zabbix.* to zabbix@'10.0.0.%';
mysql> set global log_bin_trust_function_creators = 1;
#用于控制在二进制日志中记录创建函数和存储过程的权限检查。当该变量的值为 1 时，MySQL 将信任具有创建函数和存储过程权限的非特权用户，允许他们创建这些对象并将其记录在二进制日志中。
mysql> \! zcat /root/server.sql.gz | mysql -uzabbix -p123456 -h 10.0.0.130 zabbix
#在这里我试了很多次，发现是我创建用户只允许10网段，我使用127登不进去，这个要注意用户授权，我本机IP是130
mysql> set global log_bin_trust_function_creators = 0;
```
##### 配置nginx
```bash
vim /etc/zabbix/nginx.conf 
        listen          8080;
#        server_name     example.com;
#这里我没有配置域名，80已经被另一个配置文件监听，如果要配置80，要么增加default_server，删除另一个配置文件的default_server，要么添加域名，以请求头区分访问页面。
```
##### 启动zabbix，配置自启动
```bash
systemctl start zabbix-server zabbix-agent nginx php8.1-fpm  &&  systemctl enable zabbix-server zabbix-agent nginx php8.1-fpm
```
##### 替换中文
```bash
win + R 输入fonts
#随便拷贝一个字体，这里我选择的微软雅黑
cd /usr/share/zabbix/assets/fonts
#zabbix字体引用目录，实际是软链接
mv graphfont.ttf graphfont.ttf.bak
#备份原字体
cp  msyh.ttc  graphfont.ttf
#替换，也可-b选项
```
##### 登录配置
![起始页面检测](/images/zabbix检测.jpg)
##### web服务连接mysql，配置文件是zabbix-server连接数据库
![连接mysql配置](/images/zabbix-mysql.jpg)
##### 时区选择，一般是上海
![时区选择](/images/zabbix-时区选择.jpg)
##### 登录页面，账号是 "Admin" 密码是 "zabbix"，注意只有"是"，表示zabbix-server运行正常
![首页](/images/zabbix-login.jpg)
##### 客户端安装agent
```bash
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-4+ubuntu22.04_all.deb
dpkg -i zabbix-release_6.0-4+ubuntu22.04_all.deb
apt update
apt install zabbix-agent
```
##### 配置agent
```bash
Server=10.0.0.100 
#必须指向Zabbix Server,此为必须项
ServerActive=10.0.0.100 
#主动模式才需要指向Zabbix Server,此处无需修改
Hostname=10.0.0.131 
#修改为当前主机的IP,主动模式才需要,此处可选
Timeout=30 
#建议修改此值
```
##### 启动agent
```bash
system enable --now zabbix-agent
```
##### 添加client，这里我是修改，如果是新client应为添加。
![首页](/images/add-client.jpg)
##### 查看client
![客户端部分信息](/images/client-mesg.jpg)
到这里我们就简单完成了zabbix的配置，zabbix还有很多扩展，比如自定义监控某项参数....这里不再过多叙述，其次zabbix不仅仅可以监控linux，也可以监控windows，或者是交换机，路由器等等软硬件。感兴趣可自行尝试参考官方安装教程使用

##### 参考连接
- 官方安装教程[https://www.zabbix.com/download](https://www.zabbix.com/download)
- 官方包仓库[https://repo.zabbix.com](https://repo.zabbix.com)
- 阿里云镜像源[(https://mirrors.aliyun.com/zabbix](https://mirrors.aliyun.com/zabbix)
- 清华大学镜像源[https://mirrors.tuna.tsinghua.edu.cn/zabbix](https://mirrors.tuna.tsinghua.edu.cn/zabbix)
