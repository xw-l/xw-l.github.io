---
title: Zabbix和Grafana
date: 2023-06-06 19:00:10
tags: [Zabbix,Grafana,图形化监控]
categories: 实用教程
cover: https://pan.alybaba.top/images/zabbix_logo.png
---

前面介绍过简单安装zabbix了，想必zabbix自带的图形化可能无法满足需要，那么今天简单介绍下使用grafana来生成zabbix的图形化界面。

1. Grafana 是一个流行的开源数据可视化和监控平台，它提供了丰富的仪表盘和图表，用于可视化各种数据源的指标和日志。
2. Grafana 可以用于各种监控和数据可视化场景，如服务器监控、应用性能监控、网络监控、物联网数据可视化等。
3.  Grafana 提供了一个强大而灵活的平台，使用户能够通过仪表盘和图表更好地理解和分析数据。

`部署步骤`
ubuntu
```bash
apt install -y adduser libfontconfig1
#安装添加用户，组工具和字体配置和渲染的库
```
下载grafana安装装包
```bash
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_9.5.2_amd64.deb
```
本地安装grafana包
```bash
dpkg -i grafana-enterprise_9.5.2_amd64.deb
```
redhat
```bash
yum install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-9.5.2-1.x86_64.rpm
#官方源安装，Appstream版本过低，这里安装最新版的
```
或者清华源加速
```bash
yum install -y https://mirrors.tuna.tsinghua.edu.cn/grafana/yum/rpm/Packages/grafana-enterprise-9.5.2-1.x86_64.rpm
```
使用grafana命令工具安装zabbix插件
```bash
grafana-cli plugins install alexanderzobnin-zabbix-app
#默认安装最新版
```
查看zabbix历史所有版本
```bash
grafana-cli plugins list-versions  alexanderzobnin-zabbix-app
```
可根据list安装指定版本号
```bash
grafana-cli plugins install alexanderzobnin-zabbix-app  版本号
```
查看已安装的插件,我是安装的最新版
```bash
grafana-cli plugins ls
```

然后在web端Plugins启用zabbix(不想贴图了ing)
然后去Data sources添加zabbix，修改以下三项
```txt
URL  http://localhost:8080/api_jsonrpc.php
user name  Admin
password   zabbix
#我用的nginx
#这里要注意的是，如果是apache，端口号后应增加zabbix，这里我使用IP了，域名同理
```
自带的面板无法加载，去[https://grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards)下载一个
打开链接后搜索zabbix，记住编号，或者下载json文件
![download zabbix dashboard](/images/dashbords.jpg)
回到web界面，点击Dashboards，最右边NEW，选择import
![setting zabbix dashboard](/images/import-dashbords.jpg)
可以选择本地文件上传，也可输入模板的ID号进行在线load,实质上输入ID也是下载json文件，所以没有网络的情况下，可使用其他电脑下载
![import zabbix dashboard](/images/import-dashbords-2.jpg)
这里我选择的load，稍等片刻加载后源选择zabbix即可。忘了说Grafana服务默认端口号为3000,如果想自定意可以按照以下操作
```bash
rpm -ql grafana-enterprise | grep etc
#查看配置文件
vim /etc/grafana/grafana.ini
;http_port = 3000
http_port = you port
```
最终得到以下界面，然后有些参数显示不正确，可以自定义进行编辑，这里不在过多叙述。
![最终效果图](/images/last-image.jpg)
参考链接
- 官方下载链接[https://grafana.com/grafana/download](https://grafana.com/grafana/download)
- 官方插件下载链接[https://grafana.com/grafana/plugins/](https://grafana.com/grafana/plugins/)
- 官方面板下载链接[https://grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards)
- 清华源加速[https://mirrors.tuna.tsinghua.edu.cn/grafana/](https://mirrors.tuna.tsinghua.edu.cn/grafana/)
