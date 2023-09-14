---
layout: post
title: Install Latest Mysql in Ubuntu21.04 in WSL
date: 2022-03-29 16:26 +0800
tags: [config, bug fix, Mysql]
categories: [Linux, Config]
author: Jebearssica
---

本篇博客有关于在Ubuntu21.04安装latest Mysql(Ver 8.0.23-3ubuntu1 for Linux on x86_64)并进行初始化的操作, 参考链接如下

* 主体部分: <https://segmentfault.com/a/1190000023081074>
* 有关Error:
  * ERROR 1064 (42000): <https://stackoverflow.com/questions/7534056/mysql-root-password-change>

## 安装

```sh
sudo apt-get install mysql-server
sudo apt-get install mysql-client
# mysql开发依赖项
sudo apt-get install libmysqlclient-dev
```

## 更改默认密码

```sh
# 查看默认配置文件, 找到默认账号与密码以进入mysql
sudo cat /etc/mysql/debian.cnf
# 登录进入
mysql -u debian-sys-maint -p

# 更改密码, 原连接中的命令在mysql5.7后被弃用
# Ubuntu 20.04 has the root user's auth plugin as: auth_socket. That plugin does not support a password. There is an answer below that talks about it. Solution is to change the plugin and password in one statement : ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Password'; The "WITH mysql_native_password" part changes the plugin. – cbmckay Aug 5 at 18:23
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Password'; 
flush privileges;
quit;
```

## 重启服务

```sh
sudo service mysql restart mysql -u root -p 新密码
```

后续进行远程配置目前不需要, 就没继续

## Failed starting mysql

warning message

```sh
su: warning: cannot change directory to /nonexistent: No such file or directory
```

Reference: <https://stackoverflow.com/questions/62987154/mysql-wont-start-error-su-warning-cannot-change-directory-to-nonexistent>

```sh
sudo service mysql stop
sudo usermod -d /var/lib/mysql/ mysql
sudo service mysql start
```
