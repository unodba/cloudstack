# LXC主机的系统要求
LXC要求Linux kernel 2.6.24开始引入的 cgroups功能。尽管不需要运行特定的发行版本，但是建议下列的版本：

CentOS / RHEL: 6.3
Ubuntu: 12.04(.1)
LXC hypervisor 要求libvirt和Qemu的版本。无论使用哪种Linux发行版，请确保满足以下要求：

libvirt: 1.0.0或更高
Qemu/KVM: 1.0 或更高版本
CloudStack中的默认使用Linux本身的桥接(bridge模块)方式实现。也可选择在CloudStack中使用OpenVswitch，具体要求如下：

libvirt: 1.0.0或更高
openvswitch: 1.7.1或更高版本
此外，硬件要求如下：

同一集群中主机必须使用相同版本的Linux系统。
同一群集中的所有节点架构必须一致。CPU的型号、数量和功能参数必须相同。
必须支持HVM(Intel-VT或者AMD-V)
64位x86 CPU(多核性能更佳)
4GB内存
至少一块网卡
在部署CloudStack时，Hypervisor主机不能运行任何虚拟机
LXC安装概述
LXC没有任何本地系统VMs，而KVM需要运行系统VMs。意思为主机需要同时支持LXC和KVM。因此，大部分的安装和配置跟KVM的安装一样。本章节不会复述KVM的安装。这里我们只会给出使KVM与CloudStack协同工作的一些特有的步骤。

> 警告: 在我们开始之前，请确保所有的主机都安装了最新的更新包。
       不建议在主机中运行与CloudStack无关的服务。

# 安装LXC主机步骤：

准备操作系统：
安装和配置libvirt
配置安全策略 (AppArmor 和 SELinux)
安装和配置Agent
准备操作系统：
主机的操作系统必须为运行CloudStack Agent和KVM实例做些准备。

使用root用户登录操作系统。
检查FQN完全合格/限定主机名。

$ hostname --fqdn
该命令会返回完全合格/限定主机名，例如”kvm1.lab.example.org”。如果没有，请编辑 /etc/hosts。
请确保主机能够访问互联网。

$ ping www.cloudstack.org
启用NTP服务以确保时间同步.

注解

NTP服务用来同步云中的服务器时间。时间不同步会带来意想不到的问题。
安装NTP

$ yum install ntp
$ apt-get install openntpd
在所有主机中重复上述步骤。
安装和配置Agent
CloudStack使用代理管理LXC实例。管理服务器与代理通信并控制主机中所有实例。

首先我们安装Agent：

在RHEL/CentOS上:

```bash
$ yum install cloudstack-agent
```
在Ubuntu上:

```bash
$ apt-get install cloudstack-agent
```
接下来更新代理配置。在 /etc/cloudstack/agent/agent.properties 中配置

设置代理运行在LXC模式下：

hypervisor.type=lxc
可选项：如果想使用直连网络(代替默认的桥接网络)，配置如下行：

libvirt.vif.driver=com.cloud.hypervisor.kvm.resource.DirectVifDriver
network.direct.source.mode=private
network.direct.device=eth0
现在主机已经为加入群集做好准备。后面的章节有介绍，请参阅 添加一个宿主机。强烈建议在添加主机之前阅读此部分内容。

安装和配置libvirt
CloudStack使用libvirt管理虚拟机。因此正确地配置libvirt至关重要。CloudStack-agent依赖于Libvirt，应提前安装完毕。

为了实现动态迁移libvirt需要监听不可靠的TCP连接。还要关闭libvirts尝试使用组播DNS进行广播。这些都可以在 /etc/libvirt/libvirtd.conf文件中进行配置。

设定下列参数：

listen_tls = 0
listen_tcp = 1
tcp_port = "16509"
auth_tcp = "none"
mdns_adv = 0
除了在libvirtd.conf中打开”listen_tcp”以外，我们还必须修改/etc/sysconfig/libvirtd中的参数:

在RHEL或者CentOS中修改 /etc/sysconfig/libvirtd：

取消如下行的注释：

