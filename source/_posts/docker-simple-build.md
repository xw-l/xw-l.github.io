---
title: 简单构建tomcat镜像
date: 2023-06-18 22:57:02
tags: [Docker,容器,Alpine,Tomcat]
categories: 实用教程
cover: https://pan.alybaba.top/images/docker-facebook.png
---

# 部署准备
- 安装docker环境
# 部署步骤
安装docker环境
```bash
apt update && sh <(curl -fsSL https://get.docker.com)
```
测试
```bash
docker version
```
拉取alpine和ubuntu镜像,这里也可以直接拉官方的，我是拉本地仓的
![pull-images](/images/pull-images.jpg)
创建目录,先编译基于alpine镜像
```bash
mkdir /Dockerfile/web/tomcat/{alpine,ubuntu} -p && cd /Dockerfile/web/tomcat/alpine
```
创建dockerfile文件
```bash
cat << EOF > dockerfile
FROM harbor.test.com/tomcat/alpine-tomcat:v1.0
LABEL autor="lee"
ENV tomcat_version=8.5.90 PATH=/usr/local/tomcat/bin:$PATH
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/'  /etc/apk/repositories \
    && apk add --no-cache openjdk8 \
    #因为alpine要支持 oracle jdk 需要安装几个库依赖文件，这里就不折腾了
    && ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo "Asia/Shanghai" > /etc/timezone \
    && addgroup tomcat -g 2023 && adduser -Gtomcat -u 2023 -D tomcat \
    && wget -P /usr/local http://pan.alybaba.top/soft/apache-tomcat-${tomcat_version}.tar.gz \
    #这里可使用 ADD 进行添加,我偷个懒...
    && tar xf /usr/local/apache-tomcat-${tomcat_version}.tar.gz -C /usr/local \
    && ln -s /usr/local/apache-tomcat-${tomcat_version} /usr/local/tomcat \
    && rm -rf /usr/local/*.tar.gz \
    && chown -R tomcat.tomcat /usr/local/apache-tomcat-${tomcat_version}/
CMD ["catalina.sh","run"]
EXPOSE 8080
EOF
```
接下来执行build，可以看到没有问题,然后执行一下，一个简单的alpine+tomcat就完成了
![build-images](/images/build-images.jpg)
web端查看,可以看到没有问题,接下来试试ubuntu
![web](/images/web-test.jpg)
进入目录
```bash
cd ../ubuntu
```
创建文件
```bash
cat << EOF > dockerfile
FROM harbor.test.com/tomcat/ubuntu-tomcat:v1.0 
LABEL autor="lee" 
ENV tomcat_version=8.5.90 jdk_version=1.8.0_371  PATH=/usr/local/tomcat/bin:$PATH  \
    #JAVA_HOME=/usr/local/jdk \
    #PATH=${PATH}:${JAVA_HOME}/bin  JRE_HOME=${JAVA_HOME}/jre \
    #CLASSPATH=.:${JAVA_HOME}/lib/:${JRE_HOME}/lib/ PATH=/usr/local/tomcat/bin:${PATH}
    #我在脚本source了，这里我也懒的给变量了
RUN sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list \
    && apt update; export DEBIAN_FRONTEND=noninteractive && apt install -y tzdata wget \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo Asia/Shanghai > /etc/timezone \
    && groupadd tomcat -g 2023 && useradd tomcat -u 2023 -g tomcat && rm -rf /var/lib/apt/lists/* \
    && wget -P /usr/local http://pan.alybaba.top/soft/jdk-${jdk_version}-linux-x64.tar.gz \
    #这里下载纯属偷懒，一般下载到构建目录用 ADD 复制进去，自动解压
    && tar xf /usr/local/jdk-${jdk_version}-linux-x64.tar.gz -C /usr/local \
    && ln -s /usr/local/jdk${jdk_version} /usr/local/jdk \
    && echo  "export JAVA_HOME=/usr/local/jdk\nexport PATH=\$PATH:\$JAVA_HOME/bin\nexport JRE_HOME=\$JAVA_HOME/jre\nexport CLASSPATH=.:\$JAVA_HOME/lib/:\$JRE_HOME/lib/" > /etc/profile.d/jdk.sh  \
    && wget -P /usr/local http://pan.alybaba.top/soft/apache-tomcat-${tomcat_version}.tar.gz \
    && tar xf /usr/local/apache-tomcat-${tomcat_version}.tar.gz -C /usr/local \
    && ln -s /usr/local/apache-tomcat-${tomcat_version} /usr/local/tomcat && apt autoremove -y  wget \
    && echo 'PATH=/usr/local/tomcat/bin:$PATH' > /etc/profile.d/tomcat.sh  && rm -rf /usr/local/*.tar.gz \
    && chown -R tomcat.tomcat /usr/local/tomcat  && echo   'JAVA_HOME=/usr/local/jdk' >  /usr/local/tomcat/conf/tomcat.conf
COPY entrypoint.sh /
ENTRYPOINT ["/entrypoint.sh"]
EXPOSE 8080
EOF
```
创建entrypoint脚本.........
```bash
cat << EOF entrypoint.sh
#!/bin/bash
. /etc/profile.d/jdk.sh
#tomcat的变量我写进去了....
sleep 1
catalina.sh run
EOF
```
构建镜像
```bash
chmod +x entrypoint  && docker build -t ubuntu-tomcat:v1.0.1 .
```
可以看到很大....
![build-images](/images/build-images-2.jpg)
那我们来访问试试，没问题。那么到这里就简单完成了docker镜像构建
![web-test-2](/images/web-test-2.jpg)

# 参考资料
- dockerfile 官方说明: [https://docs.docker.com/engine/reference/builder/](https://docs.docker.com/engine/reference/builder/)
