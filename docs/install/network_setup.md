# Network Setup

Achieving the correct networking setup is crucial to a successful
CloudStack installation. This section contains information to help you
make decisions and follow the right procedures to get your network set
up correctly.
正确的网络设置对于CloudStack能否成功安装至关重要。遵循本节包含的信息，指导你搭建正确的网络配置。

## 基本和高级网络

CloudStack提供二种网络类型:

**Basic**
For AWS-style networking. Provides a single network where guest isolation can
be provided through layer-3 means such as security groups (IP address source
filtering).
适用于AWS类型的网络。提供单一的来宾网络类型，通过三层安全组进行隔离（源IP地址过滤）。

**Advanced**
For more sophisticated network topologies. This network model provides the
most flexibility in defining guest networks, but requires more configuration
steps than basic networking.
适用于更复杂的网络拓扑。这种网络模型在定义来宾网络时提供了最大限度的灵活性，但是比基本网络需要更多的配置步骤。

Each zone has either basic or advanced networking. Once the choice of
networking model for a zone has been made and configured in CloudStack,
it can not be changed. A zone is either basic or advanced for its entire
lifetime.
每个区域都有基本或高级网络。一个区域的整个生命周期中，不论是基本或高级网络。一旦在CloudStack中选择并配置区域的网络类型，就无法再修改。

The following table compares the networking features in the two networking models.
下表比较了两种网络类型的功能

| Networking Feature       |  Basic Network                     |   Advanced Network|
| :----: | :----: |:----:|
| Number of networks       |  Single network                    |   Multiple networks
| Firewall type            |  Physical                          |   Physical and Virtual
| Load balancer            |  Physical                          |   Physical and Virtual
| Isolation type           |  Layer 3                           |   Layer 2 and Layer 3
| VPN support              |  No                                |   Yes
| Port forwarding          |  Physical                          |   Physical and Virtual
| 1:1 NAT                  |  Physical                          |   Physical and Virtual
| Source NAT               |  No                                |   Physical and Virtual
| Userdata                 |  Yes                               |   Yes
| Network usage monitoring |  sFlow / netFlow at physical router|   Hypervisor and Virtual Router
| DNS and DHCP             |  Yes                               |   Yes


The two types of networking may be in use in the same cloud. However, a
given zone must use either Basic Networking or Advanced Networking.
在一个云中可能会存在二种网络类型。但无论如何，一个给定的区域必须使用基本网络或高级网络。

Different types of network traffic can be segmented on the same physical
network. Guest traffic can also be segmented by account. To isolate
traffic, you can use separate VLANs. If you are using separate VLANs on
a single physical network, make sure the VLAN tags are in separate
numerical ranges.
单一的物理网络可以被分割不同类型的网络流量。账户也可以分割来宾流量。你可以通过划分VLAN来隔离流量。如果你在物理网络中划分了VLAN，确保VLAN标签的数值在独立范围。


## VLAN分配示例

VLANs are required for public and guest traffic. The following is an
example of a VLAN allocation scheme:
公共和来宾流量要求使用VLAN，下面是一个VLAN分配的示例：

.. cssclass:: table-striped table-bordered table-hover


| VLAN IDs   |          Traffic type        |                                        Scope|
| :----: | :----: |:----:|
| less than 500    |    Management traffic. Reserved for administrative purposes. |  CloudStack software can access this, hypervisors, system VMs.
| 500-599          |    VLAN carrying public traffic.                            |   CloudStack accounts.
| 600-799          |    VLANs carrying guest traffic.                             |  CloudStack accounts. Account-specific VLAN is chosen from this pool.
| 800-899          |    VLANs carrying guest traffic.                            |   CloudStack accounts. Account-specific VLAN chosen by CloudStack admin to assign to that account.
| 900-999           |   VLAN carrying guest traffic                              |   CloudStack accounts. Can be scoped by project, domain, or all accounts.
| greater than 1000 |   Reserved for future use|  |



## 硬件配置示例

This section contains an example configuration of specific switch models
for zone-level layer-3 switching. It assumes VLAN management protocols,
such as VTP or GVRP, have been disabled. The example scripts must be
changed appropriately if you choose to use VTP or GVRP.
本节包含了一个特定交换机型号的示例配置，作为区域级别的三层交换。假设VLAN管理协议，如VTP、GVRP等已经被禁用。如果你选择使用VTP或GVRP，那么你必须适当的修改示例脚本。

### Cisco 3750

The following steps show how a Cisco 3750 is configured for zone-level
layer-3 switching. These steps assume VLAN 201 is used to route untagged
private IPs for pod 1, and pod 1’s layer-2 switch is connected to
GigabitEthernet1/0/1.
以下步骤显示如何配置Cisco 3750作为区域级别的三层交换机。这些步骤假设VLAN 201用于Pod1中未标记的私有IP线路。并且Pod1中的二层交换机已经连接到GigabitEthernet1/0/1端口。

#. Setting VTP mode to transparent allows us to utilize VLAN IDs above
   1000. Since we only use VLANs up to 999, vtp transparent mode is not
   strictly required.
   设置VTP为透明模式，以便我们使用超过1000的VLAN。因为我们只用到999，VTP透明模式不做严格要求。

```shell
      vtp mode transparent
      vlan 200-999
      exit```

#. Configure GigabitEthernet1/0/1.
配置GigabitEthernet1/0/1。
```shell
      interface GigabitEthernet1/0/1
      switchport trunk encapsulation dot1q
      switchport mode trunk
      switchport trunk native vlan 201
      exit```

The statements configure GigabitEthernet1/0/1 as follows:
端口GigabitEthernet1/g1的配置说明

-  VLAN 201 is the native untagged VLAN for port GigabitEthernet1/0/1.
端口1/g1的本征VLAN为201（属于VLAN201）
-  Cisco passes all VLANs by default. As a result, all VLANs (300-999)
   are passed to all the pod-level layer-2 switches.
