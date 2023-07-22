---
title: LVS-NAT,DR,TUNNL模式简单实现
date: 2023-05-13 20:36:00
tags: [linux,代理服务器,负载均衡]
categories: 实用教程
cover: https://pan.alybaba.top/images/linux.jpg
---
# LVS-NAT简单实战
注意：请关闭所有主机selinux和firewalld，架构图如下
![LVS-NAT](/images/LVS-NAT.png)
## LVS配置
```bash
hostnamectl set-hostname lvs
#修改主机名
echo "1" >  /proc/sys/net/ipv4/ip_forward
#开启ip转发，回包是需要经过LVS
apt update && apt install ipvsadm
#安装软件
ipvsadm -A -t 172.18.0.100:80 -s wrr
#新增VS配置，-t为tcp，-s为模式，wrr为权重轮询
ipvsadm -a -t 172.18.0.100:80 -r 192.168.59.131:80 -m
#新增RS配置,-r表示后端服务器，-m为NAT模式
ipvsadm -a -t 172.18.0.100:80 -r 192.168.59.132:80 -m
vim /etc/netplan/00-installer-config.yaml
network:
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 172.18.0.100/24
      gateway4: 172.18.0.1
      nameservers:
         addresses: [172.18.0.1,8.8.8.8]
    ens37:
      dhcp4: no
      addresses:
        - 192.168.59.100/24
  version: 2
#修改网卡文件

ipvsadm -Ln
#查看集群配置信息
ipvsadm-save -n > ipvsadm.rule
#以数字方式保存配置
ipvsadm-restore <  ipvsadm.rule
#从文件加载配置
```

## RS1配置
```bash
apt update && apt install ngixn -y
echo "仅主机 192.168.59.131" > /var/www/html/index.nginx-debian.html
systemctl start nginx
vim /etc/netplan/00-installer-config.yaml
network:
  ethernets:
    ens33:
      dhcp4: no
      addresses:
          - 192.168.59.131/24
      gateway4: 192.168.59.100
  version: 2

netplan apply
```
## RS2配置
```bash
apt install ngixn -y
echo "仅主机 192.168.59.132" > /var/www/html/index.nginx-debian.html
systemctl start nginx
vim /etc/netplan/00-installer-config.yaml
network:
  ethernets:
    ens33:
      dhcp4: no
      addresses:
          - 192.168.59.132/24
      gateway4: 192.168.59.100
  version: 2

netplan apply
```
## 测试：
在Client上curl 172.18.0.100，在这里我们配置的权重轮询，但是没有配置weight，所以每curl一次都会换一次主机，一定程度上进行了负载。
```bash
ipvsadm -e -t 172.18.0.100:80 -r 192.168.59.131:80 -m -w 3
#这里我们修改了131主机权重为3，也就是比如客户机每访问4次，3次为131，1次为132。比例为3比1
```
## 总结：
请记得一定要关闭防火墙，如果为redhat，请禁用selinux。其次NAT模式需要在LVS主机上进行IP地址的转换，会产生额外的网络开销，降低了网络性能，所以LVS主机需要具备较高的性能和带宽。在本次实验中的NAT模式存在单点故障的风险，实际生产最好使用主备，有兴趣的可自行尝试，这里就不再做过多叙述。

