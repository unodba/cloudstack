# KVM Hypervisor 要求
KVM包含在多种基于Linux的操作系统中。尽管不作要求，但是我们推荐以下发行版：

- CentOS / RHEL: 6.3
- Ubuntu: 12.04(.1)

KVM hypervisors主要要求在于libvirt和Qemu版本。不管您使用何种Linux版本，请确保满足以下要求：

- libvirt: 0.9.4 或更高版本
- Qemu/KVM: 1.0 或更高版本
CloudStack中的默认使用Linux本身的桥接(bridge模块)方式实现。也可选择在CloudStack中使用OpenVswitch，具体要求如下：

- libvirt: 0.9.11 或更高版本
- openvswitch: 1.7.1或更高版本
此外，硬件要求如下：

- 同一集群中主机必须使用相同版本的Linux系统。
- 同一群集中的所有节点架构必须一致。CPU的型号、数量和功能参数必须相同。
- 必须支持HVM(Intel-VT或者AMD-V)
- 64位x86 CPU(多核性能更佳)
- 4GB内存
- 至少一块网卡
- 在部署CloudStack时，Hypervisor主机不能运行任何虚拟机

# KVM安装概述
如果你想使用 KVM hypervisor来运行虚拟机，请在你的云环境中安装KVM主机。本章节不会复述KVM的安装文档。它提供了KVM主机与CloudStack协同工作所要准备特有的步骤。

> 警告 在我们开始之前，请确保所有的主机都安装了最新的更新包。
> 警告 不建议在主机中运行与CloudStack无关的服务。

# 准备操作系统
主机的操作系统必须为运行CloudStack Agent和KVM实例做些准备。

使用root用户登录操作系统。
检查FQN完全合格/限定主机名。

```bash
$ hostname --fqdn
```
该命令会返回完全合格/限定主机名，例如”kvm1.lab.example.org”。如果没有，请编辑 /etc/hosts。
请确保主机能够访问互联网。

```bash
$ ping www.cloudstack.org
```
启用NTP服务以确保时间同步.

注解

NTP服务用来同步云中的服务器时间。时间不同步会带来意想不到的问题。
安装NTP

```bash
$ yum install ntp
$ apt-get install openntpd
```

确认CPU支持虚拟化技术
```bash
# cat /proc/cpuinfo| grep -E '(vmx|svm)'
```

在所有主机中重复上述步骤。

# 安装和配置Agent
CloudStack使用Agent来管理KVM实例。Agent与管理服务器通讯并控制主机上所有的虚拟机。

首先我们安装Agent：

在RHEL/CentOS上:

```bash
$ yum install cloudstack-agent
```
在Ubuntu上:

```bash
$ apt-get install cloudstack-agent
```
现在主机已经为加入群集做好准备。

# 配置KVM虚拟机的CPU型号(可选)
此外，CloudStack Agent允许主机管理员控制KVM实例中的CPU型号。默认情况下，KVM实例的CPU型号为只有少数CPU特性且版本为xxx的QEMU Virtual CPU。指定CPU型号有几个原因：

- 通过主机CPU的特性最大化提升KVM实例的性能；
- 确保所有机器的默认CPU保持一致，消除对QEMU变量的依赖。
在大多数情况下,主机管理员需要每个主机配置文件(/etc/cloudstack/agent/agent.properties)中指定guest虚拟机的CPU配置。这将通过引入两个新的配置参数来实现：

```bash
guest.cpu.mode=custom|host-model|host-passthrough
guest.cpu.model=from /usr/share/libvirt/cpu_map.xml(only valid when guest.cpu.mode=custom)
```
更改CPU型号有三个选择：

- custom:
指定一个在/usr/share/libvirt/cpu_map.xml文件中所支持的型号名称。
- host-model:
 libvirt可以识别出在/usr/share/libvirt/cpu_map.xml中与主机最接近的CPU型号，然后请求其他的CPU flags完成匹配。如果虚拟机迁移到其他CPU稍有不同的主机中，保持好的可靠性/兼容性能提供最多的功能和最大限度提示的性能。
- host-passthrough:
libvirt 会告诉KVM没有修改过CPU passthrough的主机。与host-model的差别是不仅匹配flags特性，还要匹配CPU的每一个特性。他能提供最好的性能， 同时对一些检查CPU底层特性的应用程序很重要，但这样会带来一些迁移的代价：虚拟机只会迁移到CPU完全匹配的主机上。

这里有一些示例：

- custom

```bash
guest.cpu.mode=custom
guest.cpu.model=SandyBridge
```

- host-model

```bash
guest.cpu.mode=host-model
```

- host-passthrough

```bash
guest.cpu.mode=host-passthrough
```

> 注意：host-passthrough可能会导致迁移失败，如果你遇到这个问题，你应该使用 host-model或者custom


# 安装和配置libvirt

```bash
#  yum  -y groupinstall 'Virtualization' 'Virtualization Client' 'Virtualzation Platform' 'Virtualization Tools'
# yum -y install kvm kmod-kvm qemu kvm-qemu-img virt-viewer virt-manager libvirt vconfig
# lsmod | grep kvm
```
得益于Linux内核的支持，KVM 相关包都不大，这步应该可以很快完成。装好后为保证管理节点可以正常调用，还需要开放相关端口。