Cisco默认允许所有VLAN，因此，VLAN（300-999）都被允许并访问所有的Pod级别的二层交换机。

### Layer-2 Switch
二层交换

The layer-2 switch is the access switching layer inside the pod.
在Pod中的二层交换机提供接入层功能

-  It should trunk all VLANs into every computing host.
它​通​过​trunk连​接​所​有​VLANS中​的​计​算​主​机。

-  It should switch traffic for the management network containing
   computing and storage hosts. The layer-3 switch will serve as the
   gateway for the management network.
它​为​管​理​网​络​提​供​包​含​计​算​和​存​储​主​机​的​流​量​交​换​。​三​层​交​换​机​将​作​为​这​个​管​理​网​络​的​网​关

The following sections contain example configurations for specific switch models
for pod-level layer-2 switching. It assumes VLAN management protocols
such as VTP or GVRP have been disabled. The scripts must be changed
appropriately if you choose to use VTP or GVRP.
本节包含了一个特定交换机型号的示例配置，作为Pod级别的二层交换机。它假设VLAN管理协议，例如VTP、GVRP等已经被禁用。如果你正在使用VTP或GVRP，那么你必须适当的修改示例脚本。

### Cisco 3750

The following steps show how a Cisco 3750 is configured for pod-level
layer-2 switching.
以​下​步​骤​将​演​示​如何​配​置​cisco 3750作​为​Pod级​别​的​二层交​换​机

#. Setting VTP mode to transparent allows us to utilize VLAN IDs above
   1000. Since we only use VLANs up to 999, vtp transparent mode is not
   strictly required.
   设置VTP为透明模式，以便我们使用超过1000的VLAN。因为我们只用到999，VTP透明模式并不做严格要求。

```shell
      vtp mode transparent
      vlan 300-999
      exit```

#. Configure all ports to dot1q and set 201 as the native VLAN.
配置所有的端口使用dot1q协议，并设置201为本征VLAN。

```shell
      interface range GigabitEthernet 1/0/1-24
      switchport trunk encapsulation dot1q
      switchport mode trunk
      switchport trunk native vlan 201
      exit```

By default, Cisco passes all VLANs. Cisco switches complain of the
native VLAN IDs are different when 2 ports are connected together.
That’s why you must specify VLAN 201 as the native VLAN on the layer-2
switch.
默​认​情​况下​，Cisco允​许​所有​VLAN通过。​如果本征VLAN ID不相同的2个​​端​口​连​接​在​一​起时​​，​Cisco交​换​机​提出控诉。​这​就​是​为​什​么​必​须​指​定​VLAN 201为​二交​换​机​的​本​征​VLAN。

## Hardware Firewall
## 硬件防火墙

All deployments should have a firewall protecting the management server;
see Generic Firewall Provisions. Optionally, some deployments may also
have a Juniper SRX firewall that will be the default gateway for the
guest networks; see `“External Guest Firewall Integration for Juniper SRX (Optional)” <#external-guest-firewall-integration-for-juniper-srx-optional>`_.
所有部署都应该有一个防火墙保护管理服务器；查看防火墙的一般规定。 根据情况不同，一些部署也可能用到Juniper SRX防火墙作为来宾网络的默认网关； 参阅 “集成外部来宾防火墙Juniper SRX （可选）”.

### Generic Firewall Provisions
### 防火墙的一般规定

The hardware firewall is required to serve two purposes:
硬件防火墙必需服务于两个目的：

-  Protect the Management Servers. NAT and port forwarding should be
   configured to direct traffic from the public Internet to the
   Management Servers.
   保护管理服务器。

-  Route management network traffic between multiple zones. Site-to-site
   VPN should be configured between multiple zones.
   路由多个区域之间的管理网络流量。站点到站点 VPN应该在多个区域之间配置。

To achieve the above purposes you must set up fixed configurations for
the firewall. Firewall rules and policies need not change as users are
provisioned into the cloud. Any brand of hardware firewall that supports
NAT and site-to-site VPN can be used.
为了达到上述目的，你必须设置固定配置的防火墙。用户被配置到云中，不需要改变防火墙规则和策略。任何支持NAT和站点到站点VPN的硬件防火墙，不论品牌，都可以使用。

### External Guest Firewall Integration for Juniper SRX (Optional)
### 集成外部来宾防火墙Juniper SRX（可选）

.. note::
   Available only for guests using advanced networking.
客户使用高级网络时才有效。

CloudStack provides for direct management of the Juniper SRX series of
firewalls. This enables CloudStack to establish static NAT mappings from
public IPs to guest VMs, and to use the Juniper device in place of the
virtual router for firewall services. You can have one or more Juniper
SRX per zone. This feature is optional. If Juniper integration is not
provisioned, CloudStack will use the virtual router for these services.
CloudStack中提供了对Juniper SRX系列防火墙的直接管理。这使得CloudStack能建立公共IP到客户虚拟机的静态NAT映射，并利用Juniper设备替代虚拟路由器提供防火墙服务。每个区域中可以有一个或多个Juniper SRX设备。这个特性是可选的。如果不提供Juniper设备集成，CloudStack会使用虚拟路由器提供这些服务。

The Juniper SRX can optionally be used in conjunction with an external
load balancer. External Network elements can be deployed in a
side-by-side or inline configuration.
Juniper SRX可以和任意的外部负载均衡器一起使用。外部网络元素可以部署为并排或内联结构。

![parallel-mode](parallel-mode.png)

CloudStack requires the Juniper SRX firewall to be configured as follows:

.. note::
   Supported SRX software version is 10.3 or higher.

#. Install your SRX appliance according to the vendor's instructions.

#. Connect one interface to the management network and one interface to
   the public network. Alternatively, you can connect the same interface
   to both networks and a use a VLAN for the public network.

#. Make sure "vlan-tagging" is enabled on the private interface.