# LVS-DR模式单网段案例
注意：请关闭所有主机selinux和firewalld，架构图如下
![LVS-DR-单网段](/images/LVS-DR-single.png)
## CIP配置
```bash
hostnamectl set-hostname CIP
route del default
#删除默认路由
route add default gw 172.18.0.100
#增加默认路由，也就是网关，下一跳地址
route -n
#打印路由
```
## Route配置
```bash
hostnamectl set-hostname route
echo "1" >  /proc/sys/net/ipv4/ip_forward
systemctl stop firewalld
cd /etc/sysconfig/network-scripts/
vim  ifcfg-ens160
	BOOTPROTO=static
	IPADDR=172.18.0.100
	NETMASK=255.255.255.0
	NAME=ens160
	DEVICE=ens160
	ONBOOT=yes
cp ifcfg-ens160  ifcfg-ens224
vim  ifcfg-ens224
	BOOTPROTO=static
	IPADDR=192.168.59.200
	NETMASK=255.255.255.0
	NAME=ens224
	DEVICE=ens224
	ONBOOT=yes

nmcli con reload; nmcli con up ens160;nmcli con up ens224
#重加载配置文件，并不会启动，所以后面还要加启动命令
#注意，Router用的是redhat系列，如果是debian系列，网卡配置可能稍有不同
```
## LVS配置
```bash
hostnamectl set-hostname lvs
systemctl stop firewalld
apt update && apt install ipvsadm
vim /etc/netplan/00-installer-config.yaml
network:
  ethernets:
    ens37:
      dhcp4: no
      addresses:
        - 192.168.59.130/24
      gateway4: 192.168.59.200

  version: 2

netplan apply
#重载网卡配置文件，且重启网卡
ifconfig lo:1 192.168.59.100 netmask 255.255.255.255
#为环回网口配置IP
ipvsadm -A -t 192.168.59.100:80 -s rr
#rr为轮询算法
ipvsadm -a -t 192.168.59.100:80 -r 192.168.59.131:80 -g
#-g为DR模式
ipvsadm -a -t 192.168.59.100:80 -r 192.168.59.132:80 -g
ipvsadm -Ln
```
## RS1配置
```bash
IP配置参考LVS
hostnamectl set-hostname RS1
systemctl stop firewalld
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
#ignore=1使该网卡只响应LVS的请求
#announce=2，忽略IP数据包的源IP地址
ifconfig lo:1 192.168.59.100 netmask 255.255.255.255
```
## RS2配置
```bash
hostnamectl set-hostname RS2
systemctl stop firewalld
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
ifconfig lo:1 10.0.0.100 netmask 255.255.255.255
```
LVS的eth0的网关可否不配置？如果随便配置，发现什么问题？如果不配置，怎么解决
可以，修改一下内核参数就行
```bash
ubuntu
echo "0" > /proc/sys/net/ipv4/conf/all/rp_filter
echo "0" > /proc/sys/net/ipv4/conf/LVS 网卡名/rp_filter
#关闭反向路径过滤，所有数据包均允许通过。

redhat
echo "0" > /proc/sys/net/ipv4/conf/all/rp_filter
#参数rp_filter用来控制系统是否开启对数据包源地址的校验。
#0表示不校验
#1表开启严格的反向路径校验。对每一个收到的数据包，校验其反向路径是否是最佳路径。如果反向路径不是最佳路径，则直接丢弃该数据包；
#2表示开启松散的反向路径校验，对每个收到的数据包，校验其源地址是否可以到达，即反向路径是否可以ping通，如反向路径不通，则直接丢弃该数据包。
```
LVS的VIP可以配置到lo网卡,但必须使用32位的netmask,为什么?
```bash
由于回环地址是一个单独的网络，因此需要使用32位掩码来配置回环网口的地址，以确保只有该地址的唯一性，避免因为掩码设置错误而导致不必要的网络通信和安全问题。
```
## 测试
```bash
crul 192.168.59.100
#这里访问的是lvs，终端应轮询返回RS1和RS2的网址信息
```
## 总结：
LVS-DR模式单网段可能更容易被黑客扫描，因为LVS-DR模式下，虚拟服务器和Real Server都使用了同一网段，这使得黑客更容易发现虚拟服务器所在的IP地址。而且，由于Real Server的响应包不会经过Director，所以在单网段情况下，Real Server的响应包直接返回给客户端，黑客可以通过分析这些响应包来发现Real Server的IP地址。其次LVS主机和后端RS在同一个子网内，因此可能存在ARP欺骗的风险。同时要注意的是我在这里修改内核参数只是临时修改，重启后参数会复位，回环网卡ip也是同理。如果需要永久生效，请修改网卡配置文件。ipvsadm也可配置开机自启。


