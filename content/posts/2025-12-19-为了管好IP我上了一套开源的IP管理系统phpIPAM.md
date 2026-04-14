---
title: 为了管好IP我上了一套开源的IP管理系统phpIPAM
id: 20251219001
tags:
  - 网络安全
  - 运维
categories:
  - - 技术
    - 运维
abbrlink: db0ebabe
date: 2025-12-19 14:33:19
---

通常，网络或系统管理员会使用一个电子表格来记录IP地址的分配信息，然而，随着网络中的设备越来越多依赖于电子表格并不方便，十分容易出错，想管理好这些加入网络中的设备，就得有个更方便的工具。就我家里的网络而言，接入的设备就已经接近40个了，亟需找这么一个工具来记录和管理这些IP，目前找到的比较好用又轻量化的IP管理工具就是phpIPAM。

# 一、什么是phpIPAM
phpIPAM（PHP IP Address Manager）是一个开源的网络 IP 地址管理工具，其目标是提供轻松，现代和有用的IP地址管理。它是基于php的应用程序，带有MySQL数据库后端，使用jQuery库，ajax和HTML5 / CSS3功能。主要用于企业级 IP 地址空间的规划、管理和跟踪。

**为什么要用phpIPAM**

|问题场景| 无 phpIPAM| 有 phpIPAM|
|---|---|---|
|IP分配冲突|人工记录 Excel，容易重复|系统自动管理，避免冲突|
|查找可用 IP|手动测试多个 IP|一键查找空闲 IP|
|网络规划|凭经验划分，不精确|可视化规划，最优利用|
|故障排查|不知道 IP 使用者|快速定位设备负责人|

**phpIPAM核心功能如下：**
![\[图片\]](http://image2.ishareread.com/images/2025/20251219/1-phpIPAM功能.png)

官网：
[https://www.phpipam.net/](https://www.phpipam.net/)
[https://hub.docker.com/r/phpipam/phpipam-www/](https://hub.docker.com/r/phpipam/phpipam-www/)

# 二、安装phpIPAM
安装phpIPAM可以用docker方式快速安装，根据官网的docker-compose.yml稍微做了优化，主要是加了mysql的健康检查，因为在安装的过程中数据库没有就绪或容器启动顺序有问题会导致安装失败。
docker-compose.yml文件内容如下：

```bash
version: '3.8'
services:
  phpipam-web:
    image: phpipam/phpipam-www:latest
    ports:
      - "8488:80"
    environment:
      - TZ=Asia/Shanghai
      - IPAM_DATABASE_HOST=phpipam-db
      - IPAM_DATABASE_PASS=12345678
    depends_on:
      - phpipam-db
    restart: unless-stopped

  phpipam-cron:
    image: phpipam/phpipam-cron:latest
    environment:
      - TZ=Asia/Shanghai
      - IPAM_DATABASE_HOST=phpipam-db
      - IPAM_DATABASE_PASS=12345678
      - SCAN_INTERVAL=1h
    depends_on:
      phpipam-db:
        condition: service_healthy
    restart: unless-stopped

  phpipam-db:
    image: mariadb:10.5
    environment:
      - MYSQL_ROOT_PASSWORD=12345678
      - MYSQL_DATABASE=phpipam
      - MYSQL_USER=phpipam
      - MYSQL_PASSWORD=12345678
    volumes:
      - db_data:/var/lib/mysql
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  db_data:
```

将docker-compose.yml文件考到安装目录如/home/app/phpipam
执行`docker compose up -d` 就可以顺利安装完成
安装完成后通过 `docker compose ps` 查看 phpipam的几个容器是否都正常启动了。
![docker-compose](http://image2.ishareread.com/images/2025/20251219/2-docker-compose-ps.png)

可以通过`docker compose logs`查看日志
# 三、初始化phpIPAM
因为在docker-compose.yml文件中映射的宿主机的端口是8488
在浏览器中输入 http://服务器IP：8488/  
1.选择新的phpipam安装
![新的phpipam安装](http://image2.ishareread.com/images/2025/20251219/3-新的phpipam安装.png)

2.安装pfpipam数据库
![安装pfpipam数据库](http://image2.ishareread.com/images/2025/20251219/4-自动安装数据库.png)

3.设置数据库
![设置数据库](http://image2.ishareread.com/images/2025/20251219/5-root数据库账号密码.png)

4.提示数据库安装成功
![数据库安装成功](http://image2.ishareread.com/images/2025/20251219/6-数据库安装成功.png)

5.设置管理员密码
![设置管理员密码](http://image2.ishareread.com/images/2025/20251219/7-设置Admin密码.png)

6.用前面设置的管理员密码进行登录
![用户名密码登录](http://image2.ishareread.com/images/2025/20251219/8-用户名密码登录.png)

7.可以在用户的账户详情中选择语言为“Chinese(zh_CN.UTF-8)”切换成中文
![设置成中文](http://image2.ishareread.com/images/2025/20251219/9-切换成中文.png)

# 四、使用phpIPAM
典型的IP管理流程如下：
![IP管理流程](http://image2.ishareread.com/images/2025/20251219/10-phpIPAM管理流程.png)

1.添加子网![添加子网](http://image2.ishareread.com/images/2025/20251219/11-添加子网.png)

2.添加子网信息，注意可以将Check hosts status、Discover new hosts、Resolve DNS name都打开。
![添加子网信息](http://image2.ishareread.com/images/2025/20251219/12-编辑子网.png)

3.在子网管理界面点击“Scan subnet for new hosts” 扫描子网的主机
![扫描子网](http://image2.ishareread.com/images/2025/20251219/13-点击扫描子网的主机.png)

4.点击“Scan subnet”
phpIPAM就会通过Ping scan的方式探测子网内存活的主机
![扫描子网](http://image2.ishareread.com/images/2025/20251219/14-子网扫描.png)

在子网的管理界面就可以看到子网内存活的IP和空闲的IP
![子网管理界面](http://image2.ishareread.com/images/2025/20251219/15-phpipam管理界面.png)

IP列表
![IP列表](http://image2.ishareread.com/images/2025/20251219/16-IP列表.png)

子网的可视化界面，可以很直观的看到子网内在用的IP和空闲的IP
![子网可视化界面](http://image2.ishareread.com/images/2025/20251219/17-子网可视化显示.png)


总的来说，phpIPAM安装简单使用方便，是个不错的IP管理工具。phpIPAM 本质上是一个 "网络的 CMDB"，让无形的 IP 地址变得可视、可控、可管理。对于任何有一定规模网络环境，它都是提升管理效率和规范性的重要工具。

-----------------------------
作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)
<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>