#. Record the public and private interface names. If you used a VLAN for
   the public interface, add a ".[VLAN TAG]" after the interface name.
   For example, if you are using ge-0/0/3 for your public interface and
   VLAN tag 301, your public interface name would be "ge-0/0/3.301".
   Your private interface name should always be untagged because the
   CloudStack software automatically creates tagged logical interfaces.

#. Create a public security zone and a private security zone. By
   default, these will already exist and will be called "untrust" and
   "trust". Add the public interface to the public zone and the private
   interface to the private zone. Note down the security zone names.

#. Make sure there is a security policy from the private zone to the
   public zone that allows all traffic.

#. Note the username and password of the account you want the CloudStack
   software to log in to when it is programming rules.

#. Make sure the "ssh" and "xnm-clear-text" system services are enabled.

#. If traffic metering is desired:

   #. Create an incoming firewall filter and an outgoing firewall
      filter. These filters should be the same names as your public
      security zone name and private security zone name respectively.
      The filters should be set to be "interface-specific". For example,
      here is the configuration where the public zone is "untrust" and
      the private zone is "trust":

      .. sourcecode:: bash

         root@cloud-srx# show firewall
         filter trust {
             interface-specific;
         }
         filter untrust {
             interface-specific;
         }

   #. Add the firewall filters to your public interface. For example, a
      sample configuration output (for public interface ge-0/0/3.0,
      public security zone untrust, and private security zone trust) is:

      .. sourcecode:: bash

         ge-0/0/3 {
             unit 0 {
                 family inet {
                     filter {
                         input untrust;
                         output trust;
                     }
                     address 172.25.0.252/16;
                 }
             }
         }

#. Make sure all VLANs are brought to the private interface of the SRX.

#. After the CloudStack Management Server is installed, log in to the
   CloudStack UI as administrator.

#. In the left navigation bar, click Infrastructure.

#. In Zones, click View More.

#. Choose the zone you want to work with.

#. Click the Network tab.

#. In the Network Service Providers node of the diagram, click
   Configure. (You might have to scroll down to see this.)

#. Click SRX.

#. Click the Add New SRX button (+) and provide the following:

   -  IP Address: The IP address of the SRX.

   -  Username: The user name of the account on the SRX that CloudStack
      should use.

   -  Password: The password of the account.

   -  Public Interface. The name of the public interface on the SRX. For
      example, ge-0/0/2. A ".x" at the end of the interface indicates
      the VLAN that is in use.

   -  Private Interface: The name of the private interface on the SRX.
      For example, ge-0/0/1.

   -  Usage Interface: (Optional) Typically, the public interface is
      used to meter traffic. If you want to use a different interface,
      specify its name here

   -  Number of Retries: The number of times to attempt a command on the
      SRX before failing. The default value is 2.

   -  Timeout (seconds): The time to wait for a command on the SRX
      before considering it failed. Default is 300 seconds.

   -  Public Network: The name of the public network on the SRX. For
      example, trust.

   -  Private Network: The name of the private network on the SRX. For
      example, untrust.

   -  Capacity: The number of networks the device can handle

   -  Dedicated: When marked as dedicated, this device will be dedicated
      to a single account. When Dedicated is checked, the value in the
      Capacity field has no significance implicitly, its value is 1

#. Click OK.

#. Click Global Settings. Set the parameter
   external.network.stats.interval to indicate how often you want
   CloudStack to fetch network usage statistics from the Juniper SRX. If
   you are not using the SRX to gather network usage statistics, set to 0.


   CloudStack要求按照如下信息配置Juniper SRX防火墙：

   注解

   支持SRX软件版本为10.3或者更高。
   根据供应商提供的说明书安装你的SRX设备。
   分别使用两个接口连接管理和公共网络，或者，你可以使用同一个接口来连接这两个网络，但需要为公共网络设置一个VLAN。
   确保在专有接口上开启了 “vlan-tagging” 。
   记录公共和专用接口名称。如果你在公共接口中使用了VLAN，应该在接口名称后面添加一个”.[VLAN TAG]”。例如，如果你使用 ge-0/0/3 作为公共接口并且VLAN 标签为301。那么你的公共接口名称应为”ge-0/0/3.301”。你的专用接口应始终不添加任何标签，因为CloudStack会自动在逻辑接口上添加标签。
   创建公共安全区域和专用安全区域。默认情况下，这些已经存在并分别称为”untrust” 和 “trust”。为公共区域添加公共接口和为专用区域添加专用接口。并记录下安全区域名称。
   确保专用区域到公共区域的安全策略允许所有流量通过。
   请注意CloudStack登录所用账户的用户名和密码符合编程规则。
   确保已经开启 “ssh” 和 “xnm-clear-text” 等系统服务。
   如果需要统计流量：

   创建传入的防火墙过滤和传出的防火墙过滤。这些策略应该有相同的名称分别作为你的公共安全区域名称和专用安全区域名称。过滤应该设置到 “特定接口”。 例如，在这个配置中公共区域是 “untrust”，专用区域是 “trust”:

   root@cloud-srx# show firewall
   filter trust {
       interface-specific;
   }
   filter untrust {
       interface-specific;
   }
   在你的公共接口添加防火墙过滤。例如，一个示例配置（ge-0/0/3.0为公共接口，untrust为公共安全区域，trust为专用安全区域）

   ge-0/0/3 {
       unit 0 {
           family inet {
               filter {
                   input untrust;
                   output trust;
               }
               address 172.25.0.252/16;
           }
       }
   }
   确保SRX中的专用接口允许所有VLAN通过。
   安装好CloudStack管理端后，使用管理员帐号登录CloudStack用户界面。
   在左侧导航栏中，点击基础架构
   点击区域中的查看更多。
   选择你要设置的区域。
   点击网络选项卡。
   点击示意图网络服务提供程序中的配置（你可能需要向下滚动才能看到）。
   点击SRX。
   点击（+）按钮添加一个新的SRX，并提供如下信息：

   IP地址：SRX设备的IP地址。
   用户名：CloudStack需要使用SRX设备中的账户。
   密码：该账户的密码。
   公共接口：SRX中的公共接口名称。例如，ge-0/0/2。 最后一个 ”.x” 表示该VLAN正在使用这个接口。
   专用接口: SRX中的专用接口名称。例如, ge-0/0/1。
   Usage Interface: (可选) 通常情况下，公共接口用来统计流量。如果你想使用其他的接口，则在此处修改。
   重试次数：尝试控制SRX设备失败时重试的次数。默认值是2。
   超时(秒): 尝试控制SRX设备失败时的重试间隔。默认为300秒。
   公共网络: SRX中公共网络的名字。 例，trust。
   专用网络：SRX中专用网络的名字。 例如，untrust。
   容量：该设备能处理的网络数量。
   专用: 当标记为专用后，这个设备只对单个帐号专用。该选项被勾选后，容量选项就没有了实际意义且值会被置为1。
   点击确定。
   点击全局设置。设置external.network.stats.interval参数，指定CloudStack从Juniper SRX收集网络使用情况的时间间隔。如果你不使用SRX统计网络使用情况，即设置为0.



