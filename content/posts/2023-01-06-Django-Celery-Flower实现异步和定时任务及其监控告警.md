---
title: Django+Celery+Flower实现异步和定时任务及其监控告警
id: 20230106001
tags:
  - Python
  - Django
categories:
  - - 技术
    - 开发
abbrlink: c2fa9556
date: 2023-01-06 21:00:15
---
用Django框架进行web开发非常的快捷方便，但Django框架请求/响应是同步的。但我们在实际项目中经常会碰到一些耗时的不能立即返回请求结果任务如：数据爬取、发邮件等，如果常时间等待对用户体验不是很好，在这种情况下就需要实现异步实现，马上返回响应请求，但真正的耗时任务在后台异步执行。Django框架本身无法实现异步响应但可以通过Celery很快的实现异步和定时任务。本文将介绍如何通过Django+Celery+Flower实现异步和定时任务及其任务的监控告警。

常见的任务有两类，一类是异步任务，一类是定时任务（定时执行或按一定周期执行）。Celery都能很好的支持。

Celery 是一个 基于python开发的分布式异步消息任务队列，通过它可以轻松的实现任务的异步处理， 如果你的业务场景中需要用到异步任务，就可以考虑使用celery， 举几个实例场景中可用的例子:

 - 异步任务：将耗时的操作任务提交给Celery去异步执行，比如发送短信/邮件、消息推送、音频处理等等
 - 做一个定时任务，比如每天定时执行爬虫爬取指定内容

Celery 在执行任务时需要通过一个消息中间件（Broker）来接收和发送任务消息，以及存储任务结果， 一般使用rabbitMQ、Redis或其他DB。

本文使用redis作为消息中间件和结果存储，在后面的通过数据库监控任务执行案例将介绍用到数据库作为结果存储。

## 一、在Django中引入Celary
### 1、安装库

```python
pip install celery
pip install redis
pip install eventlet  #在windows环境下需要安装eventlet包
```

### 2、引入celary
在主项目目录下，新建celary.py文件，内容如下：

```python
import os
import django
from celery import Celery
from django.conf import settings

# 设置系统环境变量，否则在启动celery时会报错
# taskproject 是当前项目名
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'taskproject.settings')
django.setup()

celery_app = Celery('taskproject')
celery_app.config_from_object('django.conf:settings')
celery_app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/1_引入celery.png)

在主目录的__init__.py中添加如下代码：

```python
from .celery import celery_app

__all__ = ['celery_app']
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/2___init__py添加代码.png)

### 3、在settings.py中设置celery的相关参数

```python
###----Celery redis 配置-----###
# Broker配置，使用Redis作为消息中间件
BROKER_URL = 'redis://:redispassword@127.0.0.1:6379/0'

# BACKEND配置，使用redis
CELERY_RESULT_BACKEND = 'redis://:redispassword@127.0.0.1:6379/1'


CELERY_ACCEPT_CONTENT=['json']
CELERY_TASK_SERIALIZER='json'
# 结果序列化方案
CELERY_RESULT_SERIALIZER = 'json'

# 任务结果过期时间，秒
CELERY_TASK_RESULT_EXPIRES = 60 * 60 * 24

# 时区配置
CELERY_TIMEZONE = 'Asia/Shanghai'
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/3_setting中设置celery的相关参数.png)

这时候Celery的基本配置完成了，可以实现并添加任务了。

## 二、实现异步任务
### 1、创建tasks.py
在子应用下建立各自对应的任务文件tasks.py(`必须是tasks.py这个名字，不允许修改`)

```python
import datetime
from time import sleep
from celery import shared_task
import logging
logger = logging.getLogger(__name__)


@shared_task()
def task1(x):
    for i in range(int(x)):
        sleep(1)
        logger.info('this is task1 '+str(i))
    return x


@shared_task
def scheduletask1():
    now = datetime.datetime.now()
    logger.info('this is scheduletask '+now.strftime("%Y-%m-%d %H:%M:%S"))
    return None
```

在tasks.py中我们定义了两个任务，这两个任务要用@shared_task装饰起来，否则celery无法管理。
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/4_@shared_task.png)


为了放便执行我们通过views把这两个任务通过函数方法调用起来，用URL进行发布。

```python
views.py
from django.http import JsonResponse
from . import tasks
# Create your views here.


def runtask(request):
    x=request.GET.get('x')
    tasks.task1.delay(x)
    content= {'200': 'run task1 success!---'+str(x)}
    return JsonResponse(content)


def runscheduletask(request):
    tasks.scheduletask1.delay()
    content= {'200': 'success！'}
    return JsonResponse(content)
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/5_views发布任务.png)

