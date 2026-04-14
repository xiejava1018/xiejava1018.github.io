---
title: Django的restframework接口框架自定义返回数据格式
id: 20220714001
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: 1dea176f
date: 2022-07-14 09:30:59
---
在前后端分离是大趋势的背景下，前端获取数据都是通过调用后台的接口来获取数据微服务的应用越来越多。Django是Python进行web应用开发常用的web框架，用Django框架进行web应用框架减少了很多工作，通常用很少量的代码就可以实现数据的增、删、改、查的业务应用，同样用Django的restframework的框架对外发布接口也是非常的简单方便，几行代码就可以将数据对象通过接口的方式提供服务。因为在实际开发过程中接口的返回数据有一定的格式，本文介绍通过自定义Response返回对象来自定义接口返回数据格式。

以下示例将数据对象Friend通过restframework框架进行接口发布。
只要定义Friend数据对象

```python
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

定义一个序列化类将返回的字段序列化

```python
class FriendModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = Friend
        fields = "__all__"
```

定义一个接口视图类获取数据

```python
class FriendView(viewsets.ModelViewSet):
    queryset = Friend.objects.all()
    serializer_class = FriendModelSerializer
```

定义接口路由就可以通过httprestfull的接口进行访问了

```python
friend_list=views.FriendView.as_view({'get':'list',})
urlpatterns = [
    path('friend/',friend_list),
]
```

接口访问效果如下：
http://localhost:8000/api/friend/
![httprestfull的接口](http://image2.ishareread.com/images/2022/20220713/接口发布.png)

但是在项目中经常会碰到接口格式变化的情况，restframework框架默认的返回数据格式不满足应用的需求。比如一般的接口都会有接口返回的code、msg、data，code用来标识接口返回代码比如200是正常，msg用来记录异常或其信息，data用来返回具体的数据。
通过restframework接口自定义返回数据格式也是很简单方便的。
先自定义Response返回对象，在返回对象中自定义数据返回的格式，示例代码如下：

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
        #自定义返回格式
        self.data={'code':code,'msg':msg,'data':data}
        self.data.update(kwargs)
        self.template_name=template_name
        self.exception=exception
        self.content_type=content_type

        if headers:
            for name, value in headers.items():
                self[name] = value
```

在接口接口视图类获取数据返回时，使用该自定义的Response返回对象。

```python
class FriendView(viewsets.ModelViewSet):
    queryset = Friend.objects.all()
    serializer_class = FriendModelSerializer
    #自定义list方法，自定义Response返回
  def list(self, request, *args, **kwargs):
        queryset = self.filter_queryset(self.get_queryset())
        serializer = self.get_serializer(queryset, many=True)
        return CustomResponse(data=serializer.data, code=200, msg="OK", status=status.HTTP_200_OK)
```

接口访问效果如下：
可以看到返回数据格式中增加了code,msg 数据放到了data节点
![自定义数据返回格式](http://image2.ishareread.com/images/2022/20220713/自定义接口返回数据1.png)

列表数据通常接口要提供翻页功能，在接口中要有总页数、当前页、是否有下一页的信息。
可以自定义一个分页器，在分页器中自定义需要返回的分页参数
参考示例代码如下：

```python
from rest_framework import status
from rest_framework.pagination import PageNumberPagination
from common.customresponse import CustomResponse

class MyPage(PageNumberPagination):
    page_size = 8 #每页显示数量
    max_page_size = 50 #每页最大显示数量。
    page_size_query_param = 'size' #每页数量的参数名称
    page_query_param = 'page'  #页码的参数名称

    def get_paginated_response(self, data):
        #自定义分页器的返回参数
        return CustomResponse(data=data,code=200,msg="OK",status=status.HTTP_200_OK, count=self.page.paginator.count,next=self.get_next_link(),previous=self.get_previous_link(),size=self.page_size,page=self.page.number)
```

在接口接口视图类获取数据返回时，如果有分页器则使用该分页器自定义的Response返回对象。

```python
class FriendView(viewsets.ModelViewSet):
    queryset = Friend.objects.all()
    serializer_class = FriendModelSerializer
    pagination_class = MyPage
    #自定义list方法，自定义Response返回
    def list(self, request, *args, **kwargs):
        queryset = self.filter_queryset(self.get_queryset())
        page = self.paginate_queryset(queryset)
        #如果有分页器，则进行分页后返回
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(queryset, many=True)
        return CustomResponse(data=serializer.data, code=200, msg="OK", status=status.HTTP_200_OK)
```

接口访问效果如下：
可以看到接口中自定义增加了分页信息。
![接口中自定义增加了分页信息](http://image2.ishareread.com/images/2022/20220713/自定义接口返回数据2-分页信息.png)

但是有时候可能希望分页的信息数据要放在data节点里面，这样也是可以做到的。

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

接口访问效果如下：
可以看到接口中自定义增加了分页信息，分页的信息数据放在data节点里面了
![自定义增加了分页信息，分页的信息数据放在data节点里面](http://image2.ishareread.com/images/2022/20220713/自定义接口返回数据3-分页信息在data里.png)
至此，本文介绍了通过Django的restframework接口框架自定义Response返回对象来自定义返回数据格式。Django的restframework接口框架使用简单方便，拿来即用，能够很大程度上减少代码开发量。

-----------------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>