---
title: Python使用BeautifulSoup4修改网页内容实战
id: 20220518001
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: 1ca32c3a
date: 2022-05-18 21:45:28
---
最近有个小项目，需要爬取页面上相应的资源数据后，保存到本地，然后将原始的HTML源文件保存下来，对HTML页面的内容进行修改将某些标签整个给替换掉。

对于这类需要对HTML进行操作的需求，最方便的莫过于**BeautifulSoup4**的库了。

样例的HTML代码如下：

```html
<html>
<body>
    <a class="videoslide" href="http://www.test.com/wp-content/uploads/1020/1381824922.JPG">
       <img src="http://www.test.com/wp-content/uploads/1020/1381824922_zy_compress.JPG" data-zy-media-id="zy_location_201310151613422786"/>
    </a>
    <a href="http://www.test.com/wp-content/uploads/1020/第一张_1381824798.JPG">
       <img data-zy-media-id="zy_image_201310151613169945" src="http://www.test.com/wp-content/uploads/1020/第一张_1381824798_zy_compress.JPG"/></a>
    <a href="http://www.test.com/wp-content/uploads/1020/第二张_1381824796.jpg">
       <img data-zy-media-id="zy_image_201310151613163009" src="http://www.test.com/wp-content/uploads/1020/第二张_1381824796_zy_compress.jpg"/>
    </a>
    <a href="http://www.test.com/wp-content/uploads/1020/第三张.jpg">
       <img data-zy-media-id="zy_image_201312311838584446" src="http://www.test.com/wp-content/uploads/1020/第三张_zy_compress.jpg"/>
    </a>
</body>
</html>
```

这里主要包括了`<a >`标签，`<a >`标签里面嵌入了`<img >`标签，其中有`<a class="videoslide">`的标识该标签实际是可以播放动画的。需要根据`class="videoslide"` 来判断将整个`<a >`标签换成播放器的`<video >`标签，将没有`class="videoslide"` 的`<a >`标签换成`<figure>`标签。

也就是将带有的`<a class="videoslide" ...><img ... /></a>`标签换成 

```html
<div class="video">
<video controls width="100%" poster="视频链接的图片地址.jpg">
    <source src="视频文件的静态地址.mp4" type="video/mp4" />
    您的浏览器不支持H5视频，请使用Chrome/Firefox/Edge浏览器。
</video>
</div>
```

将`<a ....><img .../></a>`标签换成

```html
<figure>
    < img src="图片地址_compressed.jpg" data-zy-media-id="图片地址.jpg">
    <figcaption>文字说明（如果有）</figcaption>
</figure>
```

这里通过BeautifulSoup4 的select()方法找到标签，通过get()方法获取标签及标签属性值，通过replaceWith来替换标签，具体代码如下：
首先安装BeautifulSoup4的库，BeautifulSoup4库依赖于lxml库，所以也需要安装lxml库。

```python
pip install bs4
pip install lxml
```

具体代码实现如下：

```python
import os
from bs4 import BeautifulSoup
htmlstr='<html><body>' \
        '<a class="videoslide" href="http://www.test.com/wp-content/uploads/1020/1381824922.JPG">' \
        '<img src="http://www.test.com/wp-content/uploads/1020/1381824922_zy_compress.JPG" data-zy-media-id="zy_location_201310151613422786"/></a>' \
        '<a href="http://www.test.com/wp-content/uploads/1020/第一张_1381824798.JPG">' \
        '<img data-zy-media-id="zy_image_201310151613169945" src="http://www.test.com/wp-content/uploads/1020/第一张_1381824798_zy_compress.JPG"/></a>' \
        '<a href="http://www.test.com/wp-content/uploads/1020/第二张_1381824796.jpg">' \
        '<img data-zy-media-id="zy_image_201310151613163009" src="http://www.test.com/wp-content/uploads/1020/第二张_1381824796_zy_compress.jpg"/></a>' \
        '<a href="http://www.test.com/wp-content/uploads/1020/第三张.jpg">' \
        '<img data-zy-media-id="zy_image_201312311838584446" src="http://www.test.com/wp-content/uploads/1020/第三张_zy_compress.jpg"/></a>' \
        '</body></html>'

def procHtml(htmlstr):
    soup = BeautifulSoup(htmlstr, 'lxml')
    a_tags=soup.select('a')
    for a_tag in a_tags:
        a_tag_src = a_tag.get('href')
        a_tag_filename = os.path.basename(a_tag_src)
        a_tag_path = os.path.join('src', a_tag_filename)
        a_tag['href']=a_tag_path
        next_tag=a_tag.next
        #判断是视频还是图片，如果a标签带了class="videoslide" 是视频否则是图片
        if a_tag.get('class') and 'videoslide'==a_tag.get('class')[0]:
            # 处理视频文件
            media_id = next_tag.get('data-zy-media-id')
            if media_id:
                media_url = 'http://www.test.com/travel/show_media/' + str(media_id)+'.mp4'
                media_filename = os.path.basename(media_url)
                media_path = os.path.join('src', media_filename)
                # 将div.video标签替换a标签
                video_html = '<div class=\"video\"><video controls width = \"100%\" poster = \"' + a_tag_path + '\" ><source src = \"' + media_path + '\" type = \"video/mp4\" /> 您的浏览器不支持H5视频，请使用Chrome / Firefox / Edge浏览器。 </video></div>'
                video_soup = BeautifulSoup(video_html, 'lxml')
                a_tag.replaceWith(video_soup.div)
        else:
            #获取图片信息
            if 'img'==next_tag.name:
                img_src=next_tag.get('src')
                # 判断是否路径是否为本地资源 data:image和file:
                if img_src.find('data:image') == -1 and img_src.find('file:') == -1:
                    img_filename = os.path.basename(img_src)
                    img_path = os.path.join('src', img_filename)
                    # 将<figure><img>标签替换a标签
                    figcaption=''
                    figure_html='<figure><img src=\"'+img_path+'\" data-zy-media-id=\"'+a_tag_path+'\"><figcaption>'+figcaption+'</figcaption></figure>'
                    figure_soup = BeautifulSoup(figure_html, 'lxml')
                    a_tag.replaceWith(figure_soup.figure)
    html_content = soup.contents[0]
    return html_content

if __name__ == '__main__':
    pro_html_str=procHtml(htmlstr)
    print(pro_html_str)
```



结果:
```html
<html>
<body>
<div class="video">
<video controls="" poster="src\1381824922.JPG" width="100%">
<source src="src\zy_location_201310151613422786.mp4" type="video/mp4"/> 您的浏览器不支持H5视频，请使用Chrome / Firefox / Edge浏览器。 
</video>
</div>
<figure>
<img data-zy-media-id="src\第一张_1381824798.JPG" src="src\第一张_1381824798_zy_compress.JPG"/>
<figcaption></figcaption>
</figure>
<figure>
<img data-zy-media-id="src\第二张_1381824796.jpg" src="src\第二张_1381824796_zy_compress.jpg"/>
<figcaption></figcaption></figure>
<figure>
<img data-zy-media-id="src\第三张.jpg" src="src\第三张_zy_compress.jpg"/>
<figcaption></figcaption>
</figure>
</body>
</html>
```

-----
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)
