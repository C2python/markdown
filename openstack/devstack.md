#Devstack安装Openstack#
-----------------------------
Centos7，stable/liberty。  

创建stack用户：  
>`adduser -m stack`  
>`passwd stack`  

`git clone  https://github.com/openstack-dev/devstack.git`下载安装包。  


运行`devstack/tools/create-stack-user.sh`赋予用户无密码sudo权限。 
`sudo chown -R stack:stack /home/stack/devstack`赋予用户所有权限。

安装pip：  
首先安装epel-release扩展包。  
`yum install epel-release`  
`yum install python-pip`  
设置pip源：  
>
mkdir ~/.pip  
cat > ~/.pip/pip.conf <<EOF  
[global]  
index-url = http://pypi.douban.com/simple/  
[install]  
trusted-host=pypi.douban.com  
EOF 

pip升级到8.0版本：  
`sudo pip isntall --upgrade pip`  

升级到最新版本后，执行./stack.sh时，会自动卸载重装pip，而且通过yum install安装的是旧版本，出现错误。因此需要将脚本中卸载pip以及安装pip的脚本注释掉。
修改/devstack/tools/install.sh：  
注释掉`uninstall_package python-pip`以及`install_get_pip`  

设置配置文件local.conf：  
>
[[local|localrc]]
>
HOST_IP=192.168.100.106
SERVICE_HOST=192.168.100.106
MYSQL_HOST=192.168.100.106
RABBIT_HOST=192.168.100.106
GLANCE_HOSTPORT=192.168.100.106:9292

>
ADMIN_PASSWORD=openstack
MYSQL_PASSWORD=openstack
RABBIT_PASSWORD=openstack
SERVICE_PASSWORD=openstack
SERVICE_TOKEN=openstack
>
\# https://wiki.openstack.org/wiki/NeutronDevstack
disable_service n-net
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta
enable_service q-fwaas
enable_service q-lbaas
enable_service q-metering
enable_service q-vpn
\# Optional, to enable tempest configuration as part of devstack
\#enable_service tempest
\#enable_service neutron
>
Q_USE_SECGROUP=True
FLOATING_RANGE="192.168.100.0/24"
FIXED_RANGE="10.0.0.0/24"
Q_FLOATING_ALLOCATION_POOL=start=192.168.100.107,end=192.168.100.220
Q_L3_ENABLED=True
PUBLIC_INTERFACE=enp8s0
Q_USE_PROVIDERNET_FOR_PUBLIC=True
OVS_PHYSICAL_BRIDGE=br-ex
PUBLIC_BRIDGE=br-ex
OVS_BRIDGE_MAPPINGS=public:br-ex
Q_PLUGIN=ml2
Q_ML2_TENANT_NETWORK_TYPE=vxlan
\#ENABLE_TENANT_TUNNELS=True
\#ENABLE_TENANT_VLANS=True

>
LOGFILE=$DEST/logs/stack.sh.log
>
\# Enable swift services
\#enable_service s-proxy
\#enable_service s-object
\#enable_service s-container
\#enable_service s-account
\#SWIFT_HASH=66a3d6b56c1f479c8b4e70ab5c2000f5
\#SWIFT_REPLICAS=1

>
\# Ceilometer - Metering Service (metering + alarming)
\#ENABLED_SERVICES+=,ceilometer-acompute,ceilometer-acentral,ceilometer-collector,ceilometer-api
\#ENABLED_SERVICES+=,ceilometer-alarm-notify,ceilometer-alarm-eval
\#enable_plugin ceilometer git://git.openstack.org/openstack/ceilometer
>
\# Heat - Orchestration Service
\#ENABLED_SERVICES+=,heat,h-api,h-api-cfn,h-api-cw,h-eng


>
\########################################################################################
\########################################################################################
\##########setup ceilometer/Heat/Trove/Sahara/manila
\#Ceilometer
\#enable_service ceilometer-acompute ceilometer-acentral ceilometer-collector ceilometer-api ceilometer-anotification
\#enable_service ceilometer-alarm-notifier ceilometer-alarm-evaluator
\#enable_plugin ceilometer https://git.openstack.org/openstack/ceilometer.git

>
\#Trove
\#enable_service trove tr-api tr-tmgr tr-cond
>
\#sahara
\#enable_service sahara
\#enable_plugin sahara git://git.openstack.org/openstack/sahara


>
\#manila
\#enable_plugin manila https://github.com/openstack/manila
\#ENABLED_SERVICES+=,manila,m-api,m-sch,m-shr

>
\# Murano
\#enable_service murano murano-api murano-engine
>
\#Barbican
\#enable_plugin barbican https://git.openstack.org/openstack/barbican
\#enable_service rabbit mysql key


执行./stack.sh，用stack用户执行。


#Notes：  

Rabbitmq连接错误：  
Error: Rabbitmq@:  
`* unbable to connect to rabbit-cli@localhost:nodedown`.  
此时需要考虑两种情况：  
1. 地址解析错误，此时应查看/etc/hosts与/var/logs/rabbitmq日志文件中，连接的IP是否相同。  
2. 端口错误，rabbitmq的默认监听端口是5672，而openstack可能连接的不是此端口，导致无法连接。此时应更改/etc/rabbitmq/rabbitmq.config配置文件，修改{tcp_listeners,port_num}。

在启动dashboard时发生静态文件的加载错误，`dashboard/developer/developer.scss couldnot be found in /horizon/static/`。我的做法是手动安装一下horizon。

stack.sh是自动安装的脚本。对于依赖的组件，会经常报版本太旧的错误，此时可以手动更新，如sudo pip install --upgrade [module]。但是在重新运行./stack.sh时，此脚本会删除原有组件重新安装，此时安装的却仍是旧的版本（自带包的原因），仍会报错。在这种情况下，应手动更新，并且经对应脚本中删除安装的步骤注释掉。