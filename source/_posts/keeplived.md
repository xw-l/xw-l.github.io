---
title: 编译安装keepalived
date: 2023-06-26 21:20:02
tags: [keepalived,编译,ubuntu]
categories: 实用教程
cover: https://pan.alybaba.top/images/ubuntu.jpg

---
# 部署准备
- ubuntu22.04
# 部署步骤
安装相关依赖和工具
```bash
apt update && apt -y install make gcc ipvsadm buildessential pkg-config \
automake autoconf libipset-dev libnl-3-dev libnl-genl-3-dev \
libssl-dev libxtables-dev libip4tc-dev libip6tc-dev libmagic-dev libsnmp-dev \
libglib2.0-dev libpcre2-dev libnftnl-dev libmnl-dev libsystemd-dev
```
下载源码包
```bash
wget https://keepalived.org/software/keepalived-2.2.8.tar.gz
```
解压
```bash
tar xf keepalived-2.2.8.tar.gz && cd keepalived-2.2.8/
```
配置安装路径，禁用FW_MARK(可选)
```bash
./configure --prefix=/usr/local/keepalived --disable-fwmark
#禁用 Keepalived 的 FW_MARK 功能，需配合防火墙打标签实现高级路由和策略
#未禁用该选项 使用nft list ruleset 可看到会生成一个基于nftables的规则集，导致 VIP 无法正常通信
#实际测试我无论是关闭还是未关闭，都会生成防火墙规则集
#需注释 vrrp_strict 可使 VIP 正常通信
#./configure 可用 --help 选项查看其他编译选项
```
编译安装
```bash
make && make install
```
拷贝 service 文件
```bash
cp -a /root/keepalived-2.2.8/keepalived/keepalived.service /lib/systemd/system/
```
拷贝 conf 文件
```bash
mv  /usr/local/keepalived/etc/keepalived/keepalived.conf.sample  /usr/local/keepalived/etc/keepalived/keepalived.conf
```
启动
```bash
systemctl damon-reload  &&  systemctl enable --now keepalived
```
查看版本和编译相关参数，这里可添加路径到默认 PATH ，或软链接
```bash
/usr/local/keepalived/sbin/keepalived -v
```
查看相关参考手册
```bash
man /usr/local/keepalived/share/man/man1/genhash.1
#keepalived 命令行工具手册页
man /usr/local/keepalived/share/man/man5/keepalived.conf.5
#keepalived conf配置手册页
man /usr/local/keepalived/share/man/man8/keepalived.8
#keepalived 系统管理员手册
```
# 参考资料
- https://keepalived.org/doc/installing_keepalived.html
