---
title: 一个人活成一个团队：python的django项目devops实战
id: 20240811001
tags:
  - Python
  - Django
  - 运维
categories:
  - 技术
abbrlink: fd8c1dd4
date: 2024-08-11 16:54:24
---

对于开发团队来说提高软件交付的速度和质量是一个永恒的话题，对于个人开发者来说同样如此。作为一个码农，一定会有几个自己私有的小项目，从需求管理到开发到测试到部署运维都得要自己来，将自己一个人活成一个团队。

DevOps（Development和Operations的组合），旨在通过自动化、协作和共享责任来提高软件开发和运维的效率、质量和安全性。作为一个人的团队，也可通过devops实践来提高对自己项目的效率和质量，使产品持续开发、持续集成、持续测试、持续部署、持续监控，非常频繁地发布新版本。本文就以一个实际的python的django项目来运用阿里的云效devops平台来进行实战。
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/1-devops.png)
DevOps平台工具有很多，最常见的就是大名顶顶的Jenkins，作为个人开发者要准备相应的硬件资源，还要要自己维护一套Jenkins有点麻烦。这里直接就选择成熟的阿里云效devops https://devops.aliyun.com/ ，这套平台基础版是免费的，对于个人开发者来说已经够用了。
## 一、需求规划
个人项目虽小，但是也得要有相应的规划，至少得有个需求清单来进行需求的规划和跟踪，哪些需求已经完成了，哪些还需要进行开发做到自己心中有数。
可以在云效的项目协作中创建一个项目进行管理。
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/2-云效工作台项目协作.png)

在这里我创建了一个xiejava的博客项目
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/3-云效项目概览.png)

在这里我们就可以将自己规划的需求录入进来做好自己的需求跟踪清单
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/4-云效项目需求列表.png)


可以规划自己的版本，将需求跟踪清单里的需求纳入到版本迭代计划。
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/5-云效项目需求与迭代关联.png)

在迭代计划中可以看到这个迭代要完成的需求清单。
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/6-云效项目迭代计划.png)


## 二、代码管理
即使是最简单的项目，建议还是通过代码仓库进行代码的版本管理，我的代码是放到码云https://gitee.com/xiejava/ishareblog  进行托管的，也可以托管到云效自己的代码管理仓库。
有了代码仓库，可以通过在云效构建流水线来进行自动构建、自动测试、自动部署了。
## 三、创建流水线
在云效中创建ishareblog的自动发布流水线，整个流水线包括获取代码、测试、构建、部署。
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/7-云效流水线执行情况.png)

### 1、配置流水线源
流水线源可以配置云效自己的代码库，也可以配置其他的代码库，如我里是配置的码云代码库。
可以开启代码源触发，开启后一旦代码库有提交操作，就会自动触发流水线工作。
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/8-云效从码云中获取代码.png)

> 需要说明的是，如果是外部的代码仓库，需要在外部的代码仓库中添加Webhook触发设置

如我的是码云的仓库，就要在码云的仓库中添加Webhook的配置
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/9-云效webhooks管理.png)

## 四、自动测试
在测试环节，配置了python代码扫描和Python单元测试。
python代码扫描用的是云效默认的配置
比较麻烦的是Python单元测试，Python单元测试需要在Python项目中写测试用例，还要配置测试命令。
在Python项目中写测试用例见《[django集成pytest进行自动化单元测试实战](https://blog.csdn.net/fullbug/article/details/140893466)》。
配置测试命令就是在测试服务其中进行发布测试的所有shell命令
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/10-云效单元测试.png)

配置测试命令就是在测试服务其中进行发布测试的所有shell命令
作为一个django的项目测试命令参考如下：

```powershell
# pytest default command
# 安装mysql客户端
sudo apt-get update
sudo apt-get install -y libmysqlclient-dev

# 安装新版本的SQLite3
# wget https://www.sqlite.org/2024/sqlite-autoconf-3460000.tar.gz

# 安装依赖
sudo pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
sudo pip install pysqlite3 -i https://pypi.tuna.tsinghua.edu.cn/simple
sudo pip install pysqlite3-binary -i https://pypi.tuna.tsinghua.edu.cn/simple

# 替换Django的sqlite3的驱动文件
sudo cp -f /root/workspace/ishareblog_J18t/change_set/base.py  /usr/local/lib/python3.8/site-packages/django/db/backends/sqlite3/base.py

# 初始化数据库
sudo python manage.py makemigrations --settings=ishareblog.settings_test
sudo python manage.py migrate --settings=ishareblog.settings_test

# 启动django服务
sudo nohup python manage.py runserver 8000 --settings=ishareblog.settings_test &

PORT=8000  # 替换为您想要检查的端口号
NEXT_COMMAND="sudo pytest --html=report/index.html"  # 通过pytest进行单元测试
 
until nc -z localhost $PORT; do
    echo "Port $PORT is not ready - waiting..."
    curl http://localhost:8000
    sleep 1
done
 
echo "Port $PORT is ready"
eval "$NEXT_COMMAND"

ps aux | grep python


# 通过pytest进行单元测试
#sudo pytest --html=report/index.html

pkill -f manage.py

ps aux | grep python
```
因为在单元测试中还做了接口测试，这里会要启动djang服务，进行接口测试，测试完成后还要停止服务。
可以在流水线执行完后查看扫描报告和测试报告
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/11-云效查看测试报告.png)

扫描报告
代码扫描报告，报出来的大部分是格式规范的问题。
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/12-云效代码扫描报告.png)

测试报告
自动化测试报告是通过pytest测试完成形成的报告。
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/13-云效自动化测试报告.png)

## 五、自动构建
自动构建将会将构建好的制品打包上传至构建服务器上。
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/14-云效自动构建.png)

## 六、自动部署
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/15-云效自动部署.png)

也可以配置部署后的通知邮件，比如部署成功或失败后发邮件通知。

![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/16-云效自动部署后发通知邮件.png)

最后通过统计报表可以看到流水线近段期间的执行情况

![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240811/17-云效流水线统计报表.png)

## 七、总结
DevOps通过自动化的流程，使得构建、测试、发布软件能够更加地快捷、频繁和可靠。本文通过一个python的django个人博客应用进行了DevOps的实战，通过DevOps拉通开发和运维，通过应用云效的DevOps平台实现自动化“软件交付”的流程，使得构建、测试、发布软件能够更加地快捷、频繁和可靠，提交研发交付效率。作为个人项目也是可以应用devops提高效率。

------------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>