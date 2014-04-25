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

		


		

