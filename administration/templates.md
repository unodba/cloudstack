创建模版
--------------------------------
- [Linux模板](#Linux模板)
- [创建Linux模板过程](#创建Linux模板过程)
  - [安装](#安装)
  - [其他配置](#其他配置)
  - [关闭模版](#关闭模版)
  - [创建模版](#创建模版)

<a name="环境要求"></a>
# Linux模板
为了准备使用模板部署你的Linux VMs，可以使用此文档来准备Linux模板。对于文档中的情况，你要通过配置模板，这会涉及”主模板”。
这个指导目前覆盖了传统的安装，但不会涉及用户数据和cloud-init还有假设在装过程中安装了openshh服务。
<a name="创建Linux模板过程"></a>
# 创建Linux模板过程
<a name="安装"></a>
## 安装
通常在安装过程中给VM命名是一个好的做法，这么做能确保某些组件如LVM不会只在一台机器中出现。推荐在在安装过程中使用”localhost”命名。

### 更新YUM源
```bash
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
wget http://192.178.102.249/help/CentOS-Base.repo -P /etc/yum.repos.d/
wget http://192.178.102.249/help/epel6.repo -P /etc/yum.repos.d/
```

### 更新、安装主模板中的包

* Ubuntu

```bash
sudo -i
apt-get update
apt-get upgrade -y
apt-get install -y acpid ntp
reboot
```

* CentOS

```bash
ifup eth0
yum update -y
yum install -y screen
reboot
```

### 安装vmware tools
```bash
mount /dev/cdrom /mnt
tar -zxvf VMwareTools-9.4.11-2400950.tar.gz -C /tmp
yum install perl -y
./vmware-install.pl
```
### 安装vnc服务端，并开启防火墙
```bash
yum install tigervnc tigervnc-server -y
vncserver :1
iptables -I INPUT -p tcp --dport 5901 -j ACCEPT
iptables -I INPUT -p tcp --dport 5801 -j ACCEPT
/etc/rc.d/init.d/iptables save
```
修改远控分辨率1280*720，修改时区为东八区

### 安装falcon-agent
```bash
yum install -y golang
# set $GOPATH and $GOROOT
export GOPATH=/usr/share
export GOROOT=/usr/lib/golang

mkdir -p $GOPATH/src/github.com/open-falcon
cd $GOPATH/src/github.com/open-falcon
git clone https://github.com/open-falcon/agent.git
cd agent
go get ./...
./control build

cd $GOPATH/src/github.com/open-falcon/agent/
mv cfg.example.json cfg.json
vim cfg.json
// 修改 transfer这个配置项的enabled为 true，表示开启向transfer发送数据的功能
// 修改 transfer这个配置项的addr为：127.0.0.1:8433 (改地址为transfer组件的监听地址)
// 默认情况下（所有组件都在同一台服务器上），保持cfg.json不变即可
// cfg.json中的各配置项，可以参考 https://github.com/open-falcon/agent/blob/master/README.md
//配置防火墙
iptables -I INPUT -p tcp --dport 1988 -j ACCEPT
iptables -I INPUT -p tcp --dport 8433 -j ACCEPT
iptables -I INPUT -p tcp --dport 6030 -j ACCEPT
/etc/rc.d/init.d/iptables save

//启动
./control start
//查看日志
./control tail
```

goto http://localhost:1988

```bash
vi /etc/init.d/open-falcon-agent

#!/bin/sh
### BEGIN INIT INFO
# Provides:          open-falcon open-falcon-agent
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Description:       open-falcon open-falcon-agent
### END INIT INFO

SCRIPT="/opt/open-falcon/agent/control"

 start() {
   $SCRIPT start
 }

 stop() {
   $SCRIPT stop
 }


 case "$1" in
   start)
     start
     ;;
   stop)
     stop
     ;;
   restart)
     stop
     start
     ;;
   *)
     echo "Usage: $0 {start|stop|restart}"
 esac
```
添加开机启动
```bash
chkconfig --add open-falcon-agent
chmod 775 /etc/init.d/open-falcon-agent
service open-falcon-agent restart
```
<a name="其他配置"></a>
## 其他配置
### 配置命令回溯

```bash
vi  /etc/profile

#Record history operation
export HISTTIMEFORMAT="[%Y-%m-%d %H:%M:%S] [`who am i 2>/dev/null| awk '{print $NF}'|sed -e 's/[()]//g'`] "
export PROMPT_COMMAND='\
if [ -z "$OLD_PWD" ];then
    export OLD_PWD=$PWD;
fi;
if [ ! -z "$LAST_CMD" ] && [ "$(history 1)" != "$LAST_CMD" ]; then
    logger -t `whoami`_shell_cmd "[$OLD_PWD]$(history 1)";
fi ;
export LAST_CMD="$(history 1)";
export OLD_PWD=$PWD;'
```

### 主机名管理

* CentOS

默认情况下CentOS在启动的时候配置主机名。Ubuntu没有此功能，对于Ubuntu，安装时使用下面步骤。

* Ubuntu

一个模板化的VM使用`/etc/dhcp/dhclient-exit-hooks.d`中的一个自定义脚本来设置主机名，这个脚本首先检查当前的主机名是是否是hostname，如果是，它将从DHCP租约文件获取host-name，domain-name和fix-ip，并且使用这些值来设置主机名并将其追加到 /etc/hosts 文件以用来本地主机名解析。一旦这个脚本或者一个用户从本地改变了主机名，那么它将不再根据新的主机名调整系统文件。此脚本同样也会重建openssh-server keys，这个keys在做模板(如下所示)之前被删除了。保存下面的脚本到`/etc/dhcp/dhclient-exit-hooks.d/sethostname`，并且调整权限。

```bash
vi /etc/dhcp/dhclient-exit-hooks.d/sethostname

#!/bin/sh
# dhclient change hostname script for Ubuntu
oldhostname=$(hostname -s)
if [ $oldhostname = 'localhost' ]
then
    sleep 10 # Wait for configuration to be written to disk
    hostname=$(cat /var/lib/dhcp/dhclient.eth0.leases  |  awk ' /host-name/ { host = $3 }  END { printf host } ' | sed     's/[";]//g' )
    fqdn="$hostname.$(cat /var/lib/dhcp/dhclient.eth0.leases  |  awk ' /domain-name/ { domain = $3 }  END { printf     domain } ' | sed 's/[";]//g')"
    ip=$(cat /var/lib/dhcp/dhclient.eth0.leases  |  awk ' /fixed-address/ { lease = $2 }  END { printf lease } ' | sed     's/[";]//g')
    echo "cloudstack-hostname: Hostname _localhost_ detected. Changing hostname and adding hosts."
    echo " Hostname: $hostname \n FQDN: $fqdn \n IP: $ip"
    # Update /etc/hosts
    awk -v i="$ip" -v f="$fqdn" -v h="$hostname" "/^127/{x=1} !/^127/ && x { x=0; print i,f,h; } { print $0; }" /etc/  hosts > /etc/hosts.dhcp.tmp
    mv /etc/hosts /etc/hosts.dhcp.bak
    mv /etc/hosts.dhcp.tmp /etc/hosts
    # Rename Host
    echo $hostname > /etc/hostname
    hostname $hostname
    # Recreate SSH2
    export DEBIAN_FRONTEND=noninteractive
    dpkg-reconfigure openssh-server
fi
### End of Script ###
```
```bash
# chmod 774 /etc/dhcp/dhclient-exit-hooks.d/sethostname
```

为了Ubuntu DHCP的脚本功能和CentOS dhclient能设置VM主机名，他们都去要设置主模板的主机名设置为“localhost”，运行下面的命令来更改主机名。

```bash
# hostname localhost
# echo "localhost" > /etc/hostname
```

### 密码管理
设置用户密码期限，这步是要在模板部署之后强制用户更改VM的密码。
```bash
# passwd --expire root
```

首先确认root用户账户是启用的并且使用了密码，然后使用root登录。
```bash
# passwd root
# logout
```

使用root，移除任何在安装过程中创建的自定义用户账户。

```bash
deluser myuser --remove-home
```
关于设置密码管理脚本的相关说明，请参阅给你的模板添加密码管理 ，这样能允许CloudStack通过web界面更改root密码。

> 警告

当你准备好做你的主模板的时候请运行下列步骤。如果主模板在这些步骤期间重启了，那么你要重新运行所有的步骤。在这个过程的最后，主模板应该关机并且将其创建为模板，然后再部署。

### 移除udev持久设备规则

这一步会移除模板的特殊信息，如网络MAC地址，租约信息和CD块设备，这个文件会在下次启动时自动生成。

* Ubuntu

```bash
# rm -f /etc/udev/rules.d/70*
# rm -f /var/lib/dhcp/dhclient.*
```

* CentOS

```bash
# rm -f /etc/udev/rules.d/70*
# rm -f /var/lib/dhclient/*
```

### 移除SSH Keys
这步是为了确认所有要作为模板的VMs的SSH Keys都不相同，否则这样会降低虚拟机的安全性。

```bash
# rm -f /etc/ssh/*key*
```
### 清除日志文件

从主模板移除旧的日志文件是一个好习惯。

```bash
cat /dev/null > /var/log/audit/audit.log 2>/dev/null
cat /dev/null > /var/log/wtmp 2>/dev/null
logrotate -f /etc/logrotate.conf 2>/dev/null
rm -f /var/log/*-* /var/log/*.gz 2>/dev/null
```

### 清除用户历史

下一步来清除曾经运行过的bash命令。

```bash
# history -c
# unset HISTFILE
```

### 网卡配置
对于CentOS，必须要修改网络接口的配置文件，在这里我们编辑/etc/sysconfig/network-scripts/ifcfg-eth0文件，更改下面的内容。
```bash
DEVICE=eth0
TYPE=Ethernet
BOOTPROTO=dhcp
ONBOOT=yes
```
<a name="关闭模版"></a>
## 关闭模版

现在可以关闭模板VM并且创建模板了！

```bash
# halt -p
```
<a name="创建模板"></a>
## 创建模板

现在可以创建模板了。

> 注
通过Ubuntu和CentOS的模板分发的虚机可能需要重启主机名才生效。
