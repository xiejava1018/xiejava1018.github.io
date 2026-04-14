---
title: django集成pytest进行自动化单元测试实战
id: 20240803001
tags:
  - Python
  - Django
categories:
  - 技术
abbrlink: 8fbbc02f
date: 2024-08-03 17:23:11
---
在Django项目中集成Pytest进行单元测试可以提高测试的灵活性和效率，相比于Django自带的测试框架，Pytest提供了更为丰富和强大的测试功能。本文通过一个实际项目ishareblog介绍django集成pytest进行自动化单元测试实战。
## 一、引入pytest相关的包

```python
pip install pytest
pip install pytest-django
pip install pytest-html
```

其中pytest-django插件，它提供了Django和Pytest之间的桥梁，pytest-html 是一个 pytest 的插件，用于生成详细的 HTML 测试报告。这个插件能够将 pytest 运行的结果转化为一个直观、易于阅读的 HTML 格式报告，这对于分享测试结果、审查测试覆盖率以及归档测试历史非常有帮助。

## 二、配置pytest
### 1、将django的配置区分测试环境、开发环境和生产环境
因为测试环境、开发环境和生产环境的环境配置参数不一样，一个好的实践是将开发、测试和生产环境通过配置区分开，django的配置主要集中在项目的settings.py文件，这里通过settings.py的配置文件将开发、测试、生产区分开，不同的环境调用不通的配置文件。

![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240803/1-django环境配置.png)


因为大部分的配置参数都是一样的，在这里我将公共的配置参数都抽到了base.py，环境配置中有差异的部分分别放到各自的配置文件中，如开发环境用的是mysql，测试环境用sqlite3，就可以将不同的配置给区分开。
测试环境是settings_test.py，这里除了数据库的配置不一样，其他都沿用基础的公共配置。settings_test.py配置如下：

```python
from .base import *

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = True

ALLOWED_HOSTS = []

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'test_db.sqlite3'),
    }
}
```

### 2、配置pytest
在Django项目根目录下，创建或编辑pytest.ini文件，来配置Pytest。
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240803/2-pytest配置.png)

pytest.ini代码如下：

```python
[pytest]
DJANGO_SETTINGS_MODULE = ishareblog.settings_test

python_files = tests.py test_*.py *_tests.py
```

DJANGO_SETTINGS_MODULE = ishareblog.settings_test  指定了pytest用到的环境配置
python_files = tests.py test_*.py *_tests.py 指定了pytest将测试以test开头的py文件中的测试用例。

## 三、编写测试用例
接下来，可以在tests.py或test_*.py文件中编写你的测试用例。由于pytest-django插件的存在，你可以像平常一样使用Django的测试机制，同时也能享受Pytest带来的便利。以下以我的ishareblog博客代码通过业务测试和接口测试来编写测试用例。

![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240803/3-pytest测试用例.png)

### 1、业务测试
我的isharebog业务相对简单，主要是测试验证业务模型模块的增删改查是否符合预期。
业务测试tests.py示例代码如下：

```python
import pytest
from django.test import TestCase
from blog.models import BlogCategory

@pytest.mark.django_db
class TestBlogCategory(TestCase):

    def setUp(self):
        self.blogcategory = BlogCategory.objects.create(id=1,title="Test Category", href='/category/1')

    def test_BogCategoryModel(self):
        blog_category = BlogCategory.objects.get(id=self.blogcategory.id)
        self.assertEqual(blog_category.title, "Test Category")
        self.assertEqual(blog_category.href, '/category/1')


@pytest.mark.django_db
def test_blog_category_create():
    blogcategory = BlogCategory.objects.create(id=1,title="Test Category", href='/category/1')
    category_count = BlogCategory.objects.count()
    assert category_count > 0, "Blog category was not created category_count=0."
    assert blogcategory.id > 0, "Blog category was not created."
    assert blogcategory.title == "Test Category", "Blog category title is wrong."
    assert blogcategory.href == "/category/1", "Blog category href is wrong."


@pytest.mark.django_db
def test_blog_category_query():
    category_count = len(BlogCategory.objects.all())
    assert category_count >= 0, "Blog category query error."


if __name__ == '__main__':
    pytest.main(["-s", "-v", "-p", "no:warnings", "--tb=short", "--html=report.html", "blog/tests.py"])
```

业务测试举了通过测试类和测试方法写的测试用例，分别对博客目录进行添加和查询编写了测试用例。
### 2、接口测试
接口是暴露给前端程序调用的，接口测试主要是测试接口正不正常，接口值是不是符合预期。

```python
import requests
import pytest

host = "http://localhost:8000"


class TestApi:
    def test_getcategory_list(self):
        url = f'{host}/api/category/'
        response = requests.get(url)
        assert response.status_code == 200, f'Expected status code 200 but got {response.status_code}'
        assert response.json() != None, f'Expected to get json response but got {response.text}'
        print(response.json())

    def test_getpost_list(self):
        url = f'{host}/api/post/list'
        response = requests.get(url)
        assert response.status_code == 200, f'Expected status code 200 but got {response.status_code}'
        assert response.json() != None, f'Expected to get json response but got {response.text}'


if __name__ == '__main__':
    pytest.main(["-s", "-v", "-p", "no:warnings", "--tb=short", "--html=report.html", "api/tests.py"])
```

接口测试部分，对获取目录的API接口和文章列表的API接口编写了测试用例。

## 四、进行测试
最后可以分别在blog目录和api目录下运行test.py 分别进行业务和接口的单元测试。
注意在进行测试之前需要执行 `python manage.py makemigrations --settings=ishareblog.settings_test` 初始化环境。
在进行api接口测试之前需要将django的应用服务启动 `python manage.py runserver 8000 --settings=ishareblog.settings_test` 启动的时候也带上测试环境的配置。
可以通过`pytest --html=report.html` 自动执行所有的单元测试，并生成可读的html的测试报告。
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240803/4-pytest执行测试.png)

pytest生成的report.html测试报告
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240803/5-pytest的report报告.png)

以上通过一个ishareblog的实际项目介绍django集成pytest进行自动化单元测试实战。
ishareblog的所有代码包括pytest的配置见 [https://gitee.com/xiejava/ishareblog](https://gitee.com/xiejava/ishareblog)

-----------------------------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>