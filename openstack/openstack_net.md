#Openstack网络架构#
-----------------------------------------
网桥：隔离广播域，2层交换机。  

compute、Neutron节点的网络架构图。  
![](https://github.com/C2python/opentsack-conf/blob/master/openstack/img/1node_net.jpg)  

Compute 节点上由Neutron-OVS-Agent负责：
>
br-int：每个虚机都通过一个Linux brige连到该OVS桥上  
br-tun：转化网络packet中的VLAN ID 和 Tunnel ID  
GRE tunnel：虚拟GRE通道  

Neutron节点上：
>  
br-tun/br-int：同Compute节点，由Neutron-OVS-Agent负责  
br-ex：连接物理网卡，用于和外网通信  
Network namespace：用于tenant 网络DHCP服务的qDHCP由Neutron-DHCP-Agent负责，和用于网络间routing的qRouter由Neutron-L3-Agent负责  

open vswitch、 networkspaces、Linux Bridge、veth pairs。  
Open vSwitch(OVS)：  
用于连接虚拟机和物理网口。VS包含bridages和ports，OVS bridges不同于与linux bridge（使用brctl命令创建）。从上图中我们可以看到OVS包含两个网桥，分别是br-int和br-tun。其中虚拟机的虚拟网口与主机TAP设备相连，TAP再通过一个网桥与br-int的网口相连。br-tun连接GRE隧道一端。TAP设备之所以需要通过网桥再与br-int相连，是因为OVS不支持iptables。iptables只能用于linux bridage而非OVS bridage。  
br-int与br-tun通过虚拟网线相连。

##Network Namespaces(netns)##
网络namespace是拥有独立的网络配置隔离容器，并且该网络不能被其他名字空间看到。  
定义一个新的namespace，命令如下：
\>`sudo ip netns add pri-net`
\>`ip netns list`  

namespace是一个独立的网路空间，可以在其中进行各种网络配置命令而不影响其他用户。命令格式如下：  
\>`ip netns exex pri-net ifconfig`  



##Linux网络虚拟化  
  
tun/tap驱动程序实现了虚拟网卡的功能，tun表示虚拟的是点对点设备，tap表示虚拟的是以太网设备，这两种设备针对网络包实施不同的封装。利用tun/tap驱动，可以将tcp/ip协议栈处理好的网络分包传给任何一个使用tun/tap驱动的进程，由进程重新处理后再发到物理链路中。  

利用ip netns [options]工具实现linux下网络的虚拟化。  

创建虚拟网络环境：  
\>`sudo ip netns add net0`：创建了一个完全隔离的新网络环境，这个环境包括独立的网卡空间，路由表，ARP表，IP地址表，iptables，etables，等。与网络相关的组件都是独立出来。  

\>`ip netns list`：列出已有的网络环境。  

进入虚拟网络环境进行处理。  
使用如下命令：  
\>`ip netns exec net0 [command]`  
\>`ip netns exec net0 bash`：打开一个新的shell，进入net0的环境中运行  

连接两个网络环境：  
\>`ip netns add net1`  
创建了另一个网络环境net1。  
\>`ip link add type veth`：创建了一对虚拟网卡veth0和veth1，相互连接，可以相互通信，如同一个双向的管道。

\>`ip link set veth0 netns net0`：将网卡veth0移动到net0环境里，对其他网络环境不可见。  
\>`ip link set veth1 netns net1`：veth1移动到net1环境里。

设置veth0和veth1的IP，连通性测试。  
\>`ip netns exec net0 ip link set veth0 up`；  
\>`ip netns exec net1 ip link set veth1 up`；  
\>`ip netns exec net0 ip address add 10.0.1.1/24 dev veth0`；设置veth0的IP  
\>`ip netns exec net1 ip address add 10.0.1.2/24 dev veth1`；设置veth1的IP

\>`ip netns exec net0 ping -c 10.0.1.2`；连通性测试。

**net0，net1连接至网桥**  
>
>`ip netns add net0`  
>`ip netns add net1`  
>`ip netns add bridge`  
>`ip link add type veth`  
>`ip link set dev veth0 name net0-bridge netns net0`：将网卡veth0加入net0网络环境，重命名为net0-bridge。
>`ip link set dev veth1 name bridge-net0 netns bridge`：将网卡veth1加入bridge网络环境，重命名为bridge-net0。
>`ip link add type veth`  
>`ip link set dev veth0 name net1-bridge netns net1`：将网卡veth0加入net1网络环境，重命名为net1-bridge。
>`ip link set dev veth1 name bridge-net1 netns bridge`：将网卡veth1加入bridge网络环境，重命名为bridge-net1。

在bridge网络环境中创建网桥：    
\>`ip netns exec bridge brctl addbr br`：在bridge网络环境中创建网桥br。
\>`ip netns exec bridge ip link set dev br up`：开启网桥。
\>`ip netns exec bridge ip link set dev bridge-net0 up`：开启端口。
\>`ip netns exec bridge ip link set dev bridge-net1 up`：
\>`ip netns exec bridge brctl addif br bridge-net0`：将端口bridge-net0连接至网桥br。
\>`ip netns exec bridge brctl addif br bridge-net0`

配置虚拟环境中的虚拟网卡：  
\>`ip netns exec net0 ip link set dev net0-bridge up`  
\>`ip netns exec net0 ip address add 10.0.1.1/24 dev net0-bridge`  
\>`ip netns exec net1 ip link set dev net1-bridge up`  
\>`ip netns exec net1 ip address add 10.0.1.2/24 dev net1-bridge`  

测试连通性：  
\>`ip netns exec net0 ping -c 3 10.0.1.2`  

如果要查看网卡的连接情况，可以利用lldp工具。在一个虚拟环境中开启一个lldp demaon，执行:`lldpcli show neighbors`。  




##TUN/TAP MACVLAN MACVTAP##
TUN设备是一种虚拟网络设备，通过该此设备，程序可以方便的模拟网络行为。物理设备的工作模式如下：  
![](https://github.com/C2python/opentsack-conf/blob/master/openstack/img/phy.png)  
所有物理网卡收到的包会交给内核的 Network Stack 处理，然后通过 Socket API 通知给用户程序。  
TUN设备的工作模式如下：  
![](https://github.com/C2python/opentsack-conf/blob/master/openstack/img/tun.png)  

普通的网卡，通过网卡收发数据包，TUN设备通过文件收发数据包。所有对这个文件的写操作会通过 TUN 设备转换成一个数据包送给内核；当内核发送一个包给 TUN 设备时，通过读这个文件可以拿到包的内容。  

利用TUN设备搭建UDP VPN，处理流程如下所示：  
![](https://github.com/C2python/opentsack-conf/blob/master/openstack/img/udp_tun.png)  

**TAP设备：**  
TAP设备与TUN设备的工作方式相同，区别在于：  
TUN设备的/dev/tunX文件收发的是IP层数据包，只能工作在IP层，无法与物理网卡做bridge，但可以通过三层设备与物理网卡连通。  
TAP设备的/dev/tapX文件收发的是MAC层数据包，拥有MAC层功能，可以与物理网卡做bridge，可以广播。

**MACVLAN**  
一个网卡绑定多个MAC地址，工作方式如下：  
![](https://github.com/C2python/opentsack-conf/blob/master/openstack/img/macvlan.png)  

MACVLAN会根据收到包的目的MAC地址判断这个包需要交给哪个网卡，配合namespace使用，构建如下网络：  
![](https://github.com/C2python/opentsack-conf/blob/master/openstack/img/nsmacvlan.png)  

由于 macvlan 与 eth0 处于不同的 namespace，拥有不同的 network stack，这样使用可以不需要建立 bridge 在 virtual namespace 里面使用网络。  

**MACVTAP**  
对MACVLAN的改进，综合MACVLAN与TAP设备，使用MACVLAN的方式收发数据包，收到的数据包交给/dev/tapX。工作方式如下：  
![](https://github.com/C2python/opentsack-conf/blob/master/openstack/img/macvtap.png)  

由于 MACVLAN 是工作在 MAC 层的，所以 MACVTAP 也只能工作在 MAC 层，不会有 MACVTUN 这样的设备