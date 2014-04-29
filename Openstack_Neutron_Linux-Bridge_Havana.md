####OpenStack Neutron Linux-Bridge Havana 单节点的安装配置记录（未完待续）

#####Author
	nate.yu <nate_yhz@outlook.com>
	
#####Requirements
	Ubuntu 12.04 LTS
	
#####说明
	安装流程参考了网上信息，个人记录，请勿使用，发生一切事情，后果自负！！！
	
#####安装环境设置（所有节点都一样）
	1、安装OpenSSH-Server
		apt-get install openssh-server
	2、增加Havana的源
		apt-get install python-software-properties
		add-apt-repository cloud-archive:havana
	3、修改默认的源
		sed -i 's/cn.archive.ubuntu.com/mirrors.yun-idc.com/g' /etc/apt/sources.list
	4、更新源
		apt-get update
	7、更新已安装的包和系统
		apt-get upgrade
		apt-get dist-upgrade(视情况)
	8、更改计算机名称
		vim /etc/hostname
		vim /etc/hosts
	9、重启系统
		reboot
		
		
#####安装
*	安装MySQL

		apt-get install -y mysql-server python-mysqldb
*	配置MySQL

		sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
*	删除所有空用户

		mysql -uroot -p
		use mysql;
		delete from user where user='';
		flush privileges;
*	创建数据库和用户

		#Keystone
		CREATE DATABASE keystone;
		GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';

		#Glance
		CREATE DATABASE glance;
		GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';

		#Neutron
		CREATE DATABASE neutron;
		GRANT ALL ON neutron.* TO 'neutronUser'@'%' IDENTIFIED BY 'neutronPass';

		#Nova
		CREATE DATABASE nova;
		GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';

		#Cinder
		CREATE DATABASE cinder;
		GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass';
		
		flush privileges;
		quit;
*	重启MySQL

		service mysql restart
*	安装RabbitMQ

		apt-get install -y rabbitmq-server
*	更改RabbitMQ的默认密码

		rabbitmqctl change_password guest nate123

*	安装NTP

		apt-get install -y ntp
*	安装vlan bridge-utils

		apt-get install -y vlan bridge-utils
*	设置IP转发

		sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
		sysctl -p
*	安装Keystone

		apt-get install -y keystone
*	修改Keystone的配置文件

		vim /etc/keystone/keystone.conf
		connection = mysql://keystoneUser:keystonePass@127.0.0.1/keystone
*	删除Sqlite db文件

		rm -rf /var/lib/keystone/keystone.db
*	重启Keystone 

		service keystone restart
*	同步数据

		keystone-manage db_sync
*	增加初始化数据(需修改脚本文件)

		wget https://raw2.github.com/Ch00k/OpenStack-Havana-Install-Guide/master/keystone_basic.sh
		wget https://raw2.github.com/Ch00k/OpenStack-Havana-Install-Guide/master/keystone_endpoints_basic.sh
		chmod a+x ./keystone_*.sh
		./keystone_basic.sh
		./keystone_endpoints_basic.sh
*	创建设置环境变量文件

		vim ./creds
		export OS_TENANT_NAME=admin
		export OS_USERNAME=admin
		export OS_PASSWORD=openstacktest
		export OS_AUTH_URL="http://127.0.0.1:5000/v2.0/"
*	测试keystone

		keystone user-list
		keystone token-get
*	安装Glance

		apt-get install -y glance
*	验证服务是否运行

		service glance-api status
		service glance-registry status
*	更新ini文件

		vim /etc/glance/glance-api-paste.ini
		vim /etc/glance/glance-registry-paste.ini
		[filter:authtoken]
		paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
		auth_host = 127.0.0.1
		auth_port = 35357
		auth_protocol = http
		admin_tenant_name = service
		admin_user = glance
		admin_password = openstacktest	
*	更新conf文件

		vim /etc/glance/glance-api.conf
		vim /etc/glance/glance-registry.conf
		[DEFAULT]
		sql_connection = mysql://glanceUser:glancePass@127.0.0.1/glance

		[keystone_authtoken]
		auth_host = 127.0.0.1
		auth_port = 35357
		auth_protocol = http
		admin_tenant_name = service
		admin_user = glance
		admin_password = openstacktest

		[paste_deploy]
		flavor = keystone
*	删除sqlite文件

		rm -rf /var/lib/glance/glance.sqlite
*	重启服务

		service glance-api restart; service glance-registry restart
*	同步数据

		glance-manage db_sync
