---
title: django+markdown2+pygments实现markdown解析及代码高亮
id: 20240814001
tags:
  - Python
  - Django
categories:
  - 技术
abbrlink: af83983b
date: 2024-08-14 18:38:22
---
随着markdown的流行，web应用系统常常会要碰到有使用markdown编辑器进行富文本编辑，然后在前台web页面进行显示。常见的博客系统当然也需要支持markdown的编辑与显示。本文就通过一个真实的博客系统来说明django+markdown2+pygments实现markdown解析及代码高亮。

## 一、后台管理支持markdown编辑
django应用的后台管理支持markdown可以用django-mdeditor，它是一个Django应用，它集成了markdown-editor，允许你在Django项目中使用富文本编辑器编写Markdown格式的内容。这个插件通常用于博客、论坛或任何需要用户输入Markdown文本的场景。
### 1、安装依赖
首先，通过pip安装django-mdeditor

```python
pip install django-mdeditor
```

### 2、添加应用到INSTALLED_APPS
在settings.py文件中，将django_mdeditor添加到INSTALLED_APPS列表中，参考如下：


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
    'django_filters',  # 注册条件查询
    'mdeditor',  # 注册markdown的应用
    'drf_yasg2',  # 接口文档
]
```

### 3、设置MEDIA_URL和MEDIA_ROOT
django-mdeditor使用文件上传功能，因此需要在settings.py中正确设置

```python
MEDIA_URL和MEDIA_ROOT：
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'uploads')
```

### 4、在模型中使用MdEditorField
在Django模型中，使用MdEditorField替换标准的TextField：

```python
'''博客文章'''
class BlogPost(BaseModel):
    id = models.AutoField(primary_key=True)
    title = models.CharField(max_length=200, verbose_name='文章标题', unique = True)
    category = models.ForeignKey(BlogCategory, blank=True,null=True, verbose_name='文章分类', on_delete=models.DO_NOTHING)
    isTop = models.BooleanField(default=False, verbose_name='是否置顶')
    isHot = models.BooleanField(default=False, verbose_name='是否热门')
    isShow = models.BooleanField(default=False, verbose_name='是否显示')
    summary = models.TextField(max_length=500, verbose_name='内容摘要', default='')
    content = MDTextField(verbose_name='内容')
    viewsCount = models.IntegerField(default=0, verbose_name="查看数")
    commentsCount = models.IntegerField(default=0, verbose_name="评论数")
    tags = models.ManyToManyField(to=Tag, related_name="tag_post", blank=True, default=None, verbose_name="标签")
    blogSource = models.CharField(max_length=200, blank=True, null=True, default='',verbose_name='文章来源')
    pubTime = models.DateTimeField(blank=True, null=True, verbose_name='发布日期')

    @property
    def tag_list(self):
        return ','.join([i.tag for i in self.tags.all()])

    def __str__(self):
        return self.title

    class Meta:
        verbose_name = '博客文章'
        verbose_name_plural = '博客文章'
```

在博客文章的模型中，文章内容需要支持markdown,所以`content = MDTextField(verbose_name='内容')`

### 5、查看效果
用django自带的后台管理admin就可以看到效果了

![django自带的后台管理admin](http://image2.ishareread.com/images/2024/20240814/1-django的admin的markdown编辑效果.png)

## 二、前台支持markdown解析及代码高亮
在后台支持markdown编辑后，前台页面的博客文章页面也得要支持对markdown得解析。
### 1、安装依赖
在Django项目中使用markdown2库实现代码高亮，安装markdown2和pygments这两个Python库。markdown2用于解析Markdown文本，而pygments用于代码高亮。

```python
pip install markdown2 pygments
```

### 2、配置Markdown解析器
在Django视图中，需要导入markdown2模块，并使用它来解析Markdown文本。同时，要启用fenced-code-blocks扩展，以便markdown2能正确识别代码块。
```python
import markdown2
```

markdown2的扩展说明见 https://github.com/trentm/python-markdown2/wiki/Extras
在这里加入了 

```python
extras=["code-color", "fenced-code-blocks", "cuddled-lists", "tables",
  "with-toc", "code-friendly"] 
```
视图实现
```python
# 详情页视图实现.
def post_detail(request, id):
    try:
        post_obj = BlogPost.objects.get(id=id)
        html_content = markdown2.markdown(post_obj.content,
                                          extras=["code-color", "fenced-code-blocks", "cuddled-lists", "tables",
                                                  "with-toc", "code-friendly"])
        html_content = html_content.replace('<table>', '<table class="table table-bordered">')
        html_content = html_content.replace('<img src=', '<img style="max-width:100%;height:auto;" src=')
        context = {"post_obj": post_obj,
                   "html_content": html_content,
                   "hot_posts": get_hot_posts(),
                   "tags": get_all_tags(),
                   "post_grouped_by_year": get_post_groped_by_year(),
                   'categories': get_categories(),
                   'social_infos': get_socialinfo()}
    except BlogPost.DoesNotExist:
        raise Http404("Post does not exist")
    return render(request, "blog/post.html", context)
```

### 3、在模板中显示HTML
在Django模板中，直接输出转换后的HTML内容。使用|safe过滤器告诉模板引擎不要转义HTML代码。

```python
<div class="lyear-arc-detail">
   {{ html_content|safe }}
</div>
```

### 4、导出高亮的css文件并引入css 
有了上面的步骤，只是可以解析了markdown成html并显示，最终代码的高亮是通过css 来控制显示的，执行以下命令将高亮的css文件导出。

```python
pygmentize -S default -f html -a .codehilite > markdown_highlighy.css
```

pyments的官方文档 https://pygments.org/ 查看一共有多少种风格，可以参考网址 https://pygments.org/docs/styles/#getting-a-list-of-available-styles 
将生成的markdown_highlighy.css文件拷入到static/blog/css下
在模板页面中引入css 

```html
<link rel="stylesheet" type="text/css" href="{% static 'blog/css/markdown_highlighy.css' %}" />
```

### 5、查看效果
可以看到markdown可以正常解析，代码也可以高亮显示了。

![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240814/2-django博客markdown解析与代码高亮显示.png)


## 三、全套代码
[https://gitee.com/xiejava/ishareblog](https://gitee.com/xiejava/ishareblog)

------------------

博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>