---
title: pandas将dataframe列中的list转换为多列
id: 20231226001
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: d97c8b95
date: 2023-12-26 17:06:29
---
在应用机器学习的过程中，很大一部分工作都是在做数据的处理，一个非常常见的场景就是将一个list序列的特征数据拆成多个单独的特征数据。

比如数据集如下所示：

```python
data = [['John', '25', 'Male',[99,100,98]],
        ['Emily', '22', 'Female',[97,99,98]],
        ['Michael', '30', 'Male',[97,99,100]]]
df_data= pd.DataFrame(data,columns=['Name', 'Age', 'Gender','Score'])
df_data
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20231226/1_一维数据集.png)

很多场景是需要将类似于Score的list序列特征，拆成多个特征值如这里的语、数、外的分数。

下面通过几个实例来将dataframe列中的list序列转换为多列。
### 1、一维序列拆成多列
可以通过在列上应用Series来进行拆分。

```python
df_score=df_data['Score'].apply(pd.Series).rename(columns={0:'English',1:'Math',2:'Chinese'})
df_score
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20231226/2_一维数据集Series拆分.png)

可以看到将Score的数组，拆分成了English、Math、Chinese三个特征字段了

```python
df_data=df_data.join(df_score)
df_data
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20231226/3_一维数据集join.png)


### 2、二维序列拆成多列
用同样的思路也可以将二维序列的特征列拆成多列
如特征列是二维序列，序列里还有多个序列

```python
data = [['John', '25', 'Male',[[99,100,98],[89,70]]],
        ['Emily', '22', 'Female',[[97,99,98],[99,96]]],
        ['Michael', '30', 'Male',[[97,99,100],[87,99]]]]
df_data= pd.DataFrame(data,columns=['Name', 'Age', 'Gender','Score'])
df_data
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20231226/4_二维数据集列表里有多个列表.png)

```python
df_score=df_data['Score'].apply(pd.Series)
df_score_1=df_score[0].apply(pd.Series).rename(columns={0:'English',1:'Math',2:'Chinese'})
df_score_2=df_score[1].apply(pd.Series).rename(columns={0:'Biology',1:'Geography'})
df_score=df_score_1.join(df_score_2)
df_data=df_data.join(df_score_1).join(df_score_2)
df_data
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20231226/5_二维数据集列表里有多个列表多次Series拆分.png)

另外一种情况就是序列里面只有一个序列的二维序列，数据如下所示：

```python
data = [['John', '25', 'Male',[[99,100,98,89,70]]],
        ['Emily', '22', 'Female',[[97,99,98,99,96]]],
        ['Michael', '30', 'Male',[[97,99,100,87,99]]]]
df_data= pd.DataFrame(data,columns=['Name', 'Age', 'Gender','Score'])
df_data
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20231226/6_二维数据集列表里只有一个列表.png)

这样也可以通过多次应用Series来进行拆分，也可以先explode()再应用Series来进行拆分。

```python
df_score=df_data['Score'].apply(pd.Series)[0].apply(pd.Series).rename(columns={0:'English',1:'Math',2:'Chinese',3:'Biology',4:'Geography'})
df_score
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20231226/7_二维数据集列表里只有一个列表多次Series拆分.png)

```python
df_score=df_data['Score'].explode().apply(pd.Series).rename(columns={0:'English',1:'Math',2:'Chinese',3:'Biology',4:'Geography'})
df_score
```
![在这里插入图片描述](http://image2.ishareread.com/images/2023/20231226/8_二维数据集列表里只有一个列表先explode再Series拆分.png)

两者效果是一样的。

--------------------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>