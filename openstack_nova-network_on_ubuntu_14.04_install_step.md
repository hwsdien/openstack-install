#####Openstack nova-network on Ubuntu 14.04 Install Step

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



		

