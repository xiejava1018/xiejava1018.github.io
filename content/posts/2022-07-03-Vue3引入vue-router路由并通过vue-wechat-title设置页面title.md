---
title: Vue3引入vue-router路由并通过vue-wechat-title设置页面title
id: 20220703002
tags:
  - Vue
categories:
  - - 技术
    - 开发
abbrlink: 5b619f34
date: 2022-07-03 16:36:53
---
对于用类似Vue前后端分离技术架构的单页应用页面之间的跳转没有非前后端分离那么来得直接，甚至连设置跳转页面的Title都要费一番周折，本文介绍Vue3引入vue-router路由并设置页面Title，通过vue-router实现页面的路由，通过vue-wechat-title来设置页面的title。

# 一、用vue-router库实现路由管理
vue-router是Vue.js官方推荐的路由管理库。它和Vue.js的核心深度集成，让构建单页应用变得轻松容易。使用Vue.js和vue-router库创建单页应用非常的简单：使用Vue.js开发，整个应用已经被拆分成了独立的组件；使用vue-router库，可以把路由映射到各个组件，并把各个组件渲染到正确的地方。下面就来介绍如何安装引入vue-router库并实现路由管理
## 1、安装vue-router库
使用如下命令安装vue-router库

```bash
npm install -save -vue-router
```

