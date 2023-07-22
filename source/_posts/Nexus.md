---
title: 开源仓库管理器Nexus
date: 2023-05-30 21:30:02
tags: [私有仓库,Nexus]
categories: 实用教程
cover: https://pan.alybaba.top/images/nexus.jpg
---

### Nexus介绍
#### Nexus是一个流行的开源仓库管理器，用于管理和发布软件构件（如库、依赖项、插件等）。它提供了一个集中式的仓库，使开发人员可以方便地共享和访问软件构件。
#### 以下是Nexus的一些主要特点和功能：
#### 1. 仓库管理：Nexus提供了本地仓库和远程仓库的管理功能。您可以配置本地仓库来存储和管理自己的构件，并设置远程仓库来访问外部的公共构件库。
#### 2. 依赖项管理：Nexus可以用作构建工具（如Maven、Gradle等）的依赖项管理器。它能够自动下载和缓存构件，并提供依赖项解析和传递功能，以确保项目的构建过程顺利进行。
#### 3. 安全和权限控制：Nexus支持用户认证和权限管理，您可以设置访问仓库的用户和角色，并对不同的仓库进行细粒度的权限控制。
#### 4. 代理和缓存：Nexus允许您设置远程仓库代理，自动从外部仓库下载构件并缓存到本地仓库中。这可以加快构建过程并减少对外部网络的依赖。
#### 5. 搜索和浏览：Nexus提供了一个直观的用户界面，可以搜索和浏览仓库中的构件。您可以通过关键字、组织、版本等进行搜索，并查看构件的详细信息和元数据。
#### 6. 企业支持：Nexus提供了专业版和企业版，提供了更多高级功能和支持，如高可用性集群、故障转移、安全审计等。
#### 总体而言，Nexus是一个强大的仓库管理器，适用于各种软件开发项目，特别是基于Maven或Gradle的项目。它能够提供可靠的构件管理和依赖项解决方案，帮助开发团队更高效地管理和共享软件构件。

### 注意事项
#### 1. 内存需要4G以上(其实可修改配置)

### 环境准备
#### 一台4G内存主机，这里我用的Rocky，一台客户机用来配置仓库

### 安装jdk
```bash
yum install java-1.8.0-openjdk
```
### 安装Nexus
```bash
wget https://download.sonatype.com/nexus/3/nexus-3.54.1-01-unix.tar.gz
```
### 解压Nexus
```bash
tar xf nexus-3.54.1-01-unix.tar.gz -C /usr/local/
cd /usr/local
```
### 添加Nexus默认路径
```bash
ln -s nexus-3.54.1-01 nexus
ln -s nexus/bin/nexus /usr/bin/
或
echo 'PATH=/usr/local/nexus/bin:$PATH' > /etc/profile.d/nexus.sh
source /etc/profile.d/nexus.sh
```
### 配置Nexus启动用户
```bash
vim /usr/local/nexus/bin/nexus.rc
run_as_user="root"
```
### 配置Nexus启动端口
```bash
vim nexus/etc/nexus-default.properties
## DO NOT EDIT - CUSTOMIZATIONS BELONG IN $data-dir/etc/nexus.properties
##
# Jetty section
application-port=8081
application-host=0.0.0.0
nexus-args=${jetty.etc}/jetty.xml,${jetty.etc}/jetty-http.xml,${jetty.etc}/jetty-requestlog.xml
nexus-context-path=/

# Nexus section
nexus-edition=nexus-pro-edition
nexus-features=\
 nexus-pro-feature

nexus.hazelcast.discovery.isEnabled=true
```
### 启动Nexus
```bash
nexus run
#前台启动
nexus start
#后台启动
nexus stop
#停止
nexus status
#查看状态
```
### 查看内存，这里不贴图了，可以看到使用了2.1G，我还没去打开网页
```bash
free -h
              total        used        free      shared  buff/cache   available
Mem:          3.5Gi       2.1Gi       856Mi        16Mi       625Mi       1.2Gi
Swap:            0B          0B          0B
```
### 查看端口监听
```bash
ss -ntpl
```
### 创建service文件
```bash
vim /lib/systemd/system/nexus.service
[Unit]
Description=nexus service
After=network.target
[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/usr/local/nexus/bin/nexus start
ExecStop=/usr/local/nexus/bin/nexus stop
User=root
#User=nexus
Restart=on-abort
[Install]
WantedBy=multi-user.target

systemctl daemon-reload
systemctl status nexus
#注意，即使前面启动了，这里也会显示没有启动
nexus stop
systemctl enable --now nexus.service
#配置自启
```
### 查看Nexus面板初始密码
```bash
cat /usr/local/sonatype-work/nexus3/admin.password
#第一次登录需要修改密码，同时提示是否开启匿名下载，一般选择还是开启，如果你的仓库不对外又没有特殊要求的话
```
### Nexus优化
#### 默认maven仓是国外，修改至国内
![修改maven-central默认仓](/images/Nexus优化.png)
![修改为maven-central为国内源](/images/Nexus优化2.png)
```
修改为https://maven.aliyun.com/mvn/guide
```
### 修改Maven配置文件指向nexus
```bash
vim /etc/maven/settings.xml
#添加以下内容
<mirror>
<id>my-nexus</id>
<mirrorOf>*</mirrorOf>
<name>My-Nexus</name>
<url>http://10.0.0.201:8081/repository/maven-central/</url>
</mirror>
```
### 添加自定义存储目录
#### names可自定义，路径会自动创建
![添加自定义存储路径](/images/数据存放目录.png)


