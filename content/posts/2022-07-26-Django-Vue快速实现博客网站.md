---
title: Django+Vue快速实现博客网站
id: 20220726001
tags:
  - Python
  - Vue
categories:
  - - 技术
    - 开发
abbrlink: 15ee23ea
date: 2022-07-26 17:18:33
---
Django是一个开放源代码的Web应用框架，由Python写成。采用了MTV的框架模式，即模型M，视图V和模版T。它最初是被开发来用于管理劳伦斯出版集团旗下的一些以新闻内容为主的网站的，即是CMS（内容管理系统）软件。对于博客网站来说是典型的CMS应用。本文介绍通过Django+Vue的博客模版快速实现一个可用的博客网站。

这里用的博客模板是Gblog是一款nice的基于 vue 的博客模板。界面简洁轻快，非常适合用作个人博客。https://gitee.com/fengziy/Gblog 后台的接口和管理界面就通过Django框架来实现了。

这里数据库用mysql，接口框架主要用到的是Django的djangorestframework，内容编辑器用的是markdown通过django-mdedior库实现。

## 一、依赖库

```bash
asgiref==3.5.2
Django==4.0.6
django-cors-headers==3.13.0
django-filter==22.1
django-mdeditor==0.1.20
djangorestframework==3.13.1
mysqlclient==2.1.1
Pillow==9.2.0
pytz==2022.1
sqlparse==0.4.2
tzdata==2022.1
```