也可以通过  `npm install -save vue-router@4` 来指定版本号@4表示版本是4
安装成功后，可以在控制台看到了安装成功的信息和版本号
![控制台看到了安装成功的信息和版本号](http://image2.ishareread.com/images/2022/20220703/Vue3引入vue-router路由并通过vue-wechat-title设置页面Title/安装vue-route库.png)
除此之外也可以在工程中的package.json中看到依赖的库中包含有vue-router及版本号。
![package.json中看到依赖的库中包含有vue-router及版本号](http://image2.ishareread.com/images/2022/20220703/Vue3引入vue-router路由并通过vue-wechat-title设置页面Title/查看vue-route版本.png)

## 2、在router文件夹下创建router.js
在工程的src目录下建立router文件夹 在router文件夹下创建router.js，该文件是Vue路由管理的核心文件，所有的各组件的路由在该文件中进行配置。
参考代码如下：

```javascript
import { createRouter,createWebHistory } from "vue-router"; //引入vue-router组件
import HelloWorld from '@/components/HelloWorld';           //引入需要路由管理的页面组件HelloWorld
import siteLogin from '@/views/user/login';                 //引入需要路由管理的页面组件login
import userInfo from "@/views/user/userinfo";               //引入需要路由管理的页面组件userinfo
const router = createRouter({
    history:createWebHistory(),
    routes:[
        {
            path:'/',              //路由的路径
            name:'Home',           //路由的名称
            component:HelloWorld,  //路由的组件
        },
        {
            path:'/login',
            name:'Login',
            component:siteLogin,
        },
        {
            path:'/userinfo',
            name:'UserInfo',
            component:userInfo,
        }
    ]
})
export default router;
```

代码组织结构如下：
![代码组织结构如下](http://image2.ishareread.com/images/2022/20220703/Vue3引入vue-router路由并通过vue-wechat-title设置页面Title/代码结构.png)

## 3、在App.vue中加入路由视图
在App.vue中加入`<router-view />`
App.vue示例代码如下：
```javascript
<template>
  <div id="app">
    <router-view />
  </div>
</template>
<script>
export default {
  name: 'App',
}
</script>
<style>
</style>
```

## 4、在项目的main.js中引入路由
参考代码如下：

```javascript
import { createApp } from 'vue';
import App from './App.vue';
import router from "@/router/router";  //引入路由，会去找router下的router.js的配置文件
createApp(App).use(router).mount('#app')  //创建应用的时候应用路由
```

## 5、验证效果
为了显示更清楚，将默认创建的src\components\HelloWorld.vue内容稍加调整

```javascript
<template>
  <div >
    第一个路由组件Home
    <p>{{ name }}</p>
  </div>
</template>

<script>
export default {
  name: 'HelloWorld',
  data() {
    return {
      name:"Hello World!"
    }
  }
}
</script>
<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
</style>
```

如果上面的步骤没有遗漏，在终端输入 npm run serve 将前端服务启动起来，在浏览器访问localhost:8080可以看到如下页面：

![localhost:8080](http://image2.ishareread.com/images/2022/20220703/Vue3引入vue-router路由并通过vue-wechat-title设置页面Title/home路由.png)

访问localhost:8080/login

![访问localhost:8080/login](http://image2.ishareread.com/images/2022/20220703/Vue3引入vue-router路由并通过vue-wechat-title设置页面Title/login.png)

访问localhost:8080/userinfo

![访问localhost:8080/userinfo](http://image2.ishareread.com/images/2022/20220703/Vue3引入vue-router路由并通过vue-wechat-title设置页面Title/userinfo.png)
可以看到访问不同的URL路由到了不同的Vue页面，上述login.vue和userinfo.vue示例代码没有给出，大家可以自行随便实现。

# 二、用vue-wechat-title实现页面title的设置
在上面实现了不同页面的路由管理，但是访问不同的URL看到的页面title所有的页面都是一样的，如何设置不同页面不同的页面Title呢？比较方便的做法是用vue-wechat-title来实现。
同样首先要安装vue-wechat-title库
## 1、安装vue-wechat-title库
使用如下命令安装vue-wechat-title库

```bash
npm install vue-wechat-title -save 
```

安装完成后在工程中的package.json中看到依赖的库中包含有vue-wechat-title及版本号
![package.json中看到依赖的库中包含有vue-wechat-title及版本号](http://image2.ishareread.com/images/2022/20220703/Vue3引入vue-router路由并通过vue-wechat-title设置页面Title/vue-wecha-title.png)

## 2、在router文件夹下的router.js中增加title的配置

```javascript
import { createRouter,createWebHistory } from "vue-router"; //引入vue-router组件
import HelloWorld from '@/components/HelloWorld';           //引入需要路由管理的页面组件HelloWorld
import siteLogin from '@/views/user/login';                 //引入需要路由管理的页面组件login
import userInfo from "@/views/user/userinfo";               //引入需要路由管理的页面组件userinfo
const router = createRouter({
    history:createWebHistory(),
    routes:[
        {
            path:'/',              //路由的路径
            name:'Home',           //路由的名称
            meta:{
                title: '首页'       //title配置
            },
            component:HelloWorld,  //路由的组件
        },
        {
            path:'/login',
            name:'Login',
            meta:{
                title:'登录'
            },
            component:siteLogin,
        },
        {
            path:'/userinfo',
            name:'UserInfo',
            meta:{
                title: '用户信息'
            },
            component:userInfo,
        }
    ]
})
export default router;
```

主要是在路由配置时设置了`meta:{title:'xxxx'}`如下图：

![router.js中增加title的配置](http://image2.ishareread.com/images/2022/20220703/Vue3引入vue-router路由并通过vue-wechat-title设置页面Title/加入title.png)
## 3、在App.vue页面中使用
App.vue代码如下：

```javascript
<template>
  <div id="app"  v-wechat-title="$route.meta.title">
    <router-view />
  </div>
</template>

<script>
export default {
  name: 'App',
}
</script>

<style>
</style>
```

主要是在`<div id="app"  v-wechat-title="$route.meta.title">` 加入了`v-wechat-title="$route.meta.title"`

## 4、在main.js中引用vue-wechat-title
在main.js中引用vue-wechat-title的时候有个坑，如果按照一般的引用会报错
mian.js代码示例如下：

```javascript
import { createApp } from 'vue';
import App from './App.vue';
import router from "@/router/router";  //引入路由，会去找router下的router.js的配置文件
import VueWechatTitle from 'vue-wechat-title'; //引入VueWechatTitle
createApp(App).use(router,VueWechatTitle).mount('#app')  //创建应用的时候应用路由
```

在终端输入 npm run serve 将前端服务启动起来会报错！
<font color=Red>Uncaught TypeError: Cannot read properties of undefined (reading 'deep')</font>

原因是在挂载app示例前，vue-wechat-title还没有加载好，一定要先应用再挂载app
将createApp(App).use(router,VueWechatTitle).mount('#app')删除或注释掉。改用

```javascript
const app=createApp(App);
app.use(VueWechatTitle);
app.use(router);
app.mount('#app')
```

main.js的参考示例代码如下：

```javascript
import { createApp } from 'vue';
import App from './App.vue';
import router from "@/router/router";  //引入路由，会去找router下的router.js的配置文件
import VueWechatTitle from 'vue-wechat-title'; //引入VueWechatTitle
//createApp(App).use(router,VueWechatTitle).mount('#app')  //指令定义在 mount('#app')之后，导致自定义指令未挂载到，会报错
const app=createApp(App);
app.use(VueWechatTitle);
app.use(router);
app.mount('#app')
```

## 5、验证效果
在终端输入 npm run serve 将前端服务启动起来
看到访问不同的URL会显示不同的title
http://localhost:8080/

![http://localhost:8080/的title](http://image2.ishareread.com/images/2022/20220703/Vue3引入vue-router路由并通过vue-wechat-title设置页面Title/home路由.png)
http://localhost:8080/login

![login的title登录](http://image2.ishareread.com/images/2022/20220703/Vue3引入vue-router路由并通过vue-wechat-title设置页面Title/login的title.png)

http://localhost:8080/userinfo

![userinfo的title用户信息](http://image2.ishareread.com/images/2022/20220703/Vue3引入vue-router路由并通过vue-wechat-title设置页面Title/userinfo的title.png)

本文通过以上实例实现了Vue3引入vue-router路由并设置页面Title，通过vue-router实现页面的路由，通过vue-wechat-title来设置页面的title都还比较方便。

--------------------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 
<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>
