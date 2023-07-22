---
title: docker构建redis
date: 2023-07-03 20:10:02
tags: docker
categories: 容器
cover: https://pan.alybaba.top/images/docker-facebook.png
---
# 部署准备
- ubuntu 或其他主流发行版本
- 部署docker
# 部署步骤
安装docker,报错的话，一般是发行版不支持(脚本没有你的发行版名字)，可到官网查看使用其他方法(比如说rocky,可自己在脚本内加个 rocky 判断,或者把红帽系改成 rocky)
```bash
sh <(curl -fsSL https://get.docker.com)
```
拉取镜像，这里我就用 alpine 了.
```bash
docker pull alpine:latest
```
编写 dockerfile 文件
```bash
mkdir /images/alpine/redis -p && cd /images/alpine/redis 
cat << EOF > dockerfile
FROM alpine:latest

LABEL autor="lee"

ENV redis_version=redis-7.0.11   install_path=/usr/local/redis/

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/'  /etc/apk/repositories \
    && export DEBIAN_FRONTEND=noninteractive \
    #配置不交互
    && apk add --no-cache  --virtual .build-redis coreutils wget dpkg-dev dpkg gcc tzdata\
        linux-headers make musl-dev openssl-dev \
    #创建一个build-redis的虚拟软件包，方便删除,其他的都是编译相关
    && ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo "Asia/Shanghai" > /etc/timezone \
    #修改时区
    && wget https://download.redis.io/releases/${redis_version}.tar.gz \
    && tar xf ${redis_version}.tar.gz && cd ${redis_version} \
    && make -j $(nproc -all) && make install \
    #拉满编译安装
    && mkdir -p ${install_path}etc ${install_path}log ${install_path}run ${install_path}data \
    # 不支持花括号创建
    && addgroup -S redis && adduser -Gredis -S -D -s /sbin/nologin  redis \
    && cp /${redis_version}/redis.conf ${install_path}etc && chown redis.redis ${install_path} \
    && sed -i -e 's/bind 127.0.0.1/bind 0.0.0.0/' \
            -e '/# requirepass/a requirepass 123456' \
            -e "s#pidfile \/var\/run\/redis_6379.pid#pidfile  ${install_path}pid\/redis_6379.pid#" \
            -e "s#logfile \"\"#logfile \"${install_path}log\/redis_6379.log\"#" \
            -e "s#dir ./#dir ${install_path}data/#" \
            ${install_path}etc/redis.conf && \
    #修改部分配置文件，也可以-v挂到主机上
    apk del --no-network .build-redis && rm -rf /redis*
    #删除之前安装的软件和下载解压的redis

CMD ["redis-server"," /usr/local/redis/etc/redis.conf"]
#简单运行的命令

EXPOSE 6379
#容器默认暴漏端口
EOF
```
构建镜像
```bash
docker build -t alpine:redis-v1.0 .
```
可以看到编译很吃cpu
<hr />

![](/images/alpine-build-redis-top.jpg)

编译成功
<hr />

![](/images/alpine-build-succecd.jpg)
查看镜像大小
```bash
docker images
    REPOSITORY   TAG          IMAGE ID       CREATED         SIZE
    alpine       redis-v1.0   0c2bbc93bd59   8 minutes ago   28.9MB
    alpine       latest       c1aabb73d233   2 weeks ago     7.33MB
```
启动看看
```bash
docker run -d -P --name redis --restart=always alpine:redis-v1.0 
#可-v 挂载 /usr/local/redis 目录
```
查看下映射端口
```bash
docker port redis
    6379/tcp -> 0.0.0.0:49162
    6379/tcp -> :::49162
```
查看端口是否监听,可以看到我主机已经安装了redis了，49162端口也被监听
```bahs
netstat -ntpl
    Active Internet connections (only servers)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
    tcp        0      0 127.0.0.1:39867         0.0.0.0:*               LISTEN      31145/containerd    
    tcp        0      0 0.0.0.0:49162           0.0.0.0:*               LISTEN      54412/docker-proxy  
    tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      30940/sshd: /usr/sb 
    tcp        0      0 0.0.0.0:6379            0.0.0.0:*               LISTEN      28950/redis-server  
    tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      30928/systemd-resol 
    tcp        0      0 0.0.0.0:16379           0.0.0.0:*               LISTEN      28950/redis-server  
    tcp6       0      0 :::49162                :::*                    LISTEN      54418/docker-proxy  
    tcp6       0      0 :::22                   :::*                    LISTEN      30940/sshd: /usr/sb 
```
访问测试
```bash
redis-cli -a 123456 -h localhost -p 49162
```
如下图,简单构建完成(ps 忘了起别名...)
<hr />

![](/images/alpine-redis-run.jpg)


# 部署问题
1. sh 和 bash 有所不同，可以apk add bash,然后再删除...
2. alpine 的 busybox 命令都是精简版的，要注意。
3. 本次构建没有做到高度定制化。
4. 后来发现我创建用户好像是多余，还是用root执行，没有指定 redis 用户运行，如果要 service 运行需要安装 openrc。我就暂时不折腾了










