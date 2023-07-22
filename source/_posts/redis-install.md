---
title: Redis安装和多实例
date: 2023-06-30 22:18:02
tags: [Redis,Databases]
categories: 实用教程
cover: https://pan.alybaba.top/images/redis.jpg
---
# 部署准备 
- ubuntu22.04 或 CentOS 8
# 部署步骤
`源安装`
如需安装较新版本，可添加对应源文件
```bash
yum install redis -y
apt install redis -y
```
`编译安装`
红帽系编译工具和依赖
```bash
yum install -y gcc make jemalloc-devel systemd-devel 
#报错清yum install epel-release
```
答辩系编译工具和依赖
```bash
apt install -y gcc make libjemalloc-dev libsystemd-dev
```
下载源码
```bash
wget https://download.redis.io/releases/redis-7.0.11.tar.gz
```
解压
```bash
tar xf redis-7.0.11.tar.gz && cd redis-7.0.11
```
编译
```bash
make -j $(nproc -all) USE_SYSTEMD=yes PREFIX=/apps/redis install
```
写变量
```bash
echo 'PATH=/apps/redis/bin:$PATH' > /etc/profiled.d/redis.sh
source /etc/profiled.d/redis.sh
或者
ln -s /apps/redis/bin/redis-* /usr/local/bin/
```
创建相关文件
```bash
mkdir /apps/redis/{etc,log,run,data}
```
复制配置文件
```bash
cp redis.conf /apps/redis/etc/
vim /apps/redis/etc/redis.conf
    pidfile /apps/redis/run/redis_6379.pid
    logfile "/apps/redis/log/redis_6379.log"
    dir /apps/redis/data   
```
添加用户
```bash
useradd -r -s /sbin/nologin redis
```
授权
```bash
chown -R redis.redis  /apps/redis/
```
写daemon文件
```bash
cat <<EOF > /lib/systemd/system/redis.service
[Unit]
Description=Redis persistent key-value database
After=network.target

[Service]
ExecStart=/apps/redis/bin/redis-server /apps/redis/etc/redis.conf --supervised systemd
ExecStop=/bin/kill -s QUIT \$MAINPID
Type=notify
User=redis
Group=redis
RuntimeDirectory=redis
RuntimeDirectoryMode=0755
LimitNOFILE=1000000

[Install]
WantedBy=multi-user.target
EOF
```
大公告成
```bash
systemctl daemon-reload 
systemctl enable --now  redis 
```
# 常见报错解决
- Tcp backlog报错
    1. echo 4096 > /proc/sys/net/core/somaxconn   全连接队列，需要511以上redis才不会报错
    2. echo net.core.somaxconn = 4096 >> /etc/sysctl.conf

- Memory overcommit 
    1. echo 1 > /proc/sys/vm/overcommit_memory
    2. echo vm.overcommit_memory = 1 >> /etc/sysctl.conf
    - 0 表示有物理内存就分配，没有就返回错误
    - 1 表示分配所有内存，无论是否有内存
    - 2 表示分配超内存和交换空间总和的内存

- transparent hugepage
    1. echo never > /sys/kernel/mm/transparent_hugepage/enabled
    2. 写入到开机执行脚本中，例如bashrc，profile
# 常见配置文件选项
sed -i '/^bind/s/.*/bind 0.0.0.0/' redis.conf  
`替换本地监听`
sed -i 's/#save 3600 1 300 100 60 10000/save 3600 1 300 100 60 10000/' redis.conf
`开启自动 bgsava`
dir /app/redis/data
`数据文件存放位置`
pidfile /apps/redis/run/redis_6379.pid
`pid 文件存放位置`
dbfilename dump6379.rdp
`数据文件名`
port 6379
`端口`
protected-mode yes
`拒绝外部访问`
requirepass "123456"
`授权密码`
··········

# 多实例配置
配置文件多实例
```bash
sed -i 's/6379/6380/g' /app/redis/etc/redis.conf > /app/redis/etc/redis6380.conf
```
daemon文件多实例
```bash
sed -i 's/redis.conf/redis6379.conf/'  /lib/systemd/system.redis.service > /lib/systemd/system.redis6380.service 
```
启动
```bash
systemctl daemon-reload && systemctl start redis*
```
`同理，只要端口，pid文件，配置文件，数据文件，service文件和日志文件名不一致，即可开启多实例`
# 参考文献
- 版本选择 https://download.redis.io/releases/
- 官方安装教程参考 https://redis.io/docs/getting-started/installation/
