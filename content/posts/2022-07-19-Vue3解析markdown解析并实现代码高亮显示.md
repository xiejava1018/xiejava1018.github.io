---
title: Vue3解析markdown解析并实现代码高亮显示
id: 20220719001
tags:
  - Vue
categories:
  - - 技术
    - 开发
abbrlink: e37e6718
date: 2022-07-19 11:07:00
---
Vue实现博客前端，需要实现markdown的解析，如果有代码则需要实现代码的高亮。
Vue的markdown解析库有很多，如markdown-it、vue-markdown-loader、marked、vue-markdown等。这些库都大同小异。这里选用的是marked，代码高亮的库选用的是highlight.js。

具体实现步骤如下：
## 一、安装依赖库
在vue项目下打开命令窗口，并输入以下命令

```javascript
npm install marked -save    // marked 用于将markdown转换成html
npm install highlight.js -save   //用于代码高亮显示
```

命令执行完后可以在控制台或package.json文件中看到有安装的版本号
![package.json文件中看到有安装的版本号](http://image2.ishareread.com/images/2022/20220719/package.png)

## 二、在main.js文件中引入highlight.js及样式并创建一个自定义的全局指令

```javascript
import hljs from 'highlight.js';
import 'highlight.js/styles/atom-one-dark.css' //样式

//创建v-highlight全局指令
Vue.directive('highlight',function (el) {
  let blocks = el.querySelectorAll('pre code');
  blocks.forEach((block)=>{
    hljs.highlightBlock(block)
  })
})
```

这样就可以在vue组件中使用v-highlight引用代码高亮的方法了。

## 三、在Vue组件中应用marked解析及实现代码高亮
代码示例如下：

```html
 <!-- 正文输出 -->
<div class="entry-content">
  <div v-highlight v-html="article"  id="content"></div>
</div>

```

```javascript
<script>
    // 将marked 引入
  import { marked }from 'marked';
    export default {
        name: 'articles',
        data(){
          return{
              article:''
          }
        },
        methods: {
          getPostDetail() {
            console.log('getPostDetail()'+this.id)
            fetchPostDetail(this.id).then(res => {
               this.postdetail=res.data
               // 调用marked()方法，将markdown转换成html
               this.article= marked(this.postdetail.content);
               console.log(res.data)
              }).catch(err => {
                  console.log(err)
              })

          },
        created() {
          //调用获取文章内容的接口方法
          this.getPostDetail()
        },
    }
 </script>
```


## 四、显示效果
markdown解析及代码高亮显示效果
![在这里插入图片描述](http://image2.ishareread.com/images/2022/20220719/markdown解析效果.png)

示例中引用的样式是 `import 'highlight.js/styles/atom-one-dark.css'  `
实际highlight.js/styles中提供了很多样式，可以根据自己的喜好选用。

![代码高亮样式](http://image2.ishareread.com/images/2022/20220719/高亮样式.png)


-----------
博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>