### External Guest Firewall Integration for Cisco VNMC (Optional)


Cisco Virtual Network Management Center (VNMC) provides centralized
multi-device and policy management for Cisco Network Virtual Services.
You can integrate Cisco VNMC with CloudStack to leverage the firewall
and NAT service offered by ASA 1000v Cloud Firewall. Use it in a Cisco
Nexus 1000v dvSwitch-enabled cluster in CloudStack. In such a
deployment, you will be able to:

-  Configure Cisco ASA 1000v firewalls. You can configure one per guest
   network.

-  Use Cisco ASA 1000v firewalls to create and apply security profiles
   that contain ACL policy sets for both ingress and egress traffic.

-  Use Cisco ASA 1000v firewalls to create and apply Source NAT, Port
   Forwarding, and Static NAT policy sets.

CloudStack supports Cisco VNMC on Cisco Nexus 1000v dvSwich-enabled
VMware hypervisors.

Cisco虚拟网络管理中心（VNMC）为Cisco网络虚拟服务提供集中式多设备和策略管理。您可以通过整合Cisco VNMC和CloudStack使用ASA 1000v云防火墙提供防火墙和NAT服务。在开启了Cisco Nexus 1000v分布式虚拟交换机功能的CloudStack群集中。这样部署，您将可以：

配置Cisco ASA 1000v防火墙，你可以为每个来宾创建一个网络。
使用思科ASA 1000v防火墙创建和应用安全配置，包含入口和出口流量的ACL策略集。
使用Cisco ASA 1000v防火墙创建和应用源地址NAT，端口转发，静态NAT策略集。
CloudStack对于Cisco VNMC的支持基于开启了Cisco Nexus 1000v分布式虚拟交换机功能的VMware虚拟化管理平台。

Using Cisco ASA 1000v Firewall, Cisco Nexus 1000v dvSwitch, and Cisco VNMC in a Deployment


