---
title: 实现基于分布式的LAMP架构，并将NFS实时同步到备份服务器
date: 2023-05-11 22:10:00
tags: [数据库,Database,DNS,wordpress,NFS]
categories: 实用教程
cover: https://pan.alybaba.top/images/linux.jpg
---

本次实验机器全部为ubuntu22.04，如果使用redhat，稍有不同架构图如下
![分布式LAMP架构](/images/实战.png)

DNS配置 ip：172.18.0.122
添加2个web的域名解析信息
```bash
bash <(curl -Ls http://pan.alybaba.top:81/script/install_dns.sh) 或者
wget -qO- http://pan.alybaba.top:81/script/install_dns.sh | bash
echo "www        A  172.18.0.129" >> /etc/bind/wang.org.zone
#该脚本只会添加一个解析，执行完毕后，请手动添加另一个解析进去
```

web1  ip:172.18.0.121
```bash
apt  update && apt install php libapache2-mod-php php-mysql  apache2  nfs-kernel-server unzip
#安装相关软件
wget https://wordpress.org/latest.zip  
#下载wordpress
unzip latest.zip
#解压
mv wordpress/    /var/www/html/
#放置网页根目录，注意这里是把目录移动过去，如果您是把wordpress目录下的文件全部移动过去，请把index.html删除，下面也不用配置主站点目录了
ufw allow 80/tcp
#开启端口
sed -i 's,/var/www/html,/var/www/html/wordpress,g' /etc/apache2/sites-available/000-default.conf
#配置主站点目录
systemctl start apache2
#重启生效

rsync -av /var/www/html/wordpress/* 172.18.0.131:/data/wordpress
#把文件拷贝至nfs服务
rm -rf   /var/www/html/wordpress
#当nfs备份完后，可以删除或者是移动到其他目录
echo "172.18.0.131:/data/    /var/www/html/    nfs  _netdev  0  0" >> /etc/fstab
#挂载nfs服务，这里是删除的wordpress目录，已经备份到nfs服务器中
mount -a
#重新挂载fstab
```

web2   ip:172.18.0.129
```bash
apt update && apt install php libapache2-mod-php php-mysql  apache2  nfs-kernel-server unzip
mv wordpress/    /var/www/html/
ufw allow 80/tcp
sed -i 's,/var/www/html,/var/www/html/wordpress,g' /etc/apache2/sites-available/000-default.conf
systemctl start apache2

rm -rf   /var/www/html/wordpress
echo "172.18.0.131:/data/ /var/www/html/ nfs _netdev 0 0" >> /etc/fstab
mount -a
```



mysql master ip：172.18.0.130，创建wordpress账户，并授权。配置为master节点
```bash
apt update && apt install mysql-server
#安装mysql服务
ufw allow 3306/tcp
#开启mysql端口
sed -i 's/^bind-address/#&/'  /etc/mysql/mysql.conf.d/mysqld.cnf
#取消localhost监听，就是本机访问
echo "server-id = 130" >> /etc/mysql/mysql.conf.d/mysqld.cnf
#增加唯一server-id
systemctl start mysql
#启动mysql服务
create user backuser@'172.18.0.%' identified by '1234567890';
#创建主从同步账号
grant replication slave on *.* to backuser@'172.18.0.%';
#给与主从同步权限
reset master;
#重置binlog，然后去配置主从同步
create user wordpress@'172.18.0.%' identified by 'wordpress123'
#创建wordpress登录用户
grant all on wordpress.* to wordpress@'172.18.0.%'
#给与相关权限，可以去检查用户是否同步
```

mysql  ip：172.18.0.133  配置slave节点
```bash
apt update && apt install mysql-server
ufw allow 3306/tcp
sed -i 's/^bind-address/#&/' /etc/mysql/mysql.conf.d/mysqld.cnf
echo -e "server-id = 133\nread-only = 1" >> /etc/mysql/mysql.conf.d/mysqld.cnf
systemctl start mysql
复制以下语句
CHANGE MASTER TO MASTER_HOST='172.18.0.130',
MASTER_USER='backuser',
MASTER_PASSWORD='1234567890',
MASTER_LOG_FILE='binlog.000001',
MASTER_LOG_POS=157;
#主从同步配置，然后去master创建wordpress用户，这里图省事，也可以用其他的写入语句检测主从同步
show slave status\G
#查看slave同步状态
select user,host from mysql.user;
#测试账号是否同步
```


