---
title: 简单使用kvm
date: 2023-06-08 15:36:42
tags: [Kvm,虚拟机]
categories: 实用教程
cover: https://pan.alybaba.top/images/kvm.jpg
---

KVM是一种开源的虚拟化技术，它是基于Linux内核的虚拟化解决方案，全称是Kernel-based Virtual Machine。KVM利用Linux内核的虚拟化扩展，将物理主机转变为可以运行多个虚拟机的虚拟化平台。那么这里我将使用centos和ubuntu来进行简单相关部署
`ubuntu 22.04部署步骤`
安装相关软件,虚拟机需开启虚拟化,我这里给的内存和核心都为8
```bash
apt -y install qemu-kvm virt-manager libvirt-daemon-system cockpit cockpit-machines  bridge-utils libosinfo-bin
```
开机自启
```bash
systemctl enble --now libvirtd.service
#开源的虚拟化管理守护进程，用于支持多种虚拟化技术，包括 KVM、QEMU、Xen、LXC 等。
```
web管理界面自启
```bash
systemctl enable --now  cockpit
systemctl status libvirtd.service
brctl show
#这里是查看网桥信息，该命令可添加，删除网桥，默认kvm是桥接了一个virbr0的网桥
```
这里使用virt-manager
```bash
export DISPLAY=10.0.0.1:0.0
#0.0为xmanager，这里我已经安装并打开了xmanager passive
```
打开xmanager
```bash
virt-manager
```
![X-manager](/images/xmanager-ok.jpg)
乱码请选择英文
```bash
localctl set-locale LANG=en-US.UTF-8
```
创建目录
```bash
mkdir /data/iso && cd /data/iso
```
安装windows
```bash
virt-install \
--virt-type=kvm \
--name win2016 \
--ram 4096 \
--vcpus 4 \
--os-variant=win2k16 \
#注意，这里是根据libosinfo-bin | grep win查看到的
--cdrom=/data/iso/cn_windows_server_2016_x64_dvd_9718765.iso \
--network=bridge=virbr0,model=virtio \
--graphics vnc,listen=0.0.0.0 --noautoconsole \
--disk path=/var/lib/libvirt/images/win2016.qcow2,size=20,bus=virtio,format=qcow2
```
进入控制台
```bash
virt-manager
#这里报错请export DISPLAY=你和虚拟机网卡的网关:0.0
```
然后一直点下一步，直到这里，因为默认是不支持windows的virtio
![缺少驱动](/images/win-1.jpg)

下载驱动
```bash
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.229-1/virtio-win-0.1.229.iso
```
然后重新来...记得删除或者暂停改个名也行
```bash
virt-install \
--virt-type=kvm \
--name win2016 \
--ram 4096 \
--vcpus 4 \
--os-variant=win2k16 \
--cdrom=/data/iso/cn_windows_server_2016_x64_dvd_9718765.iso \
--network=bridge=virbr0,model=virtio \
--graphics vnc,listen=0.0.0.0 --noautoconsole \
--disk path=/var/lib/libvirt/images/win2016.qcow2,size=20,bus=virtio,format=qcow2 \
--disk path=/data/iso/virtio-win-0.1.229.iso,device=cdrom
#这里就是下载的驱动，关联一下
```
可以看到已经加载的原镜像和驱动
![成功加载](/images/win-2.jpg)
选择加载驱动程序
![选择驱动](/images/win-3.jpg)
选择2016，下一步
![加载驱动](/images/win-4.jpg)
可以看到出来磁盘了，这里我就默认了
![配置磁盘](/images/win-5.jpg)
等待安装...
![安装中...](/images/win-6.jpg)
经过N遍重启成功进入
![安装中...](/images/win-7.jpg)
安装网卡驱动
![安装网卡驱动](/images/win-8.jpg)
安装PCI驱动
![安装PCI驱动](/images/win-9.jpg)
其他驱动可依葫芦画瓢去挂载的驱动ISO里面去找，windows克隆可能会出现冲突，所以想要克隆可执行win自带初始化程序(谁会没事在linux克隆win？？？)
![初始化](/images/win-10.jpg)
打快照，点击两个个电脑图标
![快照](/images/win-11.jpg)
接着创建快照就行了
![快照](/images/win-12.jpg)

`rocky 8.x 部署步骤`
centos我就用图形web安装了
```bash
yum -y install qemu-kvm libvirt virt-manager virt-install  virt-viewer   cockpit  cockpit-machines
systemctl start --now cockpit.socket
systemctl start --now libvirtd
```
输入本机ip加port，我这里是10.0.0.200:9090
![首页](/images/fist-page.jpg)
然后点击虚拟机，创建虚拟机，注意，我已经创建目录，上传了镜像，所有我选择本地安装，这里还能编辑配置，点击创建并运行或者安装即可
![创建虚拟机](/images/create-vm.jpg)
![更多硬件自定义配置](/images/more-setting.jpg)
点一下界面，用鼠标上下键选择安装centos7
![安装centos7](/images/create-ok.jpg)
做好配置好进行安装，这里安装的mini版，所以就300个,完成后请用 TAB 键选择reboot重启
![安装centos7-1](/images/install-centos7.jpg)
到这里就安装完成了
![成功安装centos7](/images/install-ok.jpg)

