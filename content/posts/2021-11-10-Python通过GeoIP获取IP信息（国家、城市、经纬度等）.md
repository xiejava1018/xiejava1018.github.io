---
title: Python通过GeoIP获取IP信息（国家、城市、经纬度等）
id: 20211110001
tags:
  - Python
  - 网络安全
categories:
  - - 技术
    - 开发
abbrlink: 2c5697c0
date: 2021-11-10 10:24:01
---
IP地址信息是非常重要的情报信息，通过IP可以定位到该IP所在的国家、城市、经纬度等。
获取IP信息的方式有很多，很多服务商都提供了相应的地址库或API接口服务。
如国内的ipip.net，国外的ip-api.com、maxmind.com等。
很多公司都是使用Maxmind网站的IP信息库，里面包含着IP的详细信息，有付费的也有免费的，收费与免费的区别就是精准度和覆盖率。

本文介绍下载及定时更新Maxmind的离线库用python通过GeoIP来获取IP信息 
# 一、下载GeoLite2离线地址库
## 1、注册及申请License Key
下载地址库之前先要在Maxmind网站注册同意相应的协议并登陆。

### 1）注册
访问 [https://dev.maxmind.com/geoip/geolite2-free-geolocation-data](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data)
![Maxmiand注册导航](http://image2.ishareread.com/images/2021/20211110/Maxmind注册导航页.png)
点击"Sign Up for GeoLite2" 根据输入框进行注册
![Maxmiand注册表单](http://image2.ishareread.com/images/2021/20211110/Maxmind注册表单页.png)
注意邮箱一定要正确，注册后会发邮件进行确认及修改密码。
根据注册的用户名和修改后的密码登陆就可以直接下载离线包了。
![Maxmind账号信息](http://image2.ishareread.com/images/2021/20211110/Maxmind账号概览.png)
点击"Download Databases"进入到下载页面，可以看到提供了CSV及mmdb两种格式的离线库包，最近的更新时间为2021年11月02日。
![MaxmiandGeoLite2地址库下载](http://image2.ishareread.com/images/2021/20211110/地址库下载.png)
由于IP地址信息是经常有变化的，Maxmind提供了geoipupdate工具来更新离线地址包。该工具使用需要申请账号和License Key

### 2）申请License Key
还是通过刚注册的引导页面，点击“Generate a License Key”
![Maxmind生成License导航页](http://image2.ishareread.com/images/2021/20211110/Maxmind生成License导航页.png)
进如到页面后，点击“Generate new license key”
![Generate new license key](http://image2.ishareread.com/images/2021/20211110/Maxmind生成Licensekey.png)
![License Key生成确定页](http://image2.ishareread.com/images/2021/20211110/Maxmind生成License2.png)
点击确定以后就会生成账号及License key
![License key生成](http://image2.ishareread.com/images/2021/20211110/MaxmindLicense生成页.png)

## 2、下载并配置geoipupdate
[https://github.com/maxmind/geoipupdate](https://github.com/maxmind/geoipupdate)
这里有详细的安安装及配置说明

发行版本下载地址 [https://github.com/maxmind/geoipupdate/releases](https://github.com/maxmind/geoipupdate/releases)
![在这里插入图片描述](http://image2.ishareread.com/images/2021/20211110/geoipupdate下载.png)

可以看到提供了各种平台的版本的下载链接，这里我们下载安装的是linux版本，点击下载“geoipupdate_4.8.0_linux_amd64.tar.gz”
在home目录下执行

```powershell
wget https://github.com/maxmind/geoipupdate/releases/download/v4.8.0/geoipupdate_4.8.0_linux_amd64.tar.gz
```

下载至home目录

tar -zxvf geoipupdate_4.8.0_linux_amd64.tar.gz 进行解压
cd geoipupdate_4.8.0_linux_amd64  目录执行ls -alh查看目录内容，发现有两个关键文件，一个是getipupdate命令执行文件，一个是GeoIP.conf配置文件
![geoipupdate目录](http://image2.ishareread.com/images/2021/20211110/geoipupdate目录.png)

将执行命令拷贝到命令文件夹

```bash
cp geoipupdate /usr/local/bin/
```

geoipupdate命令读配置文件默认为/usr/local/etc/GeoIP.conf将配置文件拷贝到/usr/local/etc/下
```bash
cp GeoIP.conf /usr/local/etc/
```

```bash
vi /usr/local/etc/GeoIP.conf
```
![修改GeoIP.conf](http://image2.ishareread.com/images/2021/20211110/geoipupdate配置文件.png)
如上图修改离线库文件目录及账号、LicenseKey，AccountID和LicenseKey就是开始在Maxmind网站上申请的。

## 3、运行geoipupdate命令并加入定时任务
执行geoipupdate命令，在目录下面产生了GeoLite2-City.mmdb、GeoLite2-Country.mmdb两个离线库文件。
![GeoLite2离线库文件](http://image2.ishareread.com/images/2021/20211110/GeoLite2库文件.png)
创建Linux定时任务，每周自动更新一下离线库文件
```powershell
crontab -e
0 0 * * 0 /usr/local/bin/geoipupdate
```

# 二、通过Python调用GeoIP获取IP信息
默认已经安装好了Flask环境，并激活了python虚拟环境。激活python虚拟环境安装Flask教程见[http://xiejava.ishareread.com/posts/7f405b25/](http://xiejava.ishareread.com/posts/7f405b25/)
## 1、安装geoip2

```bash
pip install geoip2
```

## 2、编写hello.py调用geoip2

```bash
vi hello.py
```

复制以下代码到hello.py

```python
from flask import Flask
import geoip2.database

app = Flask(__name__)
reader=geoip2.database.Reader('/home/geoipupdate_4.8.0_linux_amd64/GeoLite2-City.mmdb')
@app.route("/")
def hello():
    return "Hello World!"

@app.route("/getip/<ip>")
def getip(ip):
    ipinfo=reader.city(ip)
    ipinfo_json={'country':ipinfo.country.name,'city':ipinfo.city.name,'location':[ipinfo.location.longitude,ipinfo.location.latitude]}
    return ipinfo_json

if __name__ == "__main__":
    app.run(host='0.0.0.0',port=8080)
```

## 3、运行hello.py
```powershell
(flask_web) [root@localhost flask_web]# python hello.py
 * Serving Flask app 'hello' (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on all addresses.
   WARNING: This is a development server. Do not use it in a production deployment.
 * Running on http://192.168.1.18:8080/ (Press CTRL+C to quit)
```

注意：如果linux开启了防火墙请关闭防火墙，或放开192.168.1.18

## 4、验证
通过浏览器访问 http://192.168.1.18:8080/getip/128.101.101.101 
![验证IP信息](http://image2.ishareread.com/images/2021/20211110/IP信息返回验证.png)
可以看到返回IP的国家、城市、经纬度等信息。

**至此，本文介绍了如何注册并下载GeoIP离线数据包，并通过官方提供的geoipupdate进行定期更新数据。还介绍了如何通过Python调用GeoIP获取IP信息。**

----------

作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)



<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>



