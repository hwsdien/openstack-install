####OpenStack Havana Nova-Network on Centos 6.4安装记录

#####Author
	nate.yu <nate_yhz@outlook.com>
	
#####Requirements
	CentOS release 6.4 	x86_64
	
#####说明
	安装流程参考了网上信息，个人记录，请勿使用，发生一切事情，后果自负！！！
	
#####安装基础软件

*	修改源

		sed -i 's/#baseurl=http:\/\/mirror.centos.org/baseurl=http:\/\/mirrors.yun-idc.com/g' /etc/yum.repos.d/CentOS-Base.repo
		
		rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
		
		rpm -Uvh http://pkgs.repoforge.org/rpmforge-release/rpmforge-release-0.5.3-1.el6.rf.x86_64.rpm
		
		yum install http://repos.fedorapeople.org/repos/openstack/openstack-havana/rdo-release-havana-7.noarch.rpm
		
		yum update
		
*	安装vim gcc gcc-c++ make cmake

		yum install vim gcc gcc-c++ make cmake
		
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
		
#####安装MySQL

*	安装

		yum -y install mysql mysql-server MySQL-python
	
*	修改配置文件

		vim /etc/my.cnf
		[mysqld]
		bind-address = 0.0.0.0  
*	启动

		service mysqld start
*	设置开机启动

		chkconfig mysqld on
*	修改密码

		mysqladmin -uroot password '123123'; history -c
*	重启

		service mysqld restart

#####安装RabbitMQ
*	安装

		yum -y install rabbitmq-server
*	启动

		service rabbitmq-server start
*	修改密码

		rabbitmqctl change_password guest nate123

*	设置开机启动

		chkconfig rabbitmq-server on
*	重启

		service rabbitmq-server restart
















		


