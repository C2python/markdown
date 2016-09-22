#OpenStack存储系统
----------------------------------
##存储模块
>* Swift-提供对象存储（Object Storage）； 
>* Glance-提供虚拟机镜像存储和管理；
>* Cinder-提供块存储（Block Storage）。

###Swift
对象存储，提供强大的可扩展性、冗余和持久性。用于永久类型的静态数据的长期存储。
<p>主要特性如下：</p>
> <ul type="circle">
<li> 极高的数据持久性；</li>
<li> 完全对称的系统架构；</li>
<li>无限的可扩展性；</li>
<li> 无单点故障；</li>
<li> 简单、可依赖。</li>
</ul>

架构图如下所示：

![](https://github.com/C2python/opentsack-conf/blob/master/openstack/img/swift.png "Swift 架构")

主要组件：

* Proxy Server：服务器进程，提供了REST API；负责组件间的通信。对客户端的请求，在Ring中查询Account、Container、Object的位置。并转发请求。
* Storage Server：提供存储服务。有三类存储服务器：Account、Container、Object。Container负责处理Object列表，但不负责具体存储位置。
* Consistency Server:查找并解决由数据所怀和故障引起的错误。由三个Server负载：Auditor、Updater、Replicator。Auditor负责检测对象、Container以及账号的完整性。Auditor移动坏文件至隔离区，Replicator负责拷贝好数据替代坏数据。Updaters负责更新失败后的处理。Account Updater和Container Updater分别负责Account和Object列表的更新。
* Ring：记录存储对象与物理位置间的映射关系。

###镜像存储-Glance
提供虚拟机的发信啊、注册、取得服务。通过Glance，虚拟机镜像可以被存储到多种存储上，比如简单的文件存储和对象存储。
![](https://github.com/C2python/opentsack-conf/blob/master/openstack/img/Glance.png "Glance 架构")

Glance被设计为可以使用多种后端存储。前端通过API Server向多个Client提供服务。支持S3、Swift，简单的文件存储以及只读的HTTPS存储，也支持其他后端，如分布式存储系统（SheepDog和[**Ceph**](http:docs.ceph.org.cn)）。

###块存储-Cinder
Cinder架构图如下所示：

![](https://github.com/C2python/opentsack-conf/blob/master/openstack/img/Cinder.png "Cinder 架构")

**Cinder服务**
> * API service：负责接受和处理 Rest 请求，并将请求放入 RabbitMQ队列。
>* Scheduler service: 处理任务队列的任务，并根据预定策略选择合适的 Volume Service 节点来执行任务。目前版本的 cinder 仅仅提供了一个 Simple Scheduler, 该调度器选择卷数量最少的一个活跃节点来创建卷。
>* Volume service: 该服务运行在存储节点上，管理存储空间。每个存储节点都有一个 Volume Service，若干个这样的存储节点联合起来可以构成一个存储资源池。为了支持不同类型和型号的存储，当前版本的 Cinder 为 Volume Service 添加如下 drivers。当然在 Cinder 的 blueprints 当中还有一些其它的 drivers，以后的版本可能会添加进来。


#Openstack安装部署#
--------------------------------
安装步骤：
>* Centos，Eclipse；
>* 创建stack用户；
>* 克隆devstack；
>* 编辑local.conf文件
>* 运行stack.sh。

