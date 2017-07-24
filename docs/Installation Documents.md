# Installation Documents 
{{TOC}}

## Virtual Machine Related 
### Create Virtual Machines 
1. Create a virtual machine with default configuration 
2. Install CentOS 7 

### Linux Configuration 

#### Enable DHCP 

By default, the configuration file is:

```shell
/etc/sysconfig/network-scritps/ifcfg-ens160
```

However, the file name `ifcfg-ens160` may vary depends on network interface name. 

- Find the below lines in config file 
```shell
BOOTPROTO=none
ONBOOT=no
```
and replace with 

```shell
BOOTPROTO=dhcp
ONBOOT=yes
```

- Restart network service with the following command.

```shell
systemctl restart network 
```

- Now check your IP address and record it for future use. 

```shell
ip addr 

# or

ifconfig 
```

#### Disable SeLinux and Firefall 

Although SeLinux and firewall provide excellent permission and security management, they could be obstacles of development. Permission and security setting should also take other applications into consideration, so we disable them in development environment for convenience. 

1. Disable SeLinux 

```shell
vim /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted

```
change `SELINUX` variable to `disable`. This change need a **reboot** to take effect. 


#### Install VMware tools 

Vmware tools provides APIs for virtual machine infrastructure to interact with virtual machines. This could be useful for development and debugging. 

```shell
yum install open-vm-tools
```

**Reboot** the virtual machine to enable the VMware Tools integration. 


#### Install Docker 

- Install Docker 

```shell
yum install docker 
```

- Run Docker service 

```shell
systemctl start docker 
```

Now the development environment is ready, enjoy yourself. 

## Zabbix Server Installation 

Zabbix Server is a headquarter of the whole monitoring system. Concretely, Zabbix Server does everything except collecting raw data from hosts directly. 

Zabbix Server comprises server side and a web front end, so we need to set them up individually. 

### Environment
#### Step 1 - Repository Configuration Package 
As Zabbix Server is not included in the repository of CentOS 7 by default. We need to install it manually by running the following command. 

```shell 
rpm -ivh http://repo.zabbix.com/zabbix/3.2/rhel/7/x86_64/zabbix-release-3.2-1.el7.noarch.rpm
``` 


#### Step 2 - MySQL

Zabbix Server stores data in various kinds of databases, we use MySQL/MariaDB here as an example, so make sure MySQL Server is correctly installed. 

```shell
yum install mariadb-server
```

Once installation is complete, start the daemon and enable auto start on boot by running the following commands.

``` shell
systemctl start mariadb && systemctl enable mariadb 
```

Securing the MariaDB Server 

```shell
mysql_secure_installation
```

**1**: Press [Enter] as we haven’t got a root password for the new installation. 

![](subtopics/assets/mysql-1.png)

**2**: Input **y** to create a new root password. 

![](subtopics/assets/mysql-2.png)

**3**: Input **y** to remove anonymous user, disallow root login, remove test database and reload privilege tables. 

Here screenshots are omitted for space saving. 

#### Step 3 - PHP Environment 

Install Apache Httpd: 

```shell
yum install httpd
```

Install PHP and PHP Mysql plugin 

```shell 
yum install php php-mysql
```




### Installing Zabbix Server 

#### Step 1 - Installing MySQL Version Zabbix Server 

```shell 
yum install zabbix-server-mysql zabbix-web-mysql
```

#### Step 2 - Creating Initial Database for Zabbix Server 

```shell
# root password in the development environment is `zabbix-wadaro` without quotation. 
mysql -u root -p 
```

Now we are in the mysql shell 

```mysql 
mysql> create database zabbix character set utf8 collate utf8_bin;
mysql> grant all privileges on zabbix.* to zabbix@localhost identified by 'zabbix-wadaro';
mysql> quit;
```

Check your Zabbix `patch version`, which seems like `3.2.*`. The `*` is the patch version. 

```shell
rpm -q zabbix-server-mysql

# output: 
zabbix-server-mysql-3.2.6-1.el7.x86_64

# So the patch version is 6. 
```

Import initial schema and data

```shell
zcat /usr/share/doc/zabbix-server-mysql-3.2.[PATCH VERSION]/create.sql.gz | mysql -uzabbix -p zabbix

# An example is 
zcat /usr/share/doc/zabbix-server-mysql-3.2.6/create.sql.gz | mysql -uzabbix -p zabbix
```

#### Step 3 - Database Configuration for Zabbix Server

Change DB Configuration in Zabbix Server config file. 

```shell
vi /etc/zabbix/zabbix_server.conf

## Change the follow keys 
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix-wadaro
```

#### Step 4 - Starting Zabbix Server 

```shell 
systemctl start zabbix-server
systemctl enable zabbix-server
```

#### Step 5 - PHP Configuration for Zabbix frontend

The detailed configuration should fit the requirements of your own environments, such as upload limits and max input time.

Make sure that the time zone is correctly configured. 

```shell
vim  /etc/httpd/conf.d/zabbix.conf
```

Change the time zone to UTC. 

```shell
php_value max_execution_time 300
php_value memory_limit 128M
php_value post_max_size 16M
php_value upload_max_filesize 2M
php_value max_input_time 300
php_value always_populate_raw_post_data -1
# php_value date.timezone Europe/Riga

## change the below line to UTC 
php_value date.timezone UTC
```

Starting apache2 server.

```shell
systemctl start httpd
```

#### Step 6 - Installing Web Frontend 

This step is fairly straightforward, just follow the installation guide. The only thing need to do is to input the password. 

An official guide is here for reference: [Link](https://www.zabbix.com/documentation/3.2/manual/installation/install#installing_frontend)

## Dockerized Zabbix Agent Installation 

This part only presents how to setup the dockerized Zabbix Agent. 

Once finished the installation of virtual machines and installation of Docker, run the follow command.

```shell
docker run --name [NAME] -e ZBX_PASSIVESERVERS="[Zabbix Server IP]" -e ZBX_HOSTNAME="[HOST_NAME]" -e ZBX_SERVER_HOST="[Zabbix Server Address]" --privileged -d -p 10050:10050 zabbix/zabbix-agent:ubuntu-3.2.6
```

An concrete example is also provided:

```shell
docker run --name tcc -e ZBX_PASSIVESERVERS="10.3.100.107" -e ZBX_HOSTNAME="TCC" -e ZBX_SERVER_HOST="10.3.100.107" --privileged -d -p 10050:10050 zabbix/zabbix-agent:ubuntu-3.2.6
```

Following is a list of description of each parameters. 

1. [NAME]: a string character to description the Docker container
2. [Zabbix Server IP]: Domain name or IP address of a Zabbix Server
3. [HOST_NAME]: A string that names current Agent. When adding this Agent in the Zabbix web interface, you should put the same string as stated here. 

A ‘verbose’ version of how to install the agent without docker will be updated in the following days. 


