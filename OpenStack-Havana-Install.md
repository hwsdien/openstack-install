####OpenStack Havana 双节点的安装配置记录

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
		apt-get dist-upgrade
	8、重启系统
		reboot
		
#####控制节点的安装
	1、安装MySQL
		apt-get install -y mysql-server python-mysqldb
	2、配置MySQL
		sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
	3、删除所有空用户
		delete from user where user='';
		flush privileges;
	4、创建数据库和用户
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
	5、重启MySQL
		service mysql restart
		
	6、安装RabbitMQ
		apt-get install -y rabbitmq-server
	7、更改RabbitMQ的默认密码
		rabbitmqctl change_password guest nate123
		
	8、安装NTP
		apt-get install -y ntp
		
	9、安装vlan bridge-utils
		apt-get install -y vlan bridge-utils
	10、设置IP转发
		sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
		sysctl -p
	
	11、安装Keystone
		apt-get install -y keystone
	12、修改Keystone的配置文件
		vim /etc/keystone/keystone.conf
		connection = mysql://keystoneUser:keystonePass@127.0.0.1/keystone
	13、删除Sqlite db文件
		rm -rf /var/lib/keystone/keystone.db
	14、重启Keystone 
		service keystone restart
	15、同步数据
		keystone-manage db_sync
	16、增加初始化数据(需修改脚本文件)
		wget https://raw2.github.com/Ch00k/OpenStack-Havana-Install-Guide/master/keystone_basic.sh
		wget https://raw2.github.com/Ch00k/OpenStack-Havana-Install-Guide/master/keystone_endpoints_basic.sh
		chmod a+x ./keystone_*.sh
		./keystone_basic.sh
		./keystone_endpoints_basic.sh
	17、创建设置环境变量文件
		vim ./creds
		export OS_TENANT_NAME=admin
		export OS_USERNAME=admin
		export OS_PASSWORD=openstacktest
		export OS_AUTH_URL="http://127.0.0.1:5000/v2.0/"
	18、测试keystone
		keystone user-list
		keystone token-get
	
	19、安装Glance
		apt-get install -y glance
	20、验证服务是否运行
		service glance-api status
		service glance-registry status
	21、更新ini文件
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
	22、更新conf文件
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
	23、删除sqlite文件
		rm -rf /var/lib/glance/glance.sqlite
	24、重启服务
		service glance-api restart; service glance-registry restart
	25、同步数据
		glance-manage db_sync
	26、测试(可wget下来再 < 导入)
		glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
	27、列出所有映像
		glance image-list
		
	28、安装OpenVSwitch
		apt-get install -y openvswitch-controller openvswitch-switch openvswitch-datapath-dkms
	29、重启OpenVSwitch
		service openvswitch-switch restart
	30、创建网桥
		ovs-vsctl add-br br-int
		ovs-vsctl add-br br-ex
	31、eth1设置成混杂模式
		auto eth1
		iface eth1 inet manual
		up ifconfig $IFACE 0.0.0.0 up
		up ip link set $IFACE promisc on
		down ip link set $IFACE promisc off
		down ifconfig $IFACE down
	32、eth1加入br-ex
		ovs-vsctl add-port br-ex eth1
	33、设置br-ex信息
		auto br-ex
		iface br-ex inet static
		address 172.16.33.133
		netmask 255.255.255.0
		gateway 172.16.33.2
		dns-nameservers 1.2.4.8
	34、重启系统
		reboot
	
	35、安装Neutron
		apt-get install -y neutron-server neutron-plugin-openvswitch neutron-plugin-openvswitch-agent dnsmasq neutron-dhcp-agent neutron-l3-agent neutron-metadata-agent
	36、验证服务是否运行
		cd /etc/init.d/; for i in $( ls neutron-* ); do sudo service $i status; cd; done
	37、更新 /etc/neutron/api-paste.ini
		[filter:authtoken]
		paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
		auth_host = 127.0.0.1
		auth_port = 35357
		auth_protocol = http
		admin_tenant_name = service
		admin_user = neutron
		admin_password = openstacktest
	38、更新 /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini
		#Under the database section
		[DATABASE]
		connection=mysql://neutronUser:neutronPass@127.0.0.1/neutron

		#Under the OVS section
		[OVS]
		tenant_network_type = gre
		enable_tunneling = True
		tunnel_id_ranges = 1:1000
		integration_bridge = br-int
		tunnel_bridge = br-tun
		local_ip = 172.16.33.128
		
		#Firewall driver for realizing neutron security group function
		[SECURITYGROUP]
		firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
	39、更新 /etc/neutron/metadata_agent.ini
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
	40、更新 /etc/neutron/neutron.conf
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

		[DATABASE]
		connection = mysql://neutronUser:neutronPass@127.0.0.1/neutron
	41、更新 /etc/neutron/l3_agent.ini
		[DEFAULT]
		interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
		use_namespaces = True
		external_network_bridge = br-ex
		signing_dir = /var/cache/neutron
		admin_tenant_name = service
		admin_user = neutron
		admin_password = openstacktest
		auth_url = http://127.0.0.1:35357/v2.0
		l3_agent_manager = neutron.agent.l3_agent.L3NATAgentWithStateReport
		root_helper = sudo neutron-rootwrap /etc/neutron/rootwrap.conf
		interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
	42、更新 /etc/neutron/dhcp_agent.ini
		[DEFAULT]
		interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
		dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
		use_namespaces = True
		signing_dir = /var/cache/neutron
		admin_tenant_name = service
		admin_user = neutron
		admin_password = openstacktest
		auth_url = http://127.0.0.1:35357/v2.0
		dhcp_agent_manager = neutron.agent.dhcp_agent.DhcpAgentWithStateReport
		root_helper = sudo neutron-rootwrap /etc/neutron/rootwrap.conf
		state_path = /var/lib/neutron
	43、删除sqlite文件
		rm -rf /var/lib/neutron/neutron.sqlite
	44、重启服务
		cd /etc/init.d/; for i in $( ls neutron-* ); do sudo service $i restart; cd /root/; done
		service dnsmasq restart
	45、验证服务是否运行
		cd /etc/init.d/; for i in $( ls neutron-* ); do sudo service $i status; cd /root/; done
		service dnsmasq status
	46、查看所有的代理
		neutron agent-list
	
	47、检测是否支持KVM
		apt-get install -y cpu-checker
		kvm-ok
	48、加截 kvm_intel 内核模块
		modprobe kvm_intel
	49、安装KVM
		apt-get install -y kvm libvirt-bin pm-utils
	50、更新 /etc/libvirt/qemu.conf
		cgroup_device_acl = [
		"/dev/null", "/dev/full", "/dev/zero",
		"/dev/random", "/dev/urandom",
		"/dev/ptmx", "/dev/kvm", "/dev/kqemu",
		"/dev/rtc", "/dev/hpet","/dev/net/tun"
		]
	51、删除默认的网桥
		virsh net-destroy default
		virsh net-undefine default	
	52、更新 /etc/libvirt/libvirtd.conf
		listen_tls = 0
		listen_tcp = 1
		auth_tcp = "none"
	53、更新 /etc/init/libvirt-bin.conf
		env libvirtd_opts="-d -l"
	54、更新 /etc/default/libvirt-bin
		libvirtd_opts="-d -l"
	55、重启服务
		service dbus restart && service libvirt-bin restart
	56、检测服务是否运行
		service dbus status && service libvirt-bin status
		
	57、安装Nova
		apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-doc nova-conductor nova-compute-kvm
	58、验证服务是否运行
		cd /etc/init.d/; for i in $( ls nova-* ); do service $i status; cd; done
	59、更新 /etc/nova/api-paste.ini
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
	60、更新 /etc/nova/nova.conf
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

		# Auth
		use_deprecated_auth=false
		auth_strategy=keystone

		# Imaging service
		glance_api_servers=127.0.0.1:9292
		image_service=nova.image.glance.GlanceImageService

		# Vnc configuration
		novnc_enabled=true
		novncproxy_base_url=http://172.16.33.133:6080/vnc_auto.html
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
		libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
		linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
		#If you want Neutron + Nova Security groups
		#firewall_driver=nova.virt.firewall.NoopFirewallDriver
		#security_group_api=neutron
		#If you want Nova Security groups only, comment the two lines above and uncomment line -1-.
		#-1-firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

		#Metadata
		service_neutron_metadata_proxy = True
		neutron_metadata_proxy_shared_secret = helloOpenStack
		metadata_host = 127.0.0.1
		metadata_listen = 0.0.0.0
		metadata_listen_port = 8775

		# Compute #
		compute_driver=libvirt.LibvirtDriver

		# Cinder #
		volume_api_class=nova.volume.cinder.API
		osapi_volume_listen_port=5900
		cinder_catalog_info=volume:cinder:internalURL
	62、更新 /etc/nova/nova-compute.conf
		[DEFAULT]
		libvirt_type=kvm
		libvirt_ovs_bridge=br-int
		libvirt_vif_type=ethernet
		libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
		libvirt_use_virtio_for_bridges=True
	63、重启相关的服务
		cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; cd /root/;done
	64、验证服务是否运行
		cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i status; cd /root/;done
	65、删除sqlite文件
		rm -rf /var/lib/nova/nova.sqlite
	66、同步数据
		nova-manage db sync
	67、服务显示
		nova-manage service list
		
	68、安装Cinder
		apt-get install -y cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms
	69、配置iscsi服务
		sed -i 's/false/true/g' /etc/default/iscsitarget
	70、启动服务
		service iscsitarget start
		service open-iscsi start
	71、更新 /etc/cinder/api-paste.ini
		[filter:authtoken]
		paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
		service_protocol = http
		service_host = 172.16.33.128
		service_port = 5000
		auth_host = 127.0.0.1
		auth_port = 35357
		auth_protocol = http
		admin_tenant_name = service
		admin_user = cinder
		admin_password = openstacktest
	72、更新 /etc/cinder/cinder.conf
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
	73、删除sqlite文件
		rm -rf /var/lib/cinder/cinder.sqlite
	74、同步数据
		cinder-manage db sync
	75、创建硬盘
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
	76、重启相关服务
		cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i restart; cd /root/; done
	77、验证服务是否在运行
		cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i status; cd /root/; done
		
	78、安装Horizon & memcached
		apt-get -y install openstack-dashboard memcached
	79、删除ubuntu主题
		dpkg --purge openstack-dashboard-ubuntu-theme
	80、重启apache & memcached
		service apache2 restart; service memcached restart
		
	
		
		
		
		
		
		
		
		
		
		
		
		

	