### 配置apt仓
#### 这里要注意ubuntu各版本的代号，图里我填的是18版本的Bionic，22.04是jammy，请注意
![配置apt仓](/images/创建apt仓库源.png)
![复制apt仓库链接](/images/apt-link.png)
### 在客户机配置
```bash
cp  /etc/apt/sources.list ./
vim /etc/apt/sources.list
deb http://10.0.0.201:8081/repository/ubuntu22.04/ jammy main restricted universe multiverse
deb-src http://10.0.0.201:8081/repository/ubuntu22.04/ jammy main restricted universe multiverse
deb http://10.0.0.201:8081/repository/ubuntu22.04/ jammy-security main restricted universe multiverse
deb-src http://10.0.0.201:8081/repository/ubuntu22.04/ jammy-security main restricted universe multiverse
deb http://10.0.0.201:8081/repository/ubuntu22.04/ jammy-updates main restricted universe multiverse
deb-src http://10.0.0.201:8081/repository/ubuntu22.04/ jammy-updates main restricted universe multiverse
deb http://10.0.0.201:8081/repository/ubuntu22.04/ jammy-proposed main restricted universe multiverse
deb-src http://10.0.0.201:8081/repository/ubuntu22.04/ jammy-proposed main restricted universe multiverse
deb http://10.0.0.201:8081/repository/ubuntu22.04/ jammy-backports main restricted universe multiverse
deb-src http://10.0.0.201:8081/repository/ubuntu22.04/ jammy-backports main restricted universe multiverse
:wq
```
### 客户端主机测试
```bash
apt update
```
![可以看到有数据了](/images/测试下载.png)
### yum源同理，这里就不在过多叙述

#### 小结以下：实际上最简单的仓库就是apache和nginx了，我之前文章有写过。但相比之下，nexus可以在图形化界面操作，能够省去不少重复操作，为运维人员分担一定的压力，其实像nexus这样的开源仓库管理服务还有许多，例如Artifactory，Nexus 2，JFrog Bintray等等，有兴趣的可自行尝试。
### 常用Type
1. Hosted：本地仓库，通常我们会部署自己的构件到这一类型的仓库，比如公司的第三方库
2. Proxy：代理仓库，它们被用来代理远程的公共仓库，如maven 中央仓库(官方仓库)
3. Group：仓库组，用来合并多个 hosted/proxy 仓库，当你的项目希望在多个repository 使用资源时就不需要多次引用了，只需要引用一个 group 即可
### 参考文档
1. [https://help.sonatype.com/repomanager3/product-information/download/download-archives---repository-manager-3](https://help.sonatype.com/repomanager3/product-information/download/download-archives---repository-manager-3)
2. [https://help.sonatype.com/repomanager3/installation](https://help.sonatype.com/repomanager3/installation)
3. [https://help.sonatype.com/repomanager3/installation/system-requirements](https://help.sonatype.com/repomanager3/installation/system-requirements)
