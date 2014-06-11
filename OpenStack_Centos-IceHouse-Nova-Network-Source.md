#### CentOS 6.5 源码安装 OpenStack Icehouse

#####Author
	nate.yu < nate.yhz at gmail.com >

#####Requirements
	CentOS release 6.5 (Final)
	Linux localhost.localdomain 2.6.32-431.el6.x86_64 #1 SMP Fri Nov 22 03:15:09 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux

#####安装基础软件
*	增加源

		rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm

		rpm -Uvh http://pkgs.repoforge.org/rpmforge-release/rpmforge-release-0.5.3-1.el6.rf.x86_64.rpm

		yum update
*	安装相关的包

		yum install vim gcc gcc-c++ make cmake lsof libtool patch automake python-devel python-pip gcc-c++ openssl-devel git wget ncurses-devel xmlto zip unzip libxslt libxslt-devel libxml2-devel libffi-devel libvirt-python libvirt qemu-kvm scsi-target-utils lvm2 numdisplay device-mapper bridge-utils dnsmasq dnsmasq-utils kernel kernel-devel bridge-utils dnsmasq dnsmasq-utils
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
*	重启系统

		reboot
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
		
#####安装 MySQL
*	安装MySQL-python

		yum install MySQL-python
*	下载MySQL

		wget http://www.percona.com/downloads/Percona-Server-5.6/LATEST/source/tarball/percona-server-5.6.17-65.0.tar.gz
		tar zxvf percona-server-5.6.17-65.0.tar.gz
		cd percona-server-5.6.17-65.0
*	系统用户设置
		
		groupadd mysql && useradd -g mysql mysql && mkdir -p /opt/mysql && mkdir -p /data1/mysql && chown -R mysql:mysql /data1/mysql
*	cmake

		cmake -DCMAKE_INSTALL_PREFIX=/opt/mysql \
		-DMYSQL_UNIX_ADDR=/data1/mysql/mysql.sock \
		-DDEFAULT_CHARSET=gbk \
		-DDEFAULT_COLLATION=gbk_chinese_ci \
		-DWITH_EXTRA_CHARSETS:STRING=armscii8,ascii,big5,cp1250,cp1251,cp1256,cp1257,cp850,cp852,cp866,cp932,dec8,eucjpms,euckr,gb2312,gbk,geostd8,greek,hebrew,hp8,keybcs2,koi8r,koi8u,latin1,latin2,latin5,latin7,macce,macroman,sjis,swe7,tis620,ucs2,ujis,utf8 \
		-DWITH_MYISAM_STORAGE_ENGINE=1 \
		-DWITH_INNOBASE_STORAGE_ENGINE=1 \
		-DWITH_MEMORY_STORAGE_ENGINE=1 \
		-DWITH_READLINE=1 \
		-DENABLED_LOCAL_INFILE=1 \
		-DMYSQL_DATADIR=/data1/mysql \
		-DMYSQL_USER=mysql \
		-DMYSQL_TCP_PORT=3306   
*	编译安装
		
		make && make install    
*	设置启动脚本

		cp ./support-files/mysql.server /etc/init.d/mysqld && chmod 755 /etc/init.d/mysqld
*	设置开机自动启动

		vim /etc/rc.local
		/etc/init.d/mysqld start
*	生成配置信息

		https://tools.percona.com/wizard
*	更新配置文件

		vim /etc/my.cnf

*	初始化数据库

		chmod 755 ./scripts/mysql_install_db && ./scripts/mysql_install_db --user=mysql --basedir=/opt/mysql --datadir=/data1/mysql
*	增加软链接

		ln -s /opt/mysql/bin/mysql /usr/bin/mysql
*	启动

		service mysqld start
*	修改密码

		/opt/mysql/bin/mysqladmin -u root password '123123'
*	重启

		service mysqld restart
		

#####安装 RabbitMQ
*	安装python-simplejson

		yum install python-simplejson
*	安装 Erlang

		wget http://www.erlang.org/download/otp_src_17.0.tar.gz
		tar zxvf otp_src_17.0.tar.gz
		cd otp_src_17.0
		./configure --prefix=/opt/erlang
		make
		make install
		
		ln -s /opt/erlang/bin/erl /usr/bin/erl
		ln -s /opt/erlang/bin/escript /usr/bin/escript
		ln -s /opt/erlang/bin/erlc /usr/bin/erlc