CloudStack使用libvirt管理虚拟机。因此正确地配置libvirt至关重要。CloudStack-agent依赖于Libvirt，应提前安装完毕。

为了实现动态迁移libvirt需要监听不可靠的TCP连接。还要关闭libvirts尝试使用组播DNS进行广播。这些都可以在 /etc/libvirt/libvirtd.conf文件中进行配置。

设定下列参数：

```bash
# vi /etc/libvirt/libvirtd.conf
listen_tls = 0
listen_tcp = 1
tcp_port = "16509"
auth_tcp = "none"
mdns_adv = 0
```
除了在libvirtd.conf中打开”listen_tcp”以外，还必须修改/etc/sysconfig/libvirtd中的参数:

在RHEL或者CentOS中修改
```bash
vi /etc/sysconfig/libvirtd：
//取消如下行的注释：
#LIBVIRTD_ARGS="--listen"
```
在Ubuntu中修改
```bash
vi /etc/default/libvirt-bin
//在下列行添加 “-l”
libvirtd_opts="-d"
//如下所示:
libvirtd_opts="-d -l"
```
修改qemu配置，取消vnc_listen=0.0.0.0前面的注释
```bash
#  less /etc/libvirt/qemu.conf
vnc_listen = "0.0.0.0"
```

```bash
#  less /etc/cgconfig.conf
group virt {
cpu {
cpu.shares = 9216;
}
}
```

重启libvirt服务

在RHEL/CentOS上:

```bash
$ service libvirtd restart
```
在Ubuntu上:

```bash
$ service libvirt-bin restart
```

# 配置安全策略
CloudStack的会被例如AppArmor和SELinux的安全机制阻止。必须关闭安全机制并确保 Agent具有所必需的权限。

## 配置SELinux（RHEL和CentOS）：

在RHEL或者CentOS中，SELinux是默认安装并启动的。你可以使用如下命令验证：

```bash
# rpm -qa | grep selinux
```
在 /etc/selinux/config 中设置SELINUX变量值为 “permissive”。这样能确保对SELinux的设置在系统重启之后依然生效。

在RHEL/CentOS上:
```bash
# getenforce
//查看当前 selinux 状态
# sed -i 's/enforcing/disabled/' /etc/selinux/config
//修改 selinux 配置文件，重启永久禁用
# setenforce permissive
//临时设置 selinux 状态
```
然后使SELinux立即运行于permissive模式，无需重新启动系统。

### 配置AppArmor(Ubuntu)

Ubuntu中默认安装并启动AppArmor。使用如下命令验证：

```bash
$ dpkg --list 'apparmor'
```
在AppArmor配置文件中禁用libvirt