*	测试(可wget下来再 < 导入)

		glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
*	列出所有映像

		glance image-list
		
*	安装Neutron

		apt-get install -y neutron-server neutron-plugin-linuxbridge neutron-plugin-linuxbridge-agent dnsmasq neutron-dhcp-agent neutron-l3-agent 
*	验证服务是否运行

		cd /etc/init.d/; for i in $( ls neutron-* ); do sudo service $i status; cd; done
*	更新 /etc/quantum/quantum.conf 

		core_plugin = quantum.plugins.linuxbridge.lb_quantum_plugin.LinuxBridgePluginV2
*	更新 /etc/quantum/api-paste.ini

		vim /etc/neutron/api-paste.ini
		[filter:authtoken]
		paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
		auth_host = 127.0.0.1
		auth_port = 35357
		auth_protocol = http
		admin_tenant_name = service
		admin_user = neutron
		admin_password = openstacktest
*	更新 /etc/quantum/plugins/linuxbridge/linuxbridge_conf.ini

		[DATABASE]
		connection=mysql://neutronUser:neutronPass@127.0.0.1/neutron
		
		[LINUX_BRIDGE]
		physical_interface_mappings = physnet1:eth1
		
		[VLANS]
		tenant_network_type = vlan
		network_vlan_ranges = physnet1:1000:2999
		
*	更新 /etc/quantum/l3_agent.ini

		interface_driver = quantum.agent.linux.interface.BridgeInterfaceDriver
*	更新 /etc/quantum/quantum.conf

		vim /etc/neutron/neutron.conf
		#RabbitMQ IP
		rabbit_host = 127.0.0.1
		rabbit_password = nate123

		[keystone_authtoken]
		auth_host = 127.0.0.1
		auth_port = 35357
		auth_protocol = http
		admin_tenant_name = service
		admin_user = neutron
		admin_password = openstacktest
		signing_dir = /var/lib/neutron/keystone-signing
		
		[quotas]
		quota_driver=neutron.db.quota_db.DbQuotaDriver

		[DATABASE]
		connection = mysql://neutronUser:neutronPass@127.0.0.1/neutron
*	更新 /etc/quantum/dhcp_agent.ini

		interface_driver = quantum.agent.linux.interface.BridgeInterfaceDriver
*	更新 /etc/quantum/metadata_agent.ini

		vim /etc/neutron/metadata_agent.ini
		# The Neutron user information for accessing the Neutron API.
		auth_url = http://127.0.0.1:35357/v2.0
		auth_region = RegionOne
		admin_tenant_name = service
		admin_user = neutron
		admin_password = openstacktest

		# IP address used by Nova metadata server
		nova_metadata_ip = 172.16.33.128


		# TCP Port used by Nova metadata server
		nova_metadata_port = 8775

		metadata_proxy_shared_secret = helloOpenStack
		
*	删除sqlite文件

		rm -rf /var/lib/neutron/neutron.sqlite
*	重启服务

		cd /etc/init.d/; for i in $( ls neutron-* ); do sudo service $i restart; cd /root/; done
		service dnsmasq restart
*	验证服务是否运行

		cd /etc/init.d/; for i in $( ls neutron-* ); do sudo service $i status; cd /root/; done
		service dnsmasq status
*	查看所有的代理

		neutron agent-list
		
*	检测是否支持KVM

		apt-get install -y cpu-checker
		kvm-ok
*	加截 kvm_intel 内核模块

		modprobe kvm_intel
*	安装KVM

		apt-get install -y kvm libvirt-bin pm-utils
*	更新 /etc/libvirt/qemu.conf

		cgroup_device_acl = [
		"/dev/null", "/dev/full", "/dev/zero",
		"/dev/random", "/dev/urandom",
		"/dev/ptmx", "/dev/kvm", "/dev/kqemu",
		"/dev/rtc", "/dev/hpet","/dev/net/tun"
		]
*	删除默认的网桥

		virsh net-destroy default
		virsh net-undefine default	
*	更新 /etc/libvirt/libvirtd.conf

		listen_tls = 0
		listen_tcp = 1
		auth_tcp = "none"
*	更新 /etc/init/libvirt-bin.conf

		env libvirtd_opts="-d -l"
*	更新 /etc/default/libvirt-bin

		libvirtd_opts="-d -l"
*	重启服务

		service dbus restart && service libvirt-bin restart
*	检测服务是否运行

		service dbus status && service libvirt-bin status
















































	