*	安装rabbitmq

		wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.3.1/rabbitmq-server-3.3.1.tar.gz
		tar zxvf rabbitmq-server-3.3.1.tar.gz
		cd rabbitmq-server-3.3.1
		make TARGET_DIR=/opt/rabbitmq SBIN_DIR=/opt/rabbitmq/sbin MAN_DIR=/opt/rabbitmq/man DOC_INSTALL_DIR=/opt/rabbitmq/doc install
*	安装plugin -- management

		vim /opt/rabbitmq/sbin/rabbitmq-server
		CONFIG_FILE=/etc/rabbitmq/rabbitmq.config
		
		mkdir /etc/rabbitmq
		/opt/rabbitmq/sbin/rabbitmq-plugins enable rabbitmq_management

*	配置

		mkdir /var/log/rabbitmq
		mkdir /var/lib/rabbitmq
		
		ln -s /opt/rabbitmq/sbin/rabbitmq-server /usr/bin/rabbitmq-server
		ln -s /opt/rabbitmq/sbin/rabbitmq-env /usr/bin/rabbitmq-env
		ln -s /opt/rabbitmq/sbin/rabbitmqctl /usr/bin/rabbitmqctl
*	启动

		rabbitmq-server &
*	停止

		rabbitmqctl stop
*	开放端口

		vim /etc/sysconfig/iptables
		-A INPUT -m state --state NEW -m tcp -p tcp --dport 15672 -j ACCEPT
		
		service iptables restart
*	web访问

		http://10.211.55.40:15672
*	增加管理用户

		rabbitmqctl add_user nate 123
		rabbitmqctl set_user_tags nate administrator
		rabbitmqctl set_permissions -p / nate ".*" ".*" ".*"


##### 安装Keystone
*	安装keystone

		wget https://launchpad.net/keystone/icehouse/2014.1.1/+download/keystone-2014.1.1.tar.gz
		tar zxvf keystone-2014.1.1.tar.gz
		cd keystone-2014.1.1
		python setup.py install
		
		mkdir /etc/keystone
		mkdir /var/log/keystone
		cp ./etc/default_catalog.templates /etc/keystone/
		cp ./etc/keystone.conf.sample /etc/keystone/keystone.conf
		cp ./etc/logging.conf.sample /etc/keystone/logging.conf
		cp ./etc/policy.json /etc/keystone/
		cp ./etc/keystone-paste.ini /etc/keystone/
		
*	安装python-keystoneclient

		git clone https://github.com/openstack/python-keystoneclient.git
		cd python-keystoneclient
		python setup.py install
*	创建keystone数据库

		mysql -uroot -p123123 -e 'create database keystone'
*	修改Keystone的配置文件

		sed -i 's/#connection=<None>/connection=mysql:\/\/root:123123@127.0.0.1\/keystone/g' /etc/keystone/keystone.conf
*	同步数据库

		pip install pbr
		keystone-manage db_sync
*	查看数据库

		mysql -uroot -p123123 -e 'use keystone;show tables;'
*	创建设置环境变量文件

		openssl rand -hex 10
		d3fda68ea86163e734d3

		vim ~/creds
		export OS_USERNAME=admin
		export OS_TENANT_NAME=admin
		export OS_PASSWORD=123123
		export OS_AUTH_URL=http://127.0.0.1:5000/v2.0
		export SERVICE_TOKEN=d3fda68ea86163e734d3
		export SERVICE_ENDPOINT=http://127.0.0.1:35357/v2.0

		source ~/creds
*	创建密钥

		keystone-manage pki_setup --keystone-user root --keystone-group root
