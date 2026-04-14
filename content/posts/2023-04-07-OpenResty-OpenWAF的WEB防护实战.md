---
title: OpenResty+OpenWAF的WEB防护实战
id: 20230407001
tags:
  - 网络安全
categories:
  - - 技术
    - 网络安全
abbrlink: dbe7a5ab
date: 2023-04-07 16:38:57
---

OpenResty是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。本文介绍通过OpenResty+OpenWAF来搭建软WAF的应用，用来防护DVWA的靶机，然后我们通过攻击DVWA的靶机来看一下OpenWAF的防护效果。

## 一、OpenResty+OpenWAF安装
### 1、安装依赖

```bash
yum install gcc gcc-c++ wget GeoIP-devel git swig make perl perl-ExtUtils-Embed readline-devel zlib-devel -y
```

安装libcidr

```bash
cd /opt
wget http://www.over-yonder.net/~fullermd/projects/libcidr/libcidr-1.2.3.tar.xz
tar -xvf libcidr-1.2.3.tar.xz
cd /opt/libcidr-1.2.3
make && make install
```

升级openssl版本

```bash
cd /opt
wget -c http://www.openssl.org/source/openssl-1.1.1d.tar.gz --no-check-certificat
tar -zxvf openssl-1.1.1d.tar.gz
cd openssl-1.1.1d/
./config
make && make install
```

下载pcre-jit
并解压pcre-jit，后面安装OpenResty的时候引入并安装
```bash
wget https://udomain.dl.sourceforge.net/project/pcre/pcre/8.45/pcre-8.45.tar.gz --no-check-certificate
tar -zxvf pcre-8.45.tar.gz
```

### 2、安装OpenWAF

```bash
cd /opt
git clone https://github.com/titansec/OpenWAF.git
mv /opt/OpenWAF/lib/openresty/ngx_openwaf.conf /etc
mv /opt/OpenWAF/lib/openresty/configure /opt/openresty-1.19.3.1
cp -RP /opt/OpenWAF/lib/openresty/* /opt/openresty-1.19.9.1/bundle/
cd /opt/OpenWAF/
make clean
make install
ln -s /usr/local/lib/libcidr.so /opt/OpenWAF/lib/resty/libcidr.so
```

### 3、安装OpenResty
OpenResty官网的下载地址 https://openresty.org/en/download.html
目前最新版本是1.21.4.1

```bash
cd /opt
wget https://openresty.org/download/openresty-1.21.4.1.tar.gz
tar -zxvf openresty-1.21.4.1.tar.gz
cd /opt/openresty-1.21.4.1/
./configure --with-pcre-jit --with-ipv6 --with-http_stub_status_module --with-http_ssl_module --with-http_realip_module --with-http_sub_module --with-http_geoip_module --with-openssl=/opt/openssl-1.1.1d --with-pcre=/opt/pcre-8.45
gmake && gmake install
```

设置nginx开机自启动服务

```bash
vim /lib/systemd/system/nginx.service
[Unit]
Description=nginx
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/openresty/nginx/sbin/nginx
ExecReload=/usr/local/openresty/nginx/sbin/nginx -s reload
ExecStop=/usr/local/openresty/nginx/sbin/nginx -s quit
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
# 设置开机启动
systemctl enable nginx
# 查看服务当前状态
systemctl status nginx
# 启动nginx服务
systemctl start nginx
# 停止nginx服务
systemctl stop nginx
# 重启nginx服务
systemctl restart nginx
```
当我们启动nginx的时候发现启动失败了，原因是因为原来安装了apache端口是80，nginx的端口也是80，端口冲突了。解决方案要不是改nginx端口，要不就是改apache的端口。这里将apache的端口改成8080。