nfs  ip：172.18.0.131 配置nfs 
```bash
mkdir /data/wordpress && mkdir /etc/exports.d
#创建文件存放目录和扩展配置文件目录
vim /etc/exports.d/wordpress.exports
/data/wordpress 172.18.0.0/24(rw)
:wq
#配置存放目录，以及可挂载网段
exportsfs -v
#查看可挂载目录
exportsfs -r
#重载配置文件
exportsfs -v
chown -R www-data.www-data /data/wordpress
#修改相应权限，方便写入
#这里做完之后去web主机进行挂载，原文件可以删除，也可以备份

wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/sersync/sersync2.5.4_64bit_binary_stable_final.tar.gz
#下载sersync，如下载不了，请export http_proxy=http://ipaddress:port或者export ALL_PROXY=socks5://ipaddress:port
#还有很多的代理工具，如squid等等，请自行百度
tar xf sersync2.5.4_64bit_binary_stable_final.tar.gz -C /usr/local/
#解压文件至目录，可随意
ln -s /usr/local/GNU-Linux-x86/sersync2    /usr/local/bin
#添加环境变量
#ln -s GNU-Linux-x86 sersync
vim /usr/local/GNU-Linux-x86/confxml
  <!-- #默认解压出来，只需要修改有注释的几行-->
        <exclude expression="^static/*"></exclude>
    </filter>
    <inotify>
        <delete start="true"/>
        <createFolder start="true"/>
        <createFile start="false"/>
        <closeWrite start="true"/>
        <moveFrom start="true"/>
        <moveTo start="true"/>
        <attrib start="true"/>
        <modify start="false"/>
    </inotify>

    <sersync>
        <localpath watch="/data/wordpress">                                 
                                   <!-- #本地目录-->
                <remote ip="172.18.0.132" name="/data/backup"/>  
  <!--#备份服务器ip和备份目录，注意，如果是rsync，name则需要改成相应的模块名称-->
            <!--<remote ip="192.168.8.39" name="tongbu"/>-->
            <!--<remote ip="192.168.8.40" name="tongbu"/>-->
        </localpath>
        <rsync>
            <commonParams params="-artuz"/>
            <auth start="false" users="rsyncuser" passwordfile="/etc/rsync.pas"/> 
<!--rsync配置，这边就使用ssh了，所以rsync为false，如果使用rsync，则为true。ssh则需要改为false-->
            <userDefinedPort start="false" port="874"/><!-- port=874 -->
            <timeout start="false" time="100"/><!-- timeout=100 -->
            <ssh start="true"/>                                                                       #开启ssh同步协议
        </rsync>
        <failLog path="/tmp/rsync_fail_log.sh" timeToExecute="60"/><!--default every 60mins execute once-->
        <crontab start="false" schedule="600"><!--600mins-->
            <crontabfilter start="false">
                <exclude expression="*.php"></exclude>
                <exclude expression="info/*"></exclude>
            </crontabfilter>
        </crontab>
        <plugin start="false" name="command"/>
    </sersync>

    <plugin name="command">
        <param prefix="/bin/sh" suffix="" ignoreError="true"/>  <!--prefix /opt/tongbu/mmm.sh suffix-->
        <filter start="false">
            <include expression="(.*)\.php"/>
            <include expression="(.*)\.sh"/>
        </filter>
    </plugin>
<plugin name="socket">
        <localpath watch="/opt/tongbu">
            <deshost ip="192.168.138.20" port="8009"/>
        </localpath>
    </plugin>
    <plugin name="refreshCDN">
        <localpath watch="/data0/htdocs/cms.xoyo.com/site/">
            <cdninfo domainname="ccms.chinacache.com" port="80" username="xxxx" passwd="xxxx"/>
            <sendurl base="http://pic.xoyo.com/cms"/>
            <regexurl regex="false" match="cms.xoyo.com/site([/a-zA-Z0-9]*).xoyo.com/images"/>
        </localpath>
    </plugin>
</head>

nohup sersync2 -dro /usr/local/sersync/confxml.xml &>/dev/null &
#后台运行sersync程序
```

backup 备份nfs ip：172.18.0.132
```bash
mkdir /data/backup 
#创建备份目录
```
在这里测试的话，我们可以开2个终端一个打开nfs，一个打开backup查看创建删除nfs下的/data/wordpress文件，然后查看backup下的/data/backup/下是否进行同步

到这里已经搭建完成了，我们大概就实现了分布式的LAMP架构，总结一下，我们在dns服务也可配置主从，并且还可配置mysql中间件，实现更安全，更可靠的LAMP分布式架构，有兴趣的可以自行尝试。其次使用dns服务器作为负载均衡不是一个很好的选项，因为基于DNS的负载均衡具有一定的局限性，例如客户端可能会缓存DNS解析结果，导致一段时间内不会重新解析域名，这可能会导致负载不均衡。一般来说，推荐使用 nginx upstream。同时我们在配置的时候需要注意理清思路，一步一个脚印，先配置什么，后配置什么，不能乱了阵脚。同时注意的时在开启各种服务时，应该养成使用 systemctl enable server_name 配置开机自启动的好习惯，这里时实验环境，这个在实际生产环境要切记。