*	配置

		sed -i 's/#admin_token=ADMIN/admin_token=d3fda68ea86163e734d3/g' /etc/keystone/keystone.conf
		sed -i 's/#public_bind_host=0.0.0.0/public_bind_host=0.0.0.0/g' /etc/keystone/keystone.conf
		sed -i 's/#admin_bind_host=0.0.0.0/admin_bind_host=0.0.0.0/g' /etc/keystone/keystone.conf
		sed -i 's/#compute_port=8774/compute_port=8774/g' /etc/keystone/keystone.conf
		sed -i 's/#admin_port=35357/admin_port=35357/g' /etc/keystone/keystone.conf
		sed -i 's/#public_port=5000/public_port=5000/g' /etc/keystone/keystone.conf
		sed -i 's/#verbose=false/verbose=true/g' /etc/keystone/keystone.conf
		sed -i 's/#debug=false/debug=true/g' /etc/keystone/keystone.conf
		sed -i 's/#log_file=<None>/log_file=keystone.log/g' /etc/keystone/keystone.conf
		sed -i 's/#log_dir=<None>/log_dir=\/var\/log\/keystone/g' /etc/keystone/keystone.conf
		sed -i 's/#use_syslog=false/use_syslog=false/g' /etc/keystone/keystone.conf
		sed -i 's/#driver=keystone.identity.backends.sql.Identity/driver=keystone.identity.backends.sql.Identity/g' /etc/keystone/keystone.conf
		sed -i 's/#template_file=default_catalog.templates/template_file=\/etc\/keystone\/default_catalog.templates/g' /etc/keystone/keystone.conf
		sed -i 's/#driver=keystone.catalog.backends.sql.Catalog/driver=keystone.catalog.backends.sql.Catalog/g' /etc/keystone/keystone.conf
		sed -i 's/#driver=keystone.token.backends.sql.Token/driver=keystone.token.backends.sql.Token/g' /etc/keystone/keystone.conf
		sed -i 's/#expiration=3600/expiration=3600/g' /etc/keystone/keystone.conf
		sed -i 's/#driver=keystone.policy.backends.sql.Policy/driver=keystone.policy.backends.sql.Policy/g' /etc/keystone/keystone.conf
		sed -i 's/#driver=keystone.contrib.ec2.backends.kvs.Ec2/driver=keystone.contrib.ec2.backends.kvs.Ec2/g' /etc/keystone/keystone.conf
*	启动

		keystone-all --config-file=/etc/keystone/keystone.conf &
*	查看端口

		lsof -i :5000
		lsof -i :35357
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
		export ip=10.211.55.40
		
		获取 service id

		keystone service-list       
		keystone endpoint-create --service-id=上面命令获取的service_id --publicurl=http://$ip:5000/v2.0 --internalurl=http://$ip:5000/v2.0 --adminurl=http://$ip:35357/v2.0


#####安装 Glance
*	安装python-swiftclient

		git clone https://github.com/openstack/python-swiftclient.git
		cd python-swiftclient
		python setup.py install
*	安装python-glanceclient

		git clone https://github.com/openstack/python-glanceclient.git
		cd python-glanceclient
		python setup.py install

*	安装Glance

		wget https://launchpad.net/glance/icehouse/2014.1.1/+download/glance-2014.1.1.tar.gz
		tar zxvf glance-2014.1.1.tar.gz
		cd glance-2014.1.1
		python setup.py install
		
		mkdir /etc/glance
		mkdir /var/log/glance
		mkdir /var/lib/glance
		
		cp ./etc/glance-api.conf /etc/glance/
		cp ./etc/glance-api-paste.ini /etc/glance/
		cp ./etc/glance-registry.conf /etc/glance/
		cp ./etc/glance-registry-paste.ini /etc/glance/
		cp ./etc/logging.cnf.sample /etc/glance/logging.cnf
		cp ./etc/policy.json /etc/glance/
		cp ./etc/schema-image.json /etc/glance/
*	创建glance用户

		keystone user-create --name=glance --pass=123123 --email=nate_yhz@outlook.com
*	绑定用户

		keystone user-role-add --user=glance --tenant=service --role=admin
*	创建服务

		keystone service-create --name=glance --type=image --description="Glance ImageService"
*	创建endpoint

		外部IP
		export ip=10.211.55.40

		获取 service id 
		keystone service-list
		keystone endpoint-create --service-id=上面命令获取的service_id --publicurl=http://$ip:9292 --internalurl=http://$ip:9292 --adminurl=http://$ip:9292
