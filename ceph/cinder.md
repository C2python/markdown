#Openstack Cinder存储管理#
----------------------------------------
Cinder的核心是对卷的管理，允许对卷、卷的类型、卷的快照进行处理。本身并没有实现对块设备的管理是实际服务，而是通过加载插件的方式来支持不同的存储后端，并向用于提供一个统一的存储接口。
  
- Cinder提供 REST API 使用户能够查询和管理 volume、volume snapshot 以及 volume type
- 提供 scheduler 调度 volume 创建请求，合理优化存储资源的分配
- Cinder结合不同的存储驱动，支持不同的存储设备，对外提供统一的存储接口

##Openctack Cinder调度流程##
Cinder架构：  
![cinder](/img/cinder.png)

![](/img/cinder-arc.jpg)  

- API service：负责接受和处理 Rest 请求，并将请求放入 RabbitMQ队列。  
- Scheduler service: 处理任务队列的任务，并根据预定策略选择合适的 Volume Service 节点来执行任务。目前版本的 cinder 仅仅提供了一个 Simple Scheduler, 该调度器选择卷数量最少的一个活跃节点来创建卷。  
- Volume service: 该服务运行在存储节点上，管理存储空间。每个存储节点都有一个 Volume Service，若干个这样的存储节点联合起来可以构成一个存储资源池。
###1. 使用###
cinder与各节点相关的模块图：  
![cinder-module](/img/cindermodule.jpg)

Cinder的服务部署在两类节点上，分别是controller节点和compute节点。在控制节点上部署有`cinder-scheduler、cinder-api、cinder-volume`三类服务。其中部署有`cinder-volume`服务说明此节点即作为控制节点也作为存储节点。  

创建卷的流程图：  
![cindercreate](/img/cindercreate.jpg)  
> 1. 首先client通过rest接口向API发送创建卷的请求；
> 2. API解析请求，并将创建volume的消息放到RabbitMQ中；
> 3. scheduler通过RabbitMQ获得创建卷的消息，执行调度算法，筛选出最合适的节点创建卷；
> 4. scheduler向RabbitMQ发送消息，"节点X创建卷"；
> 5. 节点X中的cinder volume获得此消息后，通过相应的后端存储设备创建卷。

