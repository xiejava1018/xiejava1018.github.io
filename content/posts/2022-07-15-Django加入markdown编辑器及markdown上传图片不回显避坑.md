---
title: Django加入markdown编辑器及markdown上传图片不回显避坑
id: 20220715001
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: 462af16b
date: 2022-07-15 23:29:21
---
一般来说一个CMS系统如博客系统都需要一个好的富文本编辑器，现在大家更多的是选择MarkDown编辑器来编辑内容。Django作为python的主流web开发框架当然少不了markdown的插件。本文介绍如何在Django框架中引入markdown编辑器及在使用markdown时的注意事项。

在Django框架中引入markdown编辑器主要是通过安装引入Django-mdeditor库来实现。
Django-mdeditor 是基于 Editor.md 的一个 django Markdown 文本编辑插件应用。
其官方下载地址见 [https://pypi.org/project/django-mdeditor/](https://pypi.org/project/django-mdeditor/)
根据官方指导文档
# 一、安装使用
## 1、安装django-mdeditor

```bash
pip install django-mdeditor
```

## 2、在 settings 配置文件 INSTALLED_APPS 中添加 mdeditor

```python
INSTALLED_APPS = [
    'blog',
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'django_filters',#注册条件查询
    # 注册markdown的应用
    'mdeditor',
]
```

## 3、针对django3.0+修改 frame 配置

```python
X_FRAME_OPTIONS = 'SAMEORIGIN'  # django 3.0 + 默认为 deny
```

## 4、在 settings 中添加媒体文件的路径配置

```python
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'uploads')
```

在你项目根目录下创建 uploads/editor 目录，用于存放上传的图片。
## 5、在项目的根 urls.py 中添加扩展url和媒体文件url:
注意是在项目的根 urls.py 中添加扩展url和媒体文件url，而不是在其他项目应用的urls.py中添加

```python
from django.conf.urls import url, include
from django.conf.urls.static import static
from django.conf import settings
...

urlpatterns = [
    ...
    url(r'mdeditor/', include('mdeditor.urls'))
]

if settings.DEBUG:
    # static files (images, css, javascript, etc.)
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

## 6、在项目model中应用markdown
示例代码如下：

```python
'''博客文章'''
class BlogPost(BaseModel):
    id = models.AutoField(primary_key=True)
    title = models.CharField(max_length=200, verbose_name='文章标题', unique = True)
    category = models.ForeignKey(BlogCategory, blank=True, null=True, verbose_name='文章分类', on_delete=models.DO_NOTHING)
    isTop=models.BooleanField(default=False,verbose_name='是否置顶')
    isHot=models.BooleanField(default=False,verbose_name='是否热门')
    summary=models.CharField(max_length=500,verbose_name='内容摘要',default='')
    content=MDTextField(verbose_name='内容')
    viewsCount= models.IntegerField(default=0, verbose_name="查看数")
    commentsCount=models.IntegerField(default=0, verbose_name="评论数")

    def __str__(self):
            return self.title

    class Meta:
        verbose_name = '博客文章'
        verbose_name_plural = '博客文章'
```

见 `content=MDTextField(verbose_name='内容')` 表示博客文章的内容是MDTextField

## 7、向 admin.py 中注册model:

```python
from django.contrib import admin
from blog.models import *
# Register your models here.
@admin.register(BlogPost)
class BlogPostAdmin(admin.ModelAdmin):
    list_display = ['title','category','isTop','isHot']
```

## 8、迁移创建数据表
运行 `python manage.py makemigrations` 和 `python manage.py migrate` 来创建你的model 数据库表，可以看到默认创建的content字段是longtext类型的
![默认创建的content字段是longtext类型的](http://image2.ishareread.com/images/2022/20220715/content字段.png)

## 9、测试验证
启动应用，访问http://127.0.0.1:8000/admin/ 点击新增博客文章，可以看到内容字段是markdown编辑器输入了。
![markdown编辑器](http://image2.ishareread.com/images/2022/20220715/django显示markdown.png)

至此django应用中就可以使用markdown编辑器了。

# 二、markdown上传图片不回显避坑
按照以上步骤配置django-mdeditor,markdown编辑器可以正常使用，但是这里有个大坑，就是有些浏览器在上传图片后上传的图片不回显！
我就碰到了这样的情况。

![上传图片后上传的图片不回显](http://image2.ishareread.com/images/2022/20220715/markdown上传图片接口调用情况.png)

在添加图片界面选择本地上传图片后发现后台接口调到了 `/mdeditor/uploads/?guid=1657867564930` 接口并且返回了200，但是上传的图片地址不回显，提交报“错误：图片地址不能为空。” 这就奇了怪了。
打开浏览器的调试工具，发现报了一个错，`Uncaught SyntaxError: Unexpected token 下 in JSON at position 141`

![浏览器的调试工具，发现报了一个错](http://image2.ishareread.com/images/2022/20220715/markdown上传图片js报错.png)

点击详情，具体应该是获取的JSON无法解析。

![JSON无法解析](http://image2.ishareread.com/images/2022/20220715/markdown上传图片js报错调试代码.png)

这个JSON为什么无法解析呢？开始进一步调试，这个JSON是上传时调用的后台上传方法返回的。所以来看看是不是后台上传接口返回的JSON串有什么问题。找到/mdeditor/uploads路由所对应的源码

![/mdeditor/uploads路由所对应的源码](http://image2.ishareread.com/images/2022/20220715/markdown上传图片调试路由视图.png)

UploadView的源代码，就是返回一个成功的json报文

```python
return JsonResponse({'success': 1,
                     'message': "上传成功！",
                     'url': os.path.join(settings.MEDIA_URL,
                                         MDEDITOR_CONFIGS['image_folder'],
                                         file_full_name)})
```

实际打断点debug也是正常返回上传成功的json报文。

![打断点debug也是正常返回上传成功的json报文](http://image2.ishareread.com/images/2022/20220715/markdown上传图片调试json.png)

这就有点奇怪了，接口返回了正常的json报文怎么就解析不了了呢？接着继续调前台js代码，看究竟是什么原因。

![json串里多了几个字“下载视频”!](http://image2.ishareread.com/images/2022/20220715/markdown上传图片调试json多出字符串.png)

发现js获取的json串里多了几个字“下载视频”! 这是什么鬼？实在是没有地方有返回“下载视频”这几个字啊？看js代码是通过iframe来处理请求的，再来看看iframe的内容，发现iframe里确实有“下载视频”

![iframe里确实有“下载视频”](http://image2.ishareread.com/images/2022/20220715/markdown上传图片调试iframe.png)

原来是有个chrome浏览器插件，擅自给加了“下载视频”的内容。再来看浏览器装了些啥插件。原来是有个迅雷插件，应该就是这个插件搞的鬼了，罪魁祸首就是它了！
![罪魁祸首迅雷插件](http://image2.ishareread.com/images/2022/20220715/markdown上传不回显问题原因多了迅雷插件.png)

把这个迅雷插件删除或停用，果然一切正常！可以正常回显！！！

![可以正常回显](http://image2.ishareread.com/images/2022/20220715/markdown上传图片回显正常.png)

显示插入的图片

![显示插入的图片](http://image2.ishareread.com/images/2022/20220715/markdown上传图片正常.png)

所以，碰到markdown上传图片不回显的情况，先看下自己的浏览器是不是开启了迅雷插件应用，如果开启了迅雷插件应用先停用或删除！

------
作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注微信公众号,一起学习、成长！</center>