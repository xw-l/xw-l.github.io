---
title: 如何让nginx支持QUIC和HTTP3
date: 2023-05-21 21:01:00
tags: [nginx,http3,quic]
categories: 实用教程
cover: https://pan.alybaba.top/images/nginx-logo.png
---

# 什么是 HTTP/3 和 QUIC ？

- HTTP/3 是一种基于 QUIC（Quick UDP Internet Connections）协议的 HTTP 协议版本，它是 HTTP/2 的后继者，旨在改进 Web 性能和安全性。
- HTTP/3 与之前的 HTTP 协议有很大的不同，最明显的区别是它使用 QUIC 协议而不是 TCP 协议来传输数据。
- QUIC 是一种由 Google 开发的协议，基于 UDP，它在保持安全性的同时提供更快的连接和更少的延迟。与 TCP 不同，QUIC 允许多个请求同时在同一连接上进行，从而减少了网络拥塞和握手延迟的影响。
## 参考图
![http三种协议](/images/http.png)

## 那么接下来我将介绍几种常见nginx编译http3的方法

## 基于谷歌boringssl编译
```bash
#我直接在root家目录进行的操作，请注意你们的目录
apt update && apt install build-essential ca-certificates zlib1g-dev libpcre3 libpcre3-dev tar unzip libssl-dev wget curl git cmake ninja-build hgsubversion
#安装相关依赖
apt install software-properties-common -y
add-apt-repository ppa:longsleep/golang-backports
apt install golang
go --version
#默认源安装的go版本过低...
git clone  https://github.com/google/boringssl.git
#拉取项目，国内兄弟自己找镜像站点，我服务器在国外，这玩意大的很
cd boringssl/
mkdir build
#存放编译文件
cd build/
cmake -GNinja ..
ninja
#这个编译非常吃内存，我服务器一G内存直接原地爆炸，最好是拉到好点配置的机子上编译好，不然真的容易爆炸。
#我这里编译了一份，点击这里查看[openssl已编译打包好](http://pan.alybaba.top/openssl.tar.gz)

hg clone -b quic https://hg.nginx.org/nginx-quic
#hg跟git差不多
或者wget https://hg.nginx.org/nginx-quic/archive/quic.tar.gz
tar xf quic.tar.gz
cd nginx-quic-8057e053480a
nginx -V
./auto/configure  you nginx config  --with-http_v3_module --with-stream_quic_module --with-cc-opt="-I../boringssl/include" --with-ld-opt="-L../boringssl/build/ssl -L../boringssl/build/crypto"
#复制自己nginx的配置，然后把http3的选项加上即可，没什么问题就接着往下编译
make -j$(nproc --all)
#我用虚拟机分的8核编译，没问题的话应该会多出来一个objs的目录,里面会有一个二进制文件
cp -af objs/nginx   /sbin/nginx 
#替换当前
vim /etc/nginx/conf.d/http3.conf
server{
        listen 443 quic reuseport;
        listen 443 ssl http2;
        server_name  you domain name;
        ssl_certificate /you/path/you.pem;
        ssl_certificate_key /you/path/you.key;
        ssl_protocols  TLSv1.1 TLSv1.2 TLSv1.3;
        add_header alt-svc 'h3-23=":443"; ma=86400';
        #这个声明是必须的
}
#对了，想实现http3，必需要安装证书，推荐acme.sh或者certbot，实在不行就用openssl自签。

iptables -IINPUT -p udp --dport 443 -j ACCEPT
iptables -IINPUT -p tcp --dport 443 -j ACCEPT
#最后的话去控制台防火墙开一下443的UDP端口，(ps:命令行不起作用，可能我太菜，最好还是去控制台开一下)TCP也要开一下
netstat -nupl
#查看udp端口是否监听
systemctl restart nginx
```
## 基于OpenSSL的quictlc编译
```bash
git clone https://github.com/quictls/openssl.git
cd  openssl
mkdir build
./Configure --prefix=/root/openssl/build --openssldir=./build
#配置路径，第一个必须要绝对路径
make -j$(nproc --all) && make install_dev
#编译没问题的话，build文件夹里应该会有include和lib64的文件夹
 
cd ../nginx-quic-8057e053480a
nginx -V
./auto/configure  you nginx config  --with-http_v3_module --with-stream_quic_module  --with-cc-opt="-I /root/openssl/include"  --with-ld-opt="-L /root/openssl/lib64"
#那么没有问题的话这里也会生成objs目录，目录下也会有二进制文件

#这里有一点要注意的是他那个那个没有集成到nginx里面，所以需要把 lib64的两个库文件给复制到 /lib 目录下
cp -a ../openssl/build/lib64/*.81.3  /lib
#接下来参考谷歌boringssl编译
```
## 基于Cloudflare的quiche编译
```bash
curl https://sh.rustup.rs -sSf | sh
#安装rust
source ~/.profile
#生效变量
wget https://nginx.org/download/nginx-1.16.1.tar.gz
#支持1.19.4
tar xf nginx-1.16.1.tar.gz
git clone  https://github.com/cloudflare/quiche
cd  nginx-1.16.1
patch -p01 < ../quiche/nginx/nginx-1.16.patch
#打补丁2选1，这个是官方的，版本老点
或curl https://raw.githubusercontent.com/kn007/patch/master/nginx_with_quic.patch | patch -p1
#实测打这个补丁在make时会报错，需要从其他版本替换一个同名c文件，我替换的是1.14.1，其他自行测试

curl https://raw.githubusercontent.com/kn007/patch/master/Enable_BoringSSL_OCSP.patch | patch -p1
#支持 OCSP Stapling

./configure 你的其他nginx模块 --with-http_v3_module --with-openssl=../quiche/quiche/deps/boringssl --with-quiche=../quiche
#这是需要添加的项，后面两个请根据自己的路径来
make -j$(nproc --all)
#接下来请参考boringssl编译
```
## 基于官方安装，redhat需9以上，ubuntu需22.04
### 请参考[官网](https://quic.nginx.org/packages.html)

## 下载别人编译好的二进制文件或者安装包

## 测试
```bash
nc -z -v -u IP地址 443 
#发送一个udp数据包
tcpdump -i 你的网卡名 -A -s0 port 443 and udp
#打开浏览器的开发工具，通常是F12，点击网络，刷新也可查看当前使用的网络协议
```
## [在线测试链接](https://http3check.net/)

## 总结
 1. ~~反正我编译不下十遍了，一直不支持，怀疑是防火墙问题...~~
 2. 注意，QUICh还是一个相对较新的协议，在某些情况下可能与特定的网络环境不兼容。在使用QUIC时，建议测试和评估其在特定环境中的性能和兼容性，不建议生产环境使用...
 3. 国内对UDP不是太友好，容易qos。我指的是...其实是udp不容易溯源，且难以控制，对网络安全有一定的影响。
 4. ~~可能还有其他的安装方法，这里我就不再演示了，因为我魔怔了...~~
 5. 总的来说，HTTP/3 的设计目标是通过减少延迟和提高性能，为 Web 应用程序提供更快、更安全和更高效的用户体验。感兴趣的可以自行尝试哈，毕竟是未来的趋势。

## 参考官方文献
- [https://quic.nginx.org/readme.html](https://quic.nginx.org/readme.html)
- [http://nginx.org/en/docs/quic.html](http://nginx.org/en/docs/quic.html)
- [http://nginx.org/en/docs/http/ngx_http_v3_module.html](http://nginx.org/en/docs/http/ngx_http_v3_module.html)