*	修改/etc/glance/glance-api.conf

		sed -i 's/#connection = <None>/connection = mysql:\/\/root:123123@127.0.0.1\/glance/g' /etc/glance/glance-api.conf
		sed -i 's/admin_tenant_name = %SERVICE_TENANT_NAME%/admin_tenant_name = service/g' /etc/glance/glance-api.conf
		sed -i 's/admin_user = %SERVICE_USER%/admin_user = glance/g' /etc/glance/glance-api.conf
		sed -i 's/admin_password = %SERVICE_PASSWORD%/admin_password = 123123/g' /etc/glance/glance-api.conf
		
		sed -i 's/# notifier_strategy = default/notifier_strategy = rabbit/g' /etc/glance/glance-api.conf

		sed -i 's/rabbit_host = localhost/rabbit_host = openstack/g' /etc/glance/glance-api.conf
		sed -i 's/rabbit_userid = guest/rabbit_userid = nate/g' /etc/glance/glance-api.conf
		sed -i 's/rabbit_password = guest/rabbit_password = 123/g' /etc/glance/glance-api.conf
		
		sed -i 's/#flavor=/flavor=keystone/g' /etc/glance/glance-api.conf
		sed -i 's/#config_file = glance-api-paste.ini/config_file = \/etc\/glance\/glance-api-paste.ini/g' /etc/glance/glance-api.conf
		
		vim /etc/glance/glance-api.conf

		[filter:authtoken]
		paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
		auth_host = 127.0.0.1
		auth_port = 35357
		auth_protocol = http
		admin_tenant_name = service
		admin_user = glance
		admin_password = 123123  


*	修改/etc/glance/glance-registry.conf

		sed -i 's/#connection = <None>/connection = mysql:\/\/root:123123@127.0.0.1\/glance/g' /etc/glance/glance-registry.conf
		
		sed -i 's/admin_tenant_name = %SERVICE_TENANT_NAME%/admin_tenant_name = service/g' /etc/glance/glance-registry.conf			sed -i 's/admin_user = %SERVICE_USER%/admin_user = glance/g' /etc/glance/glance-registry.conf
		sed -i 's/admin_password = %SERVICE_PASSWORD%/admin_password = 123123/g' /etc/glance/glance-registry.conf
		
		sed -i 's/#flavor=/flavor=keystone/g' /etc/glance/glance-registry.conf

		sed -i 's/#config_file = glance-api-paste.ini/config_file = \/etc\/glance\/glance-api-paste.ini/g' /etc/glance/glance-registry.conf
		
		
		vim /etc/glance/glance-registry.conf

		[filter:authtoken]
		paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
		auth_host = 127.0.0.1
		auth_port = 35357
		auth_protocol = http
		admin_tenant_name = service
		admin_user = glance
		admin_password = 123123  
		
*	创建glance数据库

		mysql -uroot -p123123 -e 'create database glance'
*	同步数据库

		glance-manage db_sync
		mysql -uroot -p123123 -e 'use glance; alter table migrate_version convert to character set utf8 collate utf8_unicode_ci;'
		glance-manage db_sync

*	查看数据库
		
		mysql -uroot -p123123 -e 'use glance; show tables;'
*	启动

		glance-control all start
*	停止

		glance-control all stop
*	重启

		glance-control all restart

*	查看端口

		lsof -i :9191
		lsof -i :9292
*	测试(可wget下来再 < 导入)

		glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
*	列出所有映像

		glance image-list


#####安装 Nova
*	安装nova

		wget https://launchpad.net/nova/icehouse/2014.1.1/+download/nova-2014.1.1.tar.gz
		tar zxvf nova-2014.1.1.tar.gz
		cd nova-2014.1.1
		python setup.py install
		
		mkdir /etc/nova
		mkdir /var/log/nova
		mkdir -p /var/lib/nova/instances
		
		cp ./etc/nova/api-paste.ini /etc/nova/
		cp ./etc/nova/logging_sample.conf /etc/nova/
		cp ./etc/nova/policy.json /etc/nova/
		cp ./etc/nova/rootwrap.conf /etc/nova/
		cp -rp ./etc/nova/rootwrap.d/ /etc/nova/
*	安装python-novaclient

		git clone https://github.com/openstack/python-novaclient.git
		cd python-novaclient
		python setup.py install
*	创建nova用户

		keystone user-create --name=nova --pass=123123 --email=nate_yhz@outlook.com
*	绑定用户

		keystone user-role-add --user=nova --tenant=service --role=admin
*	创建服务

		keystone service-create --name=nova --type=compute --description="Nova Compute Service"
*	创建endpoint

		外部IP
		export ip=10.211.55.40

		获取 service id 
		keystone service-list
		keystone endpoint-create --service-id=上面命令获取的service_id --publicurl=http://$ip:8774/v2/%\(tenant_id\)s --internalurl=http://$ip:8774/v2/%\(tenant_id\)s --adminurl=http://$ip:8774/v2/%\(tenant_id\)s
		

