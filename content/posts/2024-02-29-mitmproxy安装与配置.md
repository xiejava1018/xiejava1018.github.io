---
title: mitmproxy安装与配置
id: 20240229001
tags:
  - Python
  - 爬虫
  - 网络安全
categories:
  - - 技术
    - 开发
abbrlink: 699b1506
date: 2024-02-29 18:33:41
---
mitmproxy是一个免费的开源交互式的HTTPS代理工具。它类似于其他抓包工具如WireShark和Fiddler，支持抓取HTTP和HTTPS协议的数据包，并可以通过控制台形式进行操作。mitmproxy具有两个非常有用的组件：mitmdump和mitmweb。mitmdump是mitmproxy的命令行接口，可以直接抓取请求数据，而mitmweb是一个web程序，可以清楚地观察mitmproxy抓取的请求数据。

此外，mitmproxy的特点之一是它支持Python自定义脚本，这使得mitmproxy的使用更加灵活和强大。通过安装mitmproxy，用户可以实时查看、记录、修改数据，引发服务端或客户端的特定行为。mitmproxy是一个功能强大的抓包工具，具有广泛的应用场景，如网络调试、安全测试、数据分析等。

本文介绍mitmproxy的安装与配置，通过mitmproxy代理进行抓包。
## 一、mitmproxy的安装
首先需要安装好python，版本需要不低于3.6，且安装了附带了包管理工具pip
在命令行中输入`pip install mitmproxy`，等待安装完成。
安装完成后，系统将拥有mitmproxy、mitmdump、mitmweb三个命令，可以通过`mitmdump`检查一下mitmproxy是否安装成功了。
![查看mitmproxy版本](http://image2.ishareread.com/images/2024/20240229/1-查看mitmproxy版本.png)
## 二、运行mitmproxy
要启动 mitmproxy 用 mitmproxy、mitmdump、mitmweb 这三个命令中的任意一个即可，这三个命令功能一致，且都可以加载自定义脚本，唯一的区别是交互界面的不同。

启动mitmproxy后，客户端要配置指定通过mitmproxy代理来访问目标网站的服务资源。
![mitmproxy代理工作原理](http://image2.ishareread.com/images/2024/20240229/2-how-mitmproxy-works-explicit.png)

### 1、配置客户端代理
运行mitmproxy默认代理的端口是8080，所以要配置浏览器通过mitmproxy代理来访问目标网站。
#### 方式一，设置全局代理
设置全局代理，在window中找到网络和Internet，点击手动设置代理，将使用代理服务器的开关打开，设置代理IP地址为127.0.0.1，端口为8080
![设置全局代理](http://image2.ishareread.com/images/2024/20240229/3-windows设置网络代理.png)

#### 方式二，设置浏览器代理
通过浏览器插件设置浏览器代理，如chrome浏览器可以通过SwitchyOmega插件设置代理
![通过SwitchyOmega插件设置代理](http://image2.ishareread.com/images/2024/20240229/4-SwitchyOmega插件设置代理.png)


通过mitmproxy命令启动mitmproxy后通过设置代理后的浏览器访问http://xiejava.ishareread.com，可以看到通过mitmproxy代理后的访问流量日志。
![http抓包日志情况](http://image2.ishareread.com/images/2024/20240229/5-访问http的网站.png)

### 2、客户端安装mitmproxy提供的CA证书
对于访问https加密的网站需要证书才能解密，所以客户端需要安装mitmproxy提供的CA证书。
通过mitmporxy代理浏览器访问http://mitm.it/将显示mitmproxy证书下载和安装指导页面
![mitmproxy证书下载和安装指导页面](http://image2.ishareread.com/images/2024/20240229/6-mitmproxy证书下载安装页面.png)

可以看到mitmproxy提供了各种操作系统的CA证书，点击"Show Instructions",将显示证书的安装指导，可以根据指导一步步安装成功。
![证书的安装指导](http://image2.ishareread.com/images/2024/20240229/7-mitmproxy证书安装步骤介绍.png)

对于Windows系统：
#### 手工安装步骤：
1. 双击mitmproxy提供的CA证书文件（通常是mitmproxy-ca.p12）。
2. 在出现的导入证书引导页中，直接点击“下一步”按钮。
3. 接下来会出现密码设置提示，这里不需要设置密码，直接点击“下一步”按钮。
4. 选择证书的存储区域。通常选择“将所有的证书都放入下列存储”，然后点击“浏览”按钮，选择证书存储位置为“受信任的根证书颁发机构”，接着点击“确定”按钮。
5. 点击“下一步”按钮完成证书的导入。
6. 如果有安全警告弹出，直接点击“是”按钮即可。

#### 自动安装步骤：
在window中用管理员权限运行PowerShell，在命令行控制台，进去到证书的目录，一般是在c:\Users\yourname\.mitmproxy目录下。执行 `certutil.exe -addstore root mitmproxy-ca-cert.cer` 如下图所示。
![自动安装证书步骤](http://image2.ishareread.com/images/2024/20240229/8-自动安装证书操作.png)

安装好证书后，通过mitmproxy代理，我们来访问https协议的网站https://www.taobao.com。可以在后台看到mitmproxy抓取的https流量日志
![mitmproxy抓取的https流量日志](http://image2.ishareread.com/images/2024/20240229/9-抓取https网站流量.png)


至此，我们成功安装了mitmproxy,并配置了相应的CA证书，通过mitmporxy代理能够获取访问http和https网站的流量数据。后续我们将通过一个实例来进行mitmproxy抓包爬取京东APP的金榜排行的数据信息。

------------------
作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>