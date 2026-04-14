---
title: 企业级私有docker镜像仓库Harbor的搭建和使用
id: 20251204001
tags:
  - 运维
  - 架构
categories:
  - - 技术
    - 架构
abbrlink: 3ea90a75
date: 2025-12-04 11:22:22
---
# 一、什么是Harbor
简单来说，Harbor 是一个开源的企业级私有 Docker 镜像仓库服务。我们可以把它理解成一个 “私有的、安全的、功能强大的 Docker Hub”。
- 私有： 它部署在你自己的基础设施（如公司的数据中心或私有云）上，你完全掌控其中的所有镜像，不对外公开。
- 企业级： 这意味着它不仅仅是一个简单的存储服务器。它提供了企业所需的高级功能，如权限控制、安全扫描、镜像复制、图形化操作界面等。
- 镜像仓库： 它是专门用来存储、管理和分发 Docker 镜像的地方。你可以向 Harbor 中推送（上传）镜像，也可以从 Harbor 中拉取（下载）镜像。

目前国内的上网环境公共的Docker Hub不能直接访问了，对于公司和个人开发者来说有必要搭建私有的Docker镜像仓库，使用 Harbor 几乎是必须的。公司自己开发的应用程序镜像可能包含敏感代码和配置，绝不能放到公共仓库。Harbor 提供了一个安全的私有环境来存放这些资产。Harbor 可以配置为 Docker Hub、Google Container Registry（GCR）等公有仓库的代理缓存。当公司内第一个开发者拉取某个公有镜像（如 nginx:latest）时，Harbor 会从公网下载并缓存到本地。后续所有开发者再拉取这个镜像时，都会直接从内网的 Harbor 服务器高速获取，极大地节省了公网带宽，加快了拉取速度。

Harbor 将一个简单的镜像存储服务，升级为一个安全、高效、可靠的企业级镜像生命周期管理平台，是现代云原生应用开发和运维不可或缺的基础设施组件。

本文介绍如何通过Harbor搭建和使用自己的私有镜像仓库。
# 二、安装Harbor
准备一台linux的服务器，我这里是ubuntu 24.04  IP地址是192.168.0.49
Harbor的docker安装要求
On a Linux host: docker 20.10.10-ce+ and docker-compose 1.18.0+ .
## 1、安装docker
执行以下命令一键安装docker 

```bash
sudo curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
service docker start
```

配置国内镜像加速

```bash
vim /etc/docker/daemon.json
```

```bash
{
    "registry-mirrors": [
        "https://docker.1ms.run"
    ]
}
```

```bash
service docker restart
```

## 2、安装Harbor
下载harbor安装包，下载地址如下：
https://github.com/goharbor/harbor/releases/download/v2.14.1/harbor-offline-installer-v2.14.1.tgz
将下载的harbor安装包上传到安装harbor的目标服务器的目录下
如/home/app/harbor
解压上传的harbor安装包

```bash
tar -zxvf harbor-offline-installer-v2.14.1.tgz
```

备份配置文件

```bash
cp harbor.yml.tmpl harbor.yml
```

修改配置文件

```bash
vim harbor.yml
```
![修改harbor.yml配置文件](http://image2.ishareread.com/images/2025/20251204/1-修改harbor配置文件.png)

 执行安装脚本

```bash
./install.sh
```

安装完成后通过浏览器即可以访问harbor的管理界面
![harbor登录界面](http://image2.ishareread.com/images/2025/20251204/2-harbor登录界面.png)

默认的用户名和密码是`admin/Harbor12345`

# 三、使用Harbor
## 1、通过Harbor代理拉取镜像
先在仓库管理中建一个目标，目标是从docker的加速镜像站点去拉取镜像，我这里配置目标URL是`https://docker.1ms.run`，点击测试链接，提示“测试连接成功”表示目标URL站点没有问题，就可以点击确定。
![新建目标](http://image2.ishareread.com/images/2025/20251204/3-harbor新建目标.png)

这样我们就有了一个目标的加速仓库
![新建目标成功](http://image2.ishareread.com/images/2025/20251204/4-成功创建仓库管理目标.png)

在“项目”中点击“新建项目”，在镜像代理中**开启**“镜像代理”，选择我们开始在仓库管理中新建的目标仓库。
![开启镜像代理](http://image2.ishareread.com/images/2025/20251204/5-新建项目开启镜像代理.png)

在配置管理生成软件物料清单中勾选“在推送镜像时自动生成软件物料清单”，这样我们在拉取镜像时harbor会自动帮我们缓存镜像到本地的harbor仓库，下次拉取时速度就非常快了。
![勾选物料清单](http://image2.ishareread.com/images/2025/20251204/6-开启推送生成物料清单.png)

在另外一台需要拉取镜像的机器上配置docker的镜像仓库，将地址配置为Harbor的地址

```bash
vim /etc/docker/daemon.json
```

```bash
{
    "registry-mirrors": [
        "http://192.168.0.49"
    ],
    "insecure-registries": ["192.168.0.49"]
}
```

重启docker服务

```bash
service docker restart
```

拉取目标镜像，这里要带上harbor中的项目名称，`docker1ms/nginx:1.22`

```bash
docker pull docker1ms/nginx:1.22
```

![拉取镜像](http://image2.ishareread.com/images/2025/20251204/7-docker拉取镜像.png)

在harbor的管理界面可以看到，通过拉取镜像，harbor通过代理镜像源对镜像进行了拉取并缓存到harbor仓库，下次拉取速度更快了。
![harbor镜像缓存](http://image2.ishareread.com/images/2025/20251204/8-harbor拉取镜像成功.png)

## 2、上传镜像到Harbor仓库
另一个在工作中最常用的场景就是将自己的镜像上传到镜像仓库，以便其他人拉取使用。
下面就以在本地已经存在的docker1ms/busybox:latest为例，上传到自己的harbor仓库。
我的harbor仓库的地址是192.168.0.49,上传到默认的library项目中
为镜像打上带仓库地址和项目名的tag

```bash
docker tag docker1ms/busybox:latest 192.168.0.49/library/busybox:latest 
```

![为镜像打标签](http://image2.ishareread.com/images/2025/20251204/9-给镜像打标签.png)

客户端登录到harbor

```bash
docker login 192.168.0.49 
```

输入harbor的用户名和口令进行登录
![登录到harbor仓](http://image2.ishareread.com/images/2025/20251204/10-登录到harbor.png)

登录成功后，通过push命令上传镜像到本地harbor仓库

```bash
docker push 192.168.0.49/library/busybox:latest
```

![push镜像](http://image2.ishareread.com/images/2025/20251204/11-push镜像.png)

到harbor管理界面查看push到仓库中的镜像
![push到仓库中的镜像](http://image2.ishareread.com/images/2025/20251204/12-查看push的项目.png)

可以看到busybox镜像已经成功push到harbor的library项目中了。
![成功push](http://image2.ishareread.com/images/2025/20251204/13-查看push的项目的镜像仓库.png)

在Artifacts中可以看到push上来的镜像物料详情，可看到这个busybox镜像的大小是2.1MB
![物料清单](http://image2.ishareread.com/images/2025/20251204/14-查看push镜像的物料.png)

我们可以直接拉取harbor中我们刚push上去的镜像
可以看到如果是默认仓库library项目直接可以docker pull 镜像名就可以拉取

```bash
docker pull busybox:latest 
```

![拉取镜像](http://image2.ishareread.com/images/2025/20251204/15-直接拉取harbor库中的镜像.png)

至此，我们通过安装自己私有的镜像仓库Harbor，实现了docker镜像代理下载和上传自己的镜像到Harbor仓库，覆盖了我们平时工作中的大部分场景。

-------------
作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>