```bash
[root@localhost OpenWAF]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
查看nginx启动状态
```bash
[root@localhost OpenWAF]# systemctl status nginx
● nginx.service - nginx
Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
Active: failed (Result: exit-code) since Tue 2023-04-04 04:00:44 PDT; 19s ago
Process: 42096 ExecStart=/usr/local/openresty/nginx/sbin/nginx (code=exited, status=1/FAILURE)
Apr 04 04:00:42 localhost.localdomain nginx[42096]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
Apr 04 04:00:42 localhost.localdomain nginx[42096]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
Apr 04 04:00:43 localhost.localdomain nginx[42096]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
Apr 04 04:00:43 localhost.localdomain nginx[42096]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
Apr 04 04:00:44 localhost.localdomain nginx[42096]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
Apr 04 04:00:44 localhost.localdomain nginx[42096]: nginx: [emerg] still could not bind()
```

修改apache的端口

```bash
vim /etc/httpd/conf/httpd.conf
Listen 8080
systemctl restart httpd
```
![修改apache端口的效果](http://image2.ishareread.com/images/2023/20230407/1-apache端口.png)

将apache的端口改成8080后，再次启动nginx就可以看到OpenResty成功启动了。

```bash
systemctl start nginx
```
![OpenResty成功启动了](http://image2.ishareread.com/images/2023/20230407/2-成功启动OpenResty.png)

## 二、配置OpenWAF的web防护
这边DVWA靶机的地址是http://192.168.1.24:8080/DVWA/  DVWA靶机的安装见另一篇博文《[CentOS7+LAMP+DVWA靶机搭建](https://blog.csdn.net/fullbug/article/details/129879670)》[https://blog.csdn.net/fullbug/article/details/129879670](https://blog.csdn.net/fullbug/article/details/129879670)
我们需要配置OpenResty+OpenWAF来对192.168.1.24:8080进行WEB防护
参考《[轻松玩转OpenWAF之入门篇](https://github.com/titansec/OpenWAF/blob/master/doc/%E8%BD%BB%E6%9D%BE%E7%8E%A9%E8%BD%ACOpenWAF%E4%B9%8B%E5%85%A5%E9%97%A8%E7%AF%87.md)》及 《[深入研究OpenWAF之nginx配置](https://github.com/titansec/OpenWAF/blob/master/doc/%E6%B7%B1%E5%85%A5%E7%A0%94%E7%A9%B6OpenWAF%E4%B9%8Bnginx%E9%85%8D%E7%BD%AE.md)》
### 1、nginx配置修改
在 nginx 的 http 级别添加如下两行：
```bash
include /opt/OpenWAF/conf/twaf_main.conf;
include /opt/OpenWAF/conf/twaf_api.conf;
```

要防护的 server 或 location 级别添加如下一行：

```bash
include /opt/OpenWAF/conf/twaf_server.conf;
```

OpenResty的nginx的配置文件在 /usr/local/openresty/nginx/conf/nginx.conf

具体配置参考下图：
![nginx.conf相关配置](http://image2.ishareread.com/images/2023/20230407/3-OpenResty的nginx配置.png)

### 2、OpenWAF接入规则修改
修改/opt/OpenWAF/conf/twaf_access_rule.json文件
具体配置参考下图：
![twaf_access_rule.json文件的配置](http://image2.ishareread.com/images/2023/20230407/4-OpenWAF的twaf_access_rule.json的配置.png)

### 3、测试验证
这时候我们访问http://192.168.1.24/DVWA/  ，注意是没有带8080端口的，因为是通过OpenResty+OpenWAF来反向代理了127.0.0.1的8080端口，访问http://192.168.1.24/DVWA/  是经过了OpenWAF防护的。
这时候我们开始通过SQL注入对DVWA的靶机进行SQL注入的攻击。
![SQL注意](http://image2.ishareread.com/images/2023/20230407/5-SQL注入.png)

防护效果：
可以看到OpenWAF提示标识为攻击并记录，提示是有次SQL注入的攻击，并进行了防护。
![SQL注入防护效果](http://image2.ishareread.com/images/2023/20230407/5-SQL注入防护效果.png)

接下来我们进行一次XSS的攻击
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230407/6-XSS攻击.png)


同样OpenWAF给出了XSS的攻击提示，并进行了防护。
![XSS的防护效果](http://image2.ishareread.com/images/2023/20230407/6-XSS防护效果.png)

至此，本文介绍了OpenResty+OpenWAF的安装，并通过配置对DVWA的靶机进行了WEB防护，通过SQL注入及XSS的攻击，验证了OpenWAF的效果。OpenResty+OpenWAF是开源的软WAF解决方案，安装和配置相对简单，对于中小企业的web防护来说不失为一个低成本的解决方案。

-------------
作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)


<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>