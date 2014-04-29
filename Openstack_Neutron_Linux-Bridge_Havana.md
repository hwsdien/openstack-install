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





	