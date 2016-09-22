#Openstack CI Installation and Confiuring#
所需安装包：  
开源puppet模块的集合可参见[Link](http://github.com/example42/puppet-modules.git)
##Gerrit##
安装包：gerrit-2.10.war  
依赖包：  
##Jenkins##
Jenkins的puppet安装脚本入口：/system-congfig/manifests/site.pp，line143：  

`node /^jenkins\d+\.openstack\.org$/ {`  
  `$group = "jenkins"`  
  `$zmq_event_receivers = ['logstash.openstack.org',
                          'nodepool.openstack.org']`
  `$zmq_iptables_rule = regsubst($zmq_event_receivers,
                                '^(.*)$', '-m state --state NEW -m tcp -p tcp --dport 8888 -s \1 -j ACCEPT')`  
  `$http_iptables_rule = '-m state --state NEW -m tcp -p tcp --dport 80 -s nodepool.openstack.org -j ACCEPT'`  
  `$https_iptables_rule = '-m state --state NEW -m tcp -p tcp --dport 443 -s nodepool.openstack.org -j ACCEPT'`  
  `$iptables_rule = flatten([$zmq_iptables_rule, $http_iptables_rule, $https_iptables_rule])`   
  `class { 'openstack_project::server':`  
    `iptables_rules6     => $iptables_rule,`  
    `iptables_rules4     => $iptables_rule,`  
    `sysadmins           => hiera('sysadmins', []),`  
    `puppetmaster_server => 'puppetmaster.openstack.org',`  
  `}`  
  `class { 'openstack_project::jenkins':`  
    `jenkins_password        => hiera  （'jenkins_jobs_password'),`  
    `jenkins_ssh_private_key => hiera('jenkins_ssh_private_key_contents'),`  
    `ssl_cert_file           => '/etc/ssl/certs/ssl-cert-snakeoil.pem',`  
    `ssl_key_file            => '/etc/ssl/private/ssl-cert-snakeoil.key',`  
    `ssl_chain_file          => '',`  
  `}`  
`}`  

其中openstack\_project::server是关于所有运行主机的基本配置。  
server.pp位于moudles/openstack\_project/manifests/下。
puppetmaster\_server可以不装

参数说明：  

jenkins_password：

##Zuul##
##Gearman##
##Nodepool##