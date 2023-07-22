---
title: 开源Docker仓库管理器Harbor
date: 2023-06-17 21:41:02
tag: 私有仓库
cover: https://pan.alybaba.top/images/harbor.png
categories: 实用教程
---

# 部署准备
- 安装docker和docker-compose
- 多台虚拟机...
# 部署步骤
10.0.0.133更新
```bash
apt update
```
这里使用官方脚本安装
```bash
sh <(curl -fsSL https://get.docker.com)
```
启动docker
```bash
docker start docker
```
安装docker-compose
```bash
wget -O /usr/bin/docker-compose -P /usr/bin  \
https://github.com/docker/compose/releases/download/v2.18.1/docker-compose-linux-x86_64
或
wget -O /usr/bin/docker-compose -P /usr/bin http://pan.alybaba.top/soft/docker-compose-linux-x86_64
```
添加执行权限
```bash
chmod +x /usr/bin/docker-compose
```
测试
```bash
docker-compose version
```
下载Harbor镜像包,这里是离线安装
```bash
wget https://github.com/goharbor/harbor/releases/download/v2.8.2/harbor-offline-installer-v2.8.2.tgz
或
wget http://pan.alybaba.top/soft/harbor-offline-installer-v2.8.2.tgz
```
解压
```bash
tar xf harbor-offline-installer-v2.8.2.tgz -C /usr/local  
```
自签证书,创建存放自签证书目录
```bash
mkdir -p /data/harbor/certs && cd  /data/harbor/certs
```
生成 CA（证书颁发机构）私钥 (ca.key) 和证书 (ca.crt)
```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650  \
-subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=test.com" \ 
-key ca.key -out ca.crt
```
生成 Harbor 服务器私钥 (harbor.test.com.key)
```bash
openssl genrsa -out harbor.test.com.key 4096
```
生成 Harbor 服务器证书签名请求 (harbor.test.com.csr)
```bash
openssl req -sha512 -new -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=harbor.test.com" \
-key harbor.test.com.key -out harbor.test.com.csr
```
创建 v3.ext 文件，指定证书扩展属性
```bash
cat << EOF > v3.ext 
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=test.com
DNS.2=test
DNS.3=harbor.test.com
EOF
```
使用 CA 证书和私钥，对之前生成的 CSR 进行签名，生成最终的 Harbor 服务器证书 (harbor.test.com.crt)
```bash
openssl x509 -req -sha512 -days 3650 -extfile v3.ext -CA ca.crt -CAkey ca.key \
-CAcreateserial -in harbor.test.com.csr -out harbor.test.com.crt
```
修改 harbor 部分相关参数
```bash
cp /usr/local/harbor/harbor.yml.tmpl /usr/local/harbor/harbor.yml && vim /usr/local/harbor/harbor.yml
hostname: www.test.com
#这里需要修改hosts文件
    certificate: /you/path/harbor.test.com.crt
    private_key: /you/path/harbor.test.com.key
#你的证书和私钥存放的位置
harbor_admin_password: Harbor12345
# Harbor12345是登录密码，可以修改成你想要的密码，账号是admin
#这里值得注意的是，会自动修改nginx的配置文件自动跳转https
:wq
```
`同时，这里要注意，因为是自签证书，浏览器访问还是会不信任，导致镜像无法推送`
`2种方法，第一，在客户端安装自签证书，第二写docker配置文件，这里就采用第二种了`
```bash
echo -e '{\n"insecure-registries": ["harbor.test.com"]\n}' > /etc/docker/daemon.json
#意思是信任不安全的镜像仓库地址
systemctl restart docker
```
启动仓库
```bash
cd /usr/local/harbor  && ./install.sh && ./prepare && docker-compose up -d
#这里也可以使用docker-compose -f /you/path/docker-compose.yml up -d 指定文件所在路径后台启动
```
写入service文件，配置自启动
```bash
cat << EOF > /lib/systemd/system/harbor.service 
[Unit]
Description=Harbor
After=docker.service systemd-networkd.service systemd-resolved.service
Requires=docker.service
Documentation=http://github.com/vmware/harbor

[Service]
Type=simple
Restart=on-failure
Restart=5
ExecStart=/usr/bin/docker-compose -f /usr/local/harbor/docker-compose.yml up
ExecStop=/usr/bin/docker-compose -f /usr/local/harbor/docker-compose.yml down

[Install]
WantedBy=multi-user.target
EOF
```
启动harbor
```bash
docker-compose down
#这里要注意，前面我们用docker-compose启动过的，这里我们关闭
systemctl daemon-reload && systemctl start harbor.service
```
linux登录仓库,输出Login Succeeded则成功，如出现其他报错，请检查日志或者报错原因
```bash
docker login -u admin -p Harbor12345 harbor.test.com
#这里可以使用本机，也可以使用客户端，记得添加hosts解析以及docker信任配置
```
接下来我们去网页端登录看看,可以看到还是提示不安全，不交钱就是这样
![logi-page](/images/harbor-login.jpg)
登录进去创建一个项目,我这边已经创建了
![create-project](/images/harbor-create.jpg)
然后我们就来推送一个镜像试试，首先看看有什么镜像
![see-what-we-got](/images/harbor-docker-images.jpg)
`可以看到，我已经操作了一边。`
首先重命名
```bash
docker tag alpine-nginx:1.24.0-v3  harbor.test.com/nginx/alpine-nginx:1.24.0-v3
#harbor.test.com表示harbor仓库地址
#nginx表示你创建的项目的地址
```
接着推送就完成了
```bash
docker push  harbor.test.com/nginx/alpine-nginx:1.24.0-v3
```
然后也可以把镜像拉下来
```bash
docker pull  harbor.test.com/nginx/alpine-nginx:1.24.0-v3
```
![pull-images](/images/harbor-pull.jpg)
在web端我们也可以看到镜像，harbor还压缩了
![see-pull-images](/images/harbor-images.jpg)
`这里虽然完成的harbor的部署，但是始终是一个单点问题，所有接下来我们继续部署高可用harbor`
使用另一台空闲机10.0.0.132安装 docker 和 docker-compose 环境
```bash
sh <(curl -fsSL https://get.docker.com) && wget -O /usr/bin/docker-compose  -P \
/usr/bin  http://pan.alybaba.top/soft/docker-compose-linux-x86_64 \
&& chmod +x /usr/bin/docker-compose
```
harbor包和service文件我已经用scp命令拷贝过来
```bash
tar xf harbor-offline-installer-v2.8.2.tgz -C /usr/local && cd /usr/local/harbor
sed -i 's/hostname: reg.mydomain.com/hostname: 10.0.0.132/'  harbor.yml
echo -e '{\n"insecure-registries": ["10.0.0.132"]\n}' > /etc/docker/daemon.json
./install.sh  && ./prepare && systemctl daemon-reload
systemctl start docker harbor.service
```
那么我登上前面配置的harbor上进行高可用配置，前面的ip地址是133，它对应的仓库地址就是132
![create-target](/images/harbor-hight-use.jpg)
做完这里，在 10.0.0.133  push镜像时已经可以同步到10.0.0.132了，那么132也同理配置，这里就不再过多叙述
![create-policy](/images/harbor-hight-use-2.jpg)
# 部分参考资料
- docker-compose 官方下载地址: https://docs.docker.com/compose/
- docker 官方文档安装地址：https://docs.docker.com/engine/install/ubuntu/
- harbor 下载地址: https://github.com/vmware/harbor/releases
- harbor 安装文档: https://github.com/goharbor/harbor/blob/master/docs/install-config/_index.md
