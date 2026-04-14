---
title: PyCharm在用Django开发时debug模式启动失败显示can't find '__main__' module的解决方法
id: 20220606001
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: aebdf141
date: 2022-06-06 08:56:51
---
初次用Django开发web应用，在试图用Pycharm进行debug的时候，出现了一个奇怪的问题。以正常模式启动或者在terminal启动都没有问题。但是以debug模式启动时，显示`can't find '__main__' module”`报错。在网上找了很久都没有看到解决方法，最后在某乎上看到一篇文章，在启动时加上`--noreload`参数，既可以debug模式启动。

**报错信息：**
![报错信息](http://image2.ishareread.com/images/2022/20220605/debug报错.png)
**解决方法：**
在启动时加上 `--noreload` 参数可以正常启动调试
![加入不重新加载参数](http://image2.ishareread.com/images/2022/20220605/--noreload.png)

debug启动正常也可以调试了。
![debug正常启动](http://image2.ishareread.com/images/2022/20220605/debug启动正常.png)

踩过的坑记录一下，希望能帮到碰到同样问题的人。

感谢大佬的文章 https://zhuanlan.zhihu.com/p/443763989 

-----
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注微信公众号,一起学习、成长！</center>