*	修改/etc/nova/api-paste.ini

		sed -i 's/%SERVICE_TENANT_NAME%/service/g' /etc/nova/api-paste.ini
		sed -i 's/%SERVICE_USER%/nova/g' /etc/nova/api-paste.ini
		sed -i 's/%SERVICE_PASSWORD%/123123/g' /etc/nova/api-paste.ini
*	修改nova.conf

		vim /etc/nova/nova.conf
		[DEFAULT]
		# logs
		verbose = True
		
		# dir
		state_path = /var/lib/nova
		lock_path = /var/lock/nova  
		logdir = /var/log/nova
		
		# authentication
		auth_strategy = keystone
		
		# scheduler
		#compute_scheduler_driver = nova.scheduler.filter_scheduler.FilterScheduler
		compute_scheduler_driver = nova.scheduler.simple.SimpleScheduler

		
		# volumns
		volume_group = nova-volumes
		volume_name_template = volume-%08x
		iscsi_helper = tgtadm
		
		# database
		sql_connection = mysql://root:123123@127.0.0.1/nova
		
		# compute
		compute_driver=libvirt.LibvirtDriver
		libvirt_type = kvm
		connection_type = libvirt
		instances_path = /var/lib/nova/instances
		instance_name_template = instance-%08x
		api_paste_config = /etc/nova/api-paste.ini
		allow_resize_to_same_host = True
		
		# rabbitmq
		rpc_backend=nova.openstack.common.rpc.impl_kombu
		rabbit_host = openstack
		rabbit_userid = nate
		rabbit_port = 5672
		rabbit_password = 123
		
		# glance
		image_service = nova.image.glance.GlanceImageService
		glance_api_servers = 10.211.55.40:9292
		
		# network
		dhcpbridge_flagfile=/etc/nova/nova.conf
		dhcpbridge=/usr/bin/nova-dhcpbridge
		my_ip = 10.211.55.40
		network_manager = nova.network.manager.FlatDHCPManager
		firewall_driver = nova.virt.firewall.NoopFirewallDriver
		multi_host = True
		flat_interface = eth1
		flat_network_bridge = br1
		public_interface = eth0
		
		
		# novnc
		novncproxy_base_url = http://10.211.55.40:6080/vnc_auto.html
		vncserver_listen = 10.211.55.40
		vncserver_proxyclient_address = 10.211.55.40
		vnc_enabled = true
		vnc_keymap = en-us
		nova_url=http://127.0.0.1:8774/v1.1/
		
		root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
		instance_usage_audit = True
		instance_usage_audit_period = hour
		notify_on_state_change = vm_and_task_state
		notification_driver = nova.openstack.common.notifier.rpc_notifier
		
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
*	创建nova数据库

		mysql -uroot -p123123 -e 'create database nova'
*	同步数据库

		nova-manage db sync

*	查看数据库
		
		mysql -uroot -p123123 -e 'use nova; show tables;'
*	启动
		
		/etc/init.d/libvirtd restart
		nova-conductor --config-file=/etc/nova/nova.conf &
		nova-cert --config-file=/etc/nova/nova.conf &
		nova-network --config-file=/etc/nova/nova.conf &
		nova-api-os-compute --config-file=/etc/nova/nova.conf &
		nova-api-metadata --config-file=/etc/nova/nova.conf &
		nova-compute --config-file=/etc/nova/nova.conf &
		nova-scheduler --config-file=/etc/nova/nova.conf &
		
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


#####安装 noVNC
*	安装

		git clone https://github.com/kanaka/noVNC.git
		
		wget https://github.com/kanaka/websockify/archive/v0.5.1.tar.gz
		tar zxvf v0.5.1.tar.gz 
		cd websockify-0.5.1
		python setup.py install
		
		cp -rp /root/noVNC /usr/share/novnc
		
*	启动

		nova-consoleauth --config-file=/etc/nova/nova.conf &
		nova-novncproxy --config-file=/etc/nova/nova.conf &
		
#####安装 Cinder
*	安装 Cinder

		wget https://launchpad.net/cinder/icehouse/2014.1.1/+download/cinder-2014.1.1.tar.gz
		tar zxvf cinder-2014.1.1.tar.gz
		cd cinder-2014.1.1
		python setup.py install
		
		mkdir /etc/cinder
		mkdir /var/log/cinder
		mkdir /var/lib/cinder
		
		cp ./etc/cinder/api-paste.ini /etc/cinder/
		cp ./etc/cinder/cinder.conf.sample /etc/cinder/cinder.conf
		cp ./etc/cinder/logging_sample.conf /etc/cinder/logging.conf
		cp ./etc/cinder/policy.json /etc/cinder/
		cp ./etc/cinder/rootwrap.conf /etc/cinder/
		cp -rp ./etc/cinder/rootwrap.d/ /etc/cinder/
		
