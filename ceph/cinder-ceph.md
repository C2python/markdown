#Cinder架构、Ceph#
-----------------------------
在openstack中，Cinder为虚拟机提供永久存储。块存储提供了管理卷的架构，以及与计算节点交互，为实例提供卷。同时，块存储服务支持卷的快照以及卷的类型等管理。  
![](/img/cinder-arc.jpg)  
块存储服务由以下的组件组成：  
**cinder-api**  
接收API请求，将其转至`cinder-volume`模块以执行相应的动作。  
**cinder-volume**  
直接与存储设备交互。与`cinder-scheduler`进程等进程通过消息队列交互。`cinder-volume`响应发往块存储设备的读写请求并维护相应的状态。并且可以通过驱动支持不同的存储设备。  
**cinder-scheduler daemon**  
选择最优的存储节点来创建卷。功能类似`nova-scheduler`。  
**cinder-backup daemon**  
`cinder-backup`服务可以为任意类型的卷提供备份服务。同样的，可以通过驱动支持不同的存储服务提供商。    
**Messaging queue**  
块存储不同模块间的通信工具。 
 
##cinder的安装和配置##
**控制节点上的配置**  
在安装和配置块存储服务之前，需要创建数据库、服务凭证，以及API终端。  
>mysql -u root -p  
创建数据库：  
>create database cinder  
授权：  
>GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';    
>GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';  

`CINDER_DBPASS`替换成自己的密码。  

启动admin凭证，接入admin的CLI：  
>. admin-openrc  

添加用户，配置admin角色：  
创建cinder用户：  
>openstack user create --domain default --password-prompt cinder  

为cinder用户添加admin角色：  
>openstack role add --project service --user cinder admin  

创建cinder和cinderv2两个服务实体：  
>openstack service create --name cinder --description "OpenStack Block Storage" volume  
>openstack service create --name cinderv2 --description "OpenStack Block Storage" volume  

创建块存储服务的API终端：  
>openstack endpoint create --region RegionOne volume public http://controller:8776/v1/%\(tenant\_id\)s  
>openstack endpoint create --region RegionOne volume internal http://controller:8776/v1/%\(tenant\_id\)s  
>openstack endpoint create --region RegionOne volume admin http://controller:8776/v1/%\(tenant\_id\)s

volumev2：  
>openstack endpoint create --region RegionOne volumev2 public http://controller:8776/v1/%\(tenant\_id\)s  
>openstack endpoint create --region RegionOne volumev2 internal http://controller:8776/v1/%\(tenant\_id\)s  
>openstack endpoint create --region RegionOne volumev2 admin http://controller:8776/v1/%\(tenant\_id\)s


**安装配置相应的组件**  
安装：  
>yum install openstack-cinder  

编辑/etc/cinder/cinder.conf配置文件：  
在数据库部分，配置接入：  
>[database]  
...  
connection = mysql+pymysql://cinder:CINDER\_DBPASS@controller/cinder

将CINDER\_DBPASS替换成自己的密码。  

在[DEFAULT]和[oslo\_messaging\_rabbit]部分，配置RabbitMQ消息队列的接入：  
>[DEFAULT]  
...  
rpc\_backend = rabbit  
> 
[oslo\_messaging\_rabbit]  
...  
rabbit\_host = controller  
rabbit\_userid = openstack  
rabbit\_password = RABBIT\_PASS  

将RABBIT\_PASS替换成在RabbitMQ中为opestack配置的账户密码。  

在[DEFAULT]和[keystone\_authtoken]部分，配置认证服务：  
>
[DEFAULT]  
...  
auth\_strategy = keystone  
>
[keystone\_authtoken]  
...  
auth\_uri = http://controller:5000  
auth\_url = http://controller:35357  
memcached\_servers = controller:11211  
auth_type = password  
project\_domain\_name = default  
user\_domain\_name = default  
project\_name = service  
username = cinder  
password = CINDER\_PASS  

将CINDER\_PASS替换成cinder用户的密码。  
>
将[keystone\_authtoken]其余部分注释掉  

在[DEFAULT]部分添加控制节点管理接口的ip地址：  
>[DEFAULT]  
...  
my_ip = 10.0.0.11  

在[oslo\_concurrency]部分配置如下：  
>[oslo\_concurrency]  
...  
lock_path = /var/lib/cinder/tmp  
 
初始化数据库：  
>su -s /bin/sh -c "cinder-manage db sync" cinder  

配置计算节点使用块存储：  
配置`/ect/nova/nova.conf`文件：  
>[cinder]  
os\_region\_name = RegionOne  

重启服务：  
>systemctl restart openstack-nova-api.service  

重启块存储相关服务，开机启动：  
>systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service  
>systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service

###存储节点的安装配置###
安装相应的包：  
>yum install openstack-cinder targetcli python-keystone  

编辑`/etc/cinder/cinder.conf`文件：  
数据库配置：  
>[database]  
...  
connection = mysql+pymysql://cinder:CINDER\_DBPASS@controller/cinder  

CINDER\_DBPASS替换自己的密码。  

在[DEFAULT]和[oslo\_messaging\_rabbit]部分，配置RabbitMQ消息队列的接入：  
>[DEFAULT]  
...
rpc\_backend = rabbit  

[oslo\_messaging\_rabbit]  
...  
rabbit\_host = controller  
rabbit\_userid = openstack  
rabbit\_password = RABBIT\_PASS  

RABBIT\_PASS替换成openstack的账户密码。  

在[DEFAULT]和[keystone\_authtoken]部分，配置认证服务信息：  
>[DEFAULT]  
...  
auth_strategy = keystone  
>  
[keystone_authtoken]    
...  
auth\_uri = http://controller:5000  
auth\_url = http://controller:35357  
memcached\_servers = controller:11211  
auth\_type = password  
project\_domain\_name = default  
user\_domain\_name = default  
project\_name = service  
username = cinder  
password = CINDER\_PASS  