`命令行配置`
创建一个20G的虚拟机磁盘文件，copy该文件，在图形化界面可导入
```bash
qemu-img create -f qcow2 /var/lib/libvirt/images/centos7.qcow2 20G
```
初始化配置
```bash
virt-install --virt-type kvm \
--name centos7 \
--ram 1024 \
--vcpus 2 \
--cdrom=/data/isos/CentOS-7-x86_64-Minimal-2009.iso \
--disk path=/var/lib/libvirt/images/centos7.qcow2 \
--network network=default \
--graphics vnc,listen=0.0.0.0 \
--noautoconsole \
--os-variant=centos7.0
#注意这里os-variant应该和支持的系统相同，这里必须先创建文件才能执行该命令
#--virt-type kvm意思是虚拟化类型为kvm
#--name centos7 虚拟机名称为centos7，可以自定义
#--ram 1024 内存分配1024mb，也可使用memory
#--vcpus 2 分配cpu核数为2核
#--cdrom= 指定光驱镜像路径为
#--disk=  指定虚拟机磁盘镜像为
#--network network=default 指定网络为系统默认
#--graphics vnc,listen=0.0.0.0 指定虚拟机使用 VNC 图形界面，并监听所有网络接口。
#--noautoconsole 不自动打开虚拟机的控制台
#--os-variant=centos7.0 指定虚拟机的操作系统为 CentOS 7
```
或者指定虚拟机磁盘文件一步到位
```bash
virt-install --virt-type kvm \
--name centos7 \
--ram 1024 \
--vcpus 2 \
--cdrom=/data/isos/CentOS-7-x86_64-Minimal-2009.iso \
--disk path=/var/lib/libvirt/images/centos7.qcow2,size=10,format=qcow2,bus=virtio \
--network network=default \
--graphics vnc,listen=0.0.0.0 \
--noautoconsole \
--os-variant=centos7.0
#size=10,format=qcow2,bus=virtio,size=10为虚拟磁盘文件为10G，格式为qcow2，总线为virtio
```
`克隆机器`
方法1，复制已经安装的虚拟机磁盘文件
```bash
cp -a  /var/lib/libvirt/images/centos7-1.qcow2  /var/lib/libvirt/images/centos7-2.qcow2
```
在图形化界面选择导入
![导入复制的虚拟机磁盘文件](/images/clone-1.jpg)
指定路径，选择操作系统
![继续操作](/images/clone-2.jpg)
配置cpu和ram
![配置硬件](/images/clone-3.jpg)
这里我改个名
![修改虚拟机名称](/images/clone-4.jpg)
然后完成了
![克隆完成](/images/clone-5.jpg)

方法2,复制虚拟机磁盘文件，注意前面的复制我删了
```bash
cp -a  /var/lib/libvirt/images/centos7-1.qcow2  /var/lib/libvirt/images/centos7-2.qcow2
```
命令行指定克隆机配置
```bash
virt-install --virt-type kvm \
--name centos7-2 \
--ram 1024 --vcpus 2 \
--disk path=/var/lib/libvirt/images/centos7-2.qcow2,bus=virtio \
--network network=default,model=virtio \
--graphics vnc,listen=0.0.0.0 \
--noautoconsole \
--autostart \
--boot hd
#--network network=default,model=virtio 使用virtio模型
#--autostart 虚拟机启动时自动启动
#--boot hd 启动顺序为从硬盘启动
```
然后在控制台也可以看见了，直接双击打开就ok
![命令行克隆](/images/clone-6.jpg)

方法3,基于现有虚拟机克隆
```bash
virt-clone -o centos7  -f /var/lib/libvirt/images/centos7-2.qcow2 -n centos7-2
#-o centos7 意思是根据现有虚拟机名称进行克隆
#-f /var/lib/libvirt/images/centos7-2.qcow2 新虚拟机磁盘文件的路径和名称
#-n centos7-2 新虚拟机的名称
#这里要注意的是，被克隆的虚拟机需要关机才行，不然会报错。然后克隆出来注意IP
#可写脚本批量创建
```
部分参考链接和术语解释
- [https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md)
- [https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/)
- [https://www.linux-kvm.org](https://www.linux-kvm.org)
- QEMU-KVM 是一种开源的虚拟化解决方案，结合了QEMU和KVM两个组件，用于创建和管理虚拟机
- virt-manager 是一个图形化的虚拟机管理工具，用于管理KVM和QEMU虚拟化环境。
- libvirt 是一个开源的虚拟化管理工具，它提供了一个统一的接口和工具集，用于管理各种虚拟化平台（如KVM、QEMU、Xen等）上的虚拟机和虚拟化资源。
- Cockpit 是一个用于管理Linux服务器的图形化Web界面工具
- Cockpit Machines（也称为Virtual Machines）是Cockpit的一个插件，它提供了一个图形化的界面来管理和操作虚拟机。
- Bridge-utils 是一个用于 Linux 系统的网络管理工具，它提供了创建和管理网络桥接的功能。
- libosinfo-bin 是一个用于操作操作系统信息库（OS information database）的命令行工具。