*	安装 python-cinderclient

		git clone https://github.com/openstack/python-cinderclient.git
		cd python-cinderclient
		python setup.py install
*	安装 iscsitarget

		wget http://jaist.dl.sourceforge.net/project/iscsitarget/iscsitarget/1.4.20.2/iscsitarget-1.4.20.2.tar.gz
		tar zxvf iscsitarget-1.4.20.2.tar.gz
		cd iscsitarget-1.4.20.2
		make
		make install
*	创建cinder用户

		keystone user-create --name=cinder --pass=123123 --email=nate_yhz@outlook.com
*	绑定用户

		keystone user-role-add --user=cinder --tenant=service --role=admin
*	创建服务

		keystone service-create --name=cinder --type=volume --description="OpenStack Block Storage"

		keystone service-create --name=cinderv2 --type=volumev2 --description="OpenStack Block Storage v2"
*	创建endpoint

		export ip=10.211.55.40
		
		keystone endpoint-create --service-id=$(keystone service-list | awk '/ volume / {print $2}') --publicurl=http://$ip:8776/v1/%\(tenant_id\)s --internalurl=http://$ip:8776/v1/%\(tenant_id\)s --adminurl=http://$ip:8776/v1/%\(tenant_id\)s

		keystone endpoint-create --service-id=$(keystone service-list | awk '/ volumev2 / {print $2}') --publicurl=http://$ip:8776/v2/%\(tenant_id\)s --internalurl=http://$ip:8776/v2/%\(tenant_id\)s --adminurl=http://$ip:8776/v2/%\(tenant_id\)s
