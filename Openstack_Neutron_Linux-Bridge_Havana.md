####OpenStack Neutron Linux-Bridge Havana 单节点的安装配置记录（未完待续）

#####Author
	nate.yu <nate_yhz@outlook.com>
	
#####Requirements
	Ubuntu 12.04 LTS
	
#####说明
	安装流程参考了网上信息，个人记录，请勿使用，发生一切事情，后果自负！！！
	
#####安装内容

*	[安装环境设置](#安装环境设置)
*	[安装MySQL](#安装mysql)
*	[安装RabbitMQ](#安装rabbitmq)
*	[安装NTP](#安装ntp)
*	[安装Keystone](#安装keystone)
*	[安装Glance](#安装glance)
*	[安装Neutron](#安装neutron)
*	[安装KVM](#安装kvm)
*	[安装Nova](#安装nova)
*	[安装Horizon](#安装horizon)

   	

#####安装环境设置
*	安装OpenSSH-Server

		apt-get install openssh-server
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
		
#####安装RabbitMQ
*	安装RabbitMQ

		apt-get install -y rabbitmq-server

*	更改RabbitMQ的默认密码

		rabbitmqctl change_password guest nate123

#####安装NTP 
*	安装NTP

		apt-get install -y ntp
*	安装vlan bridge-utils

		apt-get install -y vlan bridge-utils
*	设置IP转发

		sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
		sysctl -p
		
#####安装Keystone
*	安装Keystone

		apt-get install keystone
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
		

#####安装Neutron
*	安装Neutron

		apt-get install -y neutron-server neutron-plugin-linuxbridge neutron-plugin-linuxbridge-agent dnsmasq neutron-dhcp-agent neutron-l3-agent 
*	验证服务是否运行

		cd /etc/init.d/; for i in $( ls neutron-* ); do sudo service $i status; cd; done
*	更新 /etc/neutron/neutron.conf 

		vim /etc/neutron/neutron.conf
		core_plugin = neutron.plugins.linuxbridge.lb_neutron_plugin.LinuxBridgePluginV2
*	更新 /etc/neutron/api-paste.ini

		vim /etc/neutron/api-paste.ini
		[filter:authtoken]
		paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
		auth_host = 127.0.0.1
		auth_port = 35357
		auth_protocol = http
		admin_tenant_name = service
		admin_user = neutron
		admin_password = openstacktest
*	更新 /etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini

		vim /etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini
		[database]
		connection=mysql://neutronUser:neutronPass@127.0.0.1/neutron
		
		[linux_bridge]
		physical_interface_mappings = physnet1:eth1
		
		[vlans]
		tenant_network_type = vlan
		network_vlan_ranges = physnet1:1000:2999
		
*	更新 /etc/neutron/l3_agent.ini

		vim /etc/neutron/l3_agent.ini
		interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
		use_namespaces = False
		router_id = 1
		external_network_bridge =
*	更新 /etc/default/neutron-server

		vim /etc/default/neutron-server
		NEUTRON_PLUGIN_CONFIG="/etc/neutron/plugins/linuxbridge/linuxbridge_conf.ini"

*	更新 /etc/default/dnsmasq

		vim /etc/default/dnsmasq
		ENABLED=0

*	更新 /etc/neutron/neutron.conf

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

		[database]
		connection = mysql://neutronUser:neutronPass@127.0.0.1/neutron
*	更新 /etc/neutron/dhcp_agent.ini

		vim /etc/neutron/dhcp_agent.ini
		interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
		use_namespaces = False
*	更新 /etc/neutron/metadata_agent.ini

		vim /etc/neutron/metadata_agent.ini
		# The Neutron user information for accessing the Neutron API.
		auth_url = http://127.0.0.1:35357/v2.0
		auth_region = RegionOne
		admin_tenant_name = service
		admin_user = neutron
		admin_password = openstacktest

		# IP address used by Nova metadata server
		nova_metadata_ip = 127.0.0.1


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
		
		
#####安装KVM
*	检测是否支持KVM

		apt-get install -y cpu-checker
		kvm-ok
*	加截 kvm_intel 内核模块

		modprobe kvm_intel
*	安装KVM

		apt-get install -y kvm libvirt-bin pm-utils
*	更新 /etc/libvirt/qemu.conf

		vim /etc/libvirt/qemu.conf
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

		vim /etc/libvirt/libvirtd.conf
		listen_tls = 0
		listen_tcp = 1
		auth_tcp = "none"
*	更新 /etc/init/libvirt-bin.conf
		
		vim /etc/init/libvirt-bin.conf
		env libvirtd_opts="-d -l"
*	更新 /etc/default/libvirt-bin
		
		vim /etc/default/libvirt-bin
		libvirtd_opts="-d -l"
*	重启服务

		service dbus restart && service libvirt-bin restart
*	检测服务是否运行

		service dbus status && service libvirt-bin status
		
#####安装Nova
*	安装Nova

		apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-doc nova-conductor nova-compute-kvm
		
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
		novncproxy_base_url=http://127.0.0.1:6080/vnc_auto.html
		novncproxy_port=6080
		vncserver_proxyclient_address=127.0.0.1
		vncserver_listen=0.0.0.0

		# Network settings
		network_api_class=nova.network.neutronv2.api.API
		neutron_url=http://127.0.0.1:9696
		neutron_auth_strategy=keystone
		neutron_admin_tenant_name=service
		neutron_admin_username=neutron
		neutron_admin_password=openstacktest
		neutron_admin_auth_url=http://127.0.0.1:35357/v2.0
		libvirt_vif_driver=nova.virt.libvirt.vif.neutronLinuxBridgeVIFDriver
		linuxnet_interface_driver=nova.network.linux_net.LinuxBridgeInterfaceDriver
		firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
		
		neutron_use_dhcp=true
		network_manager=nova.network.neutron.manager.NeutronManager

		#Metadata
		service_neutron_metadata_proxy = True
		neutron_metadata_proxy_shared_secret = helloOpenStack
		#metadata_host = 127.0.0.1
		#metadata_listen = 0.0.0.0
		#metadata_listen_port = 8775

		# Compute #
		compute_driver=libvirt.LibvirtDriver

		# Cinder #
		volume_api_class=nova.volume.cinder.API
		osapi_volume_listen_port=5900
		cinder_catalog_info=volume:cinder:internalURL
		
*	更新 /etc/nova/nova-compute.conf

		vim /etc/nova/nova-compute.conf
		[DEFAULT]
		libvirt_type=kvm
		compute_driver=libvirt.LibvirtDriver
		libvirt_vif_type=ethernet
		libvirt_vif_driver=nova.virt.libvirt.vif.NeutronLinuxBridgeVIFDriver
		linuxnet_interface_driver=nova.network.linux_net.NeutronLinuxBridgeInterfaceDriver
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
*	安装Horizon & memcached

		apt-get -y install openstack-dashboard memcached
*	删除ubuntu主题

		dpkg --purge openstack-dashboard-ubuntu-theme
*	重启apache & memcached

		service apache2 restart; service memcached restart












































































	