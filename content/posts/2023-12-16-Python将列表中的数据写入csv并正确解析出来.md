---
title: Python将列表中的数据写入csv并正确解析出来
id: 20231216001
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: 9d6f73eb
date: 2023-12-16 21:18:15
---
用Python做数据处理常常会将数据写到文件中进行保存，又或将保存在文件中的数据读出来进行使用。通过Python将列表中的数据写入到csv文件中很多人都会，可以通过Python直接写文件或借助pandas很方便的实现将列表中的数据写入到csv文件中，但是写进去以后取出有些字段会有变化有些坑还是要避免。本文通过实例来介绍如何将列表中的数据写入文件如csv并正确解析出来使用。

示例数据如下：

```python
data = [['John', '25', 'Male',[99,100,98]],
        ['Emily', '22', 'Female',[97,99,98]],
        ['Michael', '30', 'Male',[97,99,100]]]
```

## 通过pandas将数据写入csv

```python
import pandas as pd
df = pd.DataFrame(data,columns=['Name', 'Age', 'Gender','Score'])
filename = 'data\pd_data.csv'
df.to_csv(filename, index=False)
df
```
![数据集](http://image2.ishareread.com/images/2023/20231216/pd_data.png)

我们对原始的数据中的分数Score字段进行求和统计总分TotalScore

```python
df['TotalScore']=df['Score'].apply(sum)
df
```
![数据集TotalScore](http://image2.ishareread.com/images/2023/20231216/pd_data_totalscore.png)

## 通过pandas将csv文件中的数据读出
用pandas将csv文件将数据读出也是非常方便的一行代码就可以搞定

```python
df_read_csv=pd.read_csv(filename)
df_read_csv
```
![数据集](http://image2.ishareread.com/images/2023/20231216/pd_data.png)

但是会发现从csv文件中读出数据后形成的dataframe数据集对数据中的分数Score字段进行求和统计总分TotalScore会报错！

```python
df_read_csv['TotalScore']=df_read_csv['Score'].app

TypeError: unsupported operand type(s) for +: 'int' and 'str'
```

原因是原数据中Score字段中的数据是list但是报错至文件读出来后这个字段变成了字符串，字符串不能求和。

**解决方案**
将字段中为字符串的值进行转换，转换成list，numpy提供了string转list的方法，当然也可以自己写。

```python
import numpy as np
def makeArray(text):
    #return [int(item) for item in text[1:-1].split(',')]  #将字串转换成列表
    return np.fromstring(text[1:-1], sep=',')  #用numpy提供的方法将字串转换成列表

df_read_csv['Score']=df_read_csv['Score'].apply(makeArray) #将Score由字符串转成列表
df_read_csv['TotalScore']=df_read_csv['Score'].apply(sum)
df_read_csv
```
![数据集TotalScore](http://image2.ishareread.com/images/2023/20231216/pd_data_totalscore.png)

可以看到这下Score字段可以正常的进行求和统计总分TotalScore了。

-------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)
 <center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注微信公众号,一起学习、成长！</center>