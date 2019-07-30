---
layout:     post
title:      centos7安装数据库mysql
subtitle:   centos7安装数据库mysql5.6、5.7
date:       2019-07-16
author:     景山
header-img: img/post-bg-netty.jpg
catalog: true
tags:
    - centos7
    - mysql
---

在新版本的CentOS7中，默认的数据库更新为Mariadb，而非MySQL，所以执行yum install mysql命令只是更新Mariadb数据库，并不会安装 MySQL，安装Mysql需要先卸载Mariadb数据库。
## mysql5.6
### 删除Mariadb
[root@localhost ~]# rpm -qa | grep -i mariadb  
[root@localhost ~]# rpm -qa | grep mariadb | xargs rpm -e --nodeps  
[root@localhost ~]# rpm -qa | grep -i mariadb  
### 安装mysql
[root@localhost ~]# wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm  
[root@localhost ~]# yum -y install mysql-community-release-el7-5.noarch.rpm  
[root@localhost ~]# yum repolist all | grep mysql #查看可安装文件  
[root@localhost ~]# yum install mysql-server  
[root@localhost ~]# systemctl start mysqld.service #启动  
### 设置密码
[root@localhost ~]# mysql -u root  
mysql> use mysql;  
mysql> update user set password=PASSWORD("这里输入root用户密码") where User='root';  
mysql> flush privileges;   
### 设置远程主机登录
[root@localhost ~]# mysql -u root  
mysql> use mysql;  
mysql> GRANT ALL PRIVILEGES ON *.* TO 'your username'@'%' IDENTIFIED BY 'your password';
## mysql5.7
### 删除Mariadb
[root@localhost ~]# rpm -qa | grep -i mariadb  
[root@localhost ~]# rpm -qa | grep mariadb | xargs rpm -e --nodeps  
[root@localhost ~]# rpm -qa | grep -i mariadb  
### 安装mysql
[root@localhost ~]# wget http://repo.mysql.com/mysql57-community-release-el7-10.noarch.rpm
[root@localhost ~]# yum -y install mysql57-community-release-el7-10.noarch.rpm  
[root@localhost ~]# yum repolist all | grep mysql #查看可安装文件  
[root@localhost ~]# yum install mysql-community-server  
[root@localhost ~]# systemctl start mysqld.service #启动  
### 设置密码
[root@localhost ~]# grep "password" /var/log/mysqld.log #查询生产的临时密码  
[root@localhost ~]# mysql -u root -p临时密码  
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'new password';  
mysql> flush privileges;     
### 设置远程主机登录
[root@localhost ~]# mysql -u root  -p密码  
mysql> use mysql;  
mysql> GRANT ALL PRIVILEGES ON *.* TO 'your username'@'%' IDENTIFIED BY 'your password';
