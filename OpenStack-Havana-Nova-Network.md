####OpenStack Nova-Network on Havana 安装记录

#####Author
	nate.yu <nate_yhz@outlook.com>
	
#####Requirements
	Ubuntu 12.04 kernel 3.2.0-58
	
#####说明
	安装流程参考了网上信息，个人记录，请勿使用，发生一切事情，后果自负！！！
	
#####安装环境设置
	1、安装OpenSSH-Server
		apt-get install openssh-server
	2、增加Havana的源
		apt-get install python-software-properties
		add-apt-repository cloud-archive:havana
	3、修改默认的源
		sed -i 's/cn.archive.ubuntu.com/mirrors.yun-idc.com/g' /etc/apt/sources.list
	4、更新源
		apt-get update
	5、安装内核(openvswitch 1.9.0最高支持到3.8的内核，默认的3.11的内核，所以要改)
		apt-get install linux-headers-3.2.0-58-generic
		apt-get install linux-image-3.2.0-58-generic
	6、修改grub默认启动的内核
		sed -i 's/default="0/default="2/g' /boot/grub/grub.cfg
	7、更新已安装的包和系统
		apt-get upgrade
		apt-get dist-upgrade(视情况)
	8、更改计算机名称
		vim /etc/hostname
		vim /etc/hosts
	9、重启系统
		rebooted

#####安装步骤
	1、设置IP转发
		sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
		sysctl -p
		
	2、更改limits
		vim /etc/security/limits.conf
		*               soft    nofile           10240 
		*               hard    nofile           10240
		
	3、检测kvm
		apt-get install cpu-checker
		kvm-ok
		modprobe kvm_intel	
	4、安装qemu
		apt-get install qemu-utils	
	5、安装网桥工具
		apt-get install bridge-utils
	
	6、安装MySQL
		apt-get install -y mysql-server python-mysqldb
	7、配置MySQL
		sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
	8、删除所有空用户
		mysql -uroot -p
		use mysql;
		delete from user where user='';
		flush privileges;
	9、创建数据库和用户
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
	10、重启MySQL
		service mysql restart
		
	11、安装RabbitMQ
		apt-get install -y rabbitmq-server
	12、更改RabbitMQ的默认密码
		rabbitmqctl change_password guest nate123
		
	13、安装Keystone
		apt-get install -y keystone
	14、修改Keystone的配置文件
		vim /etc/keystone/keystone.conf
		connection = mysql://keystoneUser:keystonePass@127.0.0.1/keystone
	15、删除Sqlite db文件
		rm -rf /var/lib/keystone/keystone.db
	16、重启Keystone 
		service keystone restart
	17、同步数据
		keystone-manage db_sync
	18、增加初始化数据(需修改脚本文件)
		wget https://raw2.github.com/Ch00k/OpenStack-Havana-Install-Guide/master/keystone_basic.sh
		wget https://raw2.github.com/Ch00k/OpenStack-Havana-Install-Guide/master/keystone_endpoints_basic.sh
		chmod a+x ./keystone_*.sh
		./keystone_basic.sh
		./keystone_endpoints_basic.sh
	19、创建设置环境变量文件
		vim ./creds
		export OS_TENANT_NAME=admin
		export OS_USERNAME=admin
		export OS_PASSWORD=openstacktest
		export OS_AUTH_URL="http://127.0.0.1:5000/v2.0/"
	20、测试keystone
		keystone user-list
		keystone token-get
	
	
	21、安装Glance
		apt-get install -y glance
	22、验证服务是否运行
		service glance-api status
		service glance-registry status
	23、更新ini文件
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
	24、更新conf文件
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
	25、删除sqlite文件
		rm -rf /var/lib/glance/glance.sqlite
	26、重启服务
		service glance-api restart; service glance-registry restart
	27、同步数据
		glance-manage db_sync
	28、测试(可wget下来再 < 导入)
		glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
	29、列出所有映像
		glance image-list
		
	30、安装Nova
	apt-get install nova-novncproxy novnc nova-api nova-ajax-console-proxy nova-cert nova-conductor nova-consoleauth nova-doc nova-scheduler nova-compute-kvm python-guestfs nova-common nova-network
	31、验证服务是否运行
		cd /etc/init.d/; for i in $( ls nova-* ); do service $i status; cd; done
	32、更新 /etc/nova/api-paste.ini
		vim /etc/nova/api-paste.ini
		[filter:authtoken]
		paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
		auth_host = 127.0.0.1
		auth_port = 35357
		auth_protocol = http
		admin_tenant_name = service
		admin_user = nova
		admin_password = openstacktest
		signing_dirname = /tmp/keystone-signing-nova
		# Workaround for https://bugs.launchpad.net/nova/+bug/1154809
		auth_version = v2.0
	33、更新 /etc/nova/nova.conf
		vim /etc/nova/nova.conf
		[DEFAULT]
		logdir=/var/log/nova
		state_path=/var/lib/nova
		lock_path=/run/lock/nova
		verbose=True
		api_paste_config=/etc/nova/api-paste.ini
		compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
		rabbit_host=127.0.0.1
		rabbit_password=123123
		nova_url=http://127.0.0.1:8774/v1.1/
		sql_connection=mysql://novaUser:novaPass@127.0.0.1/nova
		root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
		cpu_allocation_ratio=16.0

		#inject password
		libvirt_inject_password=true

		#for migrate when just one compute node
		allow_resize_to_same_host=true

		# Auth
		use_deprecated_auth=false
		auth_strategy=keystone

		# Imaging service
		glance_api_servers=127.0.0.1:9292
		image_service=nova.image.glance.GlanceImageService

		# Vnc configuration
		novnc_enabled=true
		novncproxy_base_url=http://10.211.55.4:6080/vnc_auto.html
		novncproxy_port=6080
		vncserver_proxyclient_address=127.0.0.1
		vncserver_listen=0.0.0.0
		
		# Network settings
		dhcpbridge_flagfile=/etc/nova/nova.conf
		dhcpbridge=/usr/bin/nova-dhcpbridge
		firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
		network_manager=nova.network.manager.FlatDHCPManager
		public_interface=eth0
		flat_interface=eth0
		flat_network_bridge=br100
		flat_injected=False
		force_dhcp_release=true
		fixed_range=192.168.100.0/24
		flat_network_dhcp_start=192.168.100.2
		floating_range=10.211.55.0/24

		network_size=1
		multi_host=true
		enabled_apis=metadata


		# Compute #
		compute_driver=libvirt.LibvirtDriver

		# Cinder #
		volume_api_class=nova.volume.cinder.API
		osapi_volume_listen_port=5900
		cinder_catalog_info=volume:cinder:internalURL
		
	34、重启相关的服务
		cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; cd /root/;done
	35、验证服务是否运行
		cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i status; cd /root/;done
	36、删除sqlite文件
		rm -rf /var/lib/nova/nova.sqlite
	37、同步数据
		nova-manage db sync
	38、服务显示
		nova-manage service list


	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	