Guidelines
'''''''''''

-  Cisco ASA 1000v firewall is supported only in Isolated Guest
   Networks.

-  Cisco ASA 1000v firewall is not supported on VPC.

-  Cisco ASA 1000v firewall is not supported for load balancing.

-  When a guest network is created with Cisco VNMC firewall provider, an
   additional public IP is acquired along with the Source NAT IP. The
   Source NAT IP is used for the rules, whereas the additional IP is
   used to for the ASA outside interface. Ensure that this additional
   public IP is not released. You can identify this IP as soon as the
   network is in implemented state and before acquiring any further
   public IPs. The additional IP is the one that is not marked as Source
   NAT. You can find the IP used for the ASA outside interface by
   looking at the Cisco VNMC used in your guest network.

-  Use the public IP address range from a single subnet. You cannot add
   IP addresses from different subnets.

-  Only one ASA instance per VLAN is allowed because multiple VLANS
   cannot be trunked to ASA ports. Therefore, you can use only one ASA
   instance in a guest network.

-  Only one Cisco VNMC per zone is allowed.

-  Supported only in Inline mode deployment with load balancer.

-  The ASA firewall rule is applicable to all the public IPs in the
   guest network. Unlike the firewall rules created on virtual router, a
   rule created on the ASA device is not tied to a specific public IP.

-  Use a version of Cisco Nexus 1000v dvSwitch that support the vservice
   command. For example: nexus-1000v.4.2.1.SV1.5.2b.bin

   Cisco VNMC requires the vservice command to be available on the Nexus
   switch to create a guest network in CloudStack.


   指南
   Cisco ASA 1000v防火墙仅支持隔离的来宾网络。
   Cisco ASA 1000v不支持VPC。
   Cisco ASA 1000v不支持负载均衡。
   当使用Cisco VNMC防火墙创建的来宾网络时，一个额外的公共IP和SNAT IP一起被获取。SNAT IP用于规则，而额外的IP用于ASA外部接口。确保这一额外的公共IP不会被释放。你可以在网络一旦处在执行状态和获取其他公共IP之前确定这个IP。这个额外的IP没有被标识为SNAT。在你的来宾网络中可以通过Cisco VNMC来查找ASA外部接口的IP地址。
   使用单一网络中的公共IP地址段。不能添加不同网段的IP地址。
   每个VLAN中只允许一个ASA实例。因为ASA端口不支持多VLAN的trunk模式。因此，一个来宾网络中只能有一个ASA实例。
   每个区域中，只允许一个Cisco VNMC.
   只支持在Inline模式下部署负载均衡器。
   ASA防火墙策略适用于来宾网络中所有的公共IP地址。不同于虚拟路由器创建的防火墙规则，ASA设备创建的规则不绑定特定的公共IP地址。
   使用一个支持vservice命令的Cisco Nexus 1000v分布式虚拟交换机版本。例如：nexus-1000v.4.2.1.SV1.5.2b.bin

   使用Cisco VNMC在CloudStack中创建来宾网络，要求vservice命令在Nexus交换机中是可用的。

Prerequisites
先决条件

#. Configure Cisco Nexus 1000v dvSwitch in a vCenter environment.

   Create Port profiles for both internal and external network
   interfaces on Cisco Nexus 1000v dvSwitch. Note down the inside port
   profile, which needs to be provided while adding the ASA appliance to
   CloudStack.

   For information on configuration, see
   `“Configuring a vSphere Cluster with Nexus 1000v Virtual Switch”
   <hypervisor_installation.html#configuring-a-vsphere-cluster-with-nexus-1000v-virtual-switch>`_.

#. Deploy and configure Cisco VNMC.

   For more information, see
   `Installing Cisco Virtual Network Management Center
   <http://www.cisco.com/en/US/docs/switches/datacenter/vsg/sw/4_2_1_VSG_2_1_1/install_upgrade/guide/b_Cisco_VSG_for_VMware_vSphere_Rel_4_2_1_VSG_2_1_1_and_Cisco_VNMC_Rel_2_1_Installation_and_Upgrade_Guide_chapter_011.html>`_
   and `Configuring Cisco Virtual Network Management Center
   <http://www.cisco.com/en/US/docs/unified_computing/vnmc/sw/1.2/VNMC_GUI_Configuration/b_VNMC_GUI_Configuration_Guide_1_2_chapter_010.html>`_.

#. Register Cisco Nexus 1000v dvSwitch with Cisco VNMC.

   For more information, see `Registering a Cisco Nexus 1000V with Cisco VNMC
   <http://www.cisco.com/en/US/docs/switches/datacenter/vsg/sw/4_2_1_VSG_1_2/vnmc_and_vsg_qi/guide/vnmc_vsg_install_5register.html#wp1064301>`_.

#. Create Inside and Outside port profiles in Cisco Nexus 1000v dvSwitch.

   For more information, see
   `“Configuring a vSphere Cluster with Nexus 1000v Virtual Switch”
   <hypervisor_installation.html#configuring-a-vsphere-cluster-with-nexus-1000v-virtual-switch>`_.

#. Deploy and Cisco ASA 1000v appliance.

   For more information, see `Setting Up the ASA 1000V Using VNMC
   <http://www.cisco.com/en/US/docs/security/asa/quick_start/asa1000V/setup_vnmc.html>`_.

   Typically, you create a pool of ASA 1000v appliances and register
   them with CloudStack.

   Specify the following while setting up a Cisco ASA 1000v instance:

   -  VNMC host IP.

   -  Ensure that you add ASA appliance in VNMC mode.

   -  Port profiles for the Management and HA network interfaces. This
      need to be pre-created on Cisco Nexus 1000v dvSwitch.

   -  Internal and external port profiles.

   -  The Management IP for Cisco ASA 1000v appliance. Specify the
      gateway such that the VNMC IP is reachable.

   -  Administrator credentials

   -  VNMC credentials

#. Register Cisco ASA 1000v with VNMC.

   After Cisco ASA 1000v instance is powered on, register VNMC from the
   ASA console.


Using Cisco ASA 1000v Services
''''''''''''''''''''''''''''''

#. Ensure that all the prerequisites are met.

   See `“Prerequisites” <#prerequisites>`_.

#. Add a VNMC instance.

   See `“Adding a VNMC Instance” <#adding-a-vnmc-instance>`_.

#. Add a ASA 1000v instance.

   See `“Adding an ASA 1000v Instance” <#adding-an-asa-1000v-instance>`_.

#. Create a Network Offering and use Cisco VNMC as the service provider
   for desired services.

   See `“Creating a Network Offering Using Cisco ASA 1000v”
   <#creating-a-network-offering-using-cisco-asa-1000v>`_.

#. Create an Isolated Guest Network by using the network offering you
   just created.


Adding a VNMC Instance
^^^^^^^^^^^^^^^^^^^^^^

#. Log in to the CloudStack UI as administrator.

#. In the left navigation bar, click Infrastructure.

#. In Zones, click View More.

#. Choose the zone you want to work with.

#. Click the Physical Network tab.

#. In the Network Service Providers node of the diagram, click
   Configure.

   You might have to scroll down to see this.

#. Click Cisco VNMC.

#. Click View VNMC Devices.

#. Click the Add VNMC Device and provide the following:

   -  Host: The IP address of the VNMC instance.

   -  Username: The user name of the account on the VNMC instance that
      CloudStack should use.

   -  Password: The password of the account.

#. Click OK.


Adding an ASA 1000v Instance
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Log in to the CloudStack UI as administrator.

#. In the left navigation bar, click Infrastructure.

#. In Zones, click View More.

#. Choose the zone you want to work with.

#. Click the Physical Network tab.

#. In the Network Service Providers node of the diagram, click
   Configure.

   You might have to scroll down to see this.

#. Click Cisco VNMC.

#. Click View ASA 1000v.

#. Click the Add CiscoASA1000v Resource and provide the following:

   -  **Host**: The management IP address of the ASA 1000v instance. The
      IP address is used to connect to ASA 1000V.

   -  **Inside Port Profile**: The Inside Port Profile configured on
      Cisco Nexus1000v dvSwitch.

   -  **Cluster**: The VMware cluster to which you are adding the ASA
      1000v instance.

      Ensure that the cluster is Cisco Nexus 1000v dvSwitch enabled.

#. Click OK.


Creating a Network Offering Using Cisco ASA 1000v
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To have Cisco ASA 1000v support for a guest network, create a network
offering as follows:

#. Log in to the CloudStack UI as a user or admin.

#. From the Select Offering drop-down, choose Network Offering.

#. Click Add Network Offering.

#. In the dialog, make the following choices:

   -  **Name**: Any desired name for the network offering.

   -  **Description**: A short description of the offering that can be
      displayed to users.

   -  **Network Rate**: Allowed data transfer rate in MB per second.

   -  **Traffic Type**: The type of network traffic that will be carried
      on the network.

   -  **Guest Type**: Choose whether the guest network is isolated or
      shared.

   -  **Persistent**: Indicate whether the guest network is persistent
      or not. The network that you can provision without having to
      deploy a VM on it is termed persistent network.

   -  **VPC**: This option indicate whether the guest network is Virtual
      Private Cloud-enabled. A Virtual Private Cloud (VPC) is a private,
      isolated part of CloudStack. A VPC can have its own virtual
      network topology that resembles a traditional physical network.
      For more information on VPCs, see `“About Virtual Private Clouds”
      <http://docs.cloudstack.apache.org/projects/cloudstack-administration/en/latest/networking2.html#about-virtual-private-clouds>`_.

   -  **Specify VLAN**: (Isolated guest networks only) Indicate whether
      a VLAN should be specified when this offering is used.

   -  **Supported Services**: Use Cisco VNMC as the service provider for
      Firewall, Source NAT, Port Forwarding, and Static NAT to create an
      Isolated guest network offering.

   -  **System Offering**: Choose the system service offering that you
      want virtual routers to use in this network.

   -  **Conserve mode**: Indicate whether to use conserve mode. In this
      mode, network resources are allocated only when the first virtual
      machine starts in the network.

#. Click OK

   The network offering is created.


Reusing ASA 1000v Appliance in new Guest Networks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can reuse an ASA 1000v appliance in a new guest network after the
necessary cleanup. Typically, ASA 1000v is cleaned up when the logical
edge firewall is cleaned up in VNMC. If this cleanup does not happen,
you need to reset the appliance to its factory settings for use in new
guest networks. As part of this, enable SSH on the appliance and store
the SSH credentials by registering on VNMC.

#. Open a command line on the ASA appliance:

   #. Run the following:

      .. sourcecode:: bash

         ASA1000V(config)# reload

      You are prompted with the following message:

      .. sourcecode:: bash

         System config has been modified. Save? [Y]es/[N]o:"

   #. Enter N.

      You will get the following confirmation message:

      .. sourcecode:: bash

         "Proceed with reload? [confirm]"

   #. Restart the appliance.

#. Register the ASA 1000v appliance with the VNMC:

   .. sourcecode:: bash

      ASA1000V(config)# vnmc policy-agent
      ASA1000V(config-vnmc-policy-agent)# registration host vnmc_ip_address
      ASA1000V(config-vnmc-policy-agent)# shared-secret key where key is the shared secret for authentication of the ASA 1000V connection to the Cisco VNMC


## External Guest Load Balancer Integration (Optional)
## 整合外部来宾负载均衡（可选）

CloudStack can optionally use a Citrix NetScaler or BigIP F5 load
balancer to provide load balancing services to guests. If this is not
enabled, CloudStack will use the software load balancer in the virtual
router.
CloudStack可以使用Citrix NetScaler或BigIP F5负载均衡器为客户提供负载平衡服务。如果未启用,CloudStack将使用虚拟路由器中的软件负载平衡。

To install and enable an external load balancer for CloudStack
management:
在CloudStack管理端中安装和启用外部负载均衡：

#. Set up the appliance according to the vendor's directions.

#. Connect it to the networks carrying public traffic and management
   traffic (these could be the same network).

#. Record the IP address, username, password, public interface name, and
   private interface name. The interface names will be something like
   "1.1" or "1.2".

#. Make sure that the VLANs are trunked to the management network
   interface.

#. After the CloudStack Management Server is installed, log in as
   administrator to the CloudStack UI.

#. In the left navigation bar, click Infrastructure.

#. In Zones, click View More.

#. Choose the zone you want to work with.

#. Click the Network tab.

#. In the Network Service Providers node of the diagram, click
   Configure. (You might have to scroll down to see this.)

#. Click NetScaler or F5.

#. Click the Add button (+) and provide the following:

根据供应商提供的说明书安装你的设备。
连接到承载公共流量和管理流量的网络（可以是同一个网络）。
记录IP地址、用户名、密码、公共接口名称和专用接口名称。接口名称类似 “1.1” 或 “1.2”。
确保VLAN可以通过trunk到管理网络接口。
安装好CloudStack管理端后，使用管理员帐号登录CloudStack用户界面。
在左侧导航栏中，点击基础架构
点击区域中的查看更多。
选择你要设置的区域。
点击网络选项卡。
点击示意图网络服务提供程序中的配置（你可能需要向下滚动才能看到）。
点击NetScaler或F5.
点击（+）添加按钮并提供如下信息：

   For NetScaler:
对于NetScaler:

   -  IP Address: The IP address of the SRX.

   -  Username/Password: The authentication credentials to access the
      device. CloudStack uses these credentials to access the device.

   -  Type: The type of device that is being added. It could be F5 Big
      Ip Load Balancer, NetScaler VPX, NetScaler MPX, or NetScaler SDX.
      For a comparison of the NetScaler types, see the CloudStack
      Administration Guide.

   -  Public interface: Interface of device that is configured to be
      part of the public network.

   -  Private interface: Interface of device that is configured to be
      part of the private network.

   -  Number of retries. Number of times to attempt a command on the
      device before considering the operation failed. Default is 2.

   -  Capacity: The number of networks the device can handle.

   -  Dedicated: When marked as dedicated, this device will be dedicated
      to a single account. When Dedicated is checked, the value in the
      Capacity field has no significance implicitly, its value is 1.

#. Click OK.
IP地址：SRX设备的IP地址。
用户名/密码：访问设备的身份验证凭证。CloudStack使用这些凭证来访问设备。
类型：添加设备的类型。可以是F5 BigIP负载均衡器、NetScaler VPX、NetScaler MPX或 NetScaler SDX等设备。关于NetScaler的类型比较，请参阅CloudStack管理指南。
公共接口: 配置为公共网络部分的设备接口。
专用接口: 配置为专用网络部分的设备接口。
重试次数：尝试控制设备失败时重试的次数。默认值是2
容量：该设备能处理的网络数量。
专用: 当标记为专用后，这个设备只对单个帐号专用。该选项被勾选后，容量选项就没有了实际意义且值会被置为1。
点击确定。

The installation and provisioning of the external load balancer is
finished. You can proceed to add VMs and NAT or load balancing rules.
外部负载平衡器的安装和配置完成后，你就可以开始添加虚拟机和NAT或负载均衡规则。

## Management Server Load Balancing
## 管理服务器负载均衡

CloudStack can use a load balancer to provide a virtual IP for multiple
Management Servers. The administrator is responsible for creating the
load balancer rules for the Management Servers. The application requires
persistence or stickiness across multiple sessions. The following chart
lists the ports that should be load balanced and whether or not
persistence is required.
CloudStack可以使用负载均衡器为多管理服务器提供一个虚拟IP。管理员负责创建管理服务器的负载均衡规则。应用程序需要跨多个持久性或stickiness的会话。下表列出了需要进行负载平衡的端口和是否有持久性要求。

Even if persistence is not required, enabling it is permitted.
即使不需要持久性，也使它是允许的。

.. cssclass:: table-striped table-bordered table-hover


| Source Port |  Destination Port    |        Protocol   |      Persistence Required?
| ---|--- |--- |
| 80 or 443  |   8080 (or 20400 with AJP)|    HTTP (or AJP)  |  Yes
| 8250         8250                   |     TCP           |   Yes
| 8096    |      8096                 |       HTTP     |        No


In addition to above settings, the administrator is responsible for
setting the 'host' global config value from the management server IP to
load balancer virtual IP address. If the 'host' value is not set to the
VIP for Port 8250 and one of your management servers crashes, the UI is
still available but the system VMs will not be able to contact the
management server.
除了上面的设置，管理员还负责设置‘host’全局配置值，由管理服务器IP地址更改为负载均衡虚拟IP地址。如果‘host’值未设置为VIP的8250端口并且一台管理服务器崩溃时，用户界面依旧可用，但系统虚拟机将无法与管理服务器联系。

## Topology Requirements
## 拓扑结构要求

### Security Requirements
### 安全需求

The public Internet must not be able to access port 8096 or port 8250 on
the Management Server.
公共互联网必须不能访问管理服务器的8096和8250端口

### Runtime Internal Communications Requirements
### 运行时内部通信需求

-  The Management Servers communicate with each other to coordinate
   tasks. This communication uses TCP on ports 8250 and 9090.
 管理服务器需要跟其他主机进行任务协调。该通信使用TCP协议的8250和9090端口。

-  The console proxy VMs connect to all hosts in the zone over the
   management traffic network. Therefore the management traffic network
   of any given pod in the zone must have connectivity to the management
   traffic network of all other pods in the zone.
   CPVM跟区域中的所有主机通过管理网络通信。因此区域中任何一个Pod都必须能通过管理网络连接到其他Pod。

-  The secondary storage VMs and console proxy VMs connect to the
   Management Server on port 8250. If you are using multiple Management
   Servers, the load balanced IP address of the Management Servers on
   port 8250 must be reachable.
   SSVM和CPVM通过8250端口与管理服务器联系。如果你使用了多个管理服务器，确保负载均衡器IP地址到管理服务器的8250端口是可达的。

### Storage Network Topology Requirements
### 存储网络拓扑要求

The secondary storage NFS export is mounted by the secondary storage VM.
Secondary storage traffic goes over the management traffic network, even
if there is a separate storage network. Primary storage traffic goes
over the storage network, if available. If you choose to place secondary
storage NFS servers on the storage network, you must make sure there is
a route from the management traffic network to the storage network.

SSVM需要挂载辅助存储中的NFS共享目录。即使有一个单独的存储网络，辅助存储的流量也会通过管理网络。如果存储网络可用，主存储的流量会通过存储网络。如果你选择辅助存储也使用存储网络，你必须确保有一条从管理网络到存储网络的路由。

### External Firewall Topology Requirements
### 外部防火墙拓扑要求

When external firewall integration is in place, the public IP VLAN must
still be trunked to the Hosts. This is required to support the Secondary
Storage VM and Console Proxy VM.
如果整合了外部防火墙设备，公共IP的VLAN必须通过trunk到达主机。这是支持SSVM和CPVM必须满足的。

### Advanced Zone Topology Requirements
### 高级区域拓扑要求

With Advanced Networking, separate subnets must be used for private and
public networks.
使用高级网络，专用和公共网络必须分离子网。

### XenServer Topology Requirements
### XenServer拓扑要求

The Management Servers communicate with XenServer hosts on ports 22
(ssh), 80 (HTTP), and 443 (HTTPs).
管理服务器与XenServer服务器通过22（ssh）、80（http）和443（https）端口通信。

### VMware Topology Requirements
### VMware拓扑要求

-  The Management Server and secondary storage VMs must be able to
   access vCenter and all ESXi hosts in the zone. To allow the necessary
   access through the firewall, keep port 443 open.
   管理服务器和SSVM必须能够访问区域中的vCenter和所有的ESXi主机。必须保证在防火墙中允许443端口。

-  The Management Servers communicate with VMware vCenter servers on
   port 443 (HTTPs).
   管理服务器与VMware vCenter服务器通过443（https）端口通信。

-  The Management Servers communicate with the System VMs on port 3922
   (ssh) on the management traffic network.
   管理服务器与系统VM使用管理网络通过3922（ssh）端口通信。

### Hyper-V Topology Requirements
### Hyper-V拓扑要求

CloudStack Management Server communicates with Hyper-V Agent by using
HTTPS. For secure communication between the Management Server and the
Hyper-V host, open port 8250.
CloudStack管理服务器通过https与Hyper-V代理通信。管理服务器与Hyper-V主机之间的安全通信端口为8250。

### KVM Topology Requirements
### KVM拓扑要求

The Management Servers communicate with KVM hosts on port 22 (ssh).
管理服务器与KVM主机通过22（ssh）端口通信。

### LXC Topology Requirements
### LXC拓扑要求

The Management Servers communicate with LXC hosts on port 22 (ssh).
管理服务器与LXC主机通过22（ssh）端口通信。

## Guest Network Usage Integration for Traffic Sentinel
## 通过流量哨兵整合来宾网络使用情况

To collect usage data for a guest network, CloudStack needs to pull the
data from an external network statistics collector installed on the
network. Metering statistics for guest networks are available through
CloudStack’s integration with inMon Traffic Sentinel.
为了在来宾网络中收集网络数据，CloudStack需要从网络中安装的外部网络数据收集器中提取。并通过整合CloudStack和inMon流量哨兵实现来宾网络的数据统计。

Traffic Sentinel is a network traffic usage data collection package.
CloudStack can feed statistics from Traffic Sentinel into its own usage
records, providing a basis for billing users of cloud infrastructure.
Traffic Sentinel uses the traffic monitoring protocol sFlow. Routers
and switches generate sFlow records and provide them for collection by
Traffic Sentinel, then CloudStack queries the Traffic Sentinel database
to obtain this information
网络哨兵是一个收集网络流量使用数据的套件。CloudStack可以将流量哨兵中的信息统计到自己的使用记录中,为云基础架构的计费用户提供了依据。流量哨兵使用的流量监控协议为sFlow。路由器和交换机将生成的sFlow记录提供给流量哨兵，然后CloudStack通过查询流量哨兵的数据库来获取这些信息。

To construct the query, CloudStack determines what guest IPs were in use
during the current query interval. This includes both newly assigned IPs
and IPs that were assigned in a previous time period and continued to be
in use. CloudStack queries Traffic Sentinel for network statistics that
apply to these IPs during the time period they remained allocated in
CloudStack. The returned data is correlated with the customer account
that owned each IP and the timestamps when IPs were assigned and
released in order to create billable metering records in CloudStack.
When the Usage Server runs, it collects this data.
为了构建查询，CloudStack确定了当前来宾IP的查询时间间隔。这既包含新分配的IP地址，也包含之前已被分配并且在继续使用的IP地址。CloudStack查询流量哨兵，在分配的时间段内这些IP的网络统计信息。返回的数据是账户每个IP地址被分配和释放的时间戳，以便在CloudStack中为每个账户创建计费记录。当使用服务运行时便会收集这些数据

To set up the integration between CloudStack and Traffic Sentinel:
配置整合CloudStack和流量哨兵：

#. On your network infrastructure, install Traffic Sentinel and
   configure it to gather traffic data. For installation and
   configuration steps, see inMon documentation at
   `Traffic Sentinel Documentation <http://inmon.com.>`_.
   在网络基础设施中，安装和配置流量哨兵收集流量数据。 关于安装和配置步骤，请查阅inMon 流量哨兵文档.

#. In the Traffic Sentinel UI, configure Traffic Sentinel to accept
   script querying from guest users. CloudStack will be the guest user
   performing the remote queries to gather network usage for one or more
   IP addresses.
   在流量哨兵用户界面中，配置流量哨兵允许来宾用户使用脚本查询。CloudStack将通过执行远程查询为来宾用户的一个或多个IP和收集网络使用情况。

   Click File > Users > Access Control > Reports Query, then select
   Guest from the drop-down list.
   点击 File > Users > Access Control > Reports Query, 然后从下拉列表中选择来宾。

#. On CloudStack, add the Traffic Sentinel host by calling the
   CloudStack API command addTrafficMonitor. Pass in the URL of the
   Traffic Sentinel as protocol + host + port (optional); for example,
   http://10.147.28.100:8080. For the addTrafficMonitor command syntax,
   see the API Reference at `API Documentation
   <http://cloudstack.apache.org/docs/api/index.html>`_.
   在CloudStack中，使用API中的addTrafficMonitor命令添加流量哨兵主机。传入的流量哨兵URL类似于protocol + host + port (可选)；例如， http://10.147.28.100:8080。 关于addTrafficMonitor命令用法，请参阅API文档 API Documentation.

   For information about how to call the CloudStack API, see the
   Developer’s Guide at `CloudStack API Developer's Guide
   <http://docs.cloudstack.apache.org/en/latest/index.html#developers>`_.
      关于如何调用CloudStack API，请参阅 CloudStack API 开发指南。

#. Log in to the CloudStack UI as administrator.
 作为管理员登录到CloudStack用户界面。

#. Select Configuration from the Global Settings page, and set the
   following:
   在全局设置页面中，查找如下参数进行设置：

   direct.network.stats.interval: How often you want CloudStack to query
   Traffic Sentinel.
   direct.network.stats.interval: 你希望CloudStack多久收集一次流量数据。

## Setting Zone VLAN and Running VM Maximums
## 设置区域中VLAN和虚拟机的最大值

In the external networking case, every VM in a zone must have a unique
guest IP address. There are two variables that you need to consider in
determining how to configure CloudStack to support this: how many Zone
VLANs do you expect to have and how many VMs do you expect to have
running in the Zone at any one time.
在外部网络案例中，每个虚拟机都必须有独立的来宾IP地址。在CloudStack中有两个参数你需要考虑如何进行配置：在同一时刻，你希望区域中有多少VLAN和多少虚拟机在运行。

Use the following table to determine how to configure CloudStack for
your deployment.
使用如下表格来确定如何在CloudStack部署时修改配置。

| guest.vlan.bits |   Maximum Running VMs per Zone  |  Maximum Zone VLANs
|- |- |
| 12              |   4096                         |   4094
| 11              |   8192                         |   2048
| 10              |   16384                        |   1024
| 10              |   32768                        |   512

Based on your deployment's needs, choose the appropriate value of
guest.vlan.bits. Set it as described in Edit the Global Configuration
Settings (Optional) section and restart the Management Server.
基于部署需求，选择合适的guest.vlan.bits参数值。在全局配置中编辑该参数，并重启管理服务。
