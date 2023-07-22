---
title: Tips 1
date: 2023-02-28 09:05:21
tags: linux
categories: linux基础
cover: https://pan.alybaba.top/images/linux.jpg
---
为history设置时间和用户log

vi打开/etc/profile
输入i进入输入模式
page dn 在文件最下方添加以下内容：
```bash
HISTTIMEFORMAT="%Y-%m-%d:%T:`whoami` "
export HISTTIMEFORMAT
输入esc，:wq
source /etc/profile
```
然后在终端输入history可以发现执行命令时会有时间以及角色了。（感觉无卵用）


Tips 2


快速切换ROOT用户
```bash
vi /etc/sudoers.d/fastchangeusers
输入i进入编辑模式
输入：users(你使用的普通用户名) ALL=(ALL) NOPASSWD:ALL
输入esc
:wq保存退出
```
然后使用普通user切换root就不用密码了,其实也可以通过visudo添加该内容
效果是一样的，或者直接编辑/etc/sudoers文件也可以

Tips 3

解决centos网卡问题

有时候重启虚拟机后，网卡莫名其妙就获取不到地址了，百思不得其解。（可能电脑太垃圾！！）
网上找到方法解决，如下：

临时解决方案
```bash
dhclient 网卡名称 //为网卡获取ip地址
```

永久解决方法
```bash
nmcli networking // 查看托管状态
如果显示disabled，则：

nmcli networking on //开启托管

systemctl restart NetworkManager //重启网络管理服务

ip a  //可查看能够正常获取ip地址
```

注意如果不是disabled,强烈建议点击该[button](https:www.baidu.com)获取答案

Tips 4

统计ssh登录失败的用户，ip，尝试登录次数
```bash
lastb  | tr -s ' ' |cut -d " " -f1,3 |uniq -c |head -n 2 > sshfaild.list

lastb //输出ssh失败的用户
tr -s ' ' //把空格变成1个
cut -d " " // 使用空格进行分割 -f1,3 //显示第一列和第三列
uniq -c //统计
head -n 2 去掉最后两条
·>· sshfaild.list 将输出结果重定向到该文件。
```
（ps：使用密钥的不用看了）

Tips 5

使用管道符得到"ip a"的ip
```bash
ip a| grep inet | head -n3 |tail -n1 |tr -s ' ' |cut -d " " -f3

#10.0.0.153/24
不知道怎么去掉/24

建议直接输入：hostname -I

注意，如果有docker，openvpn...等软件，可能会有多块网卡。
```
Tips 6

查看前10cpu，mem占用
```bash
ps aux | head -1;ps aux |grep -v PID |sort -rn -k +3 | head //cpu
ps aux | head -1;ps aux |grep -v PID |sort -rn -k +4 | head //mem
```

