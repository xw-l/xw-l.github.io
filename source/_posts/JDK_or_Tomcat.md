---
title: Tomcat和JDK源码简单安装
date: 2023-05-24 22:36:00
tags: java
cover: https://pan.alybaba.top/images/tomcat.jpg
categories: 实用教程
---
### 简单介绍下JDK
### JDK（Java Development Kit）是Java开发工具包的缩写，它是用于开发和编译Java应用程序的软件包。JDK是Oracle提供的官方Java开发工具，包含了用于编译、调试和运行Java程序所需的各种工具和库。JDK是Java开发的核心工具，它为开发人员提供了创建、编译和调试Java应用程序所需的环境和工具。使用JDK，开发人员可以编写跨平台的Java代码，并将其编译为可在任何支持Java的平台上运行的字节码。
### Oracle JDK Version
![Oracle JDK Version](/images/JDK.png)
# Tomcat
### Tomcat是一个开源的、轻量级的、基于Java的Web应用服务器，它提供了Java Servlet、JavaServer Pages（JSP）和Java WebSocket等技术的支持，用于构建和运行Java Web应用程序。
### Tomcat Version
![Tomcat Version](/images/tomcat.png)
### 那么接下来我们就来简单的安装一下
## JDK源安装
```bash
yum list |grep jdk
#redhat系列查看包信息，注意如果没有安装JDK，将会附加安装
apt list| grep jdk
#debian系列查看包信息
yum install jdk-version
#redhat系列安装
apt install jdk-version
#debian系列安装
java -version
#如果没问题的话，这里可以查看java版本了
```
## JDK包安装
- [https://www.oracle.com/java/technologies/downloads/#java17](https://www.oracle.com/java/technologies/downloads/#java17)
- [https://www.oracle.com/java/technologies/downloads/#java8](https://www.oracle.com/java/technologies/downloads/#java8)
### 我是在甲骨文下载的，需要注册账号，同时java17有dep包，java8只能二进制安装
```bash
yum install jdk-version.rpm
vim   /etc/profile.d/jdk.sh
export JAVA_HOME=/usr/java/default
export PATH=$JAVA_HOME/bin:$PATH
#配置变量
#export JRE_HOME=$JAVA_HOME/jre
#export CLASSPATH=$JAVA_HOME/lib/:$JRE_HOME/lib/
#这两项非必需
source  /etc/profile.d/jdk.sh或
.  /etc/profile.d/jdk.sh
#生效变量
java -version
rpm -ql jdk-version.rpm
#查看安装信息
```
## JDK二进制文件安装
### 链接在上方，tar.gz为编译好的二进制文件，下载即可。
```bash
tar xf    jdk-version.tar.gz   -C   /usr/local/
cd /usr/local
ln -s jdk-name    jdk
vim   /etc/profile.d/jdk.sh
export JAVA_HOME=/usr/local/jdk
export PATH=$PATH:$JAVA_HOME/bin
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=.:$JAVA_HOME/lib/:$JRE_HOME/lib/

source /etc/profile.d/jdk.sh
java -version
```
## Tomcat安装
```bash
wget https://dlcdn.apache.org/tomcat/tomcat-8/v8.5.89/bin/apache-tomcat-8.5.89.tar.gz
tar  xf  apache-tomcat-8.5.89.tar.gz -C /usr/local
cd /usr/local
ln -s  apache-tomcat-8.5.89  tomcat 
echo 'PATH=/usr/local/tomcat/bin:$PATH' > /etc/profile.d/tomcat.sh
.  /etc/profile.d/tomcat.sh
useradd -r -s /sbin/nologin tomcat
#创建运行用户
chown -R tomcat.tomcat /usr/local/tomcat/
echo   'JAVA_HOME=/usr/local/jdk'   /usr/local/tomcat/conf/tomcat.conf
#写配置文件
vim /lib/systemd/system/tomcat.service
[Service]
Type=forking

EnvironmentFile=/usr/local/tomcat/conf/tomcat.conf
#Environment=JAVA_HOME=/usr/local/jdk
#如果没有写配置文件可启用此选项
ExecStart=/usr/local/tomcat/bin/startup.sh
ExecStop=/usr/local/tomcat/bin/shutdown.sh
PrivateTmp=true
User=tomcat
Group=tomcat
[Install]
WantedBy=multi-user.target

systemctl daemon-reload
#只要修改了systemctl service文件，都需要运行该命令
sysemctl start tomcat
systemctl status tomcat
ss -ntl
#默认是8080tcp端口
```
### 到这里，我们的JDK和Tomcat就部署完毕了，Tomcat 是一个受欢迎的 Java Web 应用服务器，适用于中小型项目和简单的 Web 应用。它的轻量级、易用性和良好的 Java EE 兼容性是其主要优点。值得注意的是，如果你要让Tomcat运行在80端口，需要在service文件里修改用户为root(ps：不合理操作，不建议。反正tomcat也不会部署在公网)生产环境一般是使用nginx这类高并发web服务器反向代理tomcat。同时部署tomcat前需要先部署JDK环境。还有我们在选择版本时应根据项目的实际情况进行抉择。

### 部分参考文献
- [https://cwiki.apache.org/confluence/display/tomcat/](https://cwiki.apache.org/confluence/display/tomcat/)
- [https://tomcat.apache.org/whichversion.html](https://tomcat.apache.org/whichversion.html)