# LVS-DR模式多网段案例
我们可直接在单网端上稍作修改即可，架构图如下
![LVS-DR-多网段](/images/LVS-DR-more.png)
```bash
在LVS，RS1，RS2上删除原来地址，然后添加新地址
ifconfig  lo:1 del 192.168.59.100
ifconfig lo:1 10.0.0.100 netmask 255.255.255.255
```
## Router配置
```bash
ifconfig lo:1 10.0.0.100 netmask 255.255.255.255
```
## LVS配置
```bash
lvsadm -C
ipvsadm  -A -t 10.0.0.100:80 -s wrr
ipvsadm  -a -t 10.0.0.100:80 -r 192.168.59.131:80 -g -w 1
ipvsadm  -a -t 10.0.0.100:80 -r 192.168.59.132:80 -g -w 1
#-w可不写，权重默认为1
```
## 测试
```bash
crul 10.0.0.200
#终端应轮询返回RS1和RS2的网址信息
```
## 总结：
LVS-DR多网段模式是在LVS-DR单网段模式的基础上进行改进的，多个网段来增加LVS集群的可扩展性，同时避免了单网段模式中可能出现的ARP攻击问题。此外，多网段模式还可以让LVS集群在多个物理网络上分布，以提高整个集群的可靠性。但相应而来的缺点是需要在路由器上配置静态路由或动态路由协议，这可能会增加一些管理成本。其次，LVS-DR多网段模式还可能会增加一些网络延迟，特别是当LVS集群分布在多个物理网络上时。LVS-DR多网段模式适合于大型网络环境中需要高可扩展性和高可靠性的应用场景，例如互联网数据中心、大型企业应用、电子商务等场景。同时，也适合于需要通过LVS来实现负载均衡和高可用性的应用程序。


# LVS-TUNNEL隧道模式
根据DR多网段进行修改，架构图如下
![LVS-DR-隧道](/images/LVS-TUNNEL.jpg)

## 为LVS，RS1，RS2添加tun0隧道地址
```bash
ifconfig  lo:1 del 10.0.0.100
#添加tunl隧道方法1
ifconfig tunl0 192.168.59.100 netmask 255.255.255.255 up
#添加tun隧道方法2
ip addr add 192.168.59.100/32 dev tunl0
ip link set up tunl0
```
## Router配置
```bash
ip address delete  10.0.0.200/24 dev ens224
```

## LVS配置
```bash
route add default gw 192.168.59.200
#上次实验在最后把网关给删了，在这里我们给他加上
```
## 测试：
```bash
curl 192.168.59.200
#在这里我们可以使用wireshark进行抓包
```
## 抓包图
![LVS-TUNNEL-抓包](/images/LVS-TUNNEL-抓包.png)
## 原理：
### LVS的tunnel模式是指将请求的数据包封装在一个新的数据包中，再通过IP网络传输到后端的Real Server上进行处理的一种LVS工作模式。在tunnel模式下，LVS将请求的数据包封装在一个新的数据包中，新的数据包的目的IP地址是后端Real Server的IP地址，这样数据包才能传递到Real Server上。
## 总结：
tunnel模式的优点是支持跨网段的负载均衡，因为请求数据包经过封装后，可以通过IP网络传输到不同网段中的Real Server上。但是由于封装和解封装的过程会增加额外的延迟和负担，因此性能比直接路由的DR模式差。同时，tunnel模式也需要额外的配置和管理工作，增加了部署和维护的难度。


# LVS高可用
## LVS和RS故障
### 解决方案
#### LVS + keepalived：keepalived是一种常见的高可用解决方案，可以通过监测LVS服务器的状态来实现自动故障转移和负载均衡。当主LVS服务器出现故障时，keepalived会自动将VIP转移到备用LVS服务器上。

#### LVS + heartbeat/corosync：heartbeat/corosync是另一种常用的高可用解决方案，可以在多个LVS服务器之间同步状态，并实现自动故障转移和负载均衡。当主LVS服务器出现故障时，备用LVS服务器可以接管其工作。

#### LVS + ldirectord：ldirectord是一个可以管理LVS集群的工具，可以实现自动故障转移和负载均衡。与keepalived和heartbeat/corosync相比，ldirectord的配置更为灵活，可以根据实际需求来进行配置。
