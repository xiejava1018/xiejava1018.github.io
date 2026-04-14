---
title: CentOS7下配置Supervisor自启动的两种方法
id: 20211111001
tags:
  - 运维
categories:
  - - 技术
    - 开发
abbrlink: 4ba25d97
date: 2021-11-11 14:18:19
---
很多网友留言问如何配置Supervisor 自启动，现将如何在CentOS7下配置Supervisor自启动的两种方法整理如下：

# 一、方法一
**直接将启动命令加入到/etc/rc.d/rc.local中（简单但不推荐）**

```powershell
vi /etc/rc.d/rc.local
```
在现有的内容后面加入supervisor的启动命令
supervisord -c /etc/supervisord.conf
![/etc/rc.d/rc.local](http://image2.ishareread.com/images/2021/20211111/rc.local.png)
注意：一定要执行 chmod +x /etc/rc.d/rc.local

> chmod +x /etc/rc.d/rc.local

给文件加入可执行权限
根据官方的提示，该方式是不被建议的，强烈建议创建自己的systemd services或udev规则来启动自已的应用，也就是方法二。

# 二、方法二
**通过创建systemd services来实现自启动 （推荐）**
进入到/usr/lib/systemd/system/目录

```powershell
[root@localhost ~]# cd /usr/lib/systemd/system/
```

找到supervisord及supervisorctl命令的路径

```powershell
[root@localhost system]# which supervisord
/usr/local/bin/supervisord
[root@localhost system]# which supervisorctl
/usr/local/bin/supervisorctl
```

## 创建文件supervisord.service

```powershell
vi supervisord.service
```

复制以下代码。注意：supervisord及supervisorctl命令的路径根据实际情况进行修改

```powershell
#supervisord.service

[Unit]
Description=Supervisor daemon

[Service]
Type=forking
ExecStart=/usr/local/bin/supervisord -c /etc/supervisord.conf
ExecStop=/usr/local/bin/supervisorctl shutdown
ExecReload=/usr/local/bin/supervisorctl reload
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```


## 启用服务

```powershell
[root@localhost system]# systemctl enable supervisord
Created symlink from /etc/systemd/system/multi-user.target.wants/supervisord.service to /usr/lib/systemd/system/supervisord.service
```

## 启动服务

```powershell
[root@localhost ~]# systemctl start supervisord
```

## 查看服务状态

```powershell
[root@localhost ~]# systemctl status supervisord
● supervisord.service - Supervisor daemon
   Loaded: loaded (/usr/lib/systemd/system/supervisord.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2021-11-11 11:11:36 CST; 12s ago
  Process: 3822 ExecStart=/usr/local/bin/supervisord -c /etc/supervisord.conf (code=exited, status=0/SUCCESS)
 Main PID: 3850 (supervisord)
   CGroup: /system.slice/supervisord.service
           ├─3850 /usr/local/bin/python3.8 /usr/local/bin/supervisord -c /etc/supervisord.conf
           ├─3916 uwsgi --ini /home/flask_web/uwsgi.ini
           ├─3918 uwsgi --ini /home/flask_web/uwsgi.ini
           └─3919 uwsgi --ini /home/flask_web/uwsgi.ini
```

## 验证一下是否为开机启动

```powershell
[root@localhost system]# systemctl is-enabled supervisord
enabled
```

reboot重启服务器后，可以发现supervisor随服务器启动后自动启动了。

**至此，本文介绍了CentOS7下配置Supervisor自启动的两种方法，推荐使用第二种方式。**

----------

作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注微信公众号,一起学习、成长！</center>


