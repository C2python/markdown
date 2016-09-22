#Ceph的安装#
--------------------------------------
准备工作：  
virtualBox；Centos7 64bit。  
拓扑：  
4节点拓扑。如下图所示：  
![topo](/img/ceph_min.png)  
采用虚拟机的安装方式。安装四台虚拟机。

##虚拟机的配置##
才有public network和cluster network分离的形式。其中public network的网络为192.168.34.0/24。clust network的子网为10.20.0.0/24。  
虚拟机采用3网卡的形式，eth0接入public network，eth1接入cluster network，eth2用来与外部通信。eth0，eth1采用内部网络模式，eth2设置成NAT。 

/在Centos 6中注意以下的配置/  
**虚拟机的克隆**    
克隆时需选择重新初始化MAC地址。  
在virtualbox中克隆虚拟机后，出现无法找到网卡的情况。此时，需要至`/etc/udev/rules.d/70-persistent-net.rules`中更改配置信息。文件中会有eth0, eth1, eth2, eth3，其中 eth0和eth1仍是被复制的虚拟机网卡信息，需要将此两行删除，将eth2,eth3更改为eth0,eth1。  
打开/etc/sysconfig/network-scripts/下的ifcfg-eth*(*代表0~n个网卡序号),把新的MAC修改进去。  
`/etc/sysconfig/network`修改主机名；  
`reboot`。  

/Centos 7/  
可以直接复制。不过需要选择重新初始化网卡地址。

**静态IP的配置**    
配置IP，如在`/etc/sysconfig/network-scripts/ifcfg-eth0`，配置参数如下：
>  
`DEVICE=eth0`  
`HWADDR=Mac地址`  
`TYPE=Ethernet`  
`BOOTPROTO=static`  
`IPADDR=192.168.34.X`
`NETMASK=255.255.255.0`  
`ONBOOT = yes`

`service network restart`  

**主机域名解析**  
配置文件`/etc/hosts`中（admin节点）：
>`192.168.34.3 mon`  
>`192.168.34.4 osd1`  
>`192.168.34.5 osd2`  

设置主机名(所有节点)：  
>`hostnamectl set-hostname {hostname}`  

**更改visudo**  
\>`sudo visudo`  
\>`Defaults requiretty`注释掉。  

##更改源##
将所有节点默认的源改成了科大的源：  
\>`mv CentOS-Base.repo CentOS-Base.repo.bk`  
\>`wget -o CentOS-Base.repo https://lug.ustc.edu.cn/wiki/_export/code/mirrors/help/centos?codeblock=3`  
\>`yum makecache`  

设置ceph.repo的源，间文件`/etc/yum.repos.d/ceph.repo`，添加以下内容：  
>`[ceph-noarch]`  
`name=Ceph noarch packages`  
`baseurl=http://download.ceph.com/rpm-firefly/el7/noarch`  
`enabled=1`  
`gpgcheck=1`  
`type=rpm-md`  
`gpgkey=https://download.ceph.com/keys/release.asc`   

安装ceph-deploy:`yum update && yum install ceph-deploy`  
**admin节点上的ceph-deploy安装完成**  
同时将所有节点中的selinux设为permissive。在文件`/etc/selinux/config`中。  

/Centos 6需要更改iptables的规则/
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

/CentOS 7在firewalld中/
直接关闭了此服务。

/以下都在admin节点上/ 
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

在admin节点创建目录my-cluster：
>`mkdir my-cluster`  
>`cd my-cluster`  

初始化：  
>`ceph-deploy new monnode`：初始化一个管理节点

此时目录下会有Ceph 配置文件、一个 monitor 密钥环和一个日志文件。

更改配置文件：  
由于只有两个OSD，所以要添加：`osd pool default size = 2`，使得集群正常；  
指定monitor的IP，公网IP(public network)。  
指定public及cluster的网络IP：  
>`public network = 192.168.34.0/24`  
>`cluster network = 10.20.0.0/24`  

安装ceph：
>`ceph-deploy install --stable firefly adminnode monnode osd1node osd2node`：指定安装firefly。

配置monitor，收集秘钥：  
>`ceph-deploy mon create-initial`

此时目录下会出现以下文件：  
>{cluster-name}.client.admin.keyring  
{cluster-name}.bootstrap-osd.keyring  
{cluster-name}.bootstrap-mds.keyring  
{cluster-name}.bootstrap-rgw.keyring（Hammer及以后版本才有，firefly没有此文件）

**添加OSD**  

>ssh node1  
sudo mkdir /home/osd1  
exit  
  
>ssh node2  
sudo mkdir /home/osd2  
exit  

准备及激活OSD：  
>`ceph-deploy osd prepare osd1node:/home/osd1 osd2node:/home/osd2`  
>`ceph-deploy osd activate osd1node:/home/osd1 osd2node:/home/osd2`  

在激活OSD那一步，如果空间过小，会报错没有可用空间。此时只要在运行一次就OK。

用 ceph-deploy 把配置文件和 admin 密钥拷贝到管理节点和 Ceph 节点：  
>`ceph-deploy admin adminnode monnode osd1node osd2node`  

确保对ceph.client.admin.keyring 有正确的操作权限。  
>`sudo chmod +r /etc/ceph/ceph.client.admin.keyring`

检查健康状态：  
>`ceph health`  

如果出现:health_warn: ***stuck unclean等信息，说明OSD数目不足，此时添加一个OSD即可。

##扩容：##





 