#LIBVIRTD_ARGS="--listen"
在Ubuntu中:修改 /etc/default/libvirt-bin

在下列行添加 “-l”

libvirtd_opts="-d"
如下所示:

libvirtd_opts="-d -l"
为了VNC控制台正常工作，必须确保该参数绑定在0.0.0.0上。通过编辑 ``/etc/libvirt/qemu.conf``实现。

请确保这个参数配置为：

vnc_listen = "0.0.0.0"
重启libvirt服务

在RHEL/CentOS上:

$ service libvirtd restart
在Ubuntu上:

$ service libvirt-bin restart
配置安全策略
CloudStack的会被例如AppArmor和SELinux的安全机制阻止。必须关闭安全机制并确保 Agent具有所必需的权限。

配置SELinux（RHEL和CentOS）：

检查你的机器是否安装了SELinux。如果没有，请跳过此部分。

在RHEL或者CentOS中，SELinux是默认安装并启动的。你可以使用如下命令验证：

$ rpm -qa | grep selinux
在 /etc/selinux/config 中设置SELINUX变量值为 “permissive”。这样能确保对SELinux的设置在系统重启之后依然生效。

在RHEL/CentOS上:

vi /etc/selinux/config
查找如下行

SELINUX=enforcing
修改为

SELINUX=permissive
然后使SELinux立即运行于permissive模式，无需重新启动系统。

$ setenforce permissive
配置AppArmor(Ubuntu)

检查你的机器中是否安装了AppArmor。如果没有，请跳过此部分。

Ubuntu中默认安装并启动AppArmor。使用如下命令验证：

$ dpkg --list 'apparmor'
在AppArmor配置文件中禁用libvirt

$ ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
$ ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
$ apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
$ apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
配置网络桥接
警告

本章节非常重要，请务必彻底理解。
注解

本章节详细介绍了如何使用Linux自带的软件配置桥接网络。如果要使用OpenVswitch，请看下一章节。
为了转发流量到实例，至少需要两个桥接网络： public 和 private。

By default these bridges are called cloudbr0 and cloudbr1, but you do have to make sure they are available on each hypervisor.

最重要的因素是所有hypervisors上的配置要保持一致。

网络示例
配置网络有很多方法。在基本网络模式中你应该拥有2个 (V)LAN，一个用于管理网络，一个用于公共网络。

假设hypervisor中的网卡(eth0)有3个VLAN标签：

VLAN 100 作为hypervisor的管理网络
VLAN 200 for public network of the instances (cloudbr0)
VLAN 300 作为实例的专用网络 (cloudbr1)
在VLAN 100 中，配置Hypervisor的IP为 192.168.42.11/24，网关为192.168.42.1

> 注解 Hypervisor与管理服务器不需要在同一个子网！

配置网络桥接
配置方式取决于发行版类型，下面给出RHEL/CentOS和Ubuntu的配置示例。

注解

本章节的目标是配置两个名为 ‘cloudbr0’和’cloudbr1’的桥接网络。这仅仅是指导性的，实际情况还要取决于你的网络布局。
在RHEL或CentOS中配置：
网络桥接所需的软件在安装libvirt时就已被安装，继续配置网络。

首先配置eth0：

vi /etc/sysconfig/network-scripts/ifcfg-eth0
确保内容如下所示：

DEVICE=eth0
HWADDR=00:04:xx:xx:xx:xx
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
TYPE=Ethernet
现在配置3个VLAN接口：

vi /etc/sysconfig/network-scripts/ifcfg-eth0.100
DEVICE=eth0.100
HWADDR=00:04:xx:xx:xx:xx
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
TYPE=Ethernet
VLAN=yes
IPADDR=192.168.42.11
GATEWAY=192.168.42.1
NETMASK=255.255.255.0
vi /etc/sysconfig/network-scripts/ifcfg-eth0.200
DEVICE=eth0.200
HWADDR=00:04:xx:xx:xx:xx
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
TYPE=Ethernet
VLAN=yes
BRIDGE=cloudbr0
vi /etc/sysconfig/network-scripts/ifcfg-eth0.300
DEVICE=eth0.300
HWADDR=00:04:xx:xx:xx:xx
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
TYPE=Ethernet
VLAN=yes
BRIDGE=cloudbr1
配置VLAN接口以便能够附加桥接网络。