```bash
$ ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
$ ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
$ apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
$ apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

# 配置网络桥接

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
> 注解 本章节的目标是配置两个名为 ‘cloudbr0’和’cloudbr1’的桥接网络。这仅仅是指导性的，实际情况还要取决于你的网络布局。

## 在RHEL或CentOS中配置：
网络桥接所需的软件在安装libvirt时就已被安装，继续配置网络。

首先配置eth0：

```bash
vi /etc/sysconfig/network-scripts/ifcfg-eth0
//确保内容如下所示：
DEVICE=eth0
HWADDR=00:04:xx:xx:xx:xx
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
TYPE=Ethernet
```
现在配置3个VLAN接口：

```bash
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
```
配置VLAN接口以便能够附加桥接网络。

```bash
vi /etc/sysconfig/network-scripts/ifcfg-cloudbr0
//现在只配置一个没有IP的桥接。
DEVICE=cloudbr0
TYPE=Bridge
ONBOOT=yes
BOOTPROTO=none
IPV6INIT=no
IPV6_AUTOCONF=no
DELAY=5
STP=yes
```
同样建立cloudbr1

```bash
vi /etc/sysconfig/network-scripts/ifcfg-cloudbr1
DEVICE=cloudbr1
TYPE=Bridge
ONBOOT=yes
BOOTPROTO=none
IPV6INIT=no
IPV6_AUTOCONF=no
DELAY=5
STP=yes
```
配置完成之后重启网络，通过重启检查一切是否正常。

> 警告 在发生配置错误和网络故障的时，请确保可以能通过其他方式例如IPMI或ILO连接到服务器。

## 在Ubuntu中配置：
在安装libvirt时所需的其他软件也会被安装，所以只需配置网络即可。

```bash
vi /etc/network/interfaces
//如下所示修改接口文件：
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
    ```
配置完成之后重启网络，通过重启检查一切是否正常。

> 警告 在发生配置错误和网络故障的时，请确保可以能通过其他方式例如IPMI或ILO连接到服务器。

# 配置使用OpenVswitch网络

为了转发流量到实例，至少需要两个桥接网络： public 和 private。

By default these bridges are called cloudbr0 and cloudbr1, but you do have to make sure they are available on each hypervisor.

最重要的因素是所有hypervisors上的配置要保持一致。

准备
将系统自带的网络桥接模块加入黑名单，确保该模块不会与openvswitch模块冲突。请参阅你所使用发行版的modprobe文档并找到黑名单。确保该模块不会在重启时自动加载或在下一操作步骤之前卸载该桥接模块。

以下网络配置依赖ifup-ovs和ifdown-ovs脚本，安装openvswitch后会提供该脚本。安装路径为位/etc/sysconfig/network-scripts/。

网络示例
配置网络有很多方法。在基本网络模式中你应该拥有2个 (V)LAN，一个用于管理网络，一个用于公共网络。

假设hypervisor中的网卡(eth0)有3个VLAN标签：

VLAN 100 作为hypervisor的管理网络
VLAN 200 for public network of the instances (cloudbr0)
VLAN 300 作为实例的专用网络 (cloudbr1)
在VLAN 100 中，配置Hypervisor的IP为 192.168.42.11/24，网关为192.168.42.1

> 注解 Hypervisor与管理服务器不需要在同一个子网！
> 注解 本章节的目标是设置三个名为’mgmt0’, ‘cloudbr0’和’cloudbr1’ 桥接网络。这仅仅是指导性的，实际情况还要取决于你的网络状况。

## 配置OpenVswitch
使用ovs-vsctl命令创建基于OpenVswitch的网络接口。该命令将配置此接口并将信息保存在OpenVswitch数据库中。

首先我们创建一个连接至eth0接口的主桥接。然后我们创建三个虚拟桥接，每个桥接都连接指定的VLAN。

```bash
# ovs-vsctl add-br cloudbr
# ovs-vsctl add-port cloudbr eth0
# ovs-vsctl set port cloudbr trunks=100,200,300
# ovs-vsctl add-br mgmt0 cloudbr 100
# ovs-vsctl add-br cloudbr0 cloudbr 200
# ovs-vsctl add-br cloudbr1 cloudbr 300
```
## 在RHEL或CentOS中配置：
所需的安装包在安装openvswitch和libvirt的时就已经安装，继续配置网络。

首先配置eth0：

```bash
vi /etc/sysconfig/network-scripts/ifcfg-eth0
//如下所示
DEVICE=eth0
HWADDR=00:04:xx:xx:xx:xx
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
TYPE=Ethernet
```
必须将基础桥接配置为trunk模式。

```bash
vi /etc/sysconfig/network-scripts/ifcfg-cloudbr
DEVICE=cloudbr
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
DEVICETYPE=ovs
TYPE=OVSBridge
```
现在对三个VLAN桥接进行配置：

```bash
vi /etc/sysconfig/network-scripts/ifcfg-mgmt0
DEVICE=mgmt0
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=static
DEVICETYPE=ovs
TYPE=OVSBridge
IPADDR=192.168.42.11
GATEWAY=192.168.42.1
NETMASK=255.255.255.0
vi /etc/sysconfig/network-scripts/ifcfg-cloudbr0
DEVICE=cloudbr0
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
DEVICETYPE=ovs
TYPE=OVSBridge
vi /etc/sysconfig/network-scripts/ifcfg-cloudbr1
DEVICE=cloudbr1
ONBOOT=yes
HOTPLUG=no
BOOTPROTO=none
TYPE=OVSBridge
DEVICETYPE=ovs
```
配置完成之后重启网络，通过重启检查一切是否正常。

> 警告 在发生配置错误和网络故障的时，请确保可以能通过其他方式例如IPMI或ILO连接到服务器。

# 配置防火墙
hypervisor之间和hypervisor与管理服务器之间要能够通讯。为了达到这个目的，我们需要开通以下TCP端口(如果使用防火墙)：

22 (SSH)
1798
16509 (libvirt)
5900 - 6100 (VNC 控制台)
49152 - 49216 (libvirt在线迁移)

### 在RHEL/CentOS中打开端口
RHEL 及 CentOS使用iptables作为防火墙，执行以下iptables命令来开启端口：

```bash
$ iptables -I INPUT -p tcp -m tcp --dport 22 -j ACCEPT
$ iptables -I INPUT -p tcp -m tcp --dport 1798 -j ACCEPT
$ iptables -I INPUT -p tcp -m tcp --dport 16509 -j ACCEPT
$ iptables -I INPUT -p tcp -m tcp --dport 5900:6100 -j ACCEPT
$ iptables -I INPUT -p tcp -m tcp --dport 49152:49216 -j ACCEPT
//这些iptables配置并不会持久保存，重启之后将会消失，我们必须手动保存这些配置。
$ iptables-save > /etc/sysconfig/iptables
```

### 在Ubuntu中打开端口：
Ubuntu中的默认防火墙是UFW(Uncomplicated FireWall)，使用Python围绕iptables进行包装。

要打开所需端口，请执行以下命令：

```bash
$ ufw allow proto tcp from any to any port 22
$ ufw allow proto tcp from any to any port 1798
$ ufw allow proto tcp from any to any port 16509
$ ufw allow proto tcp from any to any port 5900:6100
$ ufw allow proto tcp from any to any port 49152:49216
```
> 注解 默认情况下，Ubuntu中并未启用UFW。在关闭情况下执行这些命令并不能启用防火墙。
