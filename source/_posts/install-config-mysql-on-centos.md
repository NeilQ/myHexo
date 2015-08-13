title: 'CentOS上安装配置MySql'
date: 2015-08-13 10:56:11
tags:
- MySql
categories:
- DataBase
---

#安装并开启
执行命令：
```
sudo yum install mysql-server
sudo /sbin/service mysqld start
```
配置安全信息：
```
sudo /usr/bin/mysql_secure_installation
```

#设置iptables以启用远程登陆
执行命令：
```
iptables -A INPUT -p tcp --dport 3306 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p tcp --sport 3306 -m state --state ESTABLISHED -j ACCEPT

/etc/rc.d/init.d/iptables save
service iptables restart
```

#设置开机启动
```
sudo chkconfig mysqld on
sudo chkconfig iptables on
```

#配置远程登陆账号
运行mysql shell
```
mysql -u root -p
```
在shell中授权一个远程登陆用户

```
// 授权用户从任何主机连接
mysql> GRANT ALL PRIVILEGES ON *.* TO 'myuser'@'%' IDENTIFIED BY 'mypassword' WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
```

```
// 授权用户从指定ip连接, 如192.168.1.2
mysql> GRANT ALL PRIVILEGES ON *.* TO 'myuser'@'192.168.1.2' IDENTIFIED BY 'mypassword' WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
```

```
// 授权用户从指定ip连接到指定数据库, 如从192.168.1.2连接到数据库test
mysql> GRANT ALL PRIVILEGES ON test.* TO 'myuser'@'192.168.1.2' IDENTIFIED BY 'mypassword' WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
```




