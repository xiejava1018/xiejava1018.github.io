---
title: pyinstaller打包exe后不能运行报Failed to execute script XXXX问题分析与处理
id: 20201103001
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: 19a32f6f
date: 2020-11-03 17:16:00
---
最近用python的tkinter写了个小工具，发现用pyinstaller打包成exe后运行出错。报Failed to execute script XXXX

```bash
pyinstaller -F -w worksubmit.py
```

![报错](http://image2.ishareread.com/images/2020/20201101/报错.png)

为了搞清楚报错的原因，想看到程序具体执行的情况。可以通过不带-w的参数打包在控制台看程序执行情况。
`pyinstaller -F worksubmit.py` 可以通过不带-w的参数打包，这时打包的exe运行是带控制台的命令行

![运行情况](http://image2.ishareread.com/images/2020/20201101/运行.png)

可以清楚的看到

> ModuleNotFoundError:No module named 'xlrd'

这时就要解决打包时xlrd模块没有打进去的问题，找到xlrd模块的位置，并将该模块打到运行程序包里。
先找到程序依赖的xlrd模块的位置，在PyCharm中通过"File"->"Setting",在项目设置里查看Project interpreter，可以看到xlrd的目录位置。

![找包路径](http://image2.ishareread.com/images/2020/20201101/找包路径.png)

用pyinstall打包的时候通过加-p的参数将依赖的模块打进去。

```bash
pyinstaller -F -p J:\study\python\testsubmit\venv\Lib\site-packages worksubmit.py
```

![在这里插入图片描述](http://image2.ishareread.com/images/2020/20201101/打依赖模块包.png)
这样就可以顺利将依赖的模块打进去，再执行exe文件不再报错了。

**总结一下，碰到打包成exe后运行有问题，可以通过不带-w的参数打包，这时打包的exe运行是带控制台的命令行。基本上所有的运行问题都可以通过控制台的命令定位和排查。**

--------------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)
<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>