*	修改 targets.conf

		vim /etc/tgt/targets.conf
		
		include /var/lib/cinder/volumes/*
*	修改 cinder.conf

		vim /etc/cinder/cinder.conf
		
		debug=true
		verbose=true
		use_stderr=false
		
		log_dir=/var/log/cinder
		use_syslog=false
		
		connection=mysql://root:123123@127.0.0.1/cinder
		state_path=/var/lib/cinder
		api_paste_config=/etc/cinder/api-paste.ini
		my_ip=10.211.55.40
		glance_host=$my_ip
		glance_port=9292
		glance_api_servers=$glance_host:$glance_port
		
		rootwrap_config=/etc/cinder/rootwrap.conf
		auth_strategy=keystone
		policy_file=policy.json
		
		osapi_volume_listen=0.0.0.0
		osapi_volume_listen_port=8776
		
		rpc_backend=rabbit
		rabbit_host=openstack
		rabbit_port=5672
		rabbit_hosts=$rabbit_host:$rabbit_port
		rabbit_userid=nate
		rabbit_password=123
		
		volume_group=cinder-volumes
		volumes_dir=$state_path/volumes
		volume_name_template=volume-%s
		
		iscsi_target_prefix=iqn.2010-10.org.openstack:
		iscsi_ip_address=$my_ip
		iscsi_port=3260
		iscsi_helper=tgtadm
		
		[keystone_authtoken]
		auth_uri=http://127.0.0.1:5000
		auth_host=127.0.0.1
		auth_protocol=http
		auth_port=35357
		admin_user=cinder
		admin_tenant_name=service
		admin_password=123123
		
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
*	创建数据库

		mysql -uroot -p123123 -e 'create database cinder'
*	同步数据库

		cinder-manage db sync

*	查看数据库

		mysql -uroot -p123123 -e 'use cinder; show tables;'
*	启动 iscsi

		vim /etc/init.d/iscsi-target
		#modprobe -q crc32c
		/etc/init.d/iscsi-target start
		/etc/init.d/tgtd start
*	启动 cinder

		cinder-api --config-file=/etc/cinder/cinder.conf &
		cinder-scheduler --config-file=/etc/cinder/cinder.conf &
		cinder-volume --config-file=/etc/cinder/cinder.conf &

#####安装 Horizon
*	安装 Apache

		yum install httpd mod_ssl mod_wsgi
*	安装 memcached

		yum install memcached python-memcached	
		
		service memcached restart
*	安装Node.js

		wget http://nodejs.org/dist/v0.10.28/node-v0.10.28.tar.gz
		tar zxvf node-v0.10.28.tar.gz
		cd node-v0.10.28
		
		./configure
		make
		make install
*	安装 less

		npm install -g less

*	重启 Apache

		service httpd restart
*	设置 Apache 开机启动

		chkconfig httpd on
*	安装 Horizon

		wget https://launchpad.net/horizon/icehouse/2014.1.1/+download/horizon-2014.1.1.tar.gz
		tar zxvf horizon-2014.1.1.tar.gz
		cd horizon-2014.1.1
		python setup.py install
		
		cd ..
		mv ./horizon-2014.1.1 /usr/share/

*	创建配置文件

		mv /usr/share/horizon-2014.1.1/openstack_dashboard/local/local_settings.py.example /usr/share/horizon-2014.1.1/openstack_dashboard/local/local_settings.py
*	修改配置文件

		vi /usr/share/horizon-2014.1.1/openstack_dashboard/local/local_settings.py
		
		ALLOWED_HOSTS = ['*']
		
		DATABASES = {
    		'default': {
    			'ENGINE': 'django.db.backends.mysql',
    			'NAME': 'horizon',
    			'USER': 'root',
    			'PASSWORD': '123123',
    			'HOST': '127.0.0.1',
    			'PORT': '3306',
    		},
		}
		
		#SECRET_KEY = secret_key.generate_or_read_from_file(os.path.join(LOCAL_PATH, '.secret_key_store'))
		SECRET_KEY = 'As2hyCt7iAv1Jut8hy2weM6vIefsess'
		
		CACHES = {
    		'default': {
        		'BACKEND' : 'django.core.cache.backends.memcached.MemcachedCache',
        		'LOCATION' : '127.0.0.1:11211',
    		}
		}
		
*	配置apache

		vi /etc/httpd/conf.d/horizon.conf
		<VirtualHost *:80>
    		WSGIScriptAlias / /usr/share/horizon-2014.1.1/openstack_dashboard/wsgi/django.wsgi
    		WSGIDaemonProcess horizon user=apache group=apache processes=3 threads=10 home=/usr/share/horizon-2014.1.1
    		WSGIApplicationGroup horizon

    		SetEnv APACHE_RUN_USER apache
    		SetEnv APACHE_RUN_GROUP apache
    		WSGIProcessGroup horizon

    		DocumentRoot /usr/share/horizon-2014.1.1/.blackhole/
    		Alias /media /usr/share/horizon-2014.1.1/openstack_dashboard/static

    		<Directory />
        		Options FollowSymLinks
        		AllowOverride None
   		 	</Directory>

    		<Directory /usr/share/horizon-2014.1.1/>
        		Options Indexes FollowSymLinks MultiViews
        		AllowOverride None
        		Order allow,deny
        		Allow from all
    		</Directory>

    		ErrorLog /var/log/httpd/horizon_error.log
    		LogLevel warn
    		CustomLog /var/log/httpd/horizon_access.log combined
		</VirtualHost>
		WSGISocketPrefix /var/run/horizon
*	创建目录设置权限

		mkdir -p /usr/share/horizon-2014.1.1/.blackhole
		chown -R apache:apache /usr/share/horizon-2014.1.1
		
		ln -s /usr/share/horizon-2014.1.1/openstack_dashboard/static /usr/share/horizon-2014.1.1/.blackhole/static
		ln -s /usr/share/horizon-2014.1.1/openstack_dashboard/static/bootstrap /usr/share/horizon-2014.1.1/.blackhole/bootstrap

*	创建数据库

		mysql -uroot -p123123 -e 'create database horizon'
*	同步数据库

		python /root/horizon-2014.1.1/manage.py syncdb --noinput

*	查看数据库

		mysql -uroot -p123123 -e 'use horizon; show tables;'
*	重启apache

		/etc/init.d/httpd restart


		
##### 出现的错误及解决办法
*	RequiredOptError: value required for option: lock_path

		lock_path = /var/lock/nova 
		
*	got an unexpected keyword argument 'no_parent'

		不要升级websockify 0.6.0, 安装 0.5.1
*	Unable to update stats, LVMISCSIDriver -2.0.0  driver is uninitialized.


	



		


		




		


		


		

		



		
















		


		

		

		












		









	