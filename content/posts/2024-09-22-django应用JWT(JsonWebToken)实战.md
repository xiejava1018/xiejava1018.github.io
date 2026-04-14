---
title: django应用JWT(JsonWebToken)实战
id: 20240922001
tags:
  - Python
  - Django
categories:
  - 技术
abbrlink: ca8e72
date: 2024-09-22 12:17:14
---
在前后端分离的项目中，前后端进行身份验证通常用JWT来进行，JWT 提供了一个理想的认证解决方案，用来保护 RESTful API，确保只有经过认证的用户才能访问受保护的资源。基于前端框架（如React, Angular, Vue.js）的单页面应用 (SPA)，开发者通过使用 JWT可以获得一种简单、安全、高效的方式来处理用户认证和授权的问题。本文通过django项目的实战来说明如何应用和使用JWT。

## 一、什么是JWT
JWT（JSON Web Token）是一种开放标准（RFC 7519），用于在网络各方之间以安全且紧凑的形式传输信息。JWT 是一个小型的凭证，通常用于身份验证和授权场景。JWT 由三部分组成：头部 (Header)、负载 (Payload) 和签名 (Signature)。
JWT信息由3段构成，它们之间用圆点“.”连接，格式如下：

```python
aaaaaa.bbbbbb.cccccc
```

一个典型的JWT如下所示：

```python
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbl90eXBlIjoiYWNjZXNzIiwiZXhwIjoxNzI2OTcyMDc2LCJpYXQiOjE3MjY5NzAyNzYsImp0aSI6IjMyMTFiZjdmZDlhZTRmNTBhMDNmOGM2NjcwNDM2NjFiIiwidXNlcl9pZCI6Mn0.Ej6US4Uk-sSNm9P8kTU_cDAzBpO4I-BLhPstp5sG00Q
```

- 头部 (Header)：包含了关于 JWT 类型的信息以及所使用的签名算法。
- 负载 (Payload)：是 JWT 的主体部分，包含了实际需要声明的数据。这些数据通常包括用户ID、用户名、角色等信息。
- 签名 (Signature)：用于验证 JWT 的发送者就是它声称的发送者，同时也确保了 JWT 在传输过程中没有被篡改。

## 二、为什么使用JWT
使用 JWT 的原因主要有以下几点：
- 安全性：JWT 通过签名来保证数据的完整性和防篡改性。如果有人试图修改 JWT 内容，签名会失效，接收方可以检测到这一行为。
- 无状态性：JWT 是自包含的，这意味着不需要在服务器上保存会话状态。每个 JWT 都包含了所有必要的信息，从而减少了对服务器端存储的需求。
- 跨域支持：JWT 可以轻松地在不同的域之间共享，这使得它非常适合微服务架构和分布式系统。
- 性能提升：由于 JWT 是自包含的，所以服务器可以快速地验证 JWT，而无需查询数据库来获取用户信息，这提高了应用的响应速度。
- 易于缓存和扩展：JWT 可以被缓存，并且因为它们是无状态的，所以可以很容易地扩展到多个服务器，而无需担心会话复制问题。
- CSRF 防护：使用 JWT 可以帮助缓解跨站请求伪造（CSRF）攻击的风险，因为 JWT 不依赖于 cookie，也就不会随同 HTTP 请求自动发送。

总的来说，JWT 提供了一种高效、安全的方式来处理用户认证和授权，尤其是在需要跨域操作或构建无状态服务的情况下。

## 三、在django项目中如何应用JWT
JWT（JSON Web Token）是一种用于在网络应用中安全地传输信息的令牌。它通常用于身份验证和授权，特别是在单页应用（SPA）和API服务中。在Django中应用JWT，可以使用 djangorestframework-simplejwt。
### 1、安装djangorestframework-simplejwt库：

```python
pip install djangorestframework-simplejwt
```

### 2、在settings.py中配置JWT认证：
在INSTALLED_APPS中添加rest_framework_simplejwt的应用

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
    'rest_framework_simplejwt',  # 添加 simplejwt 应用
    'django_filters',  # 注册条件查询
    'mdeditor',  # 注册markdown的应用
    'drf_yasg2',  # 接口文档
]
```

添加REST_FRAMEWORK的默认认证类为JWT认证

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}
```

