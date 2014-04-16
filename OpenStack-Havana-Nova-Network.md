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
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	