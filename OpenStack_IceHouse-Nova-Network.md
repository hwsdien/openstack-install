#####Openstack nova-network on Ubuntu 14.04 Install Step

*	安装NTP

		apt-get install -y ntp
		
		ntpdate 210.72.145.44


*	安装MySQL

		apt-get install -y mysql-server python-mysqldb
		
		sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
		
		mysql -uroot -p
    	use mysql;
   		delete from user where user='';
    	flush privileges;
    	
    	#Keystone
    	CREATE DATABASE keystone;
    	GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';

    	#Glance
    	CREATE DATABASE glance;
    	GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';

   	 	#Nova
    	CREATE DATABASE nova;
    	GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';
    	
    	flush privileges;
    	quit;
    	
    	service mysql restart
    
*	安装RabbitMQ

		apt-get install -y rabbitmq-server
		
		rabbitmqctl change_password guest nate123
		
*	安装Keystone

		apt-get install -y keystone python-keystone python-keystoneclient
		
		vim /etc/keystone/keystone.conf
    	connection = mysql://keystoneUser:keystonePass@127.0.0.1/keystone
    	
    	rm -rf /var/lib/keystone/keystone.db
    	
    	service keystone restart
    	
    	keystone-manage db_sync
    	
    	wget https://raw2.github.com/Ch00k/OpenStack-Havana-Install-Guide/master/keystone_basic.sh
    	wget https://raw2.github.com/Ch00k/OpenStack-Havana-Install-Guide/master/keystone_endpoints_basic.sh
    	chmod a+x ./keystone_*.sh
    	./keystone_basic.sh
    	./keystone_endpoints_basic.sh
    	
    	vim ./creds
    	export OS_TENANT_NAME=admin
    	export OS_USERNAME=admin
    	export OS_PASSWORD=openstacktest
    	export OS_AUTH_URL="http://127.0.0.1:5000/v2.0/"
    	
    	keystone user-list
    	keystone token-get
		
*	安装Glance

		apt-get install -y glance glance-api glance-common glance-registry python-glance python-glanceclient
		
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
    	
    	vim /etc/glance/glance-api.conf
    	  	
    	[DEFAULT]
    	rabbit_password = nate123
    	
    	[database]
    	#sqlite_db = /var/lib/glance/glance.sqlite
    	connection = mysql://glanceUser:glancePass@127.0.0.1/glance

    	[keystone_authtoken]
    	auth_host = 127.0.0.1
    	auth_port = 35357
    	auth_protocol = http
    	admin_tenant_name = service
    	admin_user = glance
    	admin_password = openstacktest

    	[paste_deploy]
    	flavor = keystone    	    	
    	
    	vim /etc/glance/glance-registry.conf
    	
    	[database]
    	#sqlite_db = /var/lib/glance/glance.sqlite
    	connection = mysql://glanceUser:glancePass@127.0.0.1/glance

    	[keystone_authtoken]
    	auth_host = 127.0.0.1
    	auth_port = 35357
    	auth_protocol = http
    	admin_tenant_name = service
    	admin_user = glance
    	admin_password = openstacktest

    	[paste_deploy]
    	flavor = keystone    	    	
    	
    	rm -rf /var/lib/glance/glance.sqlite
    	
    	service glance-api restart; service glance-registry restart
    	
    	glance-manage db_sync
    	
    	glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
    	
    	glance image-list
    	
*	安装Nova


		apt-get install -y nova-api nova-cert nova-common nova-conductor nova-consoleauth nova-novncproxy nova-scheduler python-nova python-novaclient novnc nova-compute nova-compute-kvm
		
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
    	floating_range=192.168.89.0/24

    	network_size=1
    	multi_host=true
    	enabled_apis=metadata
    	
    	metadata_listen = 0.0.0.0
    	metadata_listen_port = 8775
    	
    	# Compute #
    	compute_driver=libvirt.LibvirtDriver
    	
		cd /usr/bin/; for i in $( ls nova-* ); do sudo service $i restart; cd /root/;done    	
    	cd /usr/bin/; for i in $( ls nova-* ); do sudo service $i status; cd /root/;done
    	
    	nova-manage db sync
    	
    	chown -R nova:nova /etc/nova
    	chown -R nova:nova /var/lib/nova
    	
    	nova-manage service list


				
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		
		




		

