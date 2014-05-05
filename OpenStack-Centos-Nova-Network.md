####OpenStack Havana Nova-Network on Centos 6.4安装记录

#####Author
	nate.yu <nate_yhz@outlook.com>
	
#####Requirements
	CentOS release 6.4 	x86_64
	
#####说明
	安装流程参考了网上信息，个人记录，请勿使用，发生一切事情，后果自负！！！
	
#####安装内容
*	[网络说明](#网络说明)
*	[安装基础软件](#安装基础软件)
*	[安装MySQL](#安装mysql)
*	[安装RabbitMQ](#安装rabbitmq)
*	[安装OpenStack工具包](#安装openstack工具包)
*	[安装Keystone](#安装keystone)
*	[安装Glance](#安装glance)
*	[安装Nova](#安装nova)
*	[安装Horizon](#安装horizon)
*	[相关错误及解决方法](#相关错误及解决方法)

#####网络说明
	eth0 接外部网络
	eth1 接内部网络 禁用DHCP
	
#####安装基础软件

*	修改源

		sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-Base.repo
		
		sed -i 's/#baseurl=http:\/\/mirror.centos.org/baseurl=http:\/\/mirrors.yun-idc.com/g' /etc/yum.repos.d/CentOS-Base.repo
		
		rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
		
		rpm -Uvh http://pkgs.repoforge.org/rpmforge-release/rpmforge-release-0.5.3-1.el6.rf.x86_64.rpm
		
		yum install http://repos.fedorapeople.org/repos/openstack/openstack-havana/rdo-release-havana-8.noarch.rpm
		
		yum update
		
*	安装vim gcc gcc-c++ make cmake

		yum install vim gcc gcc-c++ make cmake
		
*	修改主机名

		vim /etc/sysconfig/network
		HOSTNAME=openstack
*	修改hosts

		vim /etc/hosts
		127.0.0.1 openstack
*	关闭selinux

		vim /etc/selinux/config
		SELINUX=disabled
*	设置转发

		vim /etc/sysctl.conf
		net.ipv4.ip_forward = 1
		
		sysctl -p 
*	安装NTP

		yum -y install ntp
		
		vim /etc/ntp.conf
		driftfile /var/lib/ntp/drift
		restrict default ignore
		restrict 127.0.0.1 
		restrict 192.168.10.0 mask 255.255.255.0 nomodify notrap
		server ntp.api.bz
		server  127.127.1.0     # local clock
		fudge   127.127.1.0 stratum 10
		keys /etc/ntp/keys
		
		service ntpd start
		 
		chkconfig ntpd on
		
#####安装MySQL

*	安装

		yum -y install mysql mysql-server MySQL-python
	
*	修改配置文件

		vim /etc/my.cnf
		[mysqld]
		bind-address = 0.0.0.0  
*	启动

		service mysqld start
*	设置开机启动

		chkconfig mysqld on
*	修改密码

		mysqladmin -uroot password '123123'; history -c
*	重启

		service mysqld restart

#####安装RabbitMQ
*	安装

		yum -y install rabbitmq-server
*	启动

		service rabbitmq-server start
*	修改密码

		rabbitmqctl change_password guest nate123

*	设置开机启动

		chkconfig rabbitmq-server on
*	重启

		service rabbitmq-server restart
		
#####安装OpenStack工具包
*	安装

		yum -y install openstack-utils

#####安装Keystone
*	安装

		yum -y install openstack-keystone
*	创建keystone 数据库

		openstack-db --init --service keystone
*	修改配置

		openstack-config --set /etc/keystone/keystone.conf sql connection mysql://keystone:keystone@localhost/keystone
*	创建设置环境变量文件

		openssl rand -hex 10
		
		vim ~/creds
		export OS_USERNAME=admin
		export OS_TENANT_NAME=admin
		export OS_PASSWORD=123123
		export OS_AUTH_URL=http://127.0.0.1:5000/v2.0
		export SERVICE_TOKEN=上面openssl得到的值
		export SERVICE_ENDPOINT=http://127.0.0.1:35357/v2.0
		
		source ~/creds
*	配置token

		openstack-config --set /etc/keystone/keystone.conf DEFAULT admin_token $SERVICE_TOKEN
*	创建密钥

		keystone-manage pki_setup --keystone-user keystone --keystone-group keystone
*	设置访问权限

		chown -R keystone:keystone /etc/keystone/* 
		chown keystone:keystone /var/log/keystone/keystone.log
*	启动

		service openstack-keystone start
*	设置开机启动

		chkconfig openstack-keystone on
*	重启

		service openstack-keystone restart
*	创建管理员

		keystone user-create --name=admin --pass=123123 --email=nate_yhz@outlook.com
*	创建管理员角色

		keystone role-create --name=admin
*	创建admin & service 的tenant

		keystone tenant-create --name=admin --description='Admin Tenant'
		keystone tenant-create --name=service --description='Service Tenant'
*	绑定用户，角色和租户

		keystone user-role-add --user=admin --tenant=admin --role=admin
*	创建服务

		keystone service-create --name=keystone --type=identity --description="KeystoneIdentity Service"
*	创建endpoint
		
		外部IP
		export ip=192.168.0.100

		获取 service id 
		keystone service-list 		
		keystone endpoint-create --service-id=上面命令获取的service_id --publicurl=http://$ip:5000/v2.0 --internalurl=http://$ip:5000/v2.0 --adminurl=http://$ip:35357/v2.0
		
#####安装Glance
*	安装

		yum -y install openstack-glance
*	创建数据库

		openstack-db --init --service glance
*	修改配置

		openstack-config --set /etc/glance/glance-api.conf DEFAULT sql_connection mysql://glance:glance@localhost/glance
		openstack-config --set /etc/glance/glance-registry.conf DEFAULT sql_connection mysql://glance:glance@localhost/glance
*	创建glance用户

		keystone user-create --name=glance --pass=123123 --email=nate_yhz@outlook.com
*	绑定用户

		keystone user-role-add --user=glance --tenant=service --role=admin
*	创建服务

		keystone service-create --name=glance --type=image --description="Glance ImageService"
*	创建endpoint

		外部IP
		export ip=192.168.0.100

		获取 service id 
		keystone service-list
		keystone endpoint-create --service-id=上面命令获取的service_id --publicurl=http://$ip:9292 --internalurl=http://$ip:9292 --adminurl=http://$ip:9292
*	修改glance-api.conf

		openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_host 127.0.0.1
		openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_port 35357
		openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_protocol http
		openstack-config --set /etc/glance/glance-api.conf keystone_authtoken admin_tenant_name service
		openstack-config --set /etc/glance/glance-api.conf keystone_authtoken admin_user glance
		openstack-config --set /etc/glance/glance-api.conf keystone_authtoken admin_password 123123
		
		openstack-config --set /etc/glance/glance-api.conf DEFAULT notifier_strategy rabbit
		openstack-config --set /etc/glance/glance-api.conf DEFAULT rabbit_password nate123
		
		
*	修改glance-registry.conf

		openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken auth_host 127.0.0.1
		openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken auth_port 35357
		openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken auth_protocol http
		openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken admin_tenant_name service
		openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken admin_user glance
		openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken admin_password 123123
*	修改ini文件

		cp /usr/share/glance/glance-api-dist-paste.ini /etc/glance/glance-api-paste.ini
		cp /usr/share/glance/glance-registry-dist-paste.ini /etc/glance/glance-registry-paste.ini
		chown -R root:glance /etc/glance/glance-api-paste.ini 
		chown -R root:glance /etc/glance/glance-registry-paste.ini

		openstack-config --set /etc/glance/glance-api.conf paste_deploy config_file /etc/glance/glance-api-paste.ini
		openstack-config --set /etc/glance/glance-api.conf paste_deploy flavor keystone
		openstack-config --set /etc/glance/glance-registry.conf paste_deploy config_file /etc/glance/glance-registry-paste.ini
		openstack-config --set /etc/glance/glance-registry.conf paste_deploy flavor keystone
		
		openstack-config --set /etc/glance/glance-api-paste.ini filter:authtoken auth_host 127.0.0.1
		openstack-config --set /etc/glance/glance-api-paste.ini filter:authtoken admin_tenant_name service
		openstack-config --set /etc/glance/glance-api-paste.ini filter:authtoken admin_user glance
		openstack-config --set /etc/glance/glance-api-paste.ini filter:authtoken admin_password 123123
		
		openstack-config --set /etc/glance/glance-registry-paste.ini filter:authtoken auth_host 127.0.0.1
		openstack-config --set /etc/glance/glance-registry-paste.ini filter:authtoken admin_tenant_name service
		openstack-config --set /etc/glance/glance-registry-paste.ini filter:authtoken admin_user glance
		openstack-config --set /etc/glance/glance-registry-paste.ini filter:authtoken admin_password 123123
		
*	启动

		service openstack-glance-api start
		service openstack-glance-registry start
*	设置开机自启动

		chkconfig openstack-glance-api on
		chkconfig openstack-glance-registry on
*	重启
		
		service openstack-glance-api restart
		service openstack-glance-registry restart
*	测试

		glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
*	列出所有映像

		glance image-list
		
#####安装Nova
*	安装

		yum -y install openstack-nova
*	创建数据库

		openstack-db --init --service nova
*	创建nova用户

		keystone user-create --name=nova --pass=123123 --email=nate_yhz@outlook.com
*	绑定用户

		keystone user-role-add --user=nova --tenant=service --role=admin
*	创建服务

		keystone service-create --name=nova --type=compute --description="Nova Compute Service"

*	创建endpoint

		外部IP
		export ip=192.168.0.100

		获取 service id 
		keystone service-list
		keystone endpoint-create --service-id=上面命令获取的service_id --publicurl=http://$ip:8774/v2/%\(tenant_id\)s --internalurl=http://$ip:8774/v2/%\(tenant_id\)s --adminurl=http://$ip:8774/v2/%\(tenant_id\)s
*	修改nova.conf

		vim /etc/nova/nova.conf
		[DEFAULT]
		my_ip = 192.168.0.100
		auth_strategy = keystone
		state_path = /var/lib/nova
		verbose=True

		allow_resize_to_same_host = true
		rpc_backend=nova.openstack.common.rpc.impl_kombu
		rabbit_host = localhost
		rabbit_port = 5672
		rabbit_password = nate123
		libvirt_type = kvm
		glance_api_servers = 192.168.0.100:9292

		novncproxy_base_url = http://192.168.0.100:6080/vnc_auto.html
		vncserver_listen = $my_ip
		vncserver_proxyclient_address = $my_ip
		vnc_enabled = true
		vnc_keymap = en-us
 
		network_manager = nova.network.manager.FlatDHCPManager
		firewall_driver = nova.virt.firewall.NoopFirewallDriver
		multi_host = True
		flat_interface = eth1
		flat_network_bridge = br1
		public_interface = eth0

		instance_usage_audit = True
		instance_usage_audit_period = hour
		notify_on_state_change = vm_and_task_state
		notification_driver = nova.openstack.common.notifier.rpc_notifier

		compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
		[hyperv]
		[zookeeper]
		[osapi_v3]
		[conductor]
		[keymgr]
		[cells]
		[database]
		[image_file_url]
		[baremetal]
		[rpc_notifier2]
		[matchmaker_redis]
		[ssl]
		[trusted_computing]
		[upgrade_levels]
		[matchmaker_ring]
		[vmware]
		[spice]
		[keystone_authtoken]
		auth_host = 127.0.0.1
		auth_port = 35357
		auth_protocol = http
		admin_user = nova
		admin_tenant_name = service
		admin_password = 123123

*	启动libvirtd

		service libvirtd start
*	删除default

		virsh net-destroy default
		virsh net-undefine default
*	设置开机启动

		chkconfig libvirtd on
*	重启

		service libvirtd restart
*	启动 messagebus

		service messagebus start
*	设置开机启动

		chkconfig messagebus on
*	启动nova

		service openstack-nova-api start
		service openstack-nova-cert start
		service openstack-nova-consoleauth start
		service openstack-nova-scheduler start
		service openstack-nova-conductor start
		service openstack-nova-novncproxy start
		service openstack-nova-compute start
		service openstack-nova-network start
*	配置nova

		chkconfig openstack-nova-api on
		chkconfig openstack-nova-cert on
		chkconfig openstack-nova-consoleauth on
		chkconfig openstack-nova-scheduler on
		chkconfig openstack-nova-conductor on
		chkconfig openstack-nova-novncproxy on
		chkconfig openstack-nova-compute on
		chkconfig openstack-nova-network on
*	重启nova

		service openstack-nova-api restart
		service openstack-nova-cert restart
		service openstack-nova-consoleauth restart
		service openstack-nova-scheduler restart
		service openstack-nova-conductor restart
		service openstack-nova-novncproxy restart
		service openstack-nova-compute restart
		service openstack-nova-network restart

*	创建内部网络

		nova network-create vmnet --fixed-range-v4=10.0.0.0/24 --bridge-interface=br1 --multi-host=T
*	创建外部网络

		nova-manage floating create --ip_range=10.211.55.0/24  --pool public_ip
*	查看网络

		nova network-list
		nova-manage network list
*	设置防火墙开放22端口和icmp协议

		nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
		nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
*	查看可用镜像

		nova image-list
*	创建实例

		nova boot --flavor 1 --image myFirstImage test_vm
*	查看运行

		nova list


#####安装Horizon
*	安装

		yum -y install openstack-dashboard
*	启动apache服务

		service httpd start
*	设置开机启动

		chkconfig httpd on
*	重启nova-api

		service openstack-nova-api restart
*	系统防火墙设置

		iptables -I INPUT -p tcp --dport 80 -j ACCEPT
		iptables -I INPUT -p tcp -m multiport --dports 5900:6000 -j ACCEPT
		iptables -I INPUT -p tcp --dport 6080 -j ACCEPT
		service iptables save


#####相关错误及解决方法
*	错误#1

		修改 notifier_strategy = rabbit
		'glance.notifier.notify_kombu.RabbitStrategy' is not an available notifier strategy.
		
		解决办法：
		yum install python-kombu
		 
*	错误#2

		Invalid HTTP_HOST header (you may need to set ALLOWED_HOSTS): 
		解决方法：
		ALLOWED_HOSTS = ['horizon.example.com', 'localhost', '*']
		service httpd restart
		


		






		
	






		
		








		




















		


