#puppet#
利用hiera实现puppet manifests静态代码与动态数据的分离。  
puppet3.7自动支持hiera。  
hiera利用键值对的方式存储数据。支持yaml及json两种格式文件。默认yaml。  
hiera文件组织形式：  
Hiera的一个核心理念是重用数据，具体体现为对数据源文件的层次化分类管理（ Hierarchy ）

层次1： 所有节点通用的数据定义在一个公共数据源文件中，且只需定义一次   

层次2：   对节点分类（主要是依据facts）。一类节点的通用数据定义在一个公共数据源文件中，且只需定义与上面一层不同的部分

层次3： 对每一个节点进行配置，每个节点一个数据源文件，且只需定义与上面几层不同的数据  

数据源的配置被定义在Hiera的主配置文件（hiera.conf）中。下面是一个配置文件的例子。  
:backends:                        #支持的数据源文件格式，默认是 yaml和 json

– yaml                            #查找yaml格式数据源文件

– json                             #查找json格式数据源文件

:yaml:                              #yaml格式数据源文件的根目录

:datadir: "/etc/puppet/environments/%{::environment}/hieradata"            #%{::environment}是指facts中的environment变量。这行定义是说yaml格式的数据源文件的根目录是"/etc/puppet/environments/%{::environment}/hieradata"    

:json:                                #json格式数据源文件的根目录

:datadir: "/etc/puppet/environments/%{::environment}/hieradata"            

:hierarchy:                        #分类和层次关系（hierarchy）

– "nodes/%{::fqdn}"         #%{::fqdn}是指facts中的fqdn变量。按fqdn分类的数据源文件存在$datadir/nodes目录下， 文件 以agent节点的fqdn命名

– "virtual/%{::virtual}"       #%{::virtual}是指facts中的virtual变量。按virtual值分类的数据源文件存在$datadir/virtual目录下， 文件 以virtual值命名

– "common"                    #所有节点的默认配置都存在common.yaml或者common.json中

注意： 如果修改了 hiera.conf的内容，Puppet master进程必须重启才能生效

如果我的production环境中，server1和server2都是运行在xen上，而server3和server4都是在vmware上，那么根据上面的配置文件，我的数据源文件目录结构就很可能是这个样子

/etc/puppet/environments/production/hieradata   
├── common.yaml                    #所有节点的通用配置

├── nodes                                #以fqdn分类的每个节点的配置

│   ├── server1.yaml                #server1的配置

│   ├── server2.yaml                #server2的配置

│   ├── server3.yaml                #server3的配置

│   └── server4.yaml                #server4的配置

└── virtual                                #以virtual分类的通用配置

├── xen.yaml                       #所有xen虚拟机的通用配置

└── vmware.yaml                #所有vmware虚拟机的通用配置

对于server1 （xen虚拟机）来说，它的最终配置会是 server1.yaml，  xen.yaml 和 common.yaml 中配置的组合。  

1、 自动参数查询（automatic parameter lookup） 
这种方式主要用于查询类的参数。

当Hiera配置好后，在manifest中声明类但不给指定参数赋值，比如使用include-like方式（无类参数），或者使用resource-like方式但不显性的给参数赋值，这两种情况下，Puppet都会 自动 使用<类名>::<参数名>作为键通过Hiera查询相应的参数值。详细信息请看 这里

注意： * 不要在template中使用自动参数查询。

* 如果想禁止这个功能，在master的puppet.conf设置data_binding_terminus = none

2、 使用Hiera内置函数或Hiera命令查询

数据类型	    Hiera内置函数	    Hiera命令  
任何数据类型	hiera(<键>)	        hiera <键>  
数组     	hiera\_array(<键>)	hiera -a <键>  
Hash	    hiera\_hash(<键>)  	hiera -h <键>  

注意：    *  Hiera内置函数可以在任意的manifest文件中调用或者在puppet apply -e中命令调用。

* Hiera命令在Hiera安装好后，就可以从shell中使用。

3、 使用 Hiera内置函数 hiera_include

hiera_include() 专门用来在site.pp中查询哪些类分配给了指定节点，等同于在节点定义中使用 include-like/resource-like 来声明类，可以作为 ENC 的一个替代方案。

Hiera查询是如何工作的？

查询时，Hiera会按下面定义的 顺序 遍历子目录下的数据源文件，寻找匹配的键。

:hierarchy:

– "nodes/%{::fqdn}"

– "virtual/%{::virtual}"

– "common"

如果:hierarchy:的定义是上面这样的，查询的顺序就是datadir下的nodes/%{::fqdn}子目录，然后是virtual/%{::virtual}子目录，最后是common.yaml文件。

如果你是使用以下的查询方式，那么在找到 第一个匹配 的键之后Hiera就返回了，不在继续查找

* 自动参数查询（automatic parameter lookup）
* Hiera内置的hiera函数   
* hiera命令（没有-a或-h）
 
如果你是使用以下的查询方式，那么Hiera会认为你在查找一个数组，它会遍历所有的数据源文件，然后将所匹配的所有数组值 合并到一个数组 中返回 

* Hiera内置的hiera\_array函数 
* Hiera内置的 hiera\_include函数
* hiera -a 命令

如果你是使用以下的查询方式，那么Hiera会认为你在查找一个hash，它会遍历所有的数据源文件，然后将所匹配的所有内容值 合并到一个hash 中返回。

* Hiera内置的hiera_hash函数
* hiera -h 命令