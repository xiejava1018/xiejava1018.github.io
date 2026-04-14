---
title: CentOS7下python3+Flask+uWSGI+Nginx+Supervisor环境搭建
id: 20201110501
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: 7f405b25
date: 2021-11-05 23:38:53
---
在生产环境中通常用uwsgi作为Flask的web服务网关，通过nginx反向代理进行负载均衡，通过supervior进行服务进行的管理。这一套搭下来还是有一些坑要踩，本文通过一个简单的Flask web应用记录了CentOS7下python3+Flask+uWSGI+Nginx+Supervisor环境搭建的全过程，以及一些注意事项，以免遗忘。

# 一、Python3环境安装
CentOS7下Python3环境安装参考 [http://xiejava.ishareread.com/posts/57cef505/](http://xiejava.ishareread.com/posts/57cef505/)

查看python版本
```bash
[root@localhost ~]# python -V
Python 3.8.12
```

# 二、安装Flask
## 1、创建Python虚拟环境
在home目录下创建flask_web目录（目录根据具体实际环境创建，本教程是/home/flask_web）
通过venv创建虚拟环境
[root@localhost flask_web]# python -m venv /home/flask_web
创建成功后可以看到在目录下自动建了一些文件夹，包括python命令及依赖库等，激活以后是个独立的python虚拟运行环境。
![python虚拟运行环境](http://image2.ishareread.com/images/2021/20211105/虚拟环境文件.png)

在目录下运行source bin/activate 激活虚拟环境
```bash
[root@localhost flask_web]# source bin/activate
(flask_web) [root@localhost flask_web]# 
```

## 2、安装Flask
通过pip install flask安装flask
```bash
(flask_web) [root@localhost flask_web]# pip install flask
```

安装的时候有可能报ModuleNotFoundError: No module named '_ctypes'的错误，原因是缺少libffi-devel包，具体可参考 https://blog.csdn.net/qq_36416904/article/details/79316972
![安装Flask报错](http://image2.ishareread.com/images/2021/20211105/安装flask报错.png)

运行yum install libffi-devel -y 并且要重新编译执行安装python
解决包依赖的问题
(flask_web) [root@localhost flask_web]# yum install libffi-devel -y
进入到python源码包目录 执行使用make&make install 命令重新编译并安装python（这里比较坑）
然后再pip install flask 进行安装
安装完成后可以尝试运行flask run，提示没有Flask应用程序，说明flask已经安装成功并且可以运行了。

```bash
(flask_web) [root@localhost flask_web]# flask run
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
Usage: flask run [OPTIONS]
Try 'flask run --help' for help.

Error: Could not locate a Flask application. You did not provide the "FLASK_APP" environment variable, and a "wsgi.py" or "app.py" module was not found in the current directory.
```

3、建立测试应用
vi hello.py创建一个hello.py的文件，copy下面的内容到文件中:wq保存退出

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == "__main__":
    app.run()
```

通过python hello.py运行测试程序

```powershell
(flask_web) [root@localhost flask_web]# python hello.py
 * Serving Flask app 'hello' (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

新开一个shell窗口执行curl http://127.0.0.1:5000/ 可以看到有Hello World返回说明应用在flask框架下运行没有问题。
```powershell
[root@localhost ~]# curl http://127.0.0.1:5000/
Hello World!
```
# 三、安装及配置uwsgi
uWSGI是一个Web Server，并且独占uwsgi协议，但是同时支持WSGI协议、HTTP协议等，它的功能是把HTTP协议转化成语言支持的网络协议供python使用。有点类似于Java的web服务容器中间件tomcat
## 1、安装uwsgi
通过pip命令安装

```powershell
(flask_web) [root@localhost flask_web]# pip install uwsgi
```

如果顺利的话会显示Successfully installed uwsgi-2.0.20，表示安装成功了。

## 2、配置uwsgi
新建一个uwsgi.ini配置文件，并将配置信息复制到配置文件
vi uwsgi.ini 

```bash
[uwsgi]
#http=127.0.0.1:3366  #如果是http,通过proxy_pass http链接
socket=127.0.0.1:3366 #如果是socket，通过nginx配置uwsgi_pass socket链接
wsgi-file=/home/flask_web/hello.py
callable=app
touch-reload=/home/flask_web/
#最大请求数，最多请求5000次就重启进程，以防止内存泄漏
max-requests=5000
#请求超时时间，超过60秒关闭请求
harakiri=60
#进程的数量
processes=1
#线程数
threads = 2
#记录pid的文件
pidfile=/home/flask_web/uwsgi.pid
buffer-size = 32768
#日志最大50M
log-maxsize=50000000
#配置虚拟环境路径，如果是在虚拟环境下启动，这个一定要配，不配会有些包找不到，应用会报错。可以在uwsgi.log文件中看报错信息
virtualenv =/home/flask_web
#uwsgi日志文件，如果是通过supervisor托管，daemonize配置需要屏蔽
#daemonize=/home/flask_web/uwsgi.log
#项目更新后，自动加载
python-autoreload=1
#状态检测地址
stats = 127.0.0.1:9191
```

## 3、运行uwsgi

```bash
(flask_web) [root@localhost flask_web]# uwsgi --ini /home/flask_web/uwsgi.ini
```

启动以后通过访问curl http://127.0.0.1:3366 有Hello World!的返回信息表示uwsgi已经成功启动，并且应用程序正常。

```bash
[root@localhost flask_web]# curl http://127.0.0.1:3366
Hello World!
```

# 四、配置Nginx反向代理
ps -ef|grep nginx 找到nginx的配置文件
![nginx配置文件](http://image2.ishareread.com/images/2021/20211105/nginx配置文件.png)
如果uwsgi配置的是socket连接
[uwsgi]
socket=127.0.0.1:3366 #如果是socket，通过nginx配置uwsgi_pass socket链接
nginx的server配置如下：
```bash
 server {
      listen 808;
      server_name localhost;

      location / {
           include uwsgi_params;
           uwsgi_pass 127.0.0.1:3366;
       }
       access_log /home/flask_web/access.log;
       error_log /home/flask_web/error.log;
   }
```
如果uwsgi配置的是http
[uwsgi]
http=127.0.0.1:3366  #如果是http,通过proxy_pass http链接
nginx的server配置如下：
```bash
server {
      listen 808;
      server_name localhost;

      location / {
           proxy_pass http://127.0.0.1:3366;
       }
       access_log /home/flask_web/access.log;
       error_log /home/flask_web/error.log;
   }
```
重新加载nginx配置后，通过浏览器访问可以正常显示访问结果


# 五、通过Supervisor进行进程托管
生产环境中，可以通过supervisor来进行uwsgi和nginx进程的托管，界面化的方式管理uwsgi和nginx，包括进程的监控、启停等。
## 1、安装supervisor
通过pip安装
```bash
pip install supervisor
```
离线安装请参考：[http://xiejava.ishareread.com/posts/d670c9b8/](http://xiejava.ishareread.com/posts/d670c9b8/)

## 2、配置supervisor
找到supervisord的安装目录在/usr/local/bin下
```bash
[root@localhost bin]# which supervisord
/usr/local/bin/supervisord
```
cd到/usr/local/bin目录下
通过echo_supervisord_conf > supervisord.conf
```bash
[root@localhost bin]# echo_supervisord_conf > supervisord.conf
```
可以看到生成了一个supervisord.conf的配置文件。
将生成的supervisord.conf配置文件放到/etc/目录下
```bash
mv supervisord.conf /etc/
```
修改supervisord.conf的配置文件，主要是将子配置文件路径开启并指定配置文件路径，按照惯例将配置文件放到/etc目录下
```bash
[include]
files = /etc/supervisord.d/*.ini
```
![supervisord.conf配置文件](http://image2.ishareread.com/images/2021/20211105/supervisord.config.png)

我们在/etc目录下建个supervisord.d目录用来保存supervisor托管进程的配置文件
```bash
[root@localhost ~]# cd /etc/
[root@localhost etc]# mkdir supervisord.d
```
建立并配置子配置文件
```bash
[root@localhost etc]# cd supervisord.d/
[root@localhost supervisord.d]# vi uwsgi.ini
```
复制以下内容至uwsgi.ini文件中

```bash
[program:uwsgi]
command =uwsgi --ini /home/flask_web/uwsgi.ini
directory=/home/flask_web
startsecs=10
startretries=5
autostart=true
autorestart=true
stdout_logfile=/home/flask_web/uwsgi_sup_log.log
stdout_logfile_maxbytes=10MB
user=root
stopasgroup=true
killasgroup=true
```

## 3、启动supervisor
在启动supervisor拉起uwsgi前两个注意事项
1) uwsgi的配置文件中daemonize一定要屏蔽掉，否则守护进程一直会重启，导致端口每次都被占用，Supervisor托管不了。
![uwsgi.ini](http://image2.ishareread.com/images/2021/20211105/uwsgi.ini.png)
2) 在启动之前先将已经启动的uwsgi进程停掉，否则通过supervisor拉起uwsgi进程时端口冲突
![kill uwsgi进程](http://image2.ishareread.com/images/2021/20211105/kill-uwsgi.png)

启动supervisord进程

```bash
[root@localhost bin]# supervisord -c /etc/supervisord.conf
```

修改配置文件后重新加载可以通过 supervisorctl reload 命令重新加载
查看supervisor托管状态

```bash
[root@localhost supervisord.d]# supervisorctl status
uwsgi                            STARTING 
```

可以看到uwsgi被supervisor托管并已经启动。如果需要通过supervisor的web控制界面进行进程的管理。需要修改/etc/supervisord.conf的配置文件将访问的IP地址限制放开，设置用户名、口令

```bash
[inet_http_server]         ; inet (TCP) server disabled by default
port=*:9001        ; ip_address:port specifier, *:port for all iface
username=user              ; default is no username (open server)
password=user@123               ; default is no password (open server)
```

重新启动supervisor，重启时会报需要验证的错误

```bash
[root@localhost supervisord.d]# supervisorctl shutdown
Server requires authentication
error: <class 'xmlrpc.client.ProtocolError'>, <ProtocolError for 127.0.0.1/RPC2: 401 Unauthorized>: file: /usr/local/lib/python3.8/site-packages/supervisor/xmlrpc.py line: 542
```

可以直接kill -9杀掉supervisor的进程再启动，也可以通过supervisorctl 输入用户名、口令通过shutdown然后再重启。
启动命令：supervisord -c /etc/supervisord.conf

这时就可以通过supervisor的web控制界面进行进程的管理了。
![Supervisor](http://image2.ishareread.com/images/2021/20211105/supervisor-web.png)
**至此，CentOS7下python3+Flask+uWSGI+Nginx+Supervisor环境全部搭建好了。**

---------
作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)



<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>





