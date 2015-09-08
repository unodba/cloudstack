本文将根据实例介绍高级网络架构, 并依次介绍高级网络的各个功能模块: DHCP, DNS, 源NAT, 端口转发, 防火墙, 静态NAT, 负载均衡。

# 网络架构情况如下
CloudStack里面物理网卡1对应XenServer上流量标签为Management的Bond 0 + 3, 用于管理。
![advanced network](images/CloudStack-Advanced-Network-Zone-view-1.png)
CloudStack里面物理网卡2对应XenServer上流量标签为Trunk的Bond 1 + 4, 用于公共和来宾。
![advanced network](images/CloudStack-Advanced-Network-Zone-view-2.png)
CloudStack里面物理网卡3对应XenServer上流量标签为Management的Bond 2 + 5, 用于存储。
![advanced network](images/CloudStack-Advanced-Network-Zone-view-3.png)

使用XenServer的管理控制台XenCenter查看网络情况，VLAN ID 为91的网络是CloudStack创建的公共网络，VLAN ID 301 – 500 之间的网络是CloudStack根据需求自动创建的来宾网络。管理和存储如前文所述不需要设置VLAN ID，所以这里CloudStack也不会创建。

# 高级网络架构
![advanced network](images/CloudStack-Advanced-Network-Zone-view-5.png)
我们使用其中一个帐户建立一个高级网络环境，举例说明CloudStack高级网络的工作方式。
使用帐户shennan下同名用户shennan创建1高级网络：
![advanced network](images/CloudStack-Advanced-Network-Zone-view-6.png)
- 网络名称: SN-Network01,
- 所属帐户: shennan,
- 类型: isolate
- VLAN: 462
- 来宾网络CIDR: 192.168.100.0/24
- VLAN说明: VLAN ID462是系统自动分配的，使用此网络的VM数据包都会打上vlan为462的标签， 所连接交换机上必须已配置有此vlan标签才能保证VM网络的正常通信。

> 注：
CloudStack会根据使用需求自动创建相应VLAN ID的网络。
本文例子中管理网络使用单独网卡, 连接到交换机Access vlan 101类型端口。
Public 网络和Guest网络使用同一块网卡, 连接到交换机的Trunk类型端口。Public网络设置VLAN ID为91。
Storage 网络使用单独网卡, 连接到交换机Access vlan 12 类型端口。

管理网络和Storage网络因为是access类型所以不需要在CloudStack里面配置VLAN ID。
Public网络设置VLAN ID为91, Guest网络设置的VLAN ID为301 – 500。

系统虚拟机启动需要使用管理网络, public网络和存储网络, 因为只有Public 网络连接为Trunk并在CloudStack里面设置了VLAN为91。所以CloudStack只需要创建Public网络即可。Guest网络将会再用到时自动创建。
![advanced network](images/CloudStack-Advanced-Network-Zone-view-7.png)
启用SN-Network01此网络时, CloudStack会自动新建VLAN ID为462的Guest网络。
![advanced network](images/CloudStack-Advanced-Network-Zone-view-8.png)
CloudStack并为SN-Network01此网络自动创建一专用的虚拟路由器. 虚拟路由器配置2块网卡, 网卡一端连接Public网络，另一端连接到隔离的来宾网络.
![advanced network](images/CloudStack-Advanced-Network-Zone-view-9.png)
使用此网络的VM将会配置一块网卡连接到VLAN ID为462的来宾网络。并且会由本网络的虚拟路由器即r-24-VM的DHCP服务自动分配1个IP地址。
![advanced network](images/CloudStack-Advanced-Network-Zone-view-10.png)

# 高级网络的各个功能模块

- DHCP
- DNS
- 源NAT
- 端口转发
- 防火墙
- 静态NAT
- 负载均衡

使用上面环境在用户shennan的高级网络SN-Network01下建立3台试验虚拟机。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-1.png)

# DHCP
虚拟路由器被设置为DHCP服务器为虚拟网络内部VM自动分配IP地址，前面文章已有图示介绍。

# DNS
虚拟路由器被设置为内部DNS服务器为虚拟网络内部VM进行域名解析，也可以直接使用外部DNS。

# 源NAT

新建网络会自动配置源NAT映射

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-2.png)

源NAT 用于将VM的内部IP映射成外部IP地址，用于VM访问外部网络。外部网络计算机无法直接访问VM的内部IP。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-3.png)

# 端口转发

外部网络的计算机需要访问内部网络的VM，需要手动配置端口转发规则。

例：将虚拟路由器外部IP：111.111.101.109的8081端口影射到内部网络VM1的80端口。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-4.png)
![advanced network](images/CloudStack-Advanced-Network-Zone-feature-5.png)
配置端口转发规则

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-6.png)
 设置专用端口范围 80 – 80，公用端口范围8081 – 8081，选择协议TCP。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-7.png)
 选择Win2003-NLB01虚拟机。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-8.png) 重复添加好3条映射规则
![advanced network](images/CloudStack-Advanced-Network-Zone-feature-9.png)
此时外部计算机还无法访问，还需要配置对应的防火墙规则。

# 防火墙

刚才设置的端口转发还无法访问，因为需要进行防火墙规则配置。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-10.png)
配置防火墙规则

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-11.png)
配置规则允许端口8081 – 8083 的访问

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-12.png)
添加成功后测试Http访问。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-13.png)


# 静态NAT

静态NAT是建立内部网络IP到外部网络IP的双向映射。因此使用静态NAT后外部计算机可以直接访问内部网络VM。

每条规则对应一VM实例，并且每条静态NAT映射规则需独占1外网IP地址。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-14.png)
![advanced network](images/CloudStack-Advanced-Network-Zone-feature-15.png)
因为静态NAT需要独占外网IP，首先需要为虚拟路由器申请3个新的外网IP。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-16.png)
重复3次申请3个新外网IP

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-17.png)
设置静态NAT映射

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-18.png)
选择Win2003-NLB01

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-19.png)
完成映射后，修改防火前规则，允许80端口访问。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-20.png)
![advanced network](images/CloudStack-Advanced-Network-Zone-feature-21.png)
重复以上操作，完成3条规则配置。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-22.png)
使用外网IP直接访问VM测试。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-23.png)


# 负载均衡

负载均衡规则

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-24.png)
![advanced network](images/CloudStack-Advanced-Network-Zone-feature-25.png)
设置负载均衡规则

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-26.png)

选择均衡算法:
（算法选择工作原理请参考专业负载均衡设备文档，如果有需要的NetScaler文档的朋友可以给我发邮件索取）

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-27.png)


设置粘性，在NetScaler设备上也称为会话持久性策略，一般用于带cookies认证Web访问。

- None: 不使用会话持久性策略.
- SorceBased: 根据客户端的IP地址, 记录访问分配.
- AppCookie: 根据数据包中服务器应用Cookie, 记录访问分配
- LBCookie: 将会有虚拟路由器向数据保证插入Cookie, 记录访问分配

一般前2种比较常用. 这里我们先择SorceBased方式。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-28.png)
![advanced network](images/CloudStack-Advanced-Network-Zone-feature-29.png)
AutoScale : 自动扩展内容较多, 并且可能需要额外的配置工具。会单独写篇文章进行讨论。

创建好后，设置防火墙规则。就可以通过浏览器进行访问测试了。我们强行关闭任意1-2台，访问会自动故障转移到其他Web Server。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-30.png)

# VPN

CloudStack帐户所有者可以创建虚拟专用网络（VPN）来访问他们的虚拟机。如果来宾网络是从提供远程访问VPN服务中实例化产生的，虚拟路由器（基于系统虚拟机）可以用于提供该服务。CloudStack为来宾虚拟网络提供L2TP-over-IPsec-based远程访问VPN服务。由于每个网络获取自己的虚拟路由器，因此VPN不能跨网络共享。Windows、Mac OS X和iOS的自身VPN客户端可用于连接客户网络。帐户的所有者可以对其用户的VPN进行创建和管理。为达此目的，CloudStack不使用其帐户数据库，而使用单独的表。VPN用户数据库之间共享帐户所有者创建的所有VPN。所有VPN用户可以访问所有帐户所有者创建的VPN。

可以在全局设置中自定义VPN相关参数。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-34.png)
使用VPN

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-31.png)
启用后获得共享密钥，创建VPN用户：shennan。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-32.png)
创建VPN连接，之后进行测试。外网计算机VPN拨号后，获得内网地址可以直接访问VM内部网络地址。

![advanced network](images/CloudStack-Advanced-Network-Zone-feature-33.png)
以上就是关于CloudStack高级网络功能应用的介绍。