vi /etc/sysconfig/network-scripts/ifcfg-cloudbr0
现在只配置一个没有IP的桥接。

DEVICE=cloudbr0
TYPE=Bridge
ONBOOT=yes
BOOTPROTO=none
IPV6INIT=no
IPV6_AUTOCONF=no
DELAY=5
STP=yes
同样建立cloudbr1

vi /etc/sysconfig/network-scripts/ifcfg-cloudbr1
DEVICE=cloudbr1
TYPE=Bridge
ONBOOT=yes
BOOTPROTO=none
IPV6INIT=no
IPV6_AUTOCONF=no
DELAY=5
STP=yes
配置完成之后重启网络，通过重启检查一切是否正常。

警告

在发生配置错误和网络故障的时，请确保可以能通过其他方式例如IPMI或ILO连接到服务器。
在Ubuntu中配置：¶
在安装libvirt时所需的其他软件也会被安装，所以只需配置网络即可。

vi /etc/network/interfaces
如下所示修改接口文件：

auto lo
iface lo inet loopback

# The primary network interface
auto eth0.100
iface eth0.100 inet static
    address 192.168.42.11
    netmask 255.255.255.240
    gateway 192.168.42.1
    dns-nameservers 8.8.8.8 8.8.4.4
    dns-domain lab.example.org

# Public network
auto cloudbr0
iface cloudbr0 inet manual
    bridge_ports eth0.200
    bridge_fd 5
    bridge_stp off
    bridge_maxwait 1

# Private network
auto cloudbr1
iface cloudbr1 inet manual
    bridge_ports eth0.300
    bridge_fd 5
    bridge_stp off
    bridge_maxwait 1
配置完成之后重启网络，通过重启检查一切是否正常。

警告

在发生配置错误和网络故障的时，请确保可以能通过其他方式例如IPMI或ILO连接到服务器。
配置防火墙
hypervisor之间和hypervisor与管理服务器之间要能够通讯。

为了达到这个目的，我们需要开通以下TCP端口(如果使用防火墙)：

22 (SSH)
1798
16509 (libvirt)
5900 - 6100 (VNC 控制台)
49152 - 49216 (libvirt在线迁移)
如何打开这些端口取决于你使用的发行版本。在RHEL/CentOS 及Ubuntu中的示例如下。

在RHEL/CentOS中打开端口
RHEL 及 CentOS使用iptables作为防火墙，执行以下iptables命令来开启端口：

$ iptables -I INPUT -p tcp -m tcp --dport 22 -j ACCEPT
$ iptables -I INPUT -p tcp -m tcp --dport 1798 -j ACCEPT
$ iptables -I INPUT -p tcp -m tcp --dport 16509 -j ACCEPT
$ iptables -I INPUT -p tcp -m tcp --dport 5900:6100 -j ACCEPT
$ iptables -I INPUT -p tcp -m tcp --dport 49152:49216 -j ACCEPT
这些iptables配置并不会持久保存，重启之后将会消失，我们必须手动保存这些配置。

$ iptables-save > /etc/sysconfig/iptables
在Ubuntu中打开端口：
Ubuntu中的默认防火墙是UFW(Uncomplicated FireWall)，使用Python围绕iptables进行包装。

要打开所需端口，请执行以下命令：

$ ufw allow proto tcp from any to any port 22
$ ufw allow proto tcp from any to any port 1798
$ ufw allow proto tcp from any to any port 16509
$ ufw allow proto tcp from any to any port 5900:6100
$ ufw allow proto tcp from any to any port 49152:49216
注解

默认情况下，Ubuntu中并未启用UFW。在关闭情况下执行这些命令并不能启用防火墙。
添加主机到CloudStack
现在主机已经为加入群集做好准备。后面的章节有介绍，请参阅 添加一个宿主机。强烈建议在添加主机之前阅读此部分内容。
