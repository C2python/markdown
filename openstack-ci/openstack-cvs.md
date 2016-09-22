#Infrastructure User Manual#
-----------------------------------
下图是协作开发的一个流程图：  
![](img/openstack-cvs-workflow.png)  

这篇文档主要是介绍为openstack贡献源码前的准备工作。主要是关于如何使用openstack开发的协同工作软件。如在指定网站注册用户、签署版权、上传ssh key、以及安装git-review。  

**创建账户**  
Gerrit Code Review系统利用创建的账户识别用户。首次进入Gerrit，需要创建一个用户名，用户名一经创建便不可更改，将作为永久ＩＤ。
　　　

Gerrit界面：  
![](img/openstack-cvs-gerrit.png)  

进入Gerrit后，在setting中上传自己的ssh key。  
配置git用户和账号信息：  
>git config --global -----  

**[安装git-review](http://www.cnblogs.com/Security-Darren/p/4383838.html)**  
git-review是git的一个子命令，主要用于和OpenStack的代码review系统Gerrit交互，git-review和Gerrit的交互是一组Git命令，在每个git review命令后添加-v选项可以打印所有运行的Git命令。
>apt-get install git-review  
>yum install git-review  

直接运行`git review0`push到Gerrit。Gerrit是OpenStack远端Git仓库的一道大门，所有的submission都要在这里经过review后才能被merge到master分支中，因此之前的工作一定不能在master分支进行，这样会产生一个merge commit，Gerrit默认是不接受merge commit的。如果提交成功，Gerrit将返回一个显示你此次提交内容的URL，打开它就可以查看commit以及reviewer的评价了。  

##工作流程##
克隆仓库：  
>git clone https://git.openstack.org/openstack/<projectname>.git  

**Starting a Change**  
确保本地库版本是最新的：  
>git remote update  
git checkout master  
git pull --ff-only origin master  

创建分支并切换。分支的命名要紧贴自己的工作。如修复bug，命名为bug/BUG\_NUMBER。  
>git checkout -b TOPIC-BRANCH  

**Committing a change**  
在提交之前，运行单元测试，参加[Running Python Unit Tests](http://docs.openstack.org/project-team-guide/project-setup/python.html#running-python-unit-tests)。  

**确认change**  
>git show  

**submitting a change for review**  
>git review  

提交之后，将会对改动部分进行自动的测试，其他开发者也会进行交叉review。

**updating a change**  
>git commit -a --amend  
>git review  

**Adding a Dependency**  
当需要基于一个正在review的commit开始新的工作时，可以添加新的提交作为依赖项。  
>git review -d $PARENT\_CHANGE\_NUMBER  
git checkout -b $DEV\_TOPIC\_BRANCH   

编辑文件，提交文件：  
>git commit -a  
git review  

当我们一各依赖的commit发生改变时，需要已提交的最新的而补丁：  
Fetch and checkout the parent change:  
>git review -d $PARENT_CHANGE_NUMBER  

cherry-pick:  
>git review -x $CHILD_CHANGE_NUMBER  
>git review  




##Core Reviewers's Guide##

**+/-2 votes**  
>  

**Approval**  

**Work In Progress**  
> change owner 设置表示此处工作仍需要进一步的更新。  
> core reviewers 设置表示告诉contributor此处change需要更多的工作。  

**Re-approval**  


##Project Driver's GUIDE##




##Zuul##
Configuration：  

+ zuul.conf  
+ layout.yaml  
+ logging.conf  

[detailed](http://docs.openstack.org/infra/zuul/zuul.html)  

zuul可以通过pip安装：pip install zuul；
zuul主要需要以下的组件：
  
+ zuul-server：调度deamon与gerrit以及Gearman通信。处理监听到的事件、实例会job、收集结果和报告；
+ zuul-merger：与gearman通信，准备git repo用于测试；
+ zuul-clone：客户端脚本，用于准备job的工作环境，clone merger准备的repo；
+ gearmand：可选的内建gearman deamon。

各个组件通信都是依赖于gearman，同时与gerrit,jenkins的通信也是如此：  
    zuul-server:
      Gerrit
      Gearman Daemon

    zuul-merger:
      Gerrit
      Gearman Daemon

    zuul-cloner:
      http hosted zuul-merger git repos

    Jenkins:
      Gearman Daemon via Jenkins Gearman Plugin


zuul的运行至少需要zuul.conf以及layout.yaml两个配置文件，位于/etc/zuul/目录下。配置文件格式如下：  
**zuul.conf：**  

    [zuul]
    layout_config=/etc/zuul/layout.yaml

    [merger]
    git_dir=/git
    zuul_url=http://zuul.example.com/p

    [gearman_server]
    start=true

    [gearman]
    server=127.0.0.1

    [connection gerrit]
    driver=gerrit
    server=git.example.com
    port=29418
    baseurl=https://git.example.com/gerrit/
    user=zuul
    sshkey=/home/zuul/.ssh/id_rsa

**layput.yaml：**  

    pipelines:
       - name: periodic
       source: gerrit
       manager: IndependentPipelineManager
       trigger:
         timer:
           - time: '0 * * * *'

    projects:
       - name: aproject
       periodic:
         - aproject-periodic-build


#各个节点的安装和配置#
[参见](https://git.openstack.org/cgit/openstack-infra/)  
包含了jenkins，Nodepool，Gerrit，etc等节点的安装和配置。
jenkins puppet代码配置信息解析：  

##手动安装gerrit##
 ` mysql`  

  `CREATE USER 'gerrit2'@'localhost' IDENTIFIED BY 'secret';`  
  `CREATE DATABASE reviewdb;`  
  `ALTER DATABASE reviewdb charset=latin1;`  
  `GRANT ALL ON reviewdb.* TO 'gerrit2'@'localhost';`  
  `FLUSH PRIVILEGES;`  

`sudo apt-get install -y nginx`
  
`sudo /etc/init.d/nginx restart`  
  
`wget http://gerrit-releases.storage.googleapis.com/gerrit-full-2.5.2.war`  

`java -jar gerrit-full-2.5.2.war init -d ~/gerrit_site`

Mysql;  
http    
SMTP: smtp.yeah.net  
smtp port:25  
smtp user:wehzhangyuan@yeah.net  
  
`cd /etc/nginx/sites-available`  
添加：  
`server {`  
  `listen *:8082;`  
  `server_name cinnode;`  
  `allow   all;`  
  `deny    all;`  
  `auth_basic "THSTACK INC. Review System Login";`  
  `auth_basic_user_file /home/gerrit2/gerrit_site/etc/htpasswd.conf;`  

  `location / {`   
    `proxy_pass  http://10.0.1.2:8081;`  
  `} `    
`}`

`touch /home/gerrit2/gerrit_site/etc/htpasswd.conf`  
`htpasswd htpasswd.conf admin`  
input passwd:  



重启启动：  

`\# /home/gerrit2/gerrit_site/bin/gerrit.sh start   //启动`

 

`\# /home/gerrit2/gerrit_site/bin/gerrit.sh stop   // 停止`

 

`\# /home/gerrit2/gerrit_site/bin/gerrit.sh restart //重启`

访问：http://10.0.1.2:8082

非HTTP认证： development\_become\_any\_account  
利用Nginx实现http认证时，无法使用sign_out，只能关闭当前浏览器结束会话。  

##Jenkins安装配置##  
`wget http://pkg.jenkins-ci.org/debian/binary/jenkins_1.544_all.deb`  



#sys-config#
实现系统的整合，对已安装的各个节点配置。
Entry-point：manifests/site.pp；  
包含了各个节点的配置信息。
    
+ *install_puppet.sh*  
安装puppet。
+ *install_modules.sh*  
下载modules.env中所有source指向的包。放在puppet的modulepath下。用于后续利用puppet进行安装。  
+ *run\_puppet.sh*  
安装节点对应的模块，其运行核心运行代码如下：  
`MODULE_DIR=${BASE_DIR}/modules`  
`MODULE_PATH=${MODULE_DIR}:/etc/puppet/modules`
`/usr/bin/git pull -q && /bin/bash install_modules.sh && /usr/bin/puppet apply -l $MANIFEST_LOG --modulepath=$MODULE_PATH manifests/site.pp`  
主要功能：更新，运行site.pp，其模块主要在两个地方1、MODULE_DIR；2、/etc/puppet/modules。当前工作目录是BASE_DIR。  
在运行intsall_modules.sh后，会在/etc/puppet/module目录下存储一些组件的安装模块。同时，/sys-config/module也有相应的模块。  




#project-config#
配置openstack项目。利用CI系统进行openstack开发环境的配置。  


#Openstack Third-Party CI System#
安装：将jenkins，zuul，nodepool，Jenkins Job Builder，os-loganalyze安装在同一台主机中。  
项目地址：[openstackci](https://git.openstack.org/cgit/openstack-infra/puppet-openstackci/)  

puppet入口：/contrib/single\_node\_ci\_site；  
类的定义位于：/manifests/single\_node\_ci.pp中。


需要对[project config](http://git.openstack.org/cgit/openstack-infra/project-config-example)进行更改。  
  
##Jenkins的安装配置##
入口single\_node\_ci.pp中：
  
    class { '::openstackci::jenkins_master':
      vhost_name              => $jenkins_vhost_name,
      serveradmin             => $serveradmin,
      jenkins_ssh_private_key => $jenkins_ssh_private_key,
      jenkins_ssh_public_key  => $jenkins_ssh_public_key,
      manage_jenkins_jobs     => true,
      jenkins_url             => 'http://127.0.0.1:8080/',
      jenkins_username        => $jenkins_username,
      jenkins_password        => $jenkins_password,
      project_config_repo     => $project_config_repo,
      log_server              => $log_server,
      jjb_git_revision        => $jjb_git_revision,
      jjb_git_url             => $jjb_git_url,
    }

配置参数：
  
+ vhostname：Jenkins运行主机的FQDN，可以用IP代替；  
+ serveradmin：CI系统拥有者的email；  
+ jenkins\_ssh\_provate_key：jenkins master用来登录slave的私钥；  
+ jenkins\_ssh\_public\_key：相对于jenkins\_ssh\_private\_key的公钥，不能包含空格，需要剔除'ssh-rsa'的头部前缀以及email address的后缀；  
+ jenkins\_url：jenkins的监听端口；
+ jenkins\_username：与jenkins secure相关。若jenkins采用了安全策略，此用户是Jenkins Job Builder用来管理所有的jenkins jos。若为采用secure，可忽略。(Jenkins在安装时，会默认生成一个用户名为jenkins的用户。)  
+ jenkins\_password：jenkins\_username对应的密码；  
project\_config\_repo：git url，对jenkins jobs，nodepool，zuul的配置。使之可以承载openstack的开发集成测试环境。(所提供的projec-config-example需要更改，参加README)；
+ log\_server：日志服务器的域名或IP。Job运行后，运行日志会被上传至日志服务器。Jenkins利用jenkins\_ssh\_provate\_key进行文件的scp传输。  
+ jjb\_git\_revision：Jenkins\_job\_builder的版本，默认最新master；  
+ jjb\_git\_url：Jenkins\_job\_builder的安装源，默认'https://git.openstack.org/openstack-infra/jenkins-job-builder'。  


Jenkins master类的定义位于/openstackci/manifests/jenkins\_master.pp中，接口定义如下：  

    class openstackci::jenkins_master (
      $serveradmin,
      $jenkins_password,
      $jenkins_username        = 'jenkins',
      $vhost_name              = $::fqdn,
      $logo                    = '', # Logo must be present in puppet-jenkins/files
      $ssl_cert_file           = '/etc/ssl/certs/ssl-cert-snakeoil.pem',
      $ssl_key_file            = '/etc/ssl/private/ssl-cert-snakeoil.key',
      $ssl_chain_file          = '',
      $ssl_cert_file_contents  = '',
      $ssl_key_file_contents   = '',
      $ssl_chain_file_contents = '',
      $jenkins_ssh_private_key = '',
      $jenkins_ssh_public_key  = '',
      $manage_jenkins_jobs     = false,
      $jenkins_url             = 'http://localhost:8080',
      $jjb_update_timeout      = 1200,
      $jjb_git_url             = 'https://git.openstack.org/openstack-infra/jenkins-job-builder',
      $jjb_git_revision        = 'master',
      $project_config_repo     = '',
      $project_config_base     = '',
      $log_server              = undef,
    )
其中ssl_key_file和ssl_cert_file在安装apache2时自动生成。  
其包含一个jenkins::master以及jenkins::job\_builder的声明，安装相应的插件，配置log\_server。
jenkins::master的声明如下：  
  
    class { '::jenkins::master':
      vhost_name              => $vhost_name,
      serveradmin             => $serveradmin,
      logo                    => $logo,
      ssl_cert_file           => $ssl_cert_file,
      ssl_key_file            => $ssl_key_file,
      ssl_chain_file          => $ssl_chain_file,
      ssl_cert_file_contents  => $ssl_cert_file_contents,
      ssl_key_file_contents   => $ssl_key_file_contents,
      ssl_chain_file_contents => $ssl_chain_file_contents,
      jenkins_ssh_private_key => $jenkins_ssh_private_key,
      jenkins_ssh_public_key  => $jenkins_ssh_public_key,
    }

在jenkins::master的定义中，主要是安装一些基础环境，安装pip，apt，httpd，openjdk7等工具。  
jenkins::job\_builder的声明如下：  

    class { '::jenkins::job_builder':
      url                         => $jenkins_url,
      username                    => $jenkins_username,
      password                    => $jenkins_password,
      jenkins_jobs_update_timeout => $jjb_update_timeout,
      git_revision                => $jjb_git_revision,
      git_url                     => $jjb_git_url,
      config_dir                  => $::project_config::jenkins_job_builder_config_dir,
      require                     => $::project_config::config_dir,
    }

主要关于jenkins job builder的安装。  


##基于puppet的Zuul安装配置##
zuul主要安装zuul-merger，以及zuul-scheduler。其配置参数如下：  

    # Zuul Configurations
    $gerrit_server                 = 'review.openstack.org',
    $gerrit_user                   = undef,
    $gerrit_user_ssh_public_key    = undef,
    $gerrit_user_ssh_private_key   = undef,
    $gerrit_ssh_host_key           = 'review.openstack.org,23.253.232.87,2001:4800:7815:104:3bc3:d7f6:ff03:bf5d b8:3c:72:82:d5:9e:59:43:54:11:ef:93:40:1f:6d:a5',
    $git_email                     = undef,
    $git_name                      = undef,
    $log_server                    = undef,
    $smtp_host                     = 'localhost',
    $smtp_default_from             = "zuul@${vhost_name}",
    $smtp_default_to               = "zuul.reports@${vhost_name}",
    $zuul_revision                 = 'master',
    $zuul_git_source_repo          = 'https://git.openstack.org/openstack-infra/zuul',

参数说明：  

+ gerrit\_server：gerrit服务器域名；    
+ gerrit\_user：用于登录gerrit server的用户名，需要在gerrit server中注册。并上传自己的公钥；    
+ gerrit\_user\_ssh\_public\_key：gerrit user的公钥；  
+ gerrit\_user\_ssh\_private\_key：gerrit user的私钥；  
+ gerrit\_ssh\_host\_key：gerrit server的host key。/etc/ssh/*中；  
+ git\_email：zuul用于内部git commit的email；  
+ git\_name：zuul用于内部git commit的email；  
+ log\_server：日志服务器的FQDN或IP；  
+ smtp\_host：zuul用于发送notification的smtp hostname；  
+ smtp\_default\_from：在发送邮件时显示的from 邮件地址；  
+ smtp\_default\_to：目的邮件地址；  
+ zuul\_revision：使用zuul版本；　　
+ zuul\_git\_source\_repo：zuul的源码文件的repo库。　　

zuul主要安装zuul merger和zuul scheduler。zuul merger的声明如下：  

    class { '::openstackci::zuul_merger':
      vhost_name           => $vhost_name,
      gearman_server       => 'localhost',
      gerrit_server        => $gerrit_server,
      gerrit_user          => $gerrit_user,
      # known_hosts_content is set by openstackci::zuul_scheduler
      known_hosts_content  => '',
      zuul_ssh_private_key => $gerrit_user_ssh_private_key,
      zuul_url             => "http://${vhost_name}/p/",
      git_email            => $git_email,
      git_name             => $git_name,
      manage_common_zuul   => false,
      revision             => $zuul_revision,
      git_source_repo      => $zuul_git_source_repo,
    }

定义文件在/openstackci/manifests/zuul\_merger.pp中：  
定义文件中包含了zuul模块的初始化声明，以及zuul:merger的声明，包括关于zuul用户known\_hosts的创建。  

    class { '::zuul':
      vhost_name           => $vhost_name,
      gearman_server       => $gearman_server,
      gerrit_server        => $gerrit_server,
      gerrit_user          => $gerrit_user,
      zuul_ssh_private_key => $zuul_ssh_private_key,
      layout_file_name     => $layout_file_name,
      zuul_url             => $zuul_url,
      git_email            => $git_email,
      git_name             => $git_name,
      revision             => $revision,
      git_source_repo      => $git_source_repo,
    }
这是关于zuul的初始化，初始化包括了一些依赖包的安装，以及zuul的安，zuul的安装通过git\_source\_repo包来进行安装的。初始化的安装中包括graphitejs包的安装，其source repo => 'https://github.com/prestontimmons/graphitejs.git'。  
zuul.scheduler包含zuul的初始化安装以及开启相应的服务，建立log文件。

##基于puppet的nodepool安装##
资源声明：  

    class {'::openstackci::nodepool':
        mysql_root_password       => hiera('mysql_root_password'),
        mysql_password            => hiera('mysql_nodepool_password'),
        nodepool_ssh_private_key  => hiera('jenkins_ssh_private_key'),
        revision                  => hiera('nodepool_revision', 'master'),
        git_source_repo           => hiera('nodepool_git_source_repo', 'https://git.openstack.org/openstack-infra/nodepool'),
        oscc_file_contents        => hiera('oscc_file_contents', ''),
        environment               => {
          # Set up the key in /etc/default/nodepool, used by the service.
          'NODEPOOL_SSH_KEY' => hiera('jenkins_ssh_public_key')
        },
        project_config_repo       => hiera('project_config_repo'),
        # Disable nodepool image logs as it conflicts with the zuul status page
        enable_image_log_via_http => false,
        jenkins_masters           => [
          { name        => hiera('nodepool_jenkins_target', 'jenkins1'),
            url         => 'http://localhost:8080/',
            user        => hiera('jenkins_username', 'jenkins'),
            apikey      => hiera('jenkins_api_key', 'XXX'),
            credentials => hiera('jenkins_credentials_id', 'XXX'),
          },
        ],
    }

参数说明：  
mysql\_root\_passwd：本机mysql root用户的密码。  
mysql\_passwd：用于设置本机nodepool 的MySQL密码。  
nodepool\_ssh\_private\_key：用于登录jenkins slave节点的私钥。  
revision：nodepool的安装版本。  
git\_source\_repo：nodepool安装包的源。
oscc\_file\_contents：os-client-config的配置信息，用于使用openstack所需的一些客户端配置信息。   
environments：公钥。  
project\_config\_repo：项目配置文件的源码库。  
jenkins\_master:jenkins master的访问地址，以及jenkins配置了secrue连接的一些认证信息。 

/modules/nodepool/manifests/init.pp初始化nedopool，并设置clounds.yaml文件。


##[os-client-config](http://docs.openstack.org/developer/os-client-config/)##
是利用客户端连接openstack所需的配置信息。配置文件clouds.yaml通常在目录

	Current Directory
	~/.config/openstack
	/etc/openstack
第一个匹配为准。其配置信息格式如下：  

	clouds:
	  mtvexx:
	    profile: vexxhost
	    auth:
	      username: mordred@inaugust.com
	      password: XXXXXXXXX
	      project_name: mordred@inaugust.com
	    region_name: ca-ymq-1
	    dns_api_version: 1
	  mordred:
	    region_name: RegionOne
	    auth:
	      username: 'mordred'
	      password: XXXXXXX
	      project_name: 'shade'
	      auth_url: 'https://montytaylor-sjc.openstack.blueboxgrid.com:5001/v2.0'
	  infra:
	    profile: rackspace
	    auth:
	      username: openstackci
	      password: XXXXXXXX
	      project_id: 610275
	    regions:
	    - DFW
	    - ORD
	    - IAD

其包含了cloud，region，以及username和passwd，endpoint。  

使用：  
构造rest api client。

	import os_client_config
	
	session = os_client_config.make_rest_client('compute', cloud='vexxhost')
	
	response = session.get('/servers')
	server_list = response.json()['servers']  

