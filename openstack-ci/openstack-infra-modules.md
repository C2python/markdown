#Openstack协作平台的搭建#
-----------------------------------
**主要目标：  
23日，厂商搭建openstack协作平台，学习记录厂商协作平台包含的组件，搭建的工具，搭建方法，学会搭建此平台。  
与openstack infra的开源平台做对比，增加或缺少的组件。**  
积极请教不明白的地方，如Host_key，jenkins jobs的配置，以及如何实现自动化测试。zuul的配置。gerrit的权限管理。Nodepool需要的配置。  


##[Openstack Infra组件](http://docs.openstack.org/infra/system-config/)##
安装工具：puppet，git  
基础组件：
  
+ Gerrit：代码审查平台，基于组的权限管理，zuul监听gerrit。本地库备份至gitlab。  
+ [Zuul](http://docs.openstack.org/infra/system-config/zuul.html)：面向pipeline的项目网关系统。响应Gerrit事件自动执行jenkins jobs进行测试。  
+ Jenkins：包含[Jenkins job builder](http://docs.openstack.org/infra/system-config/jjb.html)，JJB利用仓库中YAML文件来构建jenkins jobs。四类YAML文件，default.yaml包含了jobs的默认配置；macros.yaml宏定义，在project.yaml中引用；project.yaml，job的具体化的配置，可以利用宏，也可以使用模板。template.yaml，模板，实现job的复用。zuul直接读取JJB的配置文件定义job，从而未实际使用jenkins。  
+ Nodepool：管理虚拟机，进行devstack image的安装测试。  
+ Devstack Gate：通过Devstack部署openstack的脚本，用于测试。
+ LogStash：日志检索搜索引擎，记录Jobs测试信息。

重要组件：  

+ Cacti：记录每个虚拟机的资源使用状况。
+ Graphan：图形工具，在网站上动态展示各组件的状态。
+ Grafyaml：基于yaml文件配置Graphan的Dashboard。
+ Elastic-Recheck：利用elastic-search及logstash追踪openstack gate的recheck的信息。
+ Jeepyb：管理Gerrit的脚本集合。
+ IRC Service：通信工具。
+ Etherpad：实时的文档编辑协作工具。
+ Paste：文件共享工具。
+ Planet：用于生成静态文件，归档工具。
+ Stackalytics：可视化工具，收集项目的状态信息如commit，changes等，在Dashboard中显示。
+ Static Web Hosting：静态web服务器，官网的各种资料。
+ Bandersnatch：pypi的mirror，开发使用。
+ MailingLists：邮件服务器。
+ Wiki：百科平台。
+ OpenstackIdServer：在线认证，用户服务。
+ StoryBoard：项目跟踪管理工具。
+ Kerberos：网络认证协议，为AFS提供基于"tickets"的安全连接。
+ OpenAFS：全局的分布式文件系统。
+ Askbot：Q&A的站点。
+ Apps Site：app分类导航应用 用于openstack的各种模块和服务展示。
+ Translate：文档翻译系统，有后端应用的支持。
+ OpenStack-Health：为CI的测试结果提供一个可视化的页面展示。
+ Refstack：测试报告的展示。
+ Code Search：源码搜索平台，与版本仓库配置。
+ Signing System：签名系统。
+ Firehose：通信管道，内部服务之间的通信。


#sys-config#
安装所有模块，实现系统的整合。
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
配置openstack项目。利用CI系统进行openstack开发环境的配置，主要包括jenkins jobs的配置文件，nodepool以及zuul的配置。  


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