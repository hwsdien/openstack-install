####OpenStack Nova-Network on Havana 安装记录

#####Author
	nate.yu <nate_yhz@outlook.com>
	
#####Requirements
	Ubuntu 12.04 LTS 
		
#####说明
	安装流程参考了网上信息，个人记录，请勿使用，发生一切事情，后果自负！！！
	
#####安装内容
*	[安装环境设置](#安装环境设置)

*	[安装基础软件](#安装基础软件)

*	[安装MySQL](#安装mysql)

*	[安装RabbitMQ](#安装rabbitmq)

*	[安装Keystone](#安装keystone)

*	[安装Glance](#安装glance)

*	[安装Nova](#安装nova)

*	[安装Cinder](#安装cinder)

*	[安装Horizon](#安装horizon)

*	[相关错误](#相关错误)




	
#####安装环境设置
*	安装OpenSSH-Server

		apt-get -y install openssh-server
*	增加Havana的源

		apt-get install python-software-properties
		add-apt-repository cloud-archive:havana
*	修改默认的源

		sed -i 's/cn.archive.ubuntu.com/mirrors.yun-idc.com/g' /etc/apt/sources.list
*	更新源

		apt-get update
*	安装内核

		apt-get install linux-headers-3.2.0-58-generic
    	apt-get install linux-image-3.2.0-58-generic
*	修改grub默认启动的内核

		sed -i 's/default="0/default="2/g' /boot/grub/grub.cfg

*	更新已安装的包和系统

		apt-get upgrade
*	更改计算机名称

		vim /etc/hostname
		vim /etc/hosts
*	重启系统

		reboot

#####安装基础软件
*	设置IP转发

		sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
		sysctl -p
		
*	更改limits

		vim /etc/security/limits.conf
		*               soft    nofile           10240 
		*               hard    nofile           10240
		
*	检测kvm

		apt-get -y install cpu-checker
		kvm-ok
		modprobe kvm_intel	
*	安装qemu

		apt-get -y install qemu-utils	
*	安装网桥工具

		apt-get -y install bridge-utils
		
#####安装MySQL
	
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
		
#####安装RabbitMQ
		
*	安装RabbitMQ

		apt-get install -y rabbitmq-server
*	更改RabbitMQ的默认密码

		rabbitmqctl change_password guest nate123
		
#####安装Keystone
		
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
	
#####安装Glance	
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
		
#####安装Nova	
*	安装Nova

		apt-get install nova-novncproxy novnc nova-api nova-ajax-console-proxy nova-cert nova-conductor nova-consoleauth nova-doc nova-scheduler nova-compute-kvm python-guestfs nova-common nova-network
*	验证服务是否运行

		cd /etc/init.d/; for i in $( ls nova-* ); do service $i status; cd; done
*	更新 /etc/nova/api-paste.ini

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
*	更新 /etc/nova/nova.conf

		vim /etc/nova/nova.conf
		[DEFAULT]
		logdir=/var/log/nova
		state_path=/var/lib/nova
		lock_path=/run/lock/nova
		verbose=True

		api_paste_config=/etc/nova/api-paste.ini
		compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
		rabbit_host=127.0.0.1
		rabbit_password=nate123

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
		flat_interface=eth1
		flat_network_bridge=br100
		force_dhcp_release=True

		allow_same_net_traffic=False
		send_arp_for_ha=True
		share_dhcp_address=True

		network_size=256
		multi_host=True

		# Compute #
		compute_driver=libvirt.LibvirtDriver

		# Cinder #
		volume_api_class=nova.volume.cinder.API
		osapi_volume_listen_port=5900
		cinder_catalog_info=volume:cinder:internalURL
		
*	重启相关的服务

		cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; cd /root/;done
*	验证服务是否运行

		cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i status; cd /root/;done
*	删除sqlite文件

		rm -rf /var/lib/nova/nova.sqlite
*	同步数据

		nova-manage db sync
*	重启相关的服务

		cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; cd /root/;done

*	服务显示

		nova-manage service list
		
#####安装Cinder
*	安装Cinder

		apt-get install -y cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms
*	配置iscsi服务

		sed -i 's/false/true/g' /etc/default/iscsitarget
*	启动服务

		service iscsitarget start
		service open-iscsi start
*	更新 /etc/cinder/api-paste.ini

		vim /etc/cinder/api-paste.ini
		[filter:authtoken]
		paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
		service_protocol = http
		service_host = 127.0.0.1
		service_port = 5000
		auth_host = 127.0.0.1
		auth_port = 35357
		auth_protocol = http
		admin_tenant_name = service
		admin_user = cinder
		admin_password = openstacktest
*	更新 /etc/cinder/cinder.conf

		vim /etc/cinder/cinder.conf
		[DEFAULT]
		rootwrap_config=/etc/cinder/rootwrap.conf
		sql_connection = mysql://cinderUser:cinderPass@127.0.0.1/cinder
		api_paste_config = /etc/cinder/api-paste.ini
		iscsi_helper=ietadm
		volume_name_template = volume-%s
		volume_group = cinder-volumes
		verbose = True
		auth_strategy = keystone
		#osapi_volume_listen_port=5900
*	删除sqlite文件

		rm -rf /var/lib/cinder/cinder.sqlite
*	同步数据

		cinder-manage db sync
*	创建硬盘

		dd if=/dev/zero of=cinder-volumes bs=1 count=0 seek=2G
		losetup /dev/loop2 cinder-volumes
		fdisk /dev/loop2
		#Type in the followings:
		n
		p
		1
		ENTER
		ENTER
		t
		8e
		w
		
		pvcreate /dev/loop2
		vgcreate cinder-volumes /dev/loop2
*	重启相关服务

		cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i restart; cd /root/; done
*	验证服务是否在运行

		cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i status; cd /root/; done
		
#####安装Horizon

*	安装Dashboard

		apt-get install memcached libapache2-mod-wsgi openstack-dashboard
*	删除ubuntu主题

		apt-get remove --purge openstack-dashboard-ubuntu-theme
*	开启服务

		service apache2 restart
		service memcached restart
		
*	删除virbr0

		ifconfig virbr0 down
		brctl delbr virbr0
		virsh net-destroy default
		virsh net-undefine default
*	创建内部网络

		nova network-create vmnet --fixed-range-v4=10.0.0.0/24 --bridge-interface=br100 --multi-host=T
*	创建外部网络

		nova-manage floating create --ip_range=10.211.55.0/24  --pool public_ip
*	查看网络

		nova network-list
		nova-manage network list
*	设置防火墙开放22端口和icmp协议

		nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
		nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0


		
#####相关错误
	qemu-kvm: wrong dependency on librdb1 (kvm: symbol lookup error: kvm: undefined symbol: rbd_aio_discard)
	解决办法: apt-get install librbd1
	
	ConnectionFailed: Connection to neutron failed: Maximum attempts reached
	解决办法: 
		keystone 删除 neutron 相关的 user & service
		keystone catalog 查下有没有
		浏览器清空cookies重新登录


####增加计算结点

#####网络说明
	eth0 接外网
	eth1 接内网跟控制节点（192.168.0.100）组成内网	

#####安装环境设置
*	安装OpenSSH-Server

		apt-get -y install openssh-server
*	增加Havana的源

		apt-get install python-software-properties
		add-apt-repository cloud-archive:havana
*	修改默认的源

		sed -i 's/cn.archive.ubuntu.com/mirrors.yun-idc.com/g' /etc/apt/sources.list
*	更新源

		apt-get update
		
*	更新已安装的包和系统

		apt-get upgrade
		apt-get dist-upgrade
*	更改计算机名称

		vim /etc/hostname
		vim /etc/hosts
*	重启系统

		reboot

#####安装基础软件
*	设置IP转发

		sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
		sysctl -p
		
*	更改limits

		vim /etc/security/limits.conf
		*               soft    nofile           10240 
		*               hard    nofile           10240
		
*	检测kvm

		apt-get -y install cpu-checker
		kvm-ok
		modprobe kvm_intel	
*	安装qemu

		apt-get -y install qemu-utils	
*	安装网桥工具

		apt-get -y install bridge-utils
*	创建设置环境变量文件

		vim ~/creds
		export OS_TENANT_NAME=admin
		export OS_USERNAME=admin
		export OS_PASSWORD=openstacktest
		export OS_AUTH_URL="http://192.168.0.100:5000/v2.0/"
		
#####安装Nova	
*	安装Nova

		apt-get install nova-compute-kvm python-guestfs python-novaclient nova-network
*	验证服务是否运行

		cd /etc/init.d/; for i in $( ls nova-* ); do service $i status; cd; done
*	更新 /etc/nova/api-paste.ini

		vim /etc/nova/api-paste.ini
		[filter:authtoken]
		paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
		auth_host = 192.168.0.100
		auth_port = 35357
		auth_protocol = http
		admin_tenant_name = service
		admin_user = nova
		admin_password = openstacktest
		signing_dirname = /tmp/keystone-signing-nova
		# Workaround for https://bugs.launchpad.net/nova/+bug/1154809
		auth_version = v2.0
*	更新 /etc/nova/nova.conf

		vim /etc/nova/nova.conf
		[DEFAULT]
		logdir=/var/log/nova
		state_path=/var/lib/nova
		lock_path=/run/lock/nova
		verbose=True

		api_paste_config=/etc/nova/api-paste.ini
		compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
		rabbit_host=192.168.0.100
		rabbit_password=nate123

		nova_url=http://192.168.0.100:8774/v1.1/
		sql_connection=mysql://novaUser:novaPass@192.168.0.100/nova
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
		glance_api_servers=192.168.0.100:9292
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
		flat_interface=eth1
		flat_network_bridge=br100
		force_dhcp_release=True

		allow_same_net_traffic=False
		send_arp_for_ha=True
		share_dhcp_address=True

		network_size=256
		multi_host=True

		# Compute #
		compute_driver=libvirt.LibvirtDriver

		# Cinder #
		volume_api_class=nova.volume.cinder.API
		osapi_volume_listen_port=5900
		cinder_catalog_info=volume:cinder:internalURL
		
*	重启相关的服务

		cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; cd /root/;done
*	验证服务是否运行

		cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i status; cd /root/;done
*	删除sqlite文件

		rm -rf /var/lib/nova/nova.sqlite
*	同步数据

		nova-manage db sync
*	重启相关的服务

		cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; cd /root/;done

*	服务显示

		nova-manage service list



		
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	