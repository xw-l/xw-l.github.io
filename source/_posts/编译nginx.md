---
title: nginx 目录优化保姆级教程
date: 2023-03-21 14:31:23
tags: [Nginx,编译]
categories: 实用教程
cover: https://pan.alybaba.top/images/nginx-logo.png
---
记录一次nginx编译
	众所周知，做一个最简单的网盘就是使用web服务了，但是不管是nginx和apache或者apache2的默认网页目录都太丑了，所以怎么能自定义修改默认的目录风格呢？
以nginx为例，官方的autoindex模块只是简单的提供了以下几个选项
```bash
	命令			默认值	可选值		作用域			作用

	autoindex		off	on/off		http/server/location	开启目录浏览功能
	
	autoindex_format	html	html/xml	same			html是以网页功能展示目录内容
					json/jsonp
	autoindex_exact_size	on	on/off		same			开启为以字节，关闭为KB,MB,GB显示文件
										autoindex_format为html时有效
	autoindex_localtime	off	on/off		same			开启以服务器的文件时间作为显示的时间
										autoindex_format为html时有效
```
首先安装nginx,	 注意！这里演示的是安装后编译再增加,也可以编译时添加模块再安装,这里就不在做过多的展示
```bash
	yum or apt update && yum or apt upgrade
	yum or apt install nginx -y
	systemctl start nginx或者/sbin/nginx -s start
```
根据根据官方给的autoindex模版，一个简单的web网盘配置如下
```bash
	http
	{ 
#注意一条配置接一个分号，其次一般来说一行一个配置
		root	/you/path/www/;	//root为nginx根目录，这里是指定nginx目录所在，一般来说自动配置了，这里是自定义
		autoindex		on;	//开启目录浏览功能
		autoindex_format	html;	//以html风格将目录展示在浏览器中，一般都是html
		autoindex_exact_size	off;	//假如开启将会以字节显示文件大小，最好是关闭
		autoindex_localtime	on;	//以服务器的时间作为显示的时间
		charset utf-8,gbk;		//开启中文，假如有中文没有该选项将乱码显示
	}
```
接着我们vi /etc/nginx/nginx.conf把上面目录配置复制进去
![nginx 配置](/images/11.jpg)
然后输入nginx -t或/sbin/nginx -t测试，输出success则继续输入systemctl restart nginx或/sbin/nginx  -s reload
![nginx 测试](/images/2.png)
接着输入hostname -I，查看本机地址，注意可能会有多个地址，自己分辨
![查看本地ip](/images/3.png)
注意:上面只是获取局域网地址，如果你是部署云请使用以下命令或脚本
```bash
curl -Ls ip.tool.lu
```
然后打开浏览器输入你的ip地址,但是忘了一件事，开启端口，我使用的是rocky：
同时这里要注意，如果是云，图省事就直接在控制台把nginx配置所对应的端口打开，我这里以常见80为例
```bash
	firewall-cmd  --zone=public --add-port=80/tcp --permanent
```
规则生效需要重启
```bash
	firewall-cmd --reload
```
ufw和iptables
```bash
	ufw allow 80
	iptables -I INPUT -s 0.0.0.0/0 -p tcp --dport 80 -j ACCEPT
	或者iptables -F
```
接着刷新一下，可以看见了
![nginx 默认目录](/images/4.png)
可以看到，很鸡儿丑。接下来我们就编译一个主题给他
	首先继续输入nginx -V或者/sbin/nginx -V,我们可以看到版本号和编译的模块
![nginx 版本和已编译模块](/images/5.png)
	接着打开nginx.org，选择相应的版本使用wget下载该链接
![nginx 源码下载](/images/6.png)

解压到当前目录
```bash
	tar -xf  nginx-1.14.2.tar.gz
```	
进入该目录
```bash
	cd nginx-1.14.2	
```
把nginx主题模块拖进来
```bash
	git clone https://github.com/aperezdc/ngx-fancyindex/releases/download/v0.5.2/ngx-fancyindex-0.5.2.tar.xz
```
解压到当前目录
```bash
	tar -xf ngx-fancyindex-0.5.2.tar.xz
```	
然后前面的nginx -V的已编译信息直接复制过去，再跟上添加的模块路径执行脚本即可
```bash
	./configure  你已经安装编译的模块,这里太长，我就不复制了后面跟上要添加的模块就ok  --add-module=./ngx-fancyindex-0.5.2
```	
接着编译，千万不要install，不然就nginx就费了因为你是安装后编译这个要注意
```bash
	make 
```
注意:如果make这里很大几率报错，原因是缺少相关的依赖，所以请点击[这里](https:www.baidu.com)查询需要的依赖,再安装对应的编译依赖包
		
完事后把编译好的二进制程序nginx替换掉现有的nginx，请先暂停nginx service
编译成功会在objs目录下生成一个新的nginx二进制可执行程序
```bash	
	/sbin/nginx -s stop 或者 systemctl stop nginx
	cp -a objs/nginx /sbin/nginx
	/sbin/nginx -s reload
```	
现在打开网址可以看到一个雏形，还是丑。我们继续，先进入nginx目录里
```bash
	cd /you/path/www
```	
接着下载主题
```bash
	git clone https://github.com/lanffy/Nginx-Fancyindex-Theme
```	
再让fancyindex配置文件生效,打开nginx配置文件增加一行
```bash
	include /you/path/fancyindex.conf;
```	
测试nginx重启生效
```bash
	/sbin/nginx -t && /sbin/nginx -s reload
```	
最终效果，也可以访问[这里](http://pan.alybaba.top)查看我做的效果

![优化后的nginx目录](/images/7.png)

总结:其实还可以编译其他相关的nginx模块，这里就不再过多叙述，有兴趣的可前往github搜索...而且nginx还有很多有意思的配置，比如负载均衡，反向代理，正向代理等等...以后慢慢分享相关的玩法配置，在这里为什么我不使用apache呢？想必大家都知道log4j2零day漏洞了，这里就不再过多说明。但是编译过程最难的还是缺少相关依赖，对于一般人来说还是比较困难。总而言之，言而总之nginx是一个很高效且优质的代理web站点，学好不愁吃穿，当然在使用nginx时也要遵纪守法，不然就真的"不愁吃穿"了
