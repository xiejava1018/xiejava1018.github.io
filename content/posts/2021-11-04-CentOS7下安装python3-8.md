---
title: CentOS7下安装python3.8
id: 20201110401
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: 57cef505
date: 2021-11-04 17:21:57
---

环境的搭建是进行开发的第一步，python因为存在python2和python3两个版本，让在建立python环境时造成不便，并且由于在Linux环境下不像Window环境安装那么友好，存在一些小坑。本教程记录了CentOS7下安装python3.8的过程和注意事项。
# 一、查看系统版本
```bash
[root@localhost ~]# cat /etc/centos-release
CentOS Linux release 7.2.1511 (Core)
```
```bash
[root@localhost ~]# uname -a
Linux localhost.localdomain 3.10.0-327.el7.x86_64 #1 SMP Thu Nov 19 22:10:57 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
```
查看python版本
```bash
[root@localhost ~]# python -V
Python 2.7.5
```

系统默认安装了Python 2.7.5

# 二、安装依赖

```bash
yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gcc make
```

如果有提示一路选择Y就可以

# 三、下载python源码包
python官网https://www.python.org/ 目前python最新版本是python3.10
![下载Python源码包](http://image2.ishareread.com/images/2021/20211104/下载python.png)

下载稳定版本3.8版

```bash
wget https://www.python.org/ftp/python/3.8.12/Python-3.8.12.tgz
```

# 四、解压安装python源码包
## 1、解压
```bash
tar -zxvf Python-3.8.12.tgz
```
## 2、安装
进入解压后的目录进行编译和安装
```bash
cd Python-3.8.12/
[root@localhost Python-3.8.12]#
```
```bash
[root@localhost Python-3.8.12]# ./configure
```
```bash
[root@localhost Python-3.8.12]# make&&make install
```
执行完后显示安装成功
![pyhont安装成功](http://image2.ishareread.com/images/2021/20211104/python安装成功.png)
## 3、建立命令软链接
虽然python3.8.12安装成功了，但默认输入python还是显示是2.7版本的。如果要用python3.8.12需要输入python3即可，有时候不太方便。可以通过修改软链接的方式将默认的python指向python3.8.12。
先看一下默认的python及新安装的python3都安装在哪里
```bash
[root@localhost Python-3.8.12]# which python
/usr/bin/python
```
```bash
[root@localhost Python-3.8.12]# which python3
/usr/local/bin/python3
```
可以看到默认的python路径为/usr/bin/python，python3的路径为/usr/local/bin/python3
将python3的软链接加到python上

```bash
[root@localhost Python-3.8.12]# mv /usr/bin/python /usr/bin/python.bak
[root@localhost Python-3.8.12]# ln -s /usr/local/bin/python3 /usr/bin/python
```
通过python -V命令查看python版号，这时python的版本已经是3.8.12了。
```bash
[root@localhost Python-3.8.12]# python -V
Python 3.8.12
```
pip命令也可以修改，python3.8.12默认的pip是pip3，CentOS7的python2.7默认没有安装pip.
输入pip命令的时候提示命令没有找到
```bash
[root@localhost Python-3.8.12]# pip
bash: pip: command not found...
```
这时也可以通过建立软链接的方式将pip命令链接到pip3上。首先看pip3命令在哪?
```bash
[root@localhost Python-3.8.12]# which pip3
/usr/local/bin/pip3
```
然后建立pip到pip3的软链接
```bash
[root@localhost Python-3.8.12]# ln -s /usr/local/bin/pip3 /usr/bin/pip
```
```bash
[root@localhost Python-3.8.12]# pip -V
pip 21.1.1 from /usr/local/lib/python3.8/site-packages/pip (python 3.8)
```
# 五、配置yum
安装python3改完软链接以后发现yum命令报错了，yum是依赖python2.7的，你把python改成了3.8了，所以报错了。

```bash
[root@localhost Python-3.8.12]# yum
  File "/usr/bin/yum", line 30
    except KeyboardInterrupt, e:
                            ^
SyntaxError: invalid syntax
```

可以修改yum里对python2的依赖即可。虽然安装了python3但是系统里python2依旧还在系统里，可以通过python2来指定用python2.7的命令，首先来看下python2的命令在哪里
```bash
[root@localhost ~]# which python2
/usr/bin/python2
```
可以cd到/usr/bin目录下 通过ls -alh|grep python查看python命令的详细情况。
```bash
[root@localhost bin]# ls -alh|grep python
```
![python命令软链接](http://image2.ishareread.com/images/2021/20211104/python软链接.png)
可以看到python软连接是执行的python3命令，python2是执行的python2.7的命令

```bash
vi /usr/libexec/urlgrabber-ext-down 
```
修改对python的依赖，修改成python2或python2.7都可以。
![修改依赖](http://image2.ishareread.com/images/2021/20211104/修改yun依赖1.png)

```bash
vi /usr/bin/yum
```
![修改依赖](http://image2.ishareread.com/images/2021/20211104/修改yum依赖2.png)

修改完这两个文件后，再敲yum命令就不会报错了。

**至此CentOS7环境下python3.8.12已经成功安装！**

作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)


<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>