[DEFAULT]部分配置存储节点管理接口的ip
>[DEFAULT]  
...  
my\_ip = MANAGEMENT\_INTERFACE\_IP\_ADDRESS  

同时，若需要指定对应的存储服务，仍需要进行相应的配置。在后面会详细描述ceph作为后端存储的配置方法。  

[DEFAULT]部分配置镜像服务API的位置：  
>[DEFAULT]  
...  
glance\_api\_servers = http://controller:9292  

配置[oslo\_concurrency]部分：  
>[oslo\_concurrency]  
...  
lock\_path = /var/lib/cinder/tmp  

完成安装，启动服务：  
> systemctl enable openstack-cinder-volume.service target.service  
systemctl start openstack-cinder-volume.service target.service  

#ceph+openstack的整合#
-----------------------------------------

利用Ceph存储openstack镜像需要通过`libvirt`。`libvirt`配置QEMU至librbd的接口，实现条带化存储。  

要在openstack中使用ceph的块存储，需要安装QEMU，libvirt。  
将利用ceph存储openstack三个部分：  

+ Images：Openstack glance管理VMs的镜像。
+ Volumes：Volumes是块设备。Openstack可以利用volumes来启动VMs，或者将volume挂载到VMs。Openstack cinder管理卷。  
+ Guest Disk：为客户操作系统的硬盘。在Havana版本之前，在ceph中启动VM只能通过cinder的boot-from-volume函数启动。Havana版本之后可以直接在通过ceph启动。但是ceph只支持RAW格式的虚拟机硬盘，若要绕过cinder直接从ceph启动虚拟机，Glance image只能是RAW格式的。  

**CREATE POOL**  
创建四种不同的存储池。
>ceph osd pool create volumes 128  
ceph osd pool create images 128  
ceph osd pool create backups 128  
ceph osd pool create vms 128  

**配置openstack ceph 客户端**  
所有的客户端节点都需要ceph.conf的配置文件。将ceph.conf配置文件拷贝至运行有`glance-api`、`cinder-volume`、`nova-compute`、`cinder-backup`的节点，这些节点也是接入ceph的客户端。其中`glance-api`运行在controller节点上，`cinder-volume`运行在每个存储节点上、`nova-compute`运行在每个计算节点上。`cinder-backup`属于cinder的服务。  
>ssh {client-node} sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf  

安装ceph-client包：  
在glance-api节点上，安装librbd的python接口：  
>sudo yun install python-rbd  

在nova-compute，cinder-backup，cinder-volume节点上，安装python接口和命令行工具：  
>sudo yum install ceph-common  

如果开启了ceph的认证体系（cephx），需要为Nova/Cinder和Glance创建新用户：  
>ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object\_prefix rbd\_children, allow rwx pool=volumes, allow rwx pool=vms, allow rx pool=images'  
ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object\_prefix rbd\_children, allow rwx pool=images'  
ceph auth get-or-create client.cinder-backup mon 'allow r' osd 'allow class-read object\_prefix rbd\_children, allow rwx pool=backups'  

将用户client.cinder、client.glance、client.cinder-backup的kering添加至客户端节点，并改变所有权：  
>ceph auth get-or-create client.glance | ssh {your-glance-api-server} sudo tee /etc/ceph/ceph.client.glance.keyring  
ssh {your-glance-api-server} sudo chown glance:glance /etc/ceph/ceph.client.glance.keyring  
ceph auth get-or-create client.cinder | ssh {your-volume-server} sudo tee /etc/ceph/ceph.client.cinder.keyring  
ssh {your-cinder-volume-server} sudo chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring  
ceph auth get-or-create client.cinder-backup | ssh {your-cinder-backup-server} sudo tee /etc/ceph/ceph.client.cinder-backup.keyring  
ssh {your-cinder-backup-server} sudo chown cinder:cinder /etc/ceph/ceph.client.cinder-backup.keyring  

在运行有nova-compute的节点需要keyring文件：  
>ceph auth get-or-create client.cinder | ssh {your-nova-compute-server} sudo tee /etc/ceph/ceph.client.cinder.keyring  

由于libvirt从cinder挂载块设备至虚拟机时，需要接入ceph，因此，libvirt也需要client.cinder的keyring。  
首先在计算节点上生成一个密钥的临时拷贝：  
>ceph auth get-key client.cinder | ssh {your-compute-node} tee client.cinder.key  

然后，在计算节点上，为libvirt添加密钥，删除临时拷贝：  
>`uuidgen`  
`457eb676-33da-42ec-9a8c-9293d545c337`  
>  
`cat > secret.xml <<EOF`  
`<secret ephemeral='no' private='no'>`  
  `<uuid>457eb676-33da-42ec-9a8c-9293d545c337</uuid>`  
  `<usage type='ceph'>`  
    `<name>client.cinder secret</name>`  
  `</usage>`  
`</secret>`  
`EOF`  
`sudo virsh secret-define --file secret.xml`  
`Secret 457eb676-33da-42ec-9a8c-9293d545c337 created`  
`sudo virsh secret-set-value --secret   457eb676-33da-42ec-9a8c-9293d545c337 --base64 $(cat client.cinder.key) && rm client.cinder.key secret.xml`

配置openstack使用ceph：  
配置Glance：  
在/etc/glance/glance-api.conf文件中配置如下：  
>[DEFAULT]  
...  
default_store = rbd  
...  
[glance_store]  
stores = rbd  
rbd_store_pool = images  
rbd_store_user = glance  
rbd_store_ceph_conf = /etc/ceph/ceph.conf  
rbd_store_chunk_size = 8    


