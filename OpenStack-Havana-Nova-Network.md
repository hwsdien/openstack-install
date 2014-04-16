####OpenStack Nova-Network on Havana 安装记录

#####Author
	nate.yu <nate_yhz@outlook.com>
	
#####Requirements
	Ubuntu 12.04 kernel 3.2.0-58
	
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