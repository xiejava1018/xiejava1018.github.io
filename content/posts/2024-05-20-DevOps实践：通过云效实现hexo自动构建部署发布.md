---
title: DevOps实践：通过云效实现hexo自动构建部署发布
id: 20240520001
tags:
  - 项目管理
categories:
  - - 管理
    - 项目管理
  - - 技术
    - 开发
abbrlink: f872a2f0
date: 2024-05-20 12:41:16
---

DevOps（Development和Operations的组合词）是一组过程、方法与系统的统称，用于促进开发（应用程序/软件工程）、技术运营和质量保障（QA）部门之间的沟通、协作与整合。这是一种重视“软件开发人员（Dev）”和“IT运维技术人员（Ops）”之间沟通合作的文化、运动或惯例，是一个软件开发方法论。
![DevOps](http://image2.ishareread.com/images/2024/20240519/1-DevOps.png)

DevOps的目标是通过自动化“软件交付”和“架构变更”的流程，使得构建、测试、发布软件能够更加地快捷、频繁和可靠。这种方法的出现是因为软件行业日益清晰地认识到，为了按时交付软件产品和服务，开发和运维工作必须紧密合作。

《[研发管理之认识DevOps](http://xiejava.ishareread.com/posts/5efe604f/)》介绍了DevOps的概念和价值，本文我们来做一个小小的实践，通过阿里的云效DevOps平台来实现我的xiejava.ishareread.com的hexo博客网站的自动化部署实现持续集成和持续部署（CI/CD）。

## 一、没有用DevOps之前
我的xiejava.ishareread.com个人博客是通过hexo搭建的。因为是个人博客用hexo搭建比较简单而且还可以通过github的pages服务不用购买自己的服务器都可让大家访问。hexo博客的搭建教程见《[通过Git Pages+Hexo搭建自己的博客](http://xiejava.ishareread.com/posts/79ebd763/)》毕竟github在国内访问不是很稳定，所以我是购买了一台阿里的云主机ECS来部署我的hexo应用。

在没有用DevOps之前，我是通过自己本地用VS Code来编辑hexo的markdown博文，让后在本地通过执行hexo g命令生成构建hexo网站，然后登录到自己的阿里云主机上将文件传到服务器目录，再在服务器上构建并部署hexo应用。hexo很简单，这样构建发布也很容易，这几年也是这样做的。唯一不方便的就是有时候写的博文要修改，改一次就要重新上传发布到服务器上。毕竟咱也是搞IT的，感觉这样还是很不优雅。能不能提交markdown博文的.md文件候就自动给我发布了，省得每次还要登服务器，上传文件，编译，重启服务搞一系列的操作。

## 二、用了DevOps之后
为了体现程序员的优雅，我的个人博客xiejava.ishareread.com，小小的hexo用上了DevOps的牛刀。
![hexo的DevOps](http://image2.ishareread.com/images/2024/20240519/2-hexo网站devops.png)

用了DevOps后，我只需要在本地用VS Code来编辑hexo的markdown博文，提交到代码仓库。后面的构建，打包，部署我都不用管了，直接通过DevOps平台自动做了。

## 三、如何实践DevOps实现hexo网站的自动发布
DevOps平台工具有很多，最常见的就是大名顶顶的Jenkins，本来想搭一套Jenkins的我的云服务器本来就只有1核1G的内存，更本就扛不住。后来在阿里的网站上看到阿里推出了自己的DevOps平台云效，赶紧来试试。
云效DevOps平台功能很多，包括有项目协作、代码管理、流水线、测试管理、制品仓库、知识库。
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240519/3-云效平台.png)

对于我这个小小的hexo个人博客的应用，其他都用不上，我就用了代码管理和流水线。
### 1、代码管理
将我的hexo博客代码托管到云效的代码管理Codeup代码库。
![代码管理](http://image2.ishareread.com/images/2024/20240519/4-云效Codeup代码库.png)

可以支持从github、gitee、coding等代码仓库导入代码库。
![云效从其他代码库导入代码](http://image2.ishareread.com/images/2024/20240519/5-云效从其他代码库导入代码.png)


### 2、构建流水线
新建一条流水线用于拉代码、构建、部署。
![构建流水线](http://image2.ishareread.com/images/2024/20240519/6-云效流水线.png)

#### 1）设置流水线源
这里流水线源就是设置代码源流水线将从代码管理的Codeup代码库获取代码。其实也可以不用托管到云效的Codeup代码库，流水线代码源支持github、gitee、coding等代码仓库。
![设置流水线源](http://image2.ishareread.com/images/2024/20240519/7-云效从其他仓库获取代码源.png)

#### 2）添加并设置构建环节
![云效添加构建环节](http://image2.ishareread.com/images/2024/20240519/8-云效添加构建环节.png)

因为hexo是通过Node.js构建的，所以这里添加构建任务Node.js构建。
![Nodejs构建写入自己的构建命令](http://image2.ishareread.com/images/2024/20240519/9-Nodejs构建写入自己的构建命令.png)

在node.js构建节点，可以选择Node的版版本，将自己hexo应用的构建命令写入到构建命令。
我这里只用到了三条命令，安装hexo，从代码库中拉取hexo的主题，通过hexo g生成hexo的博客应用。

```powershell
npm install -g hexo-cli --unsafe-perm
git clone https://gitee.com/xiejava/hexo-theme-next.git themes/hexo-theme-next
hexo g
```

这一步的任务输出就会生成hexo的博客应用的制品，给你打好包。下一步的动作就是将打好的制品包上传至自己的服务器进行部署。
#### 3）添加并设置部署环节
云效支持很多种部署，因为我是要部署到自己的云主机，所以选择“主机部署”
![添加主机部署环节](http://image2.ishareread.com/images/2024/20240519/10-添加主机部署环节.png)

要部署到自己的云主机，就要让云效知道你的主机在哪里，可以点击新建主机组，将自己的主机添加进来。
![新建主机组](http://image2.ishareread.com/images/2024/20240519/11-新建主机组.png)

将需要部署的主机添加到主机组
![将主机添加到主机组](http://image2.ishareread.com/images/2024/20240519/12-将主机添加到主机组.png)

然后添加在主机上执行的部署脚本
![添加在主机上执行的部署脚本](http://image2.ishareread.com/images/2024/20240519/13-添加在主机上执行的部署脚本.png)

我在主机上hexo 服务进程用supervisor进行了托管，在这里部署脚本只有三条命令。

```powershell
supervisorctl stop hexo
tar zxvf /home/admin/app/package.tgz -C /home/myhexo/myblog
supervisorctl start hexo
```

停止hexo的服务，将制品包解压到hexo的目录，然后再启动hexo服务就可以了。
云效还可以定义任务插件，比如在部署成功后发个邮件通知等。
![云效邮件通知](http://image2.ishareread.com/images/2024/20240519/14-添加邮件通知.png)

到这里流水线就构建好了。

### 3、测试流水线
设置触发条件，开启Webhook触发，实现提交代码到到云效的Codeup代码库就可以触发流水线，开启自动拉取代码，自动构建、自动打包上传至主机、自动部署的流水线作业。也可以手动运行触发流水线。
![设置触发条件](http://image2.ishareread.com/images/2024/20240519/15-设置触发条件.png)

流水线执行后，可以在运行历史中看到每次流水线执行的情况。
![查看每次流水线执行情况](http://image2.ishareread.com/images/2024/20240519/16-查看每次流水线执行的情况.png)

也可以收到云效自动发过来的每次部署情况的邮件。
![云效邮件通知](http://image2.ishareread.com/images/2024/20240519/17-云效流水线执行邮件通知.png)

### 4、其他
云效针对流水线还提供了统计报表，可以看到流水线运行的概况的统计数据。
![流水线运行的概况的统计数据](http://image2.ishareread.com/images/2024/20240519/18-流水线统计概况.png)

## 四、总结
本文通过一个hexo个人博客进行了DevOps的实践，当然因为项目太小不能实践到DevOps的全部，但也可以窥豹一斑。通过DevOps拉通开发和运维，通过应用DevOps平台能实现自动化“软件交付”的流程，使得构建、测试、发布软件能够更加地快捷、频繁和可靠，提交研发交付效率。

-------------------------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>