#Ceph的安装#
--------------------------------------
准备工作：  
virtualBox；Centos7 64bit。  
拓扑：  
4节点拓扑。如下图所示：  
![topo](/img/ceph_min.png)  
采用虚拟机的安装方式。安装四台虚拟机。

##虚拟机的配置##
网卡设置为网桥以及内部网络模式。  
为了能连接外网添加了网卡3，设置为NAT模式，同时需要配置`/etc/sysconfig/network-scripts/ifcfg-eth2`，配置参数如下：
>  
`DEVICE=eth2`  
`HWADDR=Mac地址`  
`TYPE=Ethernet`  
`BOOTPROTO=dhcp`

`service network restart`

**虚拟机的克隆**    
克隆时需选择重新初始化MAC地址。  
在virtualbox中克隆虚拟机后，出现无法找到网卡的情况。此时，需要至`/etc/udev/rules.d/70-persistent-net.rules`中更改配置信息。文件中会有eth0, eth1, eth2, eth3，其中 eth0和eth1仍是被复制的虚拟机网卡信息，需要将此两行删除，将eth2,eth3更改为eth0,eth1。  
打开/etc/sysconfig/network-scripts/下的ifcfg-eth*(*代表0~n个网卡序号),把新的MAC修改进去。  
`/etc/sysconfig/network`修改主机名；  
`reboot`。  
**静态IP的配置**  
在此文件中进行配置`/etc/sysconfig/network-scripts/ifcfg-eth[number]`：  
\>`BOOTPROTO=static IPADDR=IP_Num`  
\>`service network restart`  

**主机域名解析**  
配置文件`/etc/hosts`中（Centos7在hostname，admin节点）：
>`192.168.34.3 mon`  
>`192.168.34.4 osd1`  
>`192.168.34.5 osd2`  

**iptables设置**  
针对监视器，需要打开6789端口，同时删除拒绝除 SSH 之外的所有网卡的所有入栈连接的规则如下：  
`REJECT all -- anywhere anywhere reject-with icmp-host-prohibited`
删除此规则。
`sudo iptables -L -n --line-numbers`  
`suod iptables -D {FORWARD}(根据情况对应的组)  {Num}`  
`sudo iptables -A INPUT -i eth0 -p tcp -s 192.168.34.0/24 --dport 6789 -j ACCEPT`  
`service iptables save`  

针对OSD进程，需要对公共网和集群网打开6800至7300端口：
同样需要删除一些规则，如上。  
`sudo iptables -A INPUT -i eth0  -m multiport -p tcp -s 192.168.34.0/24 --dports 6800:7300 -j ACCEPT`  
`sudo iptables -A INPUT -i eth1  -m multiport -p tcp -s 10.20.0.0/24 --dports 6800:7300 -j ACCEPT`  
`service iptables save`  
 
##配置ssh##
admin节点上：  
\>`ssh-keygen`  
\>`ssh-copy-id {node}`  
此时可以免密登录。同时在.ssh文件下配置config文件。加入如下：  
>
Host mon  
   Hostname mon  
   User zhang  
Host osd1  
   Hostname osd1  
   User zhang  
Host osd2  
   Hostname osd2  
   User zhang  

同时要更改config权限，才能正确执行：  
\>`chmod 600 /home/ceph/.ssh/config` <<-------设置config文件的权限，只有600才能正确的执行。

安装时指定版本：
`ceph-deploy install --stable firefly node`