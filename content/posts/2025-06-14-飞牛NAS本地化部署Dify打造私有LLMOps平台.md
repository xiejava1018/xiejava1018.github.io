---
title: 飞牛NAS本地化部署dify打造私有LLMOps平台
id: 20250614001
tags:
  - AI机器学习
  - NAS
categories:
  - 技术
abbrlink: a0bb8a4f
date: 2025-06-14 11:25:43
---

作为一名程序员，在AI蓬勃发展的时代，一定要拥抱AI。**Dify** 是一款开源的大语言模型(LLM) 应用开发平台。它融合了后端即服务（Backend as Service）和 LLMOps 的理念，使开发者可以快速搭建生产级的生成式 AI 应用。即使你是非技术人员，也能参与到 AI 应用的定义和数据运营过程中。

由于 Dify 内置了构建 LLM 应用所需的关键技术栈，包括对数百个模型的支持、直观的 Prompt 编排界面、高质量的 RAG 引擎、稳健的 Agent 框架、灵活的流程编排，并同时提供了一套易用的界面和 API。这为开发者节省了许多重复造轮子的时间，使其可以专注在创新和业务需求上。

NAS作为私有化的个人数据中心，基本保持不停机的状态并且功耗小，比较适合作为私有化的开发平台载体。本文介绍如何在家庭飞牛NAS上本地化部署dify打造属于自己的私有化LLMOps平台，可以随时随地的通过Dify来构建自己的AI应用。

## 一、准备工作
### 1、环境介绍
我的NAS设备为零刻ME mini全闪小主机，CPU intel N200,内存12G，装了fnOS作为NAS用。这个配置如果仅仅只是存存相片、文件等还是有点过剩了，所以为了充分发挥迷你主机的效用决定把它打造为私有的LLMOps平台。

![NAS的硬件配置](http://image2.ishareread.com/images/2025/20250614/1-nas配置.png)

### 2、打开飞牛OS的SSH登录
登录到飞牛OS，在“系统设置”中找到“SSH”，打开SSH功能

![打开SSH](http://image2.ishareread.com/images/2025/20250614/2-打开SSH.png)

### 3、通过SSH客户端连上NAS主机
打开SSH后就可以通过SSH的客户端工具连到fnOS

![SSH连接主机](http://image2.ishareread.com/images/2025/20250614/3-SSH连接主机.png)

## 二、在飞牛NAS上安装Dify
### 1、找到Dify的安装源
在github或gitee上找到dify的项目地址
https://github.com/langgenius/dify/
https://gitee.com/dify_ai/dify
在国内访问github比较慢，可以访问gitee上的difyi项目地址。
这里有安装Dify的硬件要求和安装步骤。

![gitee的dify项目](http://image2.ishareread.com/images/2025/20250614/4-gitee的dify项目.png)

### 2、在飞牛NAS上创建存储目录
在飞牛NAS上创建一个用户clone和安装dify的目录

![NAS上创建Dify目录](http://image2.ishareread.com/images/2025/20250614/5-nas创建存储目录.png)
右键点击dify文件夹，点击“详细信息”。

![查看目录详情](http://image2.ishareread.com/images/2025/20250614/6-查看目录的存储详情.png)

在详细信息中找到“复制原始路径”，这里复制的原始路径就是该文件夹在fnOS操作系统的实际路径。

![找到实际路径](http://image2.ishareread.com/images/2025/20250614/7-复制原始路径.png)

### 3、开始安装Dify
安装Dify需要docker环境，fnOS本身就已经安装好了docker环境，所以可以直接clone后进行安装
通过`sudo -i` 切换到 root 用户，进入到开始复制的原始路径。

![切换root用户](http://image2.ishareread.com/images/2025/20250614/8-切换root用户进入dify目录.png)

通过`git clone https://gitee.com/dify_ai/dify` 命令clone项目

![clone项目](http://image2.ishareread.com/images/2025/20250614/9-clonedify.png)


进入到dify下的docker目录，将.env.example 复制成.env文件

```bash
cd dify/docker
cp .env.example .env
```

.env文件中配置的是dify的一些环境变量，dify默认的web端口是80，为了避免80端口冲突，我们把dify的默认端口改成8080，修改.env的配置文件，将docker输出的Nginx端口改成8080，Nginx的SSL端口改成8443。

```bash
EXPOSE_NGINX_PORT=8080
EXPOSE_NGINX_SSL_PORT=8080
```

![修改端口](http://image2.ishareread.com/images/2025/20250614/10-修改端口.png)

然后执行 `docker compose up -d` 就可以完成dify的安装

![安装dify](http://image2.ishareread.com/images/2025/20250614/11-安装dify.png)

安装完成后通过NAS的内网IP就可以访问到dify的初始化界面
http://192.168.0.18:8080/install
通过简单的配置，配置用户名、口令就可以正式使用dify了。

![内网访问dify](http://image2.ishareread.com/images/2025/20250614/12-内网访问dify.png)

## 三、设置外网访问
为了随时随地的访问和使用dify我们要对dify进行端口映射，使用nas的ddns来进行外网访问。
### 1、外网端口映射
在路由器上添加一条映射规则，将内网的8080端口映射到外网的某个端口上。

![外网端口映射](http://image2.ishareread.com/images/2025/20250614/12-外网端口映射.png)

### 2、在飞牛nas上进行ddns配置
如有域名就可以配置阿里云或Cloudflare的ddns服务

![DDNS配置](http://image2.ishareread.com/images/2025/20250614/13-DDNS配置.png)

这样就可以通过域名+端口直接访问NAS上的dify应用，随时随地构建和调试自己的AI应用了。

![外网访问](http://image2.ishareread.com/images/2025/20250614/14-外网访问dify.png)


------------------------

作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>