## 二、工程目录组织结构
![工程目录组织结构](http://image2.ishareread.com/images/2022/20220726/工程目录组织结构.png)

## 三、代码实现
### 1、模型
模型很简单，根据Gblog前台要显示的内容包括有‘文章分类’、‘文章标签’、‘博客文章’、‘站点信息’、‘社交信息’、‘聚焦’，模型定义分别如下：
这里要说明的是因为博客文章内容准备用markdown编写，所以引入了mdeditor `from mdeditor.fields import MDTextField`
内容字段`content=MDTextField(verbose_name='内容')`
模型代码示例如下：

```python
from django.db import models
from common.basemodel import BaseModel
from mdeditor.fields import MDTextField
# Create your models here.
'''文章分类'''
class BlogCategory(BaseModel):
    id = models.AutoField(primary_key=True)
    title=models.CharField(max_length=50,verbose_name='分类名称',default='')
    href=models.CharField(max_length=100,verbose_name='分类路径',default='')

    def __str__(self):
        return self.title

    class Meta:
        verbose_name='文章分类'
        verbose_name_plural='文章分类'


'''文章标签'''
class Tag(BaseModel):
    id=models.AutoField(primary_key=True)
    tag=models.CharField(max_length=20, verbose_name='标签')

    def __str__(self):
        return self.tag

    class Meta:
        verbose_name='标签'
        verbose_name_plural='标签'


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
    tags=models.ManyToManyField(to=Tag, related_name="tag_post", blank=True, default=None,verbose_name="标签")


    @property
    def tag_list(self):
        return ','.join([i.tag for i in self.tags.all()])

    def __str__(self):
            return self.title

    class Meta:
        verbose_name = '博客文章'
        verbose_name_plural = '博客文章'


'''站点信息'''
class Site(BaseModel):
    id = models.AutoField(primary_key=True)
    name=models.CharField(max_length=50, verbose_name='站点名称', unique = True)
    avatar=models.CharField(max_length=200, verbose_name='站点图标')
    slogan=models.CharField(max_length=200, verbose_name='站点标语')
    domain=models.CharField(max_length=200, verbose_name='站点域名')
    notice=models.CharField(max_length=200, verbose_name='站点备注')
    desc=models.CharField(max_length=200, verbose_name='站点描述')

    def __str__(self):
            return self.name

    class Meta:
        verbose_name = '站点信息'
        verbose_name_plural = '站点信息'


'''社交信息'''
class Social(BaseModel):
    id=models.AutoField(primary_key=True)
    title=models.CharField(max_length=20, verbose_name='标题')
    icon=models.CharField(max_length=200, verbose_name='图标')
    color=models.CharField(max_length=20, verbose_name='颜色')
    href=models.CharField(max_length=100, verbose_name='路径')

    def __str__(self):
        return self.title

    class Meta:
        verbose_name = '社交信息'
        verbose_name_plural = '社交信息'

'''聚焦'''
class Focus(BaseModel):
    id=models.AutoField(primary_key=True)
    title=models.CharField(max_length=20, verbose_name='标题')
    img=models.CharField(max_length=100, verbose_name='路径')

    def __str__(self):
        return self.title

    class Meta:
        verbose_name='聚焦'
        verbose_name_plural='聚焦'

'''友链'''
class Friend(BaseModel):
    id=models.AutoField(primary_key=True)
    siteName=models.CharField(max_length=20, verbose_name='友链站点名称')
    path=models.CharField(max_length=100, verbose_name='地址路径')
    desc=models.CharField(max_length=200, verbose_name='描述')

    def __str__(self):
        return self.siteName

    class Meta:
        verbose_name='友链'
        verbose_name_plural='友链'
```

### 2、admin管理
实际上只要把模型注册到admin就可以了

```python
from django.contrib import admin
from blog.models import *
# Register your models here.
@admin.register(BlogCategory)
class BlogCategoryAdmin(admin.ModelAdmin):
    admin.site.site_title="ishareblog后台"
    admin.site.site_header="ishareblog后台"
    admin.site.index_title="ishareblog管理"

    list_display = ['id', 'title', 'href']

@admin.register(BlogPost)
class BlogPostAdmin(admin.ModelAdmin):
    list_display = ['title','category','isTop','isHot']
    search_fields = ('title',)

@admin.register(Site)
class SiteAdmin(admin.ModelAdmin):
    list_display = ['name','slogan','domain','desc']

@admin.register(Social)
class SocialAdmin(admin.ModelAdmin):
    list_display = ['title','href']

@admin.register(Focus)
class FoucusAdmin(admin.ModelAdmin):
    list_display = ['title','img']

@admin.register(Friend)
class FoucusAdmin(admin.ModelAdmin):
    list_display = ['siteName','path','desc']

@admin.register(Tag)
class TagAdmin(admin.ModelAdmin):
    list_display = ['id','tag']
```

### 3、接口
前端是Vue模板展示的，所以要为前端Vue提供相应的接口。通过djangorestframework将模型通过restful接口提供是非常easy的。
#### 1）首先将需要暴露的模型通过序列化类序列化

```python
serializers.py

from blog.models import *
from rest_framework import serializers
class BlogCategoryModelSerializer(serializers.ModelSerializer):
    class Meta:
        model=BlogCategory
        fields = "__all__"

class TagModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = "__all__"


class BlogPostModelSerializer(serializers.ModelSerializer):
    create_time = serializers.DateTimeField(format="%Y-%m-%d %H:%M:%S", required=False, read_only=True)
    update_time = serializers.DateTimeField(format="%Y-%m-%d %H:%M:%S", required=False, read_only=True)
    category_id = serializers.CharField(max_length=32, source='category.id')
    pubTime=update_time
    category=BlogCategoryModelSerializer()
    tags=serializers.SerializerMethodField()

    # 多对多，钩子函数序列化,必须是以get_开头的
    def get_tags(self, obj):
        tags = obj.tags.all()
        tag = TagModelSerializer(tags, many=True)
        return tag.data

    class Meta:
        model=BlogPost
        fields="__all__"

class SiteModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = Site
        fields = "__all__"

class SocialModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = Social
        fields = "__all__"

class FocusModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = Focus
        fields = "__all__"

class FriendModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = Friend
        fields = "__all__"
```

#### 2）将序列化的对象通过视图类提供接口
custommodelviewset.py

```python
from rest_framework import status
from rest_framework import viewsets
from common.customresponse import CustomResponse

class CustomModelViewSet(viewsets.ModelViewSet):

    #CreateModelMixin->create
    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        headers = self.get_success_headers(serializer.data)
        return CustomResponse(data=serializer.data, code=201,msg="OK", status=status.HTTP_201_CREATED,headers=headers)
    #ListModelMixin->list
    def list(self, request, *args, **kwargs):
        queryset = self.filter_queryset(self.get_queryset())
        page = self.paginate_queryset(queryset)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(queryset, many=True)
        return CustomResponse(data=serializer.data, code=200, msg="OK", status=status.HTTP_200_OK)

    #RetrieveModelMixin->retrieve
    def retrieve(self, request, *args, **kwargs):
        instance = self.get_object()
        serializer = self.get_serializer(instance)
        return CustomResponse(data=serializer.data, code=200, msg="OK", status=status.HTTP_200_OK)
    #UpdateModelMixin->update
    def update(self, request, *args, **kwargs):
        partial = kwargs.pop('partial', False)
        instance = self.get_object()
        serializer = self.get_serializer(instance, data=request.data, partial=partial)
        serializer.is_valid(raise_exception=True)
        self.perform_update(serializer)

        if getattr(instance, '_prefetched_objects_cache', None):
            # If 'prefetch_related' has been applied to a queryset, we need to
            # forcibly invalidate the prefetch cache on the instance.
            instance._prefetched_objects_cache = {}

        return CustomResponse(data=serializer.data, code=200, msg="OK", status=status.HTTP_200_OK)

    #DestroyModelMixin->destroy
    def destroy(self, request, *args, **kwargs):
        instance = self.get_object()
        self.perform_destroy(instance)
        return CustomResponse(data=[], code=204, msg="OK", status=status.HTTP_204_NO_CONTENT)
```

views.py

```python
# Create your views here.
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import viewsets, status
from rest_framework import filters
from api.myfilter import BlogPostFilter
from api.serializers import *
from blog.models import BlogCategory, BlogPost,Site,Social,Focus,Friend,Tag
from api.mypage import MyPage
from common.custommodelviewset import CustomModelViewSet
from common.customresponse import CustomResponse

class BlogCategoryViewset(CustomModelViewSet):
    queryset = BlogCategory.objects.all()
    serializer_class = BlogCategoryModelSerializer

class BlogsView(CustomModelViewSet):
    queryset = BlogPost.objects.order_by('-isTop','-update_time')
    serializer_class = BlogPostModelSerializer
    pagination_class = MyPage
    filter_backends = (DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter)
    filterset_class = BlogPostFilter
    #搜索
    search_fields=('title',)
    #排序
    ordering_fields = ('isTop', 'update_time')
    #自定义获取详情接口
    def retrieve(self,request,*args, **kwargs):
        instance=self.get_object()
        instance.viewsCount+=1
        instance.save()
        serializer=self.get_serializer(instance)
        return CustomResponse(data=serializer.data,code=200,msg="success",status=status.HTTP_200_OK)


class SiteView(CustomModelViewSet):
    queryset = Site.objects.all()
    serializer_class = SiteModelSerializer

class SocialView(CustomModelViewSet):
    queryset = Social.objects.all()
    serializer_class = SocialModelSerializer

class FocusView(CustomModelViewSet):
    queryset = Focus.objects.all()
    serializer_class = FocusModelSerializer

class FriendView(viewsets.ModelViewSet):
    queryset = Friend.objects.all()
    serializer_class = FriendModelSerializer

class TagView(viewsets.ModelViewSet):
    queryset = Tag.objects.all()
    serializer_class = TagModelSerializer
```

#### 3）通过路由来实现接口地址和视图的绑定和访问 
urls.py

```python
# -*- coding: utf-8 -*-
"""
    :author: XieJava
    :url: http://ishareread.com
    :copyright: © 2021 XieJava <xiejava@ishareread.com>
    :license: MIT, see LICENSE for more details.
"""
from api import views
from django.urls import path,include
from rest_framework.routers import DefaultRouter
blogcategory_list=views.BlogCategoryViewset.as_view({'get':'list',})
blogcategory_detail=views.BlogCategoryViewset.as_view({ 'get': 'retrieve',})
blog_list=views.BlogsView.as_view({'get':'list',})
blog_detail=views.BlogsView.as_view({ 'get': 'retrieve',})
site_list=views.SiteView.as_view({'get':'list',})
site_detail=views.SiteView.as_view({'get':'retrieve',})
social_list=views.SocialView.as_view({'get':'list',})
social_detail=views.SocialView.as_view({'get':'retrieve',})
focus_list=views.FocusView.as_view({'get':'list',})
focus_detail=views.FocusView.as_view({'get':'retrieve'})
friend_list=views.FriendView.as_view({'get':'list',})
friend_detail=views.FriendView.as_view({'get':'retrieve'})
tags_list=views.TagView.as_view({'get':'list',})
# router=DefaultRouter()
# router.register('blogs',views.BlogsView)
urlpatterns = [
    path('category/',blogcategory_list),
    path('category/<pk>/',blogcategory_detail),
    path('post/list',blog_list),
    path('post/<pk>',blog_detail),
    path('social/',social_list),
    path('site/<pk>',site_detail),
    path('focus/list',focus_list),
    path('comment/',blog_list),
    path('friend/',friend_list),
    path('tags/',tags_list),
]
```

#### 4）自定义接口返回格式
接口需要根据Glog定义的格式进行定义和返回，这里就需要自定义接口返回格式。
具体实现参见：[https://xiejava.blog.csdn.net/article/details/125773730](https://xiejava.blog.csdn.net/article/details/125773730)
--自定义返回响应类customresponse.py

```python
from rest_framework.response import Response
from rest_framework.serializers import Serializer

class CustomResponse(Response):
    def __init__(self,data=None,code=None,msg=None,
                 status=None,
                 template_name=None, headers=None,
                 exception=False, content_type=None,**kwargs):
        super().__init__(None, status=status)

        if isinstance(data, Serializer):
            msg = (
                'You passed a Serializer instance as data, but '
                'probably meant to pass serialized `.data` or '
                '`.error`. representation.'
            )
            raise AssertionError(msg)
        self.data={'code':code,'msg':msg,'data':data}
        self.data.update(kwargs)
        self.template_name=template_name
        self.exception=exception
        self.content_type=content_type

        if headers:
            for name, value in headers.items():
                self[name] = value
```

--翻页实现类mypage.py

```python
from rest_framework import status
from rest_framework.pagination import PageNumberPagination
from common.customresponse import CustomResponse

class MyPage(PageNumberPagination):
    page_size = 8 #每页显示数量
    max_page_size = 50 #每页最大显示数量。
    page_size_query_param = 'size' #每页数量的参数名称
    page_query_param = 'page'  #页码的参数名称

    #自定义分页器的返回参数
    def get_paginated_response(self, data):
        ret_data = dict()
        ret_data['items'] = data
        # 加入自定义分页信息
        ret_data['total'] = self.page.paginator.count
        ret_data['hasNextPage'] = self.get_next_link()
        ret_data['size'] = self.page_size
        ret_data['page'] = self.page.number
        return CustomResponse(data=ret_data,code=200,msg="OK",status=status.HTTP_200_OK)
```

全部代码：
后台代码：[https://gitee.com/xiejava/ishareblog](https://gitee.com/xiejava/ishareblog)
前台代码：[https://gitee.com/xiejava/Gblog](https://gitee.com/xiejava/Gblog)

## 四、效果
### 1、后台管理
管理界面
![管理界面](http://image2.ishareread.com/images/2022/20220726/后台管理.png)
博客文章列表
![博客文章列表](http://image2.ishareread.com/images/2022/20220726/博客文章列表.png)
文章内容编辑，支持markdown
![文章内容编辑，支持markdown](http://image2.ishareread.com/images/2022/20220726/文章内容markdown编辑.png)
分类管理
![文章分类](http://image2.ishareread.com/images/2022/20220726/分类管理.png)
标签管理
![标签管理](http://image2.ishareread.com/images/2022/20220726/标签管理.png)
社交信息
![社交信息](http://image2.ishareread.com/images/2022/20220726/社交信息.png)
### 2、接口
接口清单
![接口清单](http://image2.ishareread.com/images/2022/20220726/接口清单.png)

文章列表接口，支持翻页

![文章列表接口](http://image2.ishareread.com/images/2022/20220726/文章列表接口.png)

文章详情接口
![文章详情接口](http://image2.ishareread.com/images/2022/20220726/文章详情接口.png)

### 3、前台展现
![前台展现](http://image2.ishareread.com/images/2022/20220726/前台展现.png)

文章列表
![文章列表](http://image2.ishareread.com/images/2022/20220726/文章列表.png)

文章详情，支持markdown显示及目录
![文章详情](http://image2.ishareread.com/images/2022/20220726/文章详情.png)

社交信息
![社交信息](http://image2.ishareread.com/images/2022/20220726/社交信息-前台.png)

博客效果地址：http://blog.ishareread.com

后续考虑
1、django原生admin的管理界面还是简陋了一点，后续可能会用其他管理界面的UI给换掉
2、现在有了一个hexo的博客了，后续可能会考虑实现hexo生成的博客内容直接同步到django的博客，或者django博客编辑的内容直接生成hexo的.md文件
有兴趣的话可以关注本博客

-------------------------
博客：http://xiejava.ishareread.com

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注微信公众号,一起学习、成长！</center>