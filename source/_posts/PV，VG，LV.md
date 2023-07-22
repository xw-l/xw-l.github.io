---
title: 创建物理卷，卷组，逻辑卷，挂载，扩容
date: 2023-03-21 19:30:10
tags: linux
categories: linux基础
cover: https://pan.alybaba.top/images/linux.jpg
---
在生产实例中，突然会碰到磁盘告警爆满，而此时生产环境不能停止，所以我们用什么方法在不关机的情况下扩容磁盘呢，LVM应运而生。
LVM是什么？
LVM(Logical Volume Manager)逻辑卷管理是在Linux2.4内核以上实现的磁盘管理技术。它是Linux环境下对磁盘分区进行管理的一种机制。
现在不仅仅是Linux系统上可以使用LVM这种磁盘管理机制，对于其它的类UNIX操作系统，以及windows操作系统都有类似与LVM这种磁盘管理软件。
更通俗的说就是动态弹性管理磁盘，下面我们就来操作操作

1-我们在虚拟机上添加一块新的硬盘，大小随意，重启或者输入partprobe刷新，输入lsblk查看，一般来说是sdb

2-使用磁盘管理工具进行操作，fdisk一般来说是mbr，gdisk一般来说是gpt，这里我们就用fdisk来操作
```bash
	fdisk /dev/sdb
	输入m可以查看帮助，p命令打印分区信息，n命令创建新分区，t命令修改现在有分区的格式，w命令保存退出，q命令退出不保存
	首先我们输入n，可以看到有两个选项，第一个是主分区，第二个是扩展分区（如果选择扩展分区，则后续的第二个选项为逻辑分区
逻辑分区属于扩展分区，也就是说后续逻辑分区不限数量，但受限于你扩展分区的容量，有点绕）
	第一个选项分区号，逻辑分区无所谓，主分区必须是1-4以内4后面都属于逻辑分区，这里直接敲回车默认就行
	第二个选项起始扇区，如没有特殊要求还是敲回车默认就好
	第三个选项我们敲+10G，也可以是+10000M，自己怎么理解怎么来，意思是分配10G空间（1G=1024M，不够严谨）
	接着提示创建成功，这里解释一下，相当于套娃又在sdb下面套了一个10G的sdb#，还是没用，需要在下面继续划分逻辑分区才能使用
	所以我我们重复一遍，选择第二个选项，再划分一个逻辑分区，我们给他+2G，同时注意大小不能超过10G，上面已经说了
	p打印出来，可以看到我们刚刚做的操作，使用w命令保存退出
	lsblk查看，可以看到扩展分区显示1kb，这就是我上面说的，实际上他是有10G可用存储的，然后sdb5的可用2G就是使用的扩展分区所划分的10GB存储
	接着我们重复一边操作，fdisk /dev/sdb，这次我们划分主分区，容量+8G，w保存退出
	到这里，我们已经做好的前期准备，虽然没有贴图（懒），但是绝对够保姆级
	忘记一个，使用t命令把主分区和逻辑分区变更一下格式”8e“，8e为LVM卷，一个简单的分区就完成了
```
3-选择之前的主分区和逻辑分区，这里就不解释pv了，可以看我关于disk的文章
pvcreate   /dev/sdb1 /dev/sdb5   

4-100M可以理解为dd命令中的bs（反正我是这样理解的），这里是指创建了一个名为”zhougege“的卷组
vgcreate -s 100M zhougege /dev/sdb1 /dev/sdb5 

5-在卷组中创建一个卷 大小2G，这里的20可以理解为dd命令中的count，所以总大小=bs*count
lvcreate -l 20 -n zhoulv zhougege
	当然我们也可以直接指定大小，在卷组中再创建一个5G的卷
lvcreate -L +5G -n zhoulv2 zhougege
	这里我们再输入pvs可以查看剩余存储，接下来我们个“zhoulv”扩容一下，还有3G空间可以使用
lvextend -L +1g /dev/zhougege/zhoulv
	或者缩减
lvreduce -L -1G /dev/zhougege/zhoulv

6-这里我们还没有挂载，挂载需要文件系统，值得注意的是要进行删除逻辑卷操作的时候，记得unmount /dev/you/device
mkfs -t /dev/zhougege/zholv 或者mkfs.ext4 /dev/zhougegezhoulv   //选择你想要的格式

7-挂载，可以创建目录进行挂载，这里就不创建了
mount /dev/zhougege/zhoulv /mnt

8-查看，可以看到文件系统和挂载点了
lsblk -f

9-这里要注意的是，想要永久挂载，必须写入fstab，同时如果是逻辑分区不要使用路径挂载，最好指定卷名或者是uuid，懂得的都懂，不懂请点击[这里](https:www.baidu.com)查看
echo "UUID=you device uuid  /monuntpoint filesystem defaults 0 0" >> /etc/fstab
echo "UUID=xxxx  /mnt ext4 defaults 0 0" >> /etc/fstab

10-快照，相信大家很多人都去过网吧，这里的快照类似于网吧的无盘系统，这样就很好理解了，50m大小只是测试，实际看你想要存储文件大小
lvcreate -n zhoulv_snap -s -L 50m -p r /dev/zhougege/zhoulv  //s=snap快照，p为权限有rw和r选项，作为快照来说当然是只读了,感觉不选也行，mount的时候加个-o r也不是不行	
	还原快照，试试往mnt里面写入或删除文件
rm  -fr /mnt/* 
	挂载快照卷，这里忘了挂载快照了，不过在创建快照的那一刻，已经保留了你被快照逻辑卷的状态，所以dont worry。挂载一下就好了
mkdir /zhoulv_snap && mount /dev/zhougege/zhoulv_snap  /zhoulv_snap
lsblk和lvs都可以看到snap的状态
	还原快照，该命令需要卸载两个卷，所以复制来的简单一点，还是写一下把，推荐cp -r，快照之后就没了，很鸡儿垃圾
cp  -rv /zhoulv_snap /mnt
umount /dev/zhougege/zhoulv_snap ; umount /dev/zhougege/zhoulv
lvconvert --merge /dev/zhougege/zhoulv_snap /dev/zhougege/zhoulv
mount /dev/zhougege/zhoulv /mnt/
ls /mnt

11-然后疯狂删除，记得先卸载
umount /mnt
lvremove /dev/zhougege/zhoulv
lvremove /dev/zhougege/zhoulv2
vgremove zhougege
pvremove
