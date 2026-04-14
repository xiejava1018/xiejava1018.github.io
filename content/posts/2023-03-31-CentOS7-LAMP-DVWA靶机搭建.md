---
title: CentOS7+LAMP+DVWA靶机搭建
id: 20230331001
tags:
  - 网络安全
categories:
  - - 技术
    - 网络安全
abbrlink: 4405dad8
date: 2023-03-31 15:43:55
---
## 一、什么是DVWA
Damn Vulnerable Web Application (DVWA)(译注：可以直译为："该死的"不安全Web应用程序)，是一个编码差的、易受攻击的 PHP/MySQL Web应用程序。 它的主要目的是帮助信息安全专业人员在合法的环境中，练习技能和测试工具，帮助 Web 开发人员更好地了解如何加强 Web 应用程序的安全性，并帮助学生和教师在可控的教学环境中了解和学习 Web 安全技术。
DVWA的中文介绍见 https://github.com/digininja/DVWA/blob/master/README.zh.md
下载地址：git clone https://github.com/digininja/DVWA.git

## 二、环境准备
### 1、LAMP环境安装
DVWA的安装依赖的软件包如下：

 - apache2 
 - libapache2-mod-php 
 - mariadb-server 
 - mariadb-client php
 - php-mysqli php-gd

就是依赖于LAMP环境，可以参考官方文档一个依赖包手工安装也可以通过下载lamp统一安装脚本一键安装。
安装 - LAMP一键安装包 
运行 `wget -c http://soft.vpser.net/lnmp/lnmp1.6.tar.gz && tar zxf lnmp1.6.tar.gz && cd lnmp1.6 && ./install.sh lamp` 一路回车选择默认项，稍等片刻，即可完成安装
如果是手工安装：`yum install -y httpd php php-mysql php-gd mariadb-server mariadb`

## 三、安装DVWA
### 1、下载DVWA的软件包
进入到默认的web发布目录

```bash
cd /var/www/html
git clone https://github.com/digininja/DVWA.git
```
直接通过地址访问DVWA
![访问DVWA](http://image2.ishareread.com/images/2023/20230331/访问DVWA.png)
他会提示需要将config/config.inc.php.dist复制成config/config.inc.php并配置环境

```bash
cp config.inc.php.dist config.inc.php
```

### 2、配置数据库

```bash
vim config.ini.php
```
找到数据库的配置信息
```bash
$_DVWA = array();
$_DVWA[ 'db_server' ]   = '127.0.0.1';
$_DVWA[ 'db_database' ] = 'dvwa';
$_DVWA[ 'db_user' ]     = 'dvwa';
$_DVWA[ 'db_password' ] = 'p@ssw0rd';
$_DVWA[ 'db_port'] = '3306';
```
根据config.inc.php的数据库配置信息配置数据库，注意不要用root来访问数据库。
先用客户端工具创建dvwa的数据库，再创建dvwa的用户
```bash
grant all on dvwa.* to 'dvwa'@'localhost' identified by 'p@ssw0rd' with grant option;
flush privileges;
```
登录数据库查看
```bash
mysql -u dvwa -p
show databases;
```
![show databases](http://image2.ishareread.com/images/2023/20230331/showdatabases.png)

### 3、修改php.ini的配置
再次访问DVWA
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230331/setupcheck.png)

这里提示要修改php.ini的配置将 `allow_url_fopen=On` 和`allow_url_include=On`。

找到环境的php.ini我这里是在/etc/php.ini进行修改，不要修改/var/www/html/DVWA/php.ini中的配置了。
配置好后重启apache 

```bash
systemctl restart httpd
```

刷新浏览器，可以看到红色的Disabled告警消失了。
![红色告警消失了](http://image2.ishareread.com/images/2023/20230331/检查check.png)

reCAPTCHA key: Missing 这个告警是因为reCAPTCHA没有配置，这个需要去谷歌的网站申请公钥和私钥。可以不用管

```bash
# ReCAPTCHA settings
#   Used for the 'Insecure CAPTCHA' module
#   You'll need to generate your own keys at: https://www.google.com/recaptcha/admin
$_DVWA[ 'recaptcha_public_key' ]  = '';
$_DVWA[ 'recaptcha_private_key' ] = '';
```
```bash
chmod 777 -R hackable
chmod 777 -R config
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230331/修复结果.png)

### 4、创建数据
配置完后点击“Create/Reset Database”

![成功创建数据库](http://image2.ishareread.com/images/2023/20230331/createdatabase.png)


### 5、登录靶机
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230331/dvwa登录.png)
DVWA的默认用户名和密码是admin /password
登录成功后就可以开始进行靶机的实验了。
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230331/DVWA登录成功.png)
后续我们将通过Open-WAF来搭建一个WAF来防护这个靶机感受一下软waf的防护情况。

--------------
作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)


<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>