在urls中加入路由进行发布

```python
from django.urls import path

from taskapp import views

urlpatterns = [
    path('task', views.runtask),
    path('runscheduletask', views.runscheduletask),
]
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/6_urls加入路由进行发布.png)

在项目的主urls中加入子项目的urls
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/7_主urls中加入子项目的urls.png)

### 2、启动celery

> 在启动celery之前，先要启动redis服务，因为celery在settings中配置要用到redis作为消息中间件和结果存储。
>windows环境下启动redis的命令为redis-server.exe redis.windows.conf

在控制台启动celery的worker

```python
celery -A taskproject worker -l debug -P eventlet
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/8_启动celery.png)

启动django访问url调用任务，看异步效果
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/9.查看异步效果.png)

### 3、查看任务
控制台查看异步任务执行的情况，可以看web的url很快返回响应结果，后台控制台一直在执行异步任务。
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/10_查看任务.png)

## 三、实现定时任务
Celery实现定时任务也很方便
### 1、定义调度器
在settings.py中加入定时任务的定义就可以实现定时任务

```python
from celery.schedules import crontab

CELERYBEAT_SCHEDULE = {
    'every_5_seconds': {
        # 任务路径
        'task': 'taskapp.tasks.scheduletask1',
        # 每5秒执行一次
        'schedule': 5,
        'args': ()
    }
}
```
这里这个scheduletask1是前面tasks.py中定义的任务，当然也可以定义多个定时任务，如加一个task1，task1是有参数的，可以在'args': ()中传入参数

```python
CELERYBEAT_SCHEDULE = {
    'every_5_seconds': {
        # 任务路径
        'task': 'taskapp.tasks.scheduletask1',
        # 每5秒执行一次
        'schedule': 5,
        'args': ()
    },
    'every_10_seconds': {
        # 任务路径
        'task': 'taskapp.tasks.task1',
        # 每10秒执行一次,task1的参数是5
        'schedule': 10,
        'args': ([5])
    }
}
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/11_带参数的任务.png)

这里定义了task1是10秒执行一次，传入的参数是5。
### 2、启动beat
需要保持worker进程，另外开一个控制台启动beat

```python
celery -A taskproject beat -l debug
```
### 3、查看任务
启动任务后看控制台打印的日志task1和scheduletask1都按计划定时执行了。
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/12_查看task和scheduletask.png)


## 三、通过数据库配置定时任务
虽然通过settings.py的配置可以实现定时任务的配置，做为实际项目中可能还是不够实用，更加工程化的做法是将定时任务的配置放到数据库里通过界面来配置。同样Celery对此也提供了很好的支持，这需要安装django-celery-beat插件。以下将介绍使用过程。
### 1、安装djiango-celery-beat

```python
pip install django-celery-beat
```

### 2、在APP中注册djiango-celery-beat

```python
INSTALLED_APPS = [
....
'django_celery_beat',
]
```

### 3、在settings.py中设置调度器及时区
在settings.py中屏蔽到原来的调度器，加入

```python
CELERYBEAT_SCHEDULER = 'django_celery_beat.schedulers.DatabaseScheduler' 
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/13_settings设置定时任务数据库配置.png)

在setings.py中设置好语言、时区等

```python
LANGUAGE_CODE = 'zh-hans'

TIME_ZONE = 'Asia/Shanghai'

USE_I18N = True

USE_TZ = False
```

### 4、进行数据库迁移

```python
python manage.py migrate django_celery_beat
```

### 5、分别启动woker和beta
在两个控制台分别启动woker和beta

```python
celery -A taskproject worker -l debug -P eventlet
```

```python
celery -A taskproject beat -l debug
```

### 6、启动django服务，访问admin的web管理端
访问 http://localhost:8000/admin/ 可以看到周期任务的管理菜单，管理定时任务非常方便。
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/14_Django管理web端.png)
### 7、配置定时任务
点击“间隔”
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/15_配置定时任务点击间隔.png)

点击“增加间隔”来增加定时任务的配置，增加一个5秒执行一次的定时器。
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/16_点击增加间隔.png)

看到有个每5秒的定时器
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/17_每5秒的定时器.png)

这时可以用这个定时器去新建调度任务了。选择周期性任务，点击“增加周期性任务”
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/18_增加周期性任务.png)

填入任务名，选择需要定时执行的任务
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/19_增加周期性任务界面.png)

因为task1需要参数，在后面参数设置中进行参数的设置。
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/20_周期性任务参数设置.png)

保存后可以看到新加了一条“每5秒执行一次task1”的调度任务。
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/21_成功添加每5秒执行一次task1.png)

### 8、查看调度效果
在woker和beta的控制台都可以看到有定时任务执行的信息，说明任务被成功调度执行了。
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/22_查看调度效果.png)


## 四、通过django的web界面监控任务执行情况
在控制台监控任务执行情况，还不是很方便，最好是能够通过web界面看到任务的执行情况，如有多少任务在执行，有多少任务执行失败了等。这个Celery也是可以做到了，就是将任务执行结果写到数据库中，通过web界面显示出来。这里要用到django-celery-results插件。通过插件可以使用Django的orm作为结果存储，这样的好处在于我们可以直接通过django的数据查看到任务状态，同时为可以制定更多的操作，下面介绍如何使用orm作为结果存储。
### 1、安装django-celery-results

```python
pip install django-celery-results
```

### 2、配置settings.py，注册app

```python
INSTALLED_APPS = (
...,
'django_celery_results',
)
```

### 3、修改backend配置，将Redis改为django-db

```python
# BACKEND配置，使用redis
#CELERY_RESULT_BACKEND = 'redis://:12345678@127.0.0.1:6379/1'
# 使用使用django orm 作为结果存储
CELERY_RESULT_BACKEND = 'django-db'  #使用django orm 作为结果存储
```

### 4、迁移数据库

```python
python manage.py migrate django_celery_results
```

可以看到创建了django_celery_results相关的表
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/23_迁移数据库.png)

### 5、查看任务
启动django服务后，执行异步和定时任务，就可以在管理界面看到任务的执行情况，执行了哪些任务，哪些任务执行失败了等。
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/24_web界面查看任务.png)

## 五、通过Flower监控任务执行情况
如果不想通django的管理界面监控任务的执行，还可以通过Flower插件来进行任务的监控。FLower的界面更加丰富，可以监控的信息更全。以下介绍通过Flower来进行任务监控。
### 1、安装flower

```python
pip install flower
```

### 2、启动flower

```python
celery -A taskproject flower --port-5566
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/25_启动flower.png)

### 3、使用flower进行任务监控
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/26_使用flower进行任务监控.png)

点击失败的我们可以看到执行失败的详情，这里是故意给task1的参数传了个‘a’字符，导致它执行报错了。可以看到任务执行的报错信息也展示出来了。
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/27_flower结果.png)

## 六、实现任务异常自动邮件告警
虽然可以通过界面来监控了，但是我们想要得更多，人不可能天天盯着界面看吧，如果能实现任务执行失败就自动发邮件告警就好了。这个Celery当然也是没有问题的。
通过钩子程序在异常的时候触发邮件通知。
### 1、加入钩子程序
对tasks.py的改造如下：

```python
import datetime
from time import sleep
from celery import shared_task
from celery import Task
from django.core.mail import send_mail
import logging
logger = logging.getLogger(__name__)

class MyHookTask(Task):
    def on_success(self, retval, task_id, args, kwargs):
        info=f'任务成功-- 0task id:{task_id} , arg:{args} , successful !'
        logger.info(info)
        send_mail('celery任务监控', info, 'sendmail@qq.com', ['tomail@qq.com'])

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        info=f'任务失败-- task id:{task_id} , arg:{args} , failed ! erros: {exc}'
        logger.info(info)
        send_mail('celery任务监控异常', info, 'sendmail@qq.com', ['tomail@qq.com'])

    def on_retry(self, exc, task_id, args, kwargs, einfo):
        logger.info(f'task id:{task_id} , arg:{args} , retry !  erros: {exc}')


@shared_task(base=MyHookTask, bind=True)
def task1(self,x):
    for i in range(int(x)):
        sleep(1)
        logger.info('this is task1 '+str(i))
    return x


@shared_task
def scheduletask1():
    now = datetime.datetime.now()
    logger.info('this is scheduletask '+now.strftime("%Y-%m-%d %H:%M:%S"))
    return None
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/28_加入钩子程序.png)

### 2、重启服务
将work和beta服务关掉，在两个控制台分别重新启动woker和beta

```python
celery -A taskproject worker -l debug -P eventlet
```

```python
celery -A taskproject beat -l debug
```

### 3、验证效果
在任务成功或失败的时候发邮件通知。
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/29_任务调度结果.png)

任务执行成功通知
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/30_任务执行成功通知.png)

任务执行异常告警通知
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20230112/31_任务执行异常告警通知.png)

Django如何发送邮件见 [https://blog.csdn.net/fullbug/article/details/128495415](https://blog.csdn.net/fullbug/article/details/128495415)

至此，本文通过几个简单的应用介绍了Django+Celery+Flower实现异步和定时任务及其监控告警。

------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>