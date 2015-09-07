CloudStack 安装和配置
--------------------------------

- [存储说明](#存储说明)
  - [主存储、二级存储](#主存储、二级存储)
  - [文件服务器](#文件服务器)
- [部署CloudStack](#部署CloudStack)
  - [配置系统相关服务](#配置系统相关服务)
  - [安装CloudStack](#安装CloudStack)
  - [安装数据库](#安装数据库)
  - [启动CloudStack](#启动CloudStack)
  - [配置系统模版](#配置系统模版)
  - [数据库复制（可选）](#数据库复制（可选）)
  - [故障切换](#故障切换)
- [配置CloudStack](#配置CloudStack)
  - [创建Basic Zone](#创建Basic Zone)
  - [创建Advanced Zone](#创建Advanced Zone)
- [FAQ](#FAQ)

<a name="存储说明"></a>
# 存储说明

CloudStack使用了两种网络存储。一种是主存储，主存储用于存放虚拟机硬盘文件。主存储也可使用本地存储，并非必需使用网络存储。二级存储用于存放虚拟机模板/快照/ISO文件，二级存储只能使用网络存储。

Primary Storage：

Primary Storage（主存储）和Cluster/Zone相关，它为运行于集群内的所有虚拟机提供磁盘存储卷。 你可以为一个区域或集群配置一个或多个主存储服务器（至少需要为一个集群配置一个主存储服务器）， 主存储一般靠近主机，以便提高性能。 Cloudstack负责把主存储分配给虚拟机。

主存储使用存储标签的概念，每个主存储可以和0个，1个或多个存储标签关联。

当某个虚拟机启动的时候，或某个数据磁盘第一个挂载到某个虚拟机的时候，存储标签就可以用于标识哪个主存储可以挂载到虚拟机。 主存储可以是静态的或动态的。

在静态情况下，管理员必须给Cloudstack预先配置一定数量的存储（例如一个SAN的一个Volumn），Cloudstack可以放很多Volumn到它的存储上。

在动态模式下，管理员可以给Cloudstack分配一个存储系统（例如一个SAN），Cloudstak通过和这个存储系统的插件配合，可以实现动态建立存储卷。

Cloudstack可以支持所有标准的iSCSI和NFS Server。

Secondary Storage：

Secondary storage（二级存储）主要保存以下数据：
Templates（模板）— OS 镜像，可用于引导虚拟机，可以包含其它配置信息，如安装好的软件。 ISO镜像 — 包含数据或引导媒介的磁盘镜像，就是可以刻盘启动的iso文件。 磁盘存储卷的快照 — 保存虚拟机数据的拷贝，可用于数据恢复或建立新的模板。

二级存储的内容对对其范围内的所有主机都是可用的，范围一般指一个区域（Zone）或一个区划（Region）。
二级存储一般配置为对一个Zone可用的NFS形式，为了使副存储对所有云的主机可用，可以引入对象存储，这样就不必在Zone间复制模板和快照。

Cloudstack提供了Swift和Amazon S3的插件。当使用以上两种存储之一，你可以首先为整个Cloudstack配置对象存储，然后为每个Zone建立NFS中间存储，中间存储的作用是为所有模板和其它副存储数据存储到对象存储系统时提供临时存储。

![store
](../images/store.png)

<a name="主存储、二级存储"></a>
## 主存储、二级存储

存储节点使用一台服务器，我们这里以NFS为例，NFS共享目录：/secondary。

### 1. 安装NFS
```shell
# yum install nfs-utils -y
# yum	install	rpcbind	-y
```
### 2. 创建存储目录
```shell
# mkdir -p /export/primary
# mkdir -p /export/secondary
```
### 3. 配置NFS
```shell
# echo "/export *(rw,async,no_root_squash,no_subtree_check)" >/etc/exports
# exportfs -a
```
编辑/etc/exports文件，设置存储的路径,Export /export 目录。
```shell
# vi  /etc/sysconfig/nfs
LOCKD_TCPPORT=32803
LOCKD_UDPPORT=32769
MOUNTD_PORT=892
RQUOTAD_PORT=875
STATD_PORT=662
STATD_OUTGOING_PORT=2020
RPCNFSDARGS="-N	4"			#	对于KVM集群是必须的,	 否则存储异常导致系统
虚机无法启动
```
修改 /etc/sysconfig/nfs，将其中的端口号全部打开。
### 4. 配置防火墙
```shell
#
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p udp --dport 111 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p tcp --dport 2049 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p tcp --dport 32803 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p udp --dport 32769 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p tcp --dport 892 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p udp --dport 892 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p tcp --dport 875 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p udp --dport 875 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p tcp --dport 662 -j ACCEPT
iptables -A INPUT -s 0.0.0.0/0 -m state --state NEW -p udp --dport 662 -j ACCEPT
# iptables-save > /etc/sysconfig/iptables
# service iptables status
```

### 5. 启动NFS服务
```shell
# service nfs start
# service rpcbind start
```

### 6. 设置服务为自动启动
```shell
# chkconfig nfs on
# chkconfig rpcbind on
```
重启服务器

<a name="文件服务器"></a>
## 文件服务器

### 1. 安装httpd服务
```shell
# yum install httpd -y
# service httpd start
```
### 2. 上传系统镜像

将系统模版、操作系统iso镜像上传至/var/www/html目录。

<a name="部署CloudStack"></a>
# 部署CloudStack

<a name="配置系统相关服务"></a>
## 配置系统相关服务

### 1. 配置IP
```shell
# echo "IPADDR=192.168.3.10
NETMASK=255.255.255.0
GATEWAY=192.168.3.254" >> /etc/sysconfig/network-scripts/ifcfg-eth0
# sed -i 's/dhcp/static' /etc/sysconfig/network-script/ifcfg-eth0
# sed -i 's/ONBOOT=no/ONBOOT=yes' /etc/sysconfig/network-script/ifcfg-eth0
```
### 2. 配置主机名
```shell
# echo "cs"> /etc/sysconfig/network
# hostname -F /etc/sysconfig/network
# echo "192.168.3.10   cs cs.cloud.com">> /etc/hosts
# hostname --fqdn
//检查配置是否有效
```
### 3. 关闭 selinux
```shell
# getenforce
//查看当前 selinux 状态
# setenforce 0
//临时设置 selinux 状态
# sed -i 's/enforcing/disabled/' /etc/selinux/config
//修改 selinux 配置文件，重启永久禁用
```
### 4. 配置系统的本地 yum 源
```shell
# yum clean all & yum makecache
```
### 5. 配置 ntp 服务器
```shell
# yum install ntp  -y
# vi /etc/ntp.conf
//编辑 ntp 配置文件，将服务器替换成可用对时服务器。
# service ntpd restart
# chkconfig ntpd on
//重启 ntp 服务，并且设置其开机启动
```
### 6. 确认java版本

管理节点不能安装JDK6或者其他较低的版本，如有安装，先卸载该JDK，再进行进行下面的步骤。（cs4.4开始改用JDK7）
```shell
# java -version
```

<a name="安装CloudStack"></a>
## 安装CloudStack

### 1. 配置cloudstack的yum源
```shell
# echo "[cloudstack-source]
name=cloudstack
baseurl=http://192.178.102.249/cloudstack/centos/4.5/
enabled=1
gpgcheck=0" > /etc/yum.repos.d/cloudstack.repo
```
实验室已将yum源同步到本地，不同版本配置不同，具体配置时以实际版本为准。
### 2. 安装cloudstack
```shell
# yum install -y cloudstack-management
```
### 3. 安装cloudstack-agent
```shell
# yum install -y cloudstack-agent
```
仅在KVM主机节点安装。

<a name="安装数据库"></a>
## 安装数据库

### 1. 安装mysql
```shell
# yum install mysql-server -y
```
CloudStack使用mysql管理数据，但安装cloudstack-management时没有包含mysql，需要手动安装，并导入数据。数据库可以被安装到其它机器上。
  注意：允许远程mysql连接，方便以后查找问题

### 2. 修改mysql配置
```shell
# echo "[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=350
log-bin=mysql-bin
binlog-format = 'ROW'
[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid" >/etc/my.cnf
```
max_connections 的参数应设置 350 乘以你准备部署的管
理节点的数量。这里假定只安装一个管理节点。

### 3. 修改mysql安全
```shell
# service mysqld start
# chkconfig mysqld on
# mysql_secure_installation
# service mysqld restart
Stopping mysqld:                                           [  OK  ]
Starting mysqld:                                           [  OK  ]
# /sbin/iptables -I INPUT -p tcp --dport 3306 -j ACCEPT
# /etc/rc.d/init.d/iptables save
```
缺省安装的 mysql 安全级别比较低，需要手工设置 mysql 下密码、禁用远程访问，删除无用账户及测试数据库。

### 4. 导入数据
```shell
# cloudstack-setup-databases cloud:111111@localhost --deploy-as=root:111111
```
数据库准备好后，需导入 CloudStack 的表及基础数据，这样云平台才能正常使用。如果没有意外的话，最后会输出 CloudStack has successfully initialized database
字样，表示数据库已经准备好了。

<a name="启动CloudStack"></a>
## 启动CloudStack

### 1. 配置管理服务器服务并启动服务，检查服务状态。
```shell
# cloudstack-setup-management
# service cloudstack-management status
# tail -n 200 -f /var/log/cloudstack/management/catalina.out
```

<a name="配置系统模版"></a>
## 配置系统模版

### 1. 确定模版版本

  CloudStack使用一组系统虚机来提供访问虚机控台，各种网络服务和管理存储的功能。当你引导云的时候，该步骤会获取这些准备用于部署的系统镜像。现在我们要从刚刚挂载的共享存储上面下载虚机模板并部署它们。管理服务器上有一个脚本来操作这些系统虚机镜像。这里的模版已装好的数据库中看到系统使用的模版地址，下载对应版本即可：
```SQL
SQL> SELECT NAME,URL FROM vm_template;
```
直接下载即可：
```shell
# wget http://download.cloud.com/templates/4.5/systemvm64template-4.5-vmware.ova
```
### 2. 挂载nfs
```shell
# mount -t nfs 192.168.50.10:/export/secondary /mnt
```
在管理服务器上挂载二级存储.
### 3. 导入系统模版
```shell
# /usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt \
-m /mnt \
-u http://192.178.102.249/cloudstack/systemvm64template-4.4.1-7-vmware.ova \
-h vmware \
-F
```

<a name="数据库复制（可选）"></a>
## 数据库复制（可选）
CloudStack支持MySQL节点间的数据库复制。通过标准的MySQL复制功能来实现。这样做可能希望防止MySQL服务器或者存储损坏。MySQL复制使用master/slave的模型。master节点是直接为管理服务器所使用。slave节点为备用，接收来自master节点的所有写操作，并将它应用于本地冗余数据库。以下是实施数据库复制的步骤。

> 注： 创建复制并不等同于备份策略，你需要另外建立有别于复制的MySQL数据的备份机制。

### 1.配置主节点
确保这是一个全新安装且没有数据的master数据库节点。
编辑master数据库的my.cnf，在[mysqld]的datadir下增加如下部分。

```bash
log_bin=mysql-bin
server_id=1
```
考虑到其他的服务器，服务器id必须是唯一的。推荐的方式是将master的ID设置为1，后续的每个slave节点序号大于1，使得所有服务器编号如：1，2，3等。
重启MySQL服务。如果是RHEL/CentOS系统，命令为：

```bash
# service mysqld restart
```
如果是Debian/Ubuntu系统，命令为：

```bash
# service mysql restart
```
在master上创建一个用于复制的账户，并赋予权限。我们创建用户”cloud-repl”，密码为”password”。假设master和slave都运行在172.16.1.0/24网段。

```bash
# mysql -u root
mysql> create user 'cloud-repl'@'172.16.1.%' identified by 'password';
mysql> grant replication slave on *.* TO 'cloud-repl'@'172.16.1.%';
mysql> flush privileges;
mysql> flush tables with read lock;
```
离开当前正在运行的MySQL会话,在新的shell中打开第二个MySQL会话。检索当前数据库的位置点。

```bash
# mysql -u root
mysql> show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      412 |              |                  |
+------------------+----------+--------------+------------------+
```

注意你数据库实例所返回的文件及位置点,退出该会话。完成master安装。返回到master的第一个会话，取消锁定并退出MySQL。

```bash
mysql> unlock tables;
```

### 2.配置从节点
安装并配置slave节点。在slave服务器上，运行如下命令。

```bash
# yum install mysql-server
# chkconfig mysqld on
```
编辑my.cnf，在[mysqld]的datadir下增加如下部分。

```bash
server_id=2
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
```
重启MySQL。对于RHEL/CentOS系统，使用”mysqld”

```bash
# service mysqld restart
```
对于Ubuntu/Debian系统，使用”mysql.”

```bash
# service mysql restart
```
引导slave连接master并进行复制。使用上面步骤中得到数据来替换IP地址，密码，日志文件，以及位置点。

```bash
mysql> change master to
    -> master_host='172.16.1.217',
    -> master_user='cloud-repl',
    -> master_password='password',
    -> master_log_file='mysql-bin.000001',
    -> master_log_pos=412;
    ```
在slave上启动复制功能。

```bash
mysql> start slave;
```
在slave上可能需要开启3306端口。这对复制来说不是必须的。但如果没有做，当需要进行数据库切换时，你仍然需要去做。

<a name="故障切换"></a>
### 故障切换
这将为管理服务器提供一个复制的数据库，用于实现手动故障切换。管理员将CloudStack从一个故障MySQL实例切换到另一个。当数据库失效发生时，你应该：

#### 1.停止管理服务器：
```bash
service cloudstack-management stop
```

将数据库的副本服务器修改为master并重启，确保数据库的副本服务器的3306端口开放给管理服务器。
更改使得管理服务器使用这个新的数据库。最简单的操作是在管理服务器的/etc/cloudstack/management/db.properties文件中写入新的数据库IP地址。

#### 2.重启管理服务器：

```bash
# service cloudstack-management start
```

<a name="配置CloudStack"></a>
# 配置CloudStack

<a name="创建Basic Zone"></a>
## 创建Basic Zone

<a name="创建Advanced Zone"></a>
## 创建Advanced Zone

<a name="FAQ"></a>
# FAQ
- 1、管理节点的webui 无法访问
Cloudstack基于tomcat提供web服务，默认使用了8080端口。如果你想改用其它端口，可以修改 /etc/tomcat6/server.xml 文件进行配置。
Cloudstack默认安装在 /etc/cloudstack/management 目录下，你可以通过修改 log4j-cloud.xml文件来调整日志的输出级别、路径等。
cloudstack默认日志在/var/log/cloudstack下

检查iptables是否阻挡了8080端口。检查cloudstack-management服务是否正常启动。
```bash
service cloudstack-management status
```
如果启动状态不正常，则需要检查一下日志。
日志位于 /var/log/cloudstack/management/catalina.out 。根据日志中的错误提示，进行相应的处理，绝大多数问题都可以得到解决。
如果日志信息不够详细，可以修改 /etc/cloudstack/management/log4j-cloud.xml来调整日志的输出级别。
- 2、登陆时提示用户名密码不正确。

默认的登陆用户名为 admin 密码是 password 。
如果登陆时提示不正确，可能是导入基础数据库时有的问题。
重新导入基础数据库：
```bash
cloudstack-setup-databases cloud:123456@localhost --deploy-as=root:root密码
```
如果还不行，参考5将数据库删掉再重新导入。
- 3、CloudStack不能添加主存储或二级存储

检查/etc/sysconfig/nfs配置文件是否把端口都开放了。
检查iptables是否有阻挡。
检查CloudStack的“全局设置”，secstorage.allowed.internal.sites属性是否设置正确。
- 4、CloudStack无法导入IOS或虚拟机模板

创建好“基础架构”后，就可以导入ISO文件或虚拟机模板，为创建虚机做准备了。
如果你发现注册ISO或注册模板时，状态字段一直不动，已就绪永远都是no，那一般都是因为二级存储有问题或Secondary Storage VM 有问题了。
选择“控制板”-<系统容量，检查二级存储容量是否正确。
检查系统VM中的Secondary Storage VM是否正常启动。
- 5、CloudStack如何重装

安装完CloudStack后，我们往往会做各种实验，可能会把系统搞得很乱。想删除的话非常麻烦，因为它们之间往往存在层级关系，必须先从最底层删起。有没简单的办法直接推倒重来呢？答案是有的，最简单只要重置下其数据库即可。
先停掉CloudStack服务：
```bash
service cloudstack-management stop
```
登陆mysql控制台，删除数据库：
```bash
mysql -u root -p123456
drop database cloud;
drop database cloud_usage;
drop database cloudbridge;
quit;
/etc/init.d/mysqld restart
```
重新导入基础数据：
```bash
cloudstack-setup-databases cloud:123456@localhost --deploy-as=root:123456
```
重新导入系统虚机：
在管理服务器上挂载二级存储：
```bash
mount 192.168.20.9:/export/secondary /mnt/
```
导入虚拟机模版

```bash
/usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt \
-m /mnt \
-u http://192.178.102.249/cloudstack/systemvm64template-4.4.1-7-vmware.ova \
-h vmware \
-F
```
卸载
```bash
umount /mnt/
```

```bash
cloudstack-setup-management
```
重启cloudstack服务
```bash
service cloudstack-management start
```
这时，你再登陆就会发现一个全新的CloudStack啦。
- 6、CloudStack的区域、提供点等无法用中文命名

CloudStack在4.1版本存在bug，请使用其他版本。