**Create Volume：**  
文件cinder/cinder/scheduler/*下存储了各种调度算法文件。经过 AvailabilityZoneFilter, CapacityFilter, CapabilitiesFilter 和 CapacityWeigher 的层层筛选。发送消息，发送消息的函数在/cinder/cinder/scheduler/filter_scheduler.py，方法为schedule_create_volume，`self.volume_rpcapi.create_volume()`。  

**Attach Volume：**  
创建volume之后，需要将volume attach到对应的instance上去。而计算节点与存储节点通常不在一个物理机上，因此，openstack利用iSCSI实现存储节点与计算节点的连接。如图所示。  
![cinderattach](/img/cinderattachvolume.jpg)  
iSCSI是client-server架构，分别对应initiator和target。  
**target**  
提供iSCSI存储资源的设备，简单的说，就是iSCSI服务器。  
**initiator**  
使用存储资源的设备，即是iSCSI客户端。  
initiator需要与target建立iSCSI连接，执行login操作，然后就可以使用target上面的块存储设备。Cinder存储节点cinder-volume默认使用tgt软件来管理和监控iSCSI target，在计算节点中使用iscsiadm执行initiator相关操作。  

Attach操作的流程如图所示：  
![cinderattach](/img/cinderattach.jpg)  
>
1. 向cinder-api发送attach请求  
2. cinder-api向RabbitMQ发送消息  
3. cinder-volume初始化volume的连接  
4. nova-compute将volume attach到instance  
  
###步骤详解###
1. 向cinder-api发送attach请求  
client向cinder-api发送attach的请求。这些请求attach在gui管理面板中。  
  - **初始化 volume 的连接** Volume 创建后，只是在 volume provider 中创建了相应存储对象（比如 LV），这时计算节点是无法使用的。Cinder-volume 需要以某种方式将 volume export 出来，计算节点才能够访问得到。这个 export 的过程就是“初始化 volume 的连接”。   
  下面是 cinder-api 的日志文件 /opt/stack/logs/c-api.log 中记录的相关信息Initialize_connection 的具体工作主要由 cinder-volume 完成。
 ![cinderconnect](/img/cinderconnect.jpg)
  - **Attach volume** 初始化 volume 连接后，计算节点将 volume 挂载到指定的 instance，完成 attach 操作。下面是 cinder-api 的日志文件 /opt/stack/logs/c-api.log 中记录的相关信息。
  ![cinderconnected](/img/cinderconnected.jpg)

Attach 的具体工作主要由 nova-compute 完成。cinder-api以及日志都在controller节点中。  

**cinder-api发送消息**  
cinder-api 分两步完成 attach 操作，所以对应地会先后向 RabbitMQ 发送了两条消息：  
1. **初始化 volume 的连接** cinder-api 没有打印发送消息的日志，只能通过源代码查看 /opt/stack/cinder/cinder/volume/api.py，方法为 initialize_connection。  
   ![cinderconnectinitialize](/img/cinderconnectinitialize.jpg)  
2. **Attach volume** cinder-api 通过源代码查看
![](/img/cinderattachcode.jpg)  


###cinder-volume初始化volume的连接###
cinder-volume接收到initialize_connection消息后，会通过tgt创建target，并将volume所对应的LV通过target export出来。日志为/opt/stack/logs/c-vol.log。如图：  
![](/img/cindertarget.jpg)  

下面的日志显示：通过命令 tgtadm --lld iscsi --op show --mode target 看到已经将 1GB（1074MB）的 LV /dev/stack-volumes-lvmdriver-1/volume-1e7f6bd7-ce11-4a73-b95e-aabd65a5b188 通过 Target 1 export 出来了。  
![](/img/cindertargetvolume.jpg)

Initialize connection完成：  
![](/img/cinderinitializeconnection.jpg)  

###nova-compute将volume attach到instance###
计算节点作为iSCSI initiator访问存储节点iSCSI Target上的volume，并将其attach到instance。日志文件/opt/stack/logs/n-cpu.log。  
![](/img/cindernovaattach.jpg)  

Nova-compute一次执行iscsiadm的new,update,login,rescan操作访问target上的volume。  
然后通过更新 instance 的 XML 配置文件将 volume 映射给 instance。  
![](/img/cinderattachxml.jpg)  
更新后的xml文件，通过`virsh edit`：  
![](/img/cinderxml.jpg)  
可以看到，instance 增加了一个类型为 block 的虚拟磁盘，source 就是要 attach 的 volume，该虚拟磁盘的设备名为 vdb。  

在存储节点上运行执行 tgt-admin --show --mode target，会看到计算节点作为 initiator 已经连接到 target 1。  
![](/img/cindercomputeconnection.jpg)


###Detach流程###
Detach的操作流程如下图所示：  
![](/img/cinderdetach.jpg)  
1. 向cinder-api发送detach请求  
2. cinde-api发送消息  
3. nova-compute detach volume  
4. cinder-volume删除target  

cinder-api将收到detach volume的请求。日志文件在/opt/stack/logs/c-api.log。如下图：  
![](/img/cinderdetachrequest.jpg)  

###cinder-api发送消息###
cinder-api发送detach消息。发送消息的源代码在/cinder/cinder/volume/api.py，方法为detach：  
![](/img/cinderdetachcode.jpg)   

Detach的操作由Nova-compute和cinder-volume共同完成：  
1、首先nova-compute将volume从instance上detach，然后断开与iSCSI target的连接；  
2、最后cinder-volume删除volume相关的iSCSI target。

nova-compute上的操作主要有如下几个步骤：  
1. 将缓存中的数据Flush到volume。
![](/img/cinderdetachflush.jpg)
2. 删除计算节点上volume对应的SCSI设备。
![](/img/cinderdetachdelscsi.jpg)
3. 通过iscsiadm的logout，delete操作断开与iSCSI target的连接。
![](/img/cinderdetachdeltarget.jpg)

###cinder-volume删除target###
存储节点 cinder-volume 通过 tgt-admin 命令删除 volume 对应的 target。日志文件为 /opt/stack/logs/c-vol.log。  
![](/img/cinderdetachvoldeltarget.jpg)


###volume delete操作###
状态为available的volume才能被delete，并且volume需要先被detach，才能被delete。delete的操作流程如下图所示：  
![](/img/cindervoulumedel.jpg)
1. 向cinder-api发送delete请求
2. cinder-api发送消息
3. cinder volume执行delete操作

cinder-api接收到delete volume的请求后，向消息队列发送delete消息。相应的源代码在`cinder/cinder/volume/api.py`的`extend`方法中。  
![](/img/cinderdeletevolumemsg.jpg)  

**cinder-volume delete volume**  
cinder-volume执行lvremove命令delete volume。日志在/opt/stack/logs/c-vol.log。  

###Snapshot volume###  
snapshot可以为volume创建快照，保存volume当前的状态。其操作流程如下图所示：  
![](/img/cindersnapshot.jpg)  

1. 向cinder-api发送sanpshot请求
2. cinder-api发送消息
3. cinder-volume执行shapshot操作

cinder-api接收到snapshot volume的请求(rest接口，post http)。发送snapshot消息。发送消息的源代码为`/cinder/cinder/volume/api.py`，方法为_create_snapshot。  
![](/img/cindersnapshotsrc.jpg)  

cinder-volume获得消息后，执行lvcreate命令创建snapshot。

###Backup Volume操作###
Backup是将volume备份到别的地方，使得可以通过restore操作恢复。  
Backup与snapshot的区别：  
1. Snapshot依赖于源volume，不能独立存在；而backup不依赖源volume，即便源volume不存在，也可以restore。  
2. Snapshot与源volume通常放在一起，都由同一个volume provider管理；而backup存放在独立的备份设备中，有自己的备份方案和实现，与volume provider没有关系。  
3. 上面两点决定了backup具有容灾功能；而snapshot则提供volume provider内便捷的回溯功能。  

**配置cinder-backup**  
Cinder的backup功能是由cinder-back服务提供的，需要手工启用。与cinder-volume类似，cinder-back也通过driver架构支持多种备份backend，包括ＰＯＳＩＸ文件系统、ＮＦＳ、Ｃｅｐｈ、GlusterFS、Swift等。支持的driver源文件放在`/cinder/cinder/backup/drivers/`。如图：  
![](/img/cinderbackup.jpg)  

1. 向cinder-api发送backup请求  
2. cinder-api发送消息  
3. cinder-backup执行back操作  

以NFS为backend，若存放volume backup的NFS远程目录为192.168.104.11:/backup，cinder-backup服务节点上mount point为 /backup_mount。  
需要在/etc/cinder/cinder.conf中作相应的配置。  
![](/img/cinderbackupconf.jpg)  

**向向cinder-api发送backup请求**  
client向cinder-api发送请求：“请backup指定的volume”。backup在CLI中执行，如下图：  
![](/img/cinderbackcli.jpg)  

cinder-api接收到backup volume的http请求。日志在/opt/stack/logs/c-api.log。  

**cinder-api发送消息**  
cinder-api发送backup消息。对应的源代码在`/cinder/cinder/backup/api.py`，方法为create。如图：  
![](/img/cinderbackupsrc.jpg)  

**cinder-backup执行backup操作**    
cinder-backup收到消息后，通过如下步骤完成backup操作，日志为/opt/stack/logs/c-vol.log。  
1、启动backup操作，mount NFS。  
![](/img/cinderbackupmount.jpg)  
2、创建volume的临时快照。  
![](/img/cindervolumesnapshot.jpg)  
3、创建存放backup的container目录。  
![](/img/cinderbackupcontainer.jpg)  
4、对临时快照数据进行压缩，并保存到container目录。
![](/img/cinderbackupshapshotcontainer.jpg)  
5、创建并保存sha256(加密)文件和metadata文件。  
![](/img/cinderbackupstore.jpg)
6、删除临时快照。
![](/img/cinderbackupsnapshotdel.jpg)  


###Restore Volume操作###
restore的过程主要分成两步：  
1. 在存储节点上创建一个空白的volume。  
2. 将backup的数据copy到空白volume上。  

restore的详细流程如下：  
![](/img/cinderrestore.jpg)  
1. 向cinder-api发送restore请求  
2. cinder-api发送restore消息  
3. cinder-scheduler挑选最合适的cinder-volume，发送消息  
4. cinder-volume创建空白的volume  
5. cinder-backup将backup数据拷贝到空白的volume上  

**向cinder-api发送restore请求**  
client向cinder-api发送请求：“请restore指定的backup。”  
目前restore只能在CLI中执行。  
\user@devstack-comtroller>`cinder backup-list`：显示backup的ID。  
\>`cinder backup-restore backup-ID`  

cinder-api接收到restore请求。日志文件/opt/stack/logs/c-api.log。  
![](/img/cinderrestoreapi.jpg)  
cinder-api转发请求，为restore创建volume。之后cinder-scheduler和cinder-volume将创建空白volume，这个过程与create-volume一样。    

**cinder-backup执行restore操作**  
日志为/opt/stack/logs/c-vol.log  
1、启动restore操作，mount NFS。  
2、读取container目录中的metadata。
3、将数据解压并写到volume中。
4、恢复volume的metadata，完成restore操作。

##Cinder的实现##
1. Cinder API  
负责接收和处理Rest请求。调用cinder-volume执行操作。  
2. Cinder scheduler    
scheduler通过调度算法，选择最合适的节点创建volume。  
3. Cinder volume    
运行在存储节点上，管理volume。与volume provider协调工作。通过加载相应的插件实现不同的后端存储。  
4. volume provider  
存储设备，提供volume的物理存储空间。  
5. Meaasge QUeue  
Cinder各个服务之间的通信和相互协作，通过消息队列，实现服务之间的松耦合。  



#模块详解#
--------------------------------
cinder代码模块：  
各个模块的启动脚本位于cmd目录下：  
`all.py`是指启动所有的服务；  
其余文件代表启动相对应的模块。  

**Backup模块**  
主要的处理函数在api.py文件中：  
`delete()`：删除指定的backup；  
`create()`：创建backup；  
`restore()`：恢复指定的backup；
`update()`：更新backup。  

其中`rpcapi.py`是对rpc消息工具的一个封装，以适用于Backup模块。其中`cctxt.cast()`广播消息从而实现与其他模块的通信。  

**Volume模块**  
  

