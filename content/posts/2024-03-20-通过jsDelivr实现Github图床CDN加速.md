---
title: 通过jsDelivr实现Github图床CDN加速
id: 20240320001
tags:
  - Python
  - Hexo
categories:
  - - 技术
    - 开发
abbrlink: a0bac159
date: 2024-03-20 17:02:16
---
最近小伙伴们是否发现访问我的个人博客[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)图片显示特别快了？
我的博客的图片是放在github上的，众所周知的原因，github访问不是很快，尤其是hexo博客用github做图床经常图片刷不出来。一直想换图床，直到找到了jsDelivr，通过jsDelivr实现Github的图床CDN加速后果然速度快了很多。
jsdelivr是一个免费的公共CDN（内容分发网络）服务，它允许网站开发者将他们的代码库、JavaScript库、字体和其他资源托管在jsdelivr上，并通过jsdelivr的CDN网络进行快速分发。使用jsdelivr可以有效地减少用户下载资源的时间，提高网页加载速度，同时减轻原始服务器的负载。
jsdelivr支持多种类型的文件托管，包括JavaScript、CSS、字体、图片等。开发者可以将自己的文件上传到jsdelivr，并获取一个指向这些文件的URL。然后，他们可以在自己的网站中引用这些URL，jsdelivr会自动处理文件的缓存、分发和版本控制。
jsdelivr的优点包括：
1. 高速分发：jsdelivr拥有全球分布的CDN网络，可以确保资源在全球范围内都能快速加载。
2. 可靠性高：jsdelivr提供了高可用性和容错性，确保资源的稳定性和可靠性。
3. 易于使用：开发者可以简单地通过上传文件或使用现有的库来获取资源的URL，并在网站上引用它们。
4. 开源和免费：jsdelivr是一个开源项目，提供免费的CDN服务，对开发者非常友好。

jsdelivr的官网地址jsdelivr.com 对于gitHub的cdn加速说明如下图所示。
![gitHub的cdn加速说明](http://image2.ishareread.com/images/2024/20240320/1-jsdelivr官网的github说明.png)

对于github的图床来说，要用jsdelivr的cdn加速服务很简单
GitHub: `https://<jsDelivr加速域名>/gh/<用户>/<项目>@<版本>/<资源路径>`
![github图床加速](http://image2.ishareread.com/images/2024/20240320/2-github图床说明.png)

拿一个实例来说明：
github的图床地址：`http://image2.ishareread.com/images/2024/20240320/1-jsdelivr官网的github说明.png`

jsdelivr的cdn加速地址为：`http://image2.ishareread.com/images/2024/20240320/1-jsdelivr官网的github说明.png`

对于hexo的博客来说，就是要把原来所有在博客md文件中的**github图床地址换成jsdelivr的cdn加速地址**。
写个Python程序很容易就能完成这个工作。
**代码如下：**

```python
import os
from logutils import logging
logger=logging.getLogger(__name__)  #定义模块日志记录器


# 修改替换文件内容，进行字符串替换
def changfile(blog_md_file,old_str,new_str):
    try:
        with open(blog_md_file, "r+",encoding='utf-8') as file:  # 打开文件
            contents = file.read()  # 读取文件内容
            contents = contents.replace(old_str, new_str)  # 替换字符串
            file.seek(0)  # 定位到文件开头
            file.write(contents)  # 将修改后的内容写入文件
            file.truncate()  # 删除文件剩余部分
            logger.info(blog_md_file+'文件中的'+old_str+'已替换成'+new_str)
    except PermissionError:
        logger.error("Permission denied when trying to open the file.")
    except FileNotFoundError:
        logger.error("File not found.")
    except UnicodeDecodeError:
        logger.error("The file was not decoded correctly.")
    return None


# 读取目录解析md文件并进行字符串替换
def changfilebypath(filepath='',old_str='',new_str=''):
    try:
        files = os.listdir(filepath)
        for file in files:
            if file.find('.md') > 0:
                blog_file = os.path.join(filepath, file)
                changfile(blog_file,old_str,new_str)
    except FileNotFoundError as e:
        logger.error('请确认输入是否正确!',e)



if __name__ == '__main__':
    old_img_url = 'http://image2.ishareread.com'  # Github图床
    new_img_url = 'http://image2.ishareread.com'  # jsdelivr加速
    changfilebypath(filepath=r'D:\CloudStation\personal\xiejavablog\myhexo\myblog\source\_posts',old_str=old_img_url,new_str=new_img_url)
```

------------------------
作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)


<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注微信公众号,一起学习、成长！</center>