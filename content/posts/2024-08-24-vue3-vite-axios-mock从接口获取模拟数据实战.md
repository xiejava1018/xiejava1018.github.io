---
title: vue3+vite+axios+mock从接口获取模拟数据实战
id: 20240824001
tags:
  - Vue
categories:
  - 技术
abbrlink: e9ff8595
date: 2024-08-24 14:44:23
---
在用Vue.js开发前端应用时通常要与后端服务进行交互，例如通过API接口获取数据，在后端服务接口还没有具备之前，可以通过mock(模拟)数据来进行开发。使用mock数据可以让前端开发人员独立于后端开发人员工作，加快开发速度。在没有真实数据的情况下，mock数据可以帮助开发者更快地看到UI的呈现效果和交互逻辑。

本文通过vue3+vite+axios+mock来介绍如何实现Vue.js的前端应用从接口获取模拟数据。
## 一、安装相关组件

```bash
npm install axios -S
npm install mockjs vite-plugin-mock -D
```

 其中`axios` 是一个基于 Promise 非常强大且灵活的 HTTP 客户端，适用于 Vue.js 应用程序中的数据获取和后端交互。它可以简化 HTTP 请求的处理，并提供丰富的功能来满足不同的需求。我们用axios来实现与接口服务的http请求。

`Mock.js` 是一个用于生成随机数据的 JavaScript 库，它主要用于前端开发过程中模拟后端接口数据。Mock.js 提供了一套简洁易用的 API，可以帮助开发者快速生成符合特定规则的假数据，从而在没有后端支持的情况下进行前端开发和测试。

`vite-plugin-mock` 是一个专为 Vite 设计的插件，用于在 Vite 项目中模拟数据。它简化了使用 Mock.js 的过程，让开发者能够更加方便地管理模拟数据。

简单来说，就是mock.js提供mock数据，通过vite-plugin-mock，将管理mock发布成服务，通过axios通过http请求接口的方式获取mock数据。

安装相关组件后，在package.json中看到相关的组件信息

```javascript
package.json
{
  "name": "mocktest",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "test": "vite --mode test",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "axios": "^1.7.5",
    "vue": "^3.4.29"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.5",
    "mockjs": "^1.1.0",
    "vite": "^5.3.1",
    "vite-plugin-mock": "^3.0.2"
  }
}
```

## 二、在vite.config.js中配置vite-plugin-mock插件
 
● viteMockServe的相关配置

```javascript
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { viteMockServe } from 'vite-plugin-mock'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [
    vue(),
    // mock 数据的 dev环境
    viteMockServe({
      // supportTs: true, // 是否开启支持ts
      mockPath: 'mock', // 设置mockPath为根目录下的mock目录
      localEnabled: true, // 设置是否监视mockPath对应的文件夹内文件中的更改
      prodEnabled: false, // 设置是否启用生产环境的mock服务
      watchFiles: true, // 是否监视文件更改
      logger: true  //是否在控制台显示请求日志
    }),
  ],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    }
  }
})
```

在viteMockServe中指定了mockPath为mock也就是根目录下的mock目录，在该目录下的mock服务都会被发布成mock服务。

## 三、实现mock服务
在根目录下新建mock目录在mock目录下新建mock文件实现mock服务，如app.js、user.js

![mock目录](http://image2.ishareread.com/images/2024/20240824/1-mock目录.png)

app.js代码如下：
```javascript
export default [{
    url: '/mock/api/getApiInfo',
    method: 'get',
    response: () => {
      return {
        code: 200,
        title: 'mock api test.'
      }
    }
  },
  {
    url: '/api/category',
    type: 'get',
    response: () => {
        return {
            code: 200,
            data: [
                {
                    id: 1,
                    title: 'JAVA',
                    href: '/category/java'
                },
                {
                    id: 2,
                    title: 'SpringBoot',
                    href: '/category/SpringBoot',
                },
                {
                    id: 3,
                    title: 'MySql',
                    href: '/category/MySql'
                },
                {
                    id: 4,
                    title: '随笔',
                    href: '/category/live'
                }
            ]
        }
    }
  }
]
```

在app.js中我们并没有用mock生产数据，只是实现了mock服务放到了mock文件目录通过viteMockServe发布出来，后面可以通过axios调用获取。
user.js代码如下：

```javascript
import Mock from 'mockjs';

// 通过Mock生成模拟数据
const userdata = Mock.mock({
  'list|10': [
    {
      'id|+1': 1,
      'name': '@cname',
      'age|18-60': 1,
      'email': '@email',
    },
  ],
}); 

export default [
  {
    url: '/mock/api/getUserInfo',
    method: 'get',
    response: () => {
      return {
        code: 200,
        data: userdata
      }
    }
  },
]
```

在user.js中，我们通过Mock生成模拟的用户列表数据

## 四、调用api接口请求mock数据
### 方法一、直接使用axios 请求mock 数据

```javascript
import axios from 'axios'
```

在方法中通过axios.get()方法直接获取请求数据

```javascript
async getData() {
         await axios.get('/mock/api/getApiInfo').then(res =>{
          console.log(res.data)
          this.msg = res.data.title
        }
      )
    },
```

### 方法二、对axios进行封装统一请求mock数据
建立一个service.js对axios进行封装，让后通过service.js来统一请求mock数据，这样做的好处是在切到真实接口的时候可以更加灵活
service.js的代码如下

```javascript
import axios from 'axios'
const api_rul = ''  //mock 接口地址可以为空字符串，真实接口配置为真实的接口地址。
// create an axios instance
const service = axios.create({
    baseURL: api_rul,
    timeout: 5000 // request timeout
})

export default service
```

通过一个统一的调用接口文件请求mock数据
如mockapi.js

```javascript
import service from '@/utils/service'

export function  getCategory() {
    return service({
        url: '/api/category',
        method: 'get',
        params: {}
    })
}

export function  getUserInfo() {
    return service({
        url: '/mock/api/getUserInfo',
        method: 'get',
        params: {}
    })
}
```

在methods中进行方法的调用

```javascript
// 方法二：通过封装后的get方法获取数据
    getUserInfo()
    {
      getUserInfo().then(res =>{
          console.log(res.data)
          this.userinfo = res.data.data
        }
      )
    },
    getCategory()
    {
      getCategory().then(res =>{
          console.log(res.data)
          this.categorys = res.data.data
        }
      )
    }
```

在vue的组件中具体的调用和展示代码如下：
HelloWorld.vue

```javascript
<template>
  <h1>{{ msg }}</h1>
  <div>
    人员列表
  </div>
  <div>
    <ul>
      <li v-for="(user) in userinfo.list" :key="index">
        {{ user.id }} : {{ user.name }}  {{ user.age }}  {{ user.email }}
      </li>
    </ul>
  </div>
  <div>
    目录
  </div>
  <div>
    <ul>
      <li v-for="(category) in categorys" :key="index">
        {{ category.id }} : {{ category.title }}  {{ category.href }} 
      </li>
    </ul>
  </div>
</template>
<script >
import axios from 'axios'
import { getCategory,getUserInfo } from '../api/mockapi'

export default {
  data() {
    return {
      msg: 'Welcome to Your Vue.js App',
      userinfo: {},
      categorys: []
    };
  },
  mounted() {
    this.getData()
    this.getUserInfo()
    this.getCategory()
  },
  methods: {
    // 方法一：直接axios请求调用获取mock数据
    async getData() {
         await axios.get('/mock/api/getApiInfo').then(res =>{
          console.log(res.data)
          this.msg = res.data.title
        }
      )
    },
    // 方法二：通过封装后的get方法获取数据
    getUserInfo()
    {
      getUserInfo().then(res =>{
          console.log(res.data)
          this.userinfo = res.data.data
        }
      )
    },
    getCategory()
    {
      getCategory().then(res =>{
          console.log(res.data)
          this.categorys = res.data.data
        }
      )
    }
  }
}
</script>
```

整个工程的目录结构说明如下：

![mocktest工程目录](http://image2.ishareread.com/images/2024/20240824/2-mocktest工程目录.png)


## 五、实际运行效果

![mock效果](http://image2.ishareread.com/images/2024/20240824/3-mock效果.png)
可以看到分别用两种方式获取mock数据的效果，其中人员列表中的数据是mock生成的模拟数。

------------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注微信公众号,一起学习、成长！</center>