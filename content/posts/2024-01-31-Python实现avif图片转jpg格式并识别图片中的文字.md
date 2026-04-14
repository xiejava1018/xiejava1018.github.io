---
title: Python实现avif图片转jpg格式并识别图片中的文字
id: 20240131001
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: '25495876'
date: 2024-01-31 21:09:23
---

在做数据分析的时候有些数据是从图片上去获取的，这就需要去识别图片上的文字。Python有很多库可以很方便的实现OCR识别图片中的文字。这里介绍用EasyOCR库进行图片文字识别。easyocr是一个比较流行的库，支持超过80种语言，识别率高，速度也比较快。
## 一、图片识别文字
### 1、导包

```python
pip install easyocr
```

### 2、代码实现

```python
import easyocr
# 用easyocr识别图片并提取文字
def easyocr_pic(pic_path):
    reader = easyocr.Reader(['ch_sim', 'en'])
    results = reader.readtext(pic_path)
    ocr_result_dict = {}
    result_list = []
    for result in results:
        result_list.append(result[1])
    ocr_result_dict['orc_reslut']=result_list
    return ocr_result_dict

if __name__ == '__main__':
    orc_result = easyocr_pic(r'waf.png')
    print(orc_result)
```

### 3、运行效果
![图片OCR识别效果](http://image2.ishareread.com/images/2024/20240131/1-防火墙识别图片.png)

可以看到图片中的中文“防火墙”和"Web应用防火墙"都正确识别出来了。

> 注意：文件名和文件路径都不能有中文，否则会报错。如：如果将waf.png改成web应用防火墙.png就会报如下的错误。
>  [WARN:0@11.296] global loadsave.cpp:248 cv::findDecoder imread_('web应用防火墙.png'): can't open/read file: check file path/integrity

在进行图片识别的时候发现如果是avif格式的也会报错。如从京东商品详情页下载的图片都是avif格式的，进行识别的时候就会报错。

![在OCR识别报错](http://image2.ishareread.com/images/2024/20240131/2-avif格式ocr识别报错.png)

但是这个图片用看图软件是可以正常显示的。

![用看图软件打开图片](http://image2.ishareread.com/images/2024/20240131/3-看图软件打开图片.png)

用画图软件另存为png或jpg格式后可以用easyocr正常识别出图片中的文字。

![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240131/4-另存为jpg后可以ocr识别.png)


注意：直接将.avif的后缀名直接改成.jpg虽然可以用看图软件可以打开，但是用easyocr识别同样会报错，所以我们需要用程序来实现将avif格式的文件转成jpg或png文件格式。
## 二、avif格式图片转jpg格式
用python来实现将avif格式的文件转成jpg也很简单，但也有些注意事项。
### 1、导包

```python
pip install pillow-avif-plugin Pillow
```

### 2、代码实现

```python
import pillow_avif  #注意一定要引入pillow_avif否则会抛异常'cannot identify image file 'XXX''
from PIL import Image
import os


# 将avif文件转成jpg文件
def convert_avif_to_jpg(input_path, output_dir):
    try:
        # 打开AVIF图像
        image = Image.open(input_path)

        # 获取输入路径的文件名及其所在目录
        file_name = os.path.basename(input_path)
        # 构建输出路径
        if not os.path.exists(output_dir):
            os.makedirs(output_dir)

        output_path = os.path.join(output_dir, f"{os.path.splitext(file_name)[0]}.jpg")
        # 保存为PNG格式
        image.save(output_path, "JPEG")
    except Exception as e:
        print(e)


if __name__ == '__main__':
    # 调用函数进行转换
    convert_avif_to_jpg(r'5e595ea90b71f7ae.jpg.avif', 'avif2jpg')
```

### 3、运行效果
![在这里插入图片描述](http://image2.ishareread.com/images/2024/20240131/5-avif文件转jpg文件.png)

可以看到正常将avif文件转成了jpg格式的文件。
### 4、注意事项

> import pillow_avif  #注意一定要引入pillow_avif否则会抛异常'cannot identify image file 'XXX'' 
> 虽然代码没有用到pillow_avif但是一定要显示的用import pillow_avif否则在运行的时候会抛异常'cannot identify image file 'XXX''

## 三、Python实现avif图片转jpg格式并识别文字全部代码
**所有代码用easyocrUtil.py实现**

```python
import easyocr
import pillow_avif  #注意一定要引入pillow_avif否则会抛异常'cannot identify image file 'XXX''
from PIL import Image
import os


# 将avif文件转成jpg文件
def convert_avif_to_jpg(input_path, output_dir):
    try:
        # 打开AVIF图像
        image = Image.open(input_path)

        # 获取输入路径的文件名及其所在目录
        file_name = os.path.basename(input_path)
        # 构建输出路径
        if not os.path.exists(output_dir):
            os.makedirs(output_dir)

        output_path = os.path.join(output_dir, f"{os.path.splitext(file_name)[0]}.jpg")
        # 保存为PNG格式
        image.save(output_path, "JPEG")
    except Exception as e:
        print(e)


# 用easyocr识别图片并提取文字
def easyocr_pic(pic_path):
    reader = easyocr.Reader(['ch_sim', 'en'])
    results = reader.readtext(pic_path)
    ocr_result_dict = {}
    result_list = []
    for result in results:
        result_list.append(result[1])
    ocr_result_dict['orc_reslut']=result_list
    return ocr_result_dict


if __name__ == '__main__':
    # 调用函数进行转换
    convert_avif_to_jpg(r'5e595ea90b71f7ae.jpg.avif', 'avif2jpg')

    # 调用函数识别图片并提取文字
    orc_result = easyocr_pic(r'avif2jpg\5e595ea90b71f7ae.jpg.jpg')
    print(orc_result)
```


-----------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)
<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注微信公众号,一起学习、成长！</center>