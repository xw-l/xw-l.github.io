---
title: 编译安装haproxy
date: 2023-06-26 20:30:02
tags: [反向代理,proxy]
categories: 实用教程
cover: https://pan.alybaba.top/images/Rocky.png
---
源安装
```bash
apt install software-properties-common
#软件源管理工具
```
添加 haproxy source 源文件
```bash
add-apt-repository ppa:vbernat/haproxy-2.0
#添加haproxy源，，红帽系列好像不被官方支持...但是源仓库是有的，版本低
```
更新一下源
```bash
apt update
```
查看选择需要安装的版本
```bash
apt-cache madison haproxy
```
安装
```bash
apt -y install haproxy=2.8.\*
```
`编译安装`
安装编译相关工具和依赖
```bash
apt -y install gcc make libssl-dev libpcre3  libpcre3-dev zlib1g-dev libreadline-dev libsystemd-dev liblua5.4-dev
```
下载源码包
```bash
wget http://www.haproxy.org/download/2.8/src/haproxy-2.8.0.tar.gz
```
解压
```bash
tar xf haproxy-2.8.0.tar.gz && cd haproxy-2.8.0
```
构建编译的参数
```bash
make ARCH=x86_64 TARGET=linux-glibc USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 USE_SYSTEMD=1 USE_PROMEX=1 USE_LUA=1
```
安装到指定目录
```bash
make install PREFIX=/apps/haproxy
```
创建软链接，也就是添加可执行路径
```bash
ln -s /apps/haproxy/sbin/haproxy /usr/local/sbin/
```
参考配置文件
```bash
global
    chroot /apps/haproxy   改变根目录
    deamon                 后台运行
    mode tcp | http        转发类型 
    stats socket /var/lib/haproxy/haproxy.sock mode 600 level admin  proxess 1    这是socket文件,可用通过命令向该文件执行命令
    user  gid | username
    group uid | groupname 指定用户和组
    nbproc n   该命令和nginx相似，开启n个进程，不过从2.5以后被弃用
    nbthread n  指定每个进程开启的线程，默认为1，同时两者同时开启会报错，Centos8的1.8无此问题
    cpu-map 1 0
    cpu-map 2 1  亲缘性，CPU绑定进程，进程从1开始，cpu从0开始
    cpu-map auto:1/1-8 0-7   自动绑定，每个进程的1-8个线程分别绑定0-7号cpu。haproxy2.4中为nbthreads
    #查看cpu绑定 ps axo pid,cmd,psr -L | grep haproxy
    #或者安装sysstat,pidstat -p  haproxy-pid -t 
    maxconn n       每个进程的最大连接数为n
    maxsslconn n    每个进程的ssl最大连接数为n
    maxconnrate n   每个进程每秒能创建的最多连接数为n
    spread-checks n 后端server状态检查，建议为2-5，默认为0
    pidfile   /some/path/haproxy.pid  PID文件路径
    log 127.0.0.1 local2 info   日志发送到本机，默认为514 udp端口
    log x.x.x.x local2 info     定义log，和rsyslog配合使用，local3为名称，info为日志等级，需要开启UDP，最多可以定义两个
        日志配置
        vim /etc/rsyslog.conf
        module(load="imudp")
        input(type="imudp" port="514")

        echo  local2.* /var/log/haproxy.log 或 local2.info  /var/log/haproxy.log > /etc/rsyslog.d/50-default.conf
        systemctl restart rsyslog
defaults
    option redispatch                   重定向其他正常服务器
    option abortonclose                 自动结束连接时长较久的连接             
    option http-keep-alive              开启与客户端的会话保持
    option forwardfor                   透传真实IP至后端服务器,可自定义修改
    mode http | tcp                     默认工作类型
    timeout http-keep-alive             会话保持时间
    timeout connect 120s                客户端请求从haproxy到后端server最长连接等待时间(TCP连接之前)默认单位 ms
    timeout server 600s                 客户端请求从haproxy到后端服务端的请求处理超时时长(TCP连接之后)，超时502
    timeout client 600s                 设置haproxy与客户端的最长非活动时间，默认单位ms，建议和timeoutserver相同
    timeout check 5s                    对后端服务器的默认检测超时时间
    default-server inter 1000 weight 3  指定后端服务器的默认设置

前后端整合
listen nginx                            使用listen替换 frontend和backend的配置方式，可以简化设置，常用于TCP协议的应用       
        bind 公网IP:端口
        mode tcp | http
        server web1 后端IP:port
        server web2 后端IP:port


前端服务器
frontend  nginx
    bind 公网IP：端口
    use_backend real-nginx

后端服务器                                                                                 
backend  real-nginx 
    redirect prefix http://www.baidu.com/                    全局重定向，只适合http
    server web1 后端IP:port check inter 5000 fail 3 rise 3   每5秒检查一次，如果3次不通就挂，上线同理
    server web2 后端IP:port weight 3                         权重
    server web2 后端IP:port check backup                     备份服务器，可当sorry server
        server后字段
            check 
                addr <IP> 可指定的健康状态监测IP，可以是专门的数据网段，减少业务网络的流量
                port <num> 指定的健康状态监测端口
                inter <num> 健康状态检查间隔时间，默认2000 ms
                fall <num> 后端服务器从线上转为线下的检查的连续失效次数，默认为3
                rise <num> 后端服务器从下线恢复上线的检查的连续有效次数，默认为2
            disable 关闭服务器
            maxconn 后端服务器最大连接数
            redir http://x.x.x  局部302临时重定向到某url，只适合http

listen mysql
        bind 0.0.0.0:3306           haproxy端口
        mode tcp                    如果不指定tcp，会以http进行连接
        server data1 x.x.x.x:3306   后端数据库地址和端口

listen stats
        mode http
        bind 0.0.0.0:9999
        stats enable
        log global                  开启日志，默认不开启
        stats uri /haproxy-status   访问后缀
        stats auth admin:123456     开启认证
        stats refresh 5             自动刷新，这里是每5s
        stats admin if TURE         如果是admin就允许访问stats页面
```
检查配置文件
```bash
haproxy -c -f /etc/haproxy/haproxy.cfg
```
创建用户
```bash
useradd -r -s /sbin/nologin -d /var/lib/haproxy haproxy
#如果没创建用户和组的话
```
创建damon.service文件
```bash
cat << EOF > /usr/lib/systemd/system/haproxy.service
[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target
[Service]
ExecStartPre=/usr/local/sbin/haproxy -f /etc/haproxy/haproxy.cfg -f /etc/haproxy/haproxy.d/ -c -q
#-f可指定文件，也可指定目录，目录一般存放子配置文件
ExecStart=/usr/local/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -f /etc/haproxy/haproxy.d/ -p  /var/lib/haproxy/haproxy.pid
ExecReload=/bin/kill -USR2 $MAINPID
LimitNOFILE=100000
[Install]
WantedBy=multi-user.target
EOF
```
```bash
systemctl daemon-reload && systemctl start haproxy
```
web端访问 http://ip:9999/haproxy-status

官方配置手册 http://docs.haproxy.org/2.8/configuration.html