添加SIMPLE_JWT的相关配置

```python
# JWT 相关设置
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=30),  # 访问令牌的有效时间
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),  # 刷新令牌的有效时间
    'ROTATE_REFRESH_TOKENS': False,  # 是否允许刷新令牌循环
    'BLACKLIST_AFTER_ROTATION': True,  # 刷新令牌后是否加入黑名单
    'UPDATE_LAST_LOGIN': False,  # 登录时是否更新最后登录时间

    'ALGORITHM': 'HS256',  # 签名算法
    'SIGNING_KEY': SECRET_KEY,  # 签名密钥
    'VERIFYING_KEY': None,  # 验证密钥
    'AUDIENCE': None,  # 观众
    'ISSUER': None,  # 发行人
    'JWK_URL': None,  # JWK URL
    'LEEWAY': 0,  # 宽限期

    'AUTH_HEADER_TYPES': ('Bearer',),  # 授权头类型
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',  # 授权头名称
    'USER_ID_FIELD': 'id',  # 用户 ID 字段
    'USER_ID_CLAIM': 'user_id',  # 用户 ID 声明
    'USER_AUTHENTICATION_RULE': 'rest_framework_simplejwt.authentication.default_user_authentication_rule',

    'AUTH_TOKEN_CLASSES': ('rest_framework_simplejwt.tokens.AccessToken',),  # 认证令牌类
    'TOKEN_TYPE_CLAIM': 'token_type',  # 令牌类型声明
    'TOKEN_USER_CLASS': 'rest_framework_simplejwt.models.TokenUser',

    'SLIDING_TOKEN_REFRESH_EXP_CLAIM': 'refresh_exp',  # 滑动令牌刷新过期声明
    'SLIDING_TOKEN_LIFETIME': timedelta(minutes=5),  # 滑动令牌有效时间
    'SLIDING_TOKEN_REFRESH_LIFETIME': timedelta(days=1),  # 滑动令牌刷新有效时间
}
```

### 3、在urls.py中配置JWT的获取和刷新路由：

```python
from django.urls import path
from rest_framework_simplejwt.views import (TokenObtainPairView, TokenRefreshView)
urlpatterns = [
    path('token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    # 其他路由...
]
```

4、在视图中需要认证的地方使用JWT认证
如下modelviweset中使用，对于查询方法如list，retrieve不做鉴权，对于其他方法需要鉴权。

```python
def get_permissions(self):
    """
    Instantiates and returns the list of permissions that this view requires.
    """
    if self.action in ['list', 'retrieve']:
        # 对于list方法，返回AllowAny权限类，表示不需要鉴权
        permission_classes = [AllowAny, ]
    else:
        # 对于其他方法，返回IsAuthenticated权限类，表示需要用户已认证
        permission_classes = [IsAuthenticated, ]
    return [permission() for permission in permission_classes]
```

## 四、JWT如何使用
通过上面的应用后，使用接口调用遇到需要鉴权的会提示需要认证。
如当我们调用删除接口时，如果没有获得鉴权，接口会返回需要认证的信息。
![接口调用需要认证](http://image2.ishareread.com/images/2024/20240922/1-需要鉴权.png)

那如何通过JWT进行认证呢？
![JWT进行认证过程](http://image2.ishareread.com/images/2024/20240922/2-JWT的验证过程.png)


下面通过postman来应用JWT的使用过程。
### 1、调用生成JWT的接口获取JWT
![调用生成JWT的接口获取JWT](http://image2.ishareread.com/images/2024/20240922/3-调用生成JWT的接口获取JWT.png)

### 2、客户端保存JWT在调用接口时带上获取的JWT
![调用接口时带上获取的JWT](http://image2.ishareread.com/images/2024/20240922/4-请求接口带上JWT.png)


至此，本文介绍了什么时JWT，为什么要使用JWT，通过django实现JWT，介绍了JWT的使用流程，最后以一个具体API接口实例的调用来说明JWT如何使用。后续将介绍VUE从前端登录获取JWT到JWT认证的实例。

-----------------------------

博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)
<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>