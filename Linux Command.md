#Centos下相关命令操作#
------------------------------
##用户建立，分配权限##
用户账号文件/etc/passwd   ：保存用户名称、宿主目录、登录shall等基本信息。
`
cat /etc/passwd or
tail -l /etc/passwd
`

`tail查看文件命令，`

用户账号文件/etc/shadow   :保存用户的密码、账号有效期等信息
`
tail -l /etc/shadow
`

`
/etc/group,/etc/gshadow组用户文件
`

添加用户的账号：
> useradd 命令：useradd [选项] 用户名
>
命令选项：

>* -u：指定用户的UID标记号
>* -d：指定宿主目录，缺省为/home/用户名
>* -e：指定账号的失效时间
>* -g：指定用户的基本组名（或UID号）
>* -G：指定用户的附加组名（或GID）
>* -M：不能为用户建立并初始化宿主目录
>* -s：指定用户的登陆shell

管理账户命令：
>
1. useradd 选项  用户名//添加新用户
2. useradd -g 组 用户名 //将用户加入某个组中
2. usermod 选项  用户名//修改已经存在的用户
3. userdel -r    用户名//删除用户表示自家目录一起删除
4. groupadd 选项  组名// 添加新组
5. groupmod 选项  组名//修改已经存在的组
6. groupdel 组名  //删除已经存在的特定组
7. passwd 用户名 //添加用户密码

`
useradd zhangtest //建立了用户zhangtest，并且默认建立了用户目录/home/zhangtest，只有zhangtest以及root有读写执行的权限。
`
`
ls -l file/directory：列出文件或目录的权限
`

`chown username file/dir：修改一个文件或者目录的所有者`

添加用户至sudo组中：
`
chmod u+w /etc/sudoers;
vim /etc/sudoers;
插入：username ALL=(ALL)   ALL;
无密码sudo权限：username ALL=(ALL)   NOPASSWD:ALL;
chmod u-w /etc/sudoers;
`

##yum、rpm使用##
配置yum源：
>cd /etc/yum.repos.d //进入源所在目录；
>
>mv CentOS-Base.repo CentOS-Base.repo.bk //备份旧的源；

>wget http://mirrors.163.com/.help/CentOS6-Base-163.repo //下载新源

>yum makecache//使源生效
>
>wget:下载工具 wget [参数] [Url 地址]


Rpm：</p>
rpm 执行安装包二进制包（Binary）以及源代码包（Source）两种。二进制包可以直接安装在计算机中，而源代码包将会由RPM自动编译、安装。源代码包经常以src.rpm作为后缀名。

常用命令组合：
>
－ivh：安装显示安装进度--install--verbose--hash</p>
－Uvh：升级软件包--Update；</p>
－qpl：列出RPM软件包内的文件信息[Query Package list]；</p>
－qpi：列出RPM软件包的描述信息[Query Package install package(s)]；</p>
－qf：查找指定文件属于哪个RPM软件包[Query File]；</p>
－Va：校验所有的RPM软件包，查找丢失的文件[View Lost]；</p>
－e：删除包
>

命令示例：
>
>rpm -q samba //查询程序是否安装</p>
>rpm -ivh  /media/cdrom/RedHat/RPMS/samba-3.0.10-1.4E.i386.rpm //按路径安装并显示进度</p>
>rpm -ivh --relocate /=/opt/gaim aim-1.3.0-1.fc4.i386.rpm    //指定安装目录</p>
>rpm -ivh --test gaim-1.3.0-1.fc4.i386.rpm　　　 //用来检查依赖关系；并不是真正的安装；</p>
>rpm -Uvh --oldpackage gaim-1.3.0-1.fc4.i386.rpm //新版本降级为旧版本</p>
>rpm -qa | grep httpd　　　　　 ＃[搜索指定rpm包是否安装]--all搜索*httpd*</p>
>rpm -ql httpd　　　　　　　　　＃[搜索rpm包]--list所有文件安装目录</p>
>rpm -qpi Linux-1.4-6.i368.rpm　＃[查看rpm包]--query--package--install package信息</p>
>rpm -qpf Linux-1.4-6.i368.rpm　＃[查看rpm包]--file</p>
>rpm -qpR file.rpm　　　　　　　＃[查看包]依赖关系</p>
>rpm2cpio file.rpm |cpio -div    ＃[抽出文件]</p>
>rpm -ivh file.rpm 　＃[安装新的rpm]--install--verbose--hash
rpm -ivh</p>

>rpm -Uvh file.rpm    ＃[升级一个rpm]--upgrade</p>
>rpm -e file.rpm      ＃[删除一个rpm包]--erase</p>

`
rpm　--recompile　vim-4.6-4.src.rpm   ＃这个命令会把源代码解包并编译、安装它，如果用户使用命令：
`
`rpm　--rebuild　vim-4.6-4.src.rpm　　＃在安装完成后，还会把编译生成的可执行文件重新包装成i386.rpm的RPM软件包。
`

##route命令##
主要用于显示和操作IP路由表。</p>
利用route来添加路由不会永久保存，重启丢失。需要配置响应的脚本。在Centos中要配置：</p>
route命令用于显示和操作IP路由表。</p>
**添加删除路由表**
>* route [-nee]
>* route add [-net|-host] [网域或主机IP] (netmask)/网域 [netmask] [gw|dev]
>* route del [-net|-host] [网域或主机IP] (netmask)/网域 [netmask] [gw|dev]

**参数说明**
>-n  ：不要使用通讯协定或主机名称，直接使用 IP 或 port number；</p>
>-ee ：使用更详细的资讯来显示</p>

增加 (add) 与删除 (del) 路由的相关参数：
>-net ：表示后面接的路由为一个网域；</p>
>-host：表示后面接的为连接到单部主机的路由；</p>
>netmask：与网域有关，可以设定 netmask 决定网域的大小；</p>
>gw ：gateway 的简写，后续接的是 IP 的数值，与 dev 不同；</p>
>dev：如果只是要指定由那一块网路卡连线出去，则使用这个设定，后面接 eth0 等</p>

例子：</p>
`sudo route add -net 10.0.3.0 netmask 255.255.255.0 dev eth0;`
`sudo route add -host 10.0.3.1 dev eth0;`
`sudo route add -host 10.0.3.1 gw [GatewayIP];`

`route -n`
其中flags表示：
>UGH：U的意思是当前的路由是运行的、有效的；G的意思是通过gateway连接的；H的意思是指对某一个特定的机器的路由

`route add -net {NETWORK-ADDRESS} netmask {NETMASK} reject`
表示网络不可达。

**静态路由的设置**

Centos中不能通过</p>

1. /etc/rc.local
2. /etc/sysconfig/network
3. /etc/sysconfig/static-router

进行永久设置。

Centos下，永久路由的设置处于/etc/sysconfig/network-scripts目录下。针对每个接口都存在一个专门的文件：`ifcfg-eth0,ifcfg-lo`。
这对eth0路由的设置需要在文件`ifcfg-eth0`中添加。

格式如下：
> `network/prefix via gateway dev intf`
> >0.0.0.0/0 via 172.16.10.2 dev eth0

其中`/etc/sysconfig/static-router`设置如下(未验证)：
> any net 192.168.3.0/24 gw 192.168.3.254</p>
> any net 10.250.228.128 netmask 255.255.255.192 gw 10.250.228.129


##[ip命令](http://www.cnblogs.com/bamboo-talking/archive/2013/01/10/2855306.html)##
网络配置工具，用于取代传统的网络管理工具，如 ifconfig、route等。

格式如下：
> ip [OPTIONS] OBJECT [COMMAND [ARGUMENTS]]

主要参数如下Options：
>
-V,-Version 打印ip的版本并退出。</p>
-s,-stats,-statistics 输出更为详尽的信息。如果这个选项出现两次或多次，则输出的信息将更为详尽。</p>
-f,-family 这个选项后面接协议种类，包括inet、inet6或link，强调使用的协议种类。如果没有足够的信息告诉ip使用的协议种类，ip就会使用默认值inet或any。link比较特殊，它表示不涉及任何网络协议。</p>
-4 是-family inet的简写。</p>
-6 是-family inet6的简写。</p>
-0 是-family link的简写。</p>
-o,-oneline 对每行记录都使用单行输出，回行用字符代替。如果需要使用wc、grep等工具处理ip的输出，则会用到这个选项。</p>
-r,-resolve 查询域名解析系统，用获得的主机名代替主机IP地址</p>

Command:指定对象执行的操作，和对象的类型有关。ip支持对象的增加(add)、删除(delete)和展示(show/list)。</p>
ARGUMENTS是命令的一些参数。

**常用命令**
-----------------------------------
1. 常用命令</P>
>
`ip link set [device] [ARGUMENTS]`  
`ip link set dev eth0 [up|down]`  
`ip link set dev eth0 txqlen [Number]`端口队列大小；  
`~~~ mtu [Number]`  
`~~~ address [Mac_address]` 修改mac地址  

ip link show --输出设备的详细信息；  
`ip -s -s link show dev eth0`；-s越多显示的信息越详细；

ip address add --添加一个新的协议地址，缩写：add、a
`ip addr add 10.0.2.3/24 brd + dev eth0 label eth0:Alias1`  
在以太网接口eth0上增加一个地址10.0.2.3，标准广播地址，标签为eth0:Alias1  
`ip addr del/show ---`删除/显示协议地址。

ip address flush --清除协议地址  
`ip -s -s addr flush to 10.0.2.0/24`/`ip -s -s a f to 10.0.2.0/24`；删除所有10.0.2.0中的所有地址。

`ip -4 addr flush label "eth0"`取消所有以太网卡的ip地址。  

*ip neighbor [command] --*  
表管理命令。  
command：  
>add/a,change/chg,replace/repl,delete/del,flush/f,show/list

`ip neigh add 10.0.0.3 lladdr 0:0:0:0:0:1 dev eth0 nud perm`：在设备eth0上，添加地址10.0.0.3的一个permanent ARP条目；  
`ip beigh chg 10.0.0.3 dev eth0 nud reachable`状态更改为reachable；  
`ip  neigh del 10.0.0.3 dev eth0`  
`ip -s neigh show [ip]`  

###路由表管理命令###

`ip route [command] --`路由管理命令；  
command：  
>add/a,change/chg,replace/repl,delete/del,flush/f,show/list

添加路由如下：  
`ip route add 10.0.2.3/24 via 10.0.4.1`设置到网络10.0.2.3/24经过网关10.0.4.1。  
`ip route chg 10.0.2.3/24 dev eth0`设置到网络10.0.2.3/24经过设备eth0。  

实现多路径路由：  
`ip route replace/add default equalize nexthop via 211.139.218.145 dev eth0 weight 1 nexthop via 211.139.218.145 dev eth1 weight 1` 211.139.218.145是下一跳的地址(一般为网关，此处从eth0或eth1都需经过此地址)。  
`ip route add/replace default scope global nexthop dev eth0 nexthop eth1`  

设置NAT路由。在转发来自192.203.80.144的数据包之前，先进行网络地址转换，把这个地址转换为193.233.7.83
　　`ip route add nat 192.203.80.142 via 193.233.7.83`

`ip route ls proto gated/bgp |wc` 计算使用gated/bgp协议的路由个数。  
`ip -o route ls cloned |wc`计算路由缓存里的条数。

列出某个路由表的内容:`ip route ls table fddi153`
擦除路由表：`ip route flush --`  

`ip route get [destination ip]`获得到达目的地址的一个路由以及它确切的内容。
与 show不同，get可能会计算出到达目的地址的路由  

`ip rule add/del --`插入和删除路由规则  
示例1: 通过路由表inr.ruhep路由来自源地址为192.203.80/24的数据包  
`ip ru add from 192.203.80/24 table inr.ruhep prio 220`

示例2:把源地址为193.233.7.83的数据报的源地址转换为192.203.80.144，并通过表1进行路由  
`ip ru add from 193.233.7.83 nat 192.203.80.144 table 1 prio 320`

`ip maddress [command] --`多播地址管理，只能进行链路层管理。

###ip隧道配置###

`ip tunnel [command] --`  
comamd:  
如上。

建立一个点对点通道，最大TTL是32  
`ip tunnel add Cisco mode sit remote 192.31.7.104 local 192.203.80.1 ttl 32`

`ip monitor/rtmon --`状态监视。用于连续监视设备、地址和路由的状态。  
`ip monitor [file FILE] [all | OBJECT-LIST]`
`rtmon file /var/log/rtmon.log`

###[SSH的安装和配置参考](http://www.centoscn.com/CentOS/config/2013/0926/1713.html)###

客户端的配置：
>
>假设VPS采用centos，再假设用较新版本6.5。
VPS上可能没有安装桌面，但一般来说都会安装ssh，并且防火墙默认开放22端口。
那就从ssh开始。
安装ssh，默认已安装好
yum install ssh
启动ssh服务器端
service sshd start
chkconfig sshd on  
ssh登陆
如果本地端是Linux
ssh root@192.168.1.1
其中root表示的是登录用户名，192.168.1.1为主机的IP地址，当然也可以使用主机名、域名来指代IP地址。
ssh 192.168.1.1
则会以当前客户端的用户名进行登录。   
**ssh无密码登录**  
但是每次输入密码登录十分麻烦，有没有一种方式可以让服务器能够确定我的身份，无需输入密码可以直接通过认证？  
ssh除了使用密码验证外，还提供了一种公私密钥的验证方式。客户端生成一个私钥，并生成一个与之对应的公钥，然后将公钥上传到服务器上。下面是Linux示例。  
在客户端生成私钥、公钥（注意，在客户端完成）：
ssh-keygen -t rsa  
-t指定要创建的密钥类型，默认就是rsa了，所以只执行ssh-keygen是一样的。
期间会提示你输入你私钥的加密密码。如果需要完全脱离密码，此处可留空，直接回车，否则以后每次连接需要本地解锁。  
完成后，会当前用户的主目录下的~/.ssh/路径下生成两个文件id_rsa与id_rsa.pub分别是私钥与公钥。  
接下来，要把生成的公钥上传到服务器上，同样还是在客户端执行以下的代码。
`ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.1.1`  
其中root可以修改为你想要自动登录的服务器端用户名，192.168.1.1修改为你的VPS主机名或IP地址。  
最后，ssh登录远程服务器。  
`ssh root@192.168.1.1`  
此时就不需要密码就可以登录了。

ssh -p 29418 -i ~/.ssh/admin 10.0.1.2 -l admin  
ssh-keygen -t rsa -C "管理员" -f ~/.ssh/admin:指定用户admin，生成密钥
>

**ssh-keygen命令打通主机之间的ssh**  
scp /root/.ssh/id_rsa.pub host-dst:/root/.ssh/  
ssh -t host-dst "cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys && rm -f /root/.ssh/id_rsa.pub"  



**[更改相关配置的操作](http://my.oschina.net/1462469/blog/265783)**

##[awk工具](http://man.linuxde.net/awk)##

awk是强大的文本分析工具，在对文本文件的处理以及生成报表，awk是无可替代的。awk认为文本文件都是结构化的，它将每一个输入行定义为一个记录，行中的每个字符串定义为一个域(段)，域和域之间使用分割符分割。  

**工作原理**
>
1. 行工作模式，读入文件的每一行，会把一行的内容，存到$0里
2. 使用内置的变量FS(段的分隔符，默认用的是空白字符)，分割这一行，把分割出来的每个段存到相应的变量$(1-100)
3. 输出的时候按照内置变量OFS(out FS)，输出
4. 读入下一行继续操作

`echo "This is a book" > test.txt`  
`awk '{print $2,$1,$3,$4}' awk.txt`  
输出：is This a book.  
$0里记录了This is a book.

awk脚本是由模式和操作组成的。  
**模式**  
可以是以下任意一个：
>* /正则表达式/：使用通配符的扩展集。 
>* 关系表达式：使用运算符进行操作，可以是字符串或数字的比较测试。 
>* 模式匹配表达式：用运算符~（匹配）和~!（不匹配）。 
>* BEGIN语句块、pattern语句块、END语句块：参见awk的工作原理

**操作**  
操作由一个或多个命令、函数、表达式组成，之间由换行符或分号隔开，并位于大括号内，主要部分是：  
>* 变量或数组赋值 
>* 输出命令 
>* 内置函数 
>* 控制流语句  

awk脚本的基本结构  
`awk 'BEGIN{ print "start"} pattern{ commands } END{ print "end"}' file`  
`awk 'BEGIN{ print "start"} pattern{ commands } END{ print "end"}' var= variable file`   
一个awk脚本通常由：BEGIN语句块、能够使用模式匹配的通用语句块、END语句块3部分组成，这三个部分是可选的。任意一个部分都可以不出现在脚本中，脚本通常是被单引号或双引号中。

awk的工作原理  
`awk BEGIN{ commands } pattern{ commands } END{ commands }`  

执行步骤：  
>* 第一步：执行BEGIN{ commands }语句块中的语句； 
>* 第二步：从文件或标准输入(stdin)读取一行，然后执行pattern{ commands }语句块，它逐行扫描文件，从第一行到最后一行重复这个过程，直到文件全部被读取完毕。 
>* 第三步：当读至输入流末尾时，执行END{ commands }语句块。

示例：  
> `echo -e "A line 1\nA line 2" | awk 'BEGIN{print "Header"} {print $0":"$1} END{print "end"}'`  
>  Header  
>  A line 1:A  
>  A line 2:A  
>  End  

**awk内置变量(预定义变量)**
>* $n 当前记录的第n个字段，比如n为1表示第一个字段，n为2表示第二个字段。 
>* $0 这个变量包含执行过程中当前行的文本内容。 
>* [N] ARGC 命令行参数的数目。 
>* [G] ARGIND 命令行中当前文件的位置（从0开始算）。 
>* [N] ARGV 包含命令行参数的数组。 
>* [G] CONVFMT 数字转换格式（默认值为%.6g）。 
>* [P] ENVIRON 环境变量关联数组。 
>* [N] ERRNO 最后一个系统错误的描述。 
>* [G] FIELDWIDTHS 字段宽度列表（用空格键分隔）。 
>* [A] FILENAME 当前输入文件的名。 
>* [P] FNR 同NR，但相对于当前文件。 
>* [A] FS 字段分隔符（默认是任何空格）。 
>* [G] IGNORECASE 如果为真，则进行忽略大小写的匹配。 
>* [A] NF 表示字段数，在执行过程中对应于当前的字段数。 
>* [A] NR 表示记录数，在执行过程中对应于当前的行号。 
>* [A] OFMT 数字的输出格式（默认值是%.6g）。 
>* [A] OFS 输出字段分隔符（默认值是一个空格）。 
>* [A] ORS 输出记录分隔符（默认值是一个换行符）。 
>* [A] RS 记录分隔符（默认是一个换行符）。 
>* [N] RSTART 由match函数所匹配的字符串的第一个位置。 
>* [N] RLENGTH 由match函数所匹配的字符串的长度。 
>* [N] SUBSEP 数组下标分隔符（默认值是34）。

示例：  
>`echo -e "line1 f2 f3\nline2 f4 f5\nline3 f6 f7" | awk '{print "Line No:"NR", No of fields:"NF, "$0="$0, "$1="$1, "$2="$2, "$3="$3}'`  
> Line No:1, No of fields:3 $0=line1 f2 f3 $1=line1 $2=f2 $3=f3   
> Line No:2, No of fields:3 $0=line2 f4 f5 $1=line2 $2=f4 $3=f5   
> Line No:3, No of fields:3 $0=line3 f6 f7 $1=line3 $2=f6 $3=f7  

`awk '{print $NF}' test.file`打印一行中最后一个字段。  
`awk 'END{print NR}' text.file`统计文件中的行数。  

**传递外部变量**  
三种方式  
>1.  
>`var=1000`  
>`echo | awk -v var1=$var '{print var1}'`
>  
2.  
>`var1="aaa"`  
>`ar2="bbb"`  
>`echo | awk '{print v1,v2}' v1=$var1 v2=$var2`  

来自文件
>`awk '{print v1,v2}' v1=$var1 v2=$var2 test.txt`  

输出到文件：  
>`echo | awk '{printf("hello word!\n") > "datafile"}'`  
> 或 `echo | awk '{printf("hello word!\n") >> "datafile"}'`

设置字段界定符，默认是空格：  
>`awk -F: '{ print $NF }' /etc/passwd`  
>`或 awk 'BEGIN{ FS=":" } { print $NF }' /etc/passwd`

###更多awk内容参照此链接[link](http://man.linuxde.net/awk)###

##find命令操作##

用于在目录结构中搜索文件，并进行指定的操作。  
命令格式：  
`find pathame -options [-print -exec -ok]`  
命令参数：  
>pathname: find命令所查找的目录路径。   
>-print： find命令将匹配的文件输出到标准输出。   
> -exec： find命令对匹配的文件执行该参数所给出的shell命令。相应命令的形式为'command' {  } \;，注意{   }和\；之间的空格。   
> -ok： 和-exec的作用相同，只不过以一种更为安全的模式来执行该参数所给出的shell命令，在执行每一个命令之前，都会给出提示，让用户来确定是否执行。  

命令选项：  
>-name   按照文件名查找文件。  
>-perm   按照文件权限来查找文件。  
>-prune  使用这一选项可以使find命令不在当前指定的目录中查找，如果同时使用-depth选项，那么-prune将被find命令忽略。  
>-user   按照文件属主来查找文件。  
>-group  按照文件所属的组来查找文件。  
>-mtime -n +n  按照文件的更改时间来查找文件， - n表示文件更改时间距现在n天以内，+ n表示文件更改时间距现在n天以前。find命令还有-atime和-ctime 选项，但它们都和-m time选项。  
>-nogroup  查找无有效所属组的文件，即该文件所属的组在/etc/groups中不存在。  
>-nouser   查找无有效属主的文件，即该文件的属主在/etc/passwd中不存在。  
>-newer file1 ! file2  查找更改时间比文件file1新但比文件file2旧的文件。  
>-type  查找某一类型的文件，诸如：  
>b - 块设备文件。  
>d - 目录。  
>c - 字符设备文件。  
>p - 管道文件。  
>l - 符号链接文件。  
>f - 普通文件。  
>-size n：[c] 查找文件长度为n块的文件，带有c时表示文件长度以字节计。  >-depth：在查找文件时，首先查找当前目录中的文件，然后再在其子目录中查找。  
>-fstype：查找位于某一类型文件系统中的文件，这些文件系统类型通常可以在配置文件/etc/fstab中找到，该配置文件中包含了本系统中有关文件系统的信息。  
>-mount：在查找文件时不跨越文件系统mount点。    
>-follow：如果find命令遇到符号链接文件，就跟踪至链接所指向的文件。  
>-cpio：对匹配的文件使用cpio命令，将这些文件备份到磁带设备中。  
另外,下面三个的区别:  
>-amin n   查找系统中最后N分钟访问的文件  
>-atime n  查找系统中最后n*24小时访问的文件  
>-cmin n   查找系统中最后N分钟被改变文件状态的文件  
>-ctime n  查找系统中最后n*24小时被改变文件状态的文件  
>-mmin n   查找系统中最后N分钟被改变文件数据的文件  
>-mtime n  查找系统中最后n*24小时被改变文件数据的文件

示例：  
`find . -name "*.txt" -exec mv {} {}d \`  
查找当前目录下.txt文件，并转换成.txtd文件，{}表示文件名。  
`find . -name "*.txt" -exec mv {} "awk.txt"`  
表示文件名转成awk.txt。  

批量更改文件后缀名：  
`rename .c  .h   *.c`当前目录下.c更改为.h  
`rename str1 str2 file/dir`是指在匹配此文件或在此目录下(~/*)的所有文件，将文件名字中的str1字符用str2替换。

批量替换：  
`ls *aw* | awk '{org=$0;gsub("aw","wa");system("mv"" "org" "$0)}'`  
将本目录下所有文件名中aw替换成wa。只能在当前目录下  
`find . "*aw*" | awk '{org=$0;gsub("aw","wa");system("mv"" "org" "$0)}`则可以递归实现。  

##[sed命令](http://man.linuxde.net/sed)##
文本编辑器，读取文件每行内容，并进行编辑输出到屏幕。但并不保存。

##[grep工具](http://man.linuxde.net/grep)##
文本搜索工具，并将匹配的内容输出。支持正则表达式。  
命令结构：  
`grep [options] "pattern" filename`  
options可选。选项如下：  
>-a 不要忽略二进制数据。  
>-A<显示列数> 除了显示符合范本样式的那一行之外，并显示该行之后的内容。   
>-b 在显示符合范本样式的那一行之外，并显示该行之前的内容。  
>-c 计算符合范本样式的列数。  
>-C<显示列数>或-<显示列数> 除了显示符合范本样式的那一列之外，并显示该列之前后的内容。   
>-d<进行动作> 当指定要查找的是目录而非文件时，必须使用这项参数，否则grep命令将回报信息并停止动作。   
>-e<范本样式> 指定字符串作为查找文件内容的范本样式。   
>-E 将范本样式为延伸的普通表示法来使用，意味着使用能使用扩展正则表达式。   
>-f<范本文件> 指定范本文件，其内容有一个或多个范本样式，让grep查找符合范本条件的文件内容，格式为每一列的范本样式。   
>-F 将范本样式视为固定字符串的列表。   
>-G 将范本样式视为普通的表示法来使用。   
>-h 在显示符合范本样式的那一列之前，不标示该列所属的文件名称。   
>-H 在显示符合范本样式的那一列之前，标示该列的文件名称。   
>-i 胡列字符大小写的差别。   
>-l 列出文件内容符合指定的范本样式的文件名称。   
>-L 列出文件内容不符合指定的范本样式的文件名称。   
>-n 在显示符合范本样式的那一列之前，标示出该列的编号。   
>-q 不显示任何信息。   
>-R/-r 此参数的效果和指定“-d recurse”参数相同。   
>-s 不显示错误信息。   
>-v 反转查找。   
>-w 只显示全字符合的列。   
>-x 只显示全列符合的列。   
>-y 此参数效果跟“-i”相同。   
>-o 只输出文件中匹配到的部分。

`echo "this is a line." | grep -o -E "[a-z]+\."`  
输出：`line.`  

##|><##
管道命令：左边的标准输出作为右边命令的标准输入。
重定向：  
<从文件而不是从键盘或句柄读入命令输入   
\>输出到指定的设备或文件，覆盖  
\>>将命令输出添加到文件末尾而不删除文件中已有的信息  

##alias、ps、top命令##
alias用来设置指令的别名。  
使用方法：  
`alias 新的命令='原命令 -选项/参数'`  

`ps [options]`显示进程状态，结合kill使用。

`top [options]` 显示系统的整体运行状况。

`free [options]` 显示系统当前已使用和未使用的内存数目。  
`free -m`以MB为单位显示。

`vmstat [options]`命令的含义为显示虚拟内存状态（“Viryual Memor Statics”），但是它可以报告关于进程、内存、I/O等系统整体运行状态。  


#虚拟机网络四种连接模式#
>1. NAT：网络地址转换模式；
>2. Bridged Adpater: 桥接模式；
>3. Internal：内部网络模式；
>4. Host-only Adapter：主机模式。

四种连接方式的区别：  
![虚拟机网络](virtual_network.png)  


##NAT##
单向访问，虚拟机可以访问主机，并且通过主机访问Internet。然后主机及其他主机无法访问虚拟机。虚拟机访问外网时需要做地址转换。  
当有端口映射或利用 VPN，则主机及外网可以访问虚拟机。
主要原理如下：  
![nat](NAT.jpg)  
内部网络共用一个全局IP实现外网的访问。  

##桥接模式##
这种方式是虚拟机的网卡与宿主机的网卡连接起来，在由虚拟机获取一个宿主机IP网段的网络IP，从而达到网络互联的效果。通过这种方式的连接，虚拟机有一个同宿主机在一个网段的iP，并且也有同宿主机一样的网络设置，所以虚拟机可以与宿主机及宿主机网络中的任何机器进行网络互联访问。

##Internal模式##
虚拟机与外网完全断开，只实现虚拟机于虚拟机之间的内部网络模式。

##Host-only Adapter##
我们可以理解为Guest在主机中模拟出一张专供虚拟机使用的网卡，所有虚拟机都是连接到该网卡上的，我们可以通过设置这张网卡来实现上网及其他很多功能，比如（网卡共享、网卡桥接等）。  
虚拟机与主机的关系：默认不能相互访问，双方不属于同一IP段，host-only网卡默认IP段为192.168.56.X 子网掩码为255.255.255.0，后面的虚拟机被分配到的也都是这个网段。通过网卡共享、网卡桥接等，可以实现虚拟机于主机相互访问。  
虚拟机与网络主机的关系：默认不能相互访问，原因同上，通过设置，可以实现相互访问。  
虚拟机与虚拟机的关系：默认可以相互访问，都是同处于一个网段。  
虚拟机访问主机，用的是主机的VirtualBox Host-Only Network网卡的IP：192.168.56.1  ，不管主机“本地连接”有无红叉，永远通。  
主机访问虚拟机，用是的虚拟机的网卡3的IP： 192.168.56.101  ，不管主机“本地连接”有无红叉，永远通。  
虚拟机访问互联网，用的是自己的网卡2， 这时主机要能通过“本地连接”有线上网，（无线网卡不行）  


##iptables##
防火墙软件。临时存储，重启丢失。若需要永久保存：则运行`/sbin/service iptables save`；生成一个`iptables.save`保存旧的规则。更改后的规则将保存在/etc/sysconfig/iptables文件中。
  
语法结构：  
`iptables [options] --`  

选项：  
>-t<表>：指定要操纵的表；   
>-A：向规则链中添加条目；   
>-D：从规则链中删除条目；   
>-i：向规则链中插入条目；   
>-R：替换规则链中的条目；   
>-L：显示规则链中已有的条目；    
>-F：清除规则链中已有的条目；   
>-Z：清空规则链中的数据包计算器和字节计数器；   
>-N：创建新的用户自定义规则链；  
>-m：加载模块，加载iptables功能模块；   
>-P：定义规则链中的默认目标；   
>-h：显示帮助信息；   
>-p：指定要匹配的数据包协议类型；   
>-s：指定要匹配的数据包源ip地址；   
>-j<目标>：指定要跳转的目标；   
>-i<网络接口>：指定数据包进入本机的网络接口；   
>-o<网络接口>：指定数据包要离开本机所使用的网络接口。  

表明包括：
>* raw：高级功能，如：网址过滤；
>* mangle：数据包修改（Qos），用于实现服务质量；
>* net：地址转换，用于网关路由器；
>* filter：包过滤，用于防火墙规则。

规则链名包括：
>* INPUT：处理输入数据包；
>* OUTPUT：处理输出数据包；
>* FORWARD：处理转发数据包；
>* PREROUTING：用于目标地址转换（DNAT）；
>* POSTROUTING：用于源地址转换（SNAT）。

动作包括：
>* ACCEPT：接收匹配规则的数据包；
>* DROP：丢弃匹配规则的数据包；
>* REDIRECT：重定向、映射、透明代理；
>* SNAT：源地址转换；
>* DNAT：目标地址转换；
>* MASQUERADE：IP伪装（ＮＡＴ），用于ＡＤＳＬ；
>* LOG ：日志记录。

操作实例：  
清除已有iptables规则  
> `iptables -F`  
> `iptables -X`  
> `iptables -Z`

开放指定的端口：  
>`iptables -A INPUT -s 127.0.0.1 -d 127.0.0.1 -j ACCEPT`：开放本地回环接口  
>`iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT`：允许已建立的或相关的连接同行  
>`iptables -A OUTPUT -j ACCEPT`：允许所有本机向外的访问  
>`iptables -A INPUT -p tcp --dsport 22 -j ACCEPT`  ：允许访问22端口  
>`iptables -A INPUT -j REJECT`：禁止其他未允许的规则访问  
>`iptables -A FORWARD -j REJECT`：禁止其他未允许的规则访问  

屏蔽IP：(-A和-I参数分别为添加到规则末尾和规则最前面，其中DROP是指丢弃数据包，不回复信息。Reject则是拒绝接入，并响应客户端)  
>`iptables -I INPUT -s 10.0.0.3 -j DROP`：屏蔽单个IP  
>`iptables -I INPUT -s 10.0.2.0/24 -j DROP`：屏蔽一个网段的IP  

查看已添加的iptables规则
> `iptables -L -n -v`：-n :数字格式显示ip和端口，v ： 显示详细信息 -v -vvv -vvvv ..可以显示更详细的信息；  


删除已添加的iptables规则：
> `iptables -L -n --line-numbers`：首先将所有iptables以序号标记显示；  
> `iptables -D [INPUT/OUTPUT/.../FORWARD] [number]`：删除序号number的规则。

##fdisk分区工具##
fdisk参考博客[Link](http://www.aboutyun.com/thread-11682-1-1.html)
>
\>>fdisk /dev/sdb；对sdb(即第二块硬盘，sda表示第一块硬盘)进行分区。  
\>>mkfs.xfs /dev/sdb1；对(第二块硬盘第一个分区)格式化为xfs文件格式。  
\>>mkdir /data;  
\>>mount /dev/sdb1 /home/data；将此硬盘分区挂载到目录/data下。 

##Linux LVM硬盘管理##
参考博客[Link](http://www.aboutyun.com/thread-11683-1-1.html)  

LVM：Logical Volume Manager，逻辑卷管理。LVM将一个或多个硬盘的分区在逻辑上集合，相当于一个大硬盘来使用，当硬盘的空间不够使用的时候，可以继续将其它的硬盘的分区加入其中，这样可以实现磁盘空间的动态管理，相对于普通的磁盘分区有很大的灵活性。  

LVM术语总结：  
>物理存储介质：系统的存储设备：硬盘，如/dev/hda、/dev/sda等。是存储系统最底层的存储单元。  
>物理卷(PV)：物理卷在逻辑卷管理系统最底层，可为整个物理硬盘或实际物理硬盘上的分区。  
>卷组(VG)： 卷组是由物理卷组成，卷组建立后可动态的添加卷到卷组中，一个逻辑卷管理系统工程中可有多个卷组。可以在卷组上创建一个或多个“LVM分区”(逻辑卷)。
>逻辑卷(LV)：LVM的逻辑卷类似于非LVM系统中的硬盘分区，在逻辑卷上可以建立文件系统(如/home、/usr等)。  
>PE(Physical extent)：物理区域是物理卷中可用于分配的最小存储单元，物理区域大小在建立卷组时指定，一旦确定不能更改，同一卷组所有物理卷的物理区域大小需一致，新的pv加入到vg后，pe的大小自动更改为vg中定义的pe大小。  
>LE(Logical extent)：逻辑区域是逻辑卷中可用于分配的最小存储单元，逻辑区域的大小取决于逻辑卷所在卷组中的物理区域的大小。

**建立逻辑卷**  
1. 创建分区
  利用fdisk创建分区，分区类型为8e。假设我们要在硬盘/dev/sdb上创建。  
`sudo fdisk /dev/sdb`  
\>`n`：创建分区  
\>`p`：分区类型为主分区
\>`+1G`：大小为1G  
\>`t`：修改分区格式  
\>`4`：指定分区编号  
\>`8e`：指定分区类型  
\>`p`：查看当前分区。
\>`w`：写入分区表  
2. 创建PV  
\>`sudo partprobe`：使分区表生效  
\>`sudo pvcreate /dev/sdb4`：创建PV  
\>`sudo pvdisplay`：查看已存在的pv  
\>`sudo vgcreate volGroup /dev/sdb4`：创建一个名为volGroup的卷组  
\>`sudo vgdisplay`：查看VG  
3. 创建LV，在VG中进行创建。
\>`sudo lvcreate -L 100M -n lvvlo volGroup`：-L指定逻辑分区的大小，-n指定逻辑分区的名字吗，volGroup表示从此卷组中进行逻辑分区的分配。  
4. LV格式化以及目录挂载  
\>`sudo mkfs.xfs /dev/sdb4/volGroup/lvvlo`：对逻辑分区lvvlo格式化。
\>`mkdir test_lv`  
\>`sudo mount /dev/sdb4/volGroup/lvvlo /home/test_lv`：挂载至目录test_lv下。

**开机自启动**  
为了使开机自启动lvvlo分区，需要将分区信息写入/etc/fstab文件中。  
`/dev/sdb4/volGroup/lvvlo(或是UUID) /home/test_lv xfs default 1,2`  
UUID可以通过运行`sudo blkid`获得。

**分区扩容**  
1. 首先创建分区
\>`fdisk /dev/sdb`  
\>`...`  
\>`sudo partprobe`：使分区表生效  
\>`mkfs.xfs /dev/sdb5`    
2. 创建pv，扩容VG，LV  
\>`pvcreate /dev/sdb5`  
\>`vgdisplay`  
\>`vgextend volGroup /dev/sdb5`：扩展，将物理卷/dev/sdb5添加至此卷组中  
\>`lvdisplay`  
\>`lvextend -L 1G /dev/volGroup/lvvlo`：扩展逻辑卷  
\>`resize2fs /dev/volGroup/lvvlo`：执行重设大小  
\>`df -h`：查看挂载状态
