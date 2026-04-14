---
title: vue3实现登录获取token并自动刷新token进行JWT认证
id: 20241010001
tags:
  - Vue
categories:
  - 技术
abbrlink: faae10b6
date: 2024-10-10 14:49:20
---
在[《django应用JWT(JSON Web Token)实战》](http://xiejava.ishareread.com/posts/ca8e72/)介绍了如何通过django实现JWT，并以一个具体API接口实例的调用来说明JWT如何使用。本文介绍如何通过vue3的前端应用来使用JWT认证调用后端的API接口，实现一下的登录认证获取JWT进行接口认证。


![账号密码验证jwt流程](http://image2.ishareread.com/images/2024/20241010/1-账号密码验证jwt流程.jpg)

## 一、账号密码登录获取JWT
通过Login.vue实现登录的用户名、密码表单信息收集，调用getToken()方法进行鉴权验证并获取jwt的token。
Login.vue

```html
<template>
    <div class="body">
        <el-form :model="form" :rules="rules" ref="loginForm" class="loginContainer">
            <h3 class="loginTitle">
            欢迎登录
            </h3>
            <el-form-item label="用户名" prop="username" :label-width="formLabelWidth"> 
                <el-input type="text" v-model="form.username" placeholder="请输入用户名"></el-input>
            </el-form-item>
            <el-form-item label="密码"  prop="password" :label-width="formLabelWidth"> 
                <el-input type="password" v-model="form.password" placeholder="请输入密码"></el-input>
            </el-form-item>
            <el-form-item :label-width="formLabelWidth"> 
                <el-button type="primary" :plain="true" @click="submitForm('form')">登录</el-button>
            </el-form-item>
        </el-form>
    </div>
</template>

<script>
import { useUserStore } from '@/stores/user'
import { ElMessage } from 'element-plus';
import {getToken} from '../api/user'
export default {
  data() {
    return {
      form: {
        username: '',
        password: '',
        err_username: "",
        err_password: "",
      },
      rules: {
        username: [
          { required: true, message: '请输入用户名', trigger: 'blur' }
        ],
        password: [
          { required: true, min:6, message: '请输入密码', trigger: 'blur' }
        ]
      },     
      formLabelWidth: '120px'
    };
  },
  methods: {
    submitForm(formName) {
        if (this.$refs.loginForm) {
            this.$refs.loginForm.validate(valid => {
            if (valid) {
                // 提交表单逻辑
                console.log('提交成功:', this.form);
                this.login();
            } else {
                console.log('验证失败');
                ElMessage.error('验证失败，请检查您的输入！');
            }
            });
      } else {
        console.error('表单未找到');
      }
    },
    login() {
      var that = this;
      this.message = "";
      // 用户名密码鉴权获取jwt的token
      getToken({
        'username': this.form.username,
        'password': this.form.password,
      }).then((Response) => {
          console.log(Response);
          if (Response && Response.access) {
            // //保存数据到本地存储
            this.username= that.form.username;
            useUserStore().login(this.username,Response.access,Response.refresh)
            this.username = "";
            this.password = "";
            this.$router.push({name:"home"}); //跳转到首页
          }
        })
        .catch(function (error) {
          console.log(error);
          if ("username" in error) {
            that.err_username = error.username[0];
          } else if ("password" in error) {
            that.err_password = error.password[0];
          } else {
            ElMessage.error('登录失败！');
          }
        });
    },

  }
};
</script>

<style scoped>
  .loginContainer{
        border-radius: 15px;
        background-clip: padding-box;
        text-align: left;
        margin: auto;
        margin-top: 180px;
        width: 450px;
        padding: 15px 35px 15px 35px;
        background: aliceblue;
        border:1px solid blueviolet;
        box-shadow: 0 0 25px #f885ff;
    }
    .loginTitle{
        margin: 0px auto 48px auto;
        text-align: center;
        font-size: 40px;
    }
    .loginRemember{
        text-align: left;
        margin: 0px 0px 15px 0px;
    }
    .loginbody{
        width: 100vw;
        height: 100vh;
        background-size:100%;
        overflow: hidden;
    }
</style>
```

在登录时调用getToken()方法获取jwt的token 
getToken的封装方法如下：

```javascript
import request from '@/utils/request'

export function getToken(data) {
    return request({
      url: 'token/',
      method: 'post',
      data
    })
  }
```

通过用户名和密码鉴权可以获得JWT的token，接口会返回access的token和refresh的token，需要将这两个token保存下来，access的token用来进行API接口的jwt认证，refresh的token用来刷新失效的access的token。

## 二、将JWT保存至本地
通过pinia将token保存至浏览器的本地存储，以便于后面请求API时带上访问的token 

```javascript
import { defineStore } from 'pinia'
import { refreshToken } from '../api/user'

export const useUserStore = defineStore('user', {
    persist: {
        enabled: true, //开启数据持久化
        strategies: [
            {
                key: "userState", //给一个要保存的名称
                storage: localStorage, //sessionStorage / localStorage 存储方式
            },
        ],
    },
    state: () => ({
        isLoggedIn: false,
        username: '',
        jwtAccessToken: null,
        jwtRefreshToken: null,
    }),
    actions: {
        login(username, accessToken,refreshToken) {
            this.username = username
            this.isLoggedIn = true
            this.setToken(accessToken, refreshToken)
        },
        logout() {
            this.username = ''
            this.jwtAccessToken = null
            this.isLoggedIn = false
        },
        setToken(accessToken, refreshToken) {
            this.jwtAccessToken = accessToken
            this.jwtRefreshToken = refreshToken
        },
        refreshToken() {
            return new Promise((resolve, reject) => {
                refreshToken({"refresh":this.jwtRefreshToken}).then((response) => {
                    this.setToken(response.access, this.jwtRefreshToken)
                    resolve(response.access)
                    console.log('return refreshToken-----------'+response.access)
                }).catch((error) => {
                    reject(error)
                })
            })
        }

    },
    getters: {
        getIsLoggedIn: (state) => state.isLoggedIn,
        getUsername: (state) => state.username,
        getUserAccessToken: (state) => state.jwtAccessToken,
        getRefreshToken: (state) => state.jwtRefreshToken,
    }
})
```

在登录的Login.vue组件中调用`useUserStore().login(this.username,Response.access,Response.refresh)`将用户名、access的token、refresh的token保存至浏览器的本地存储。

## 三、请求API带上JWT
将axios的调用封装成request.js在调用API接口时带上JWT

```javascript
import axios from 'axios'
import Router from '@/components/tools/Router'
import { useUserStore } from '@/stores/user'
import { ElMessage } from 'element-plus'
import { refreshToken } from '../api/user'

const api_rul = import.meta.env.VITE_APP_API_URL

// create an axios instance
const service = axios.create({
    baseURL: api_rul,
    timeout: 5000, // request timeout
})

// request interceptor
service.interceptors.request.use(
    config => {
        // do something before request is sent
        const { url } = config
        // 指定页面访问需要JWT认证。
        if (url.indexOf('/login')!== -1) {
            return config
        }
        let jwt = useUserStore().getUserAccessToken
        config.headers.Authorization = `Bearer ${jwt}`
        return config
    },
    error => {
        // do something with request error
        console.log(error) // for debug
        return Promise.reject(error)
    }
)

export default service
```

主要是在请求头重带着jwt的信息

```javascript
let jwt = useUserStore().getUserAccessToken
config.headers.Authorization = `Bearer ${jwt}`
```

## 四、在token失效时自动重新获取token
前面提到JWT基于安全考虑有两个token，一个是access token ,一个是refresh token 。access token的失效时间较短，可以有效降低泄露而造成的影响，两个token的区别和作用如下：

|  |	access token| 	refresh token|
|---|---|---|
|有效时间|	较短(如半小时)|	较长(如一天)|
|作用|	鉴权验证	|重新获取access token|
|什么时候使用|	每次接口鉴权验证时|access token失效时使用|

使用refresh token的逻辑如下：
![重新获取token流程](http://image2.ishareread.com/images/2024/20241010/2-重新获取token.jpg)


以下通过拦截器实现token失效后重新获取access token

```javascript
import axios from 'axios'
import Router from '@/components/tools/Router'
import { useUserStore } from '@/stores/user'
import { ElMessage } from 'element-plus'
import { refreshToken } from '../api/user'

const api_rul = import.meta.env.VITE_APP_API_URL

// create an axios instance
const service = axios.create({
    baseURL: api_rul,
    timeout: 5000, // request timeout
})

// request interceptor
service.interceptors.request.use(
    config => {
        // do something before request is sent
        const { url } = config
        // 指定页面访问需要JWT认证。
        if (url.indexOf('/login')!== -1) {
            return config
        }
        let jwt = useUserStore().getUserAccessToken
        config.headers.Authorization = `Bearer ${jwt}`
        return config
    },
    error => {
        // do something with request error
        console.log(error) // for debug
        return Promise.reject(error)
    }
)

// response interceptor
service.interceptors.response.use(
 
    response => {
        const res = response.data
        return res
    },
    async error => {
        console.log('err' + error) // for debug
        const originalRequest = error.config;
        // 授权验证失败
        if (error.response.status === 401 && originalRequest._retry!== true) {
            originalRequest._retry = true;
            // 刷新token
            let jwtRefreshToken=useUserStore().getRefreshToken
            await refreshToken({"refresh":jwtRefreshToken}).then((response) => {
                // 刷新token成功，重新请求
                let jwtToken=response.access
                useUserStore().setToken(jwtToken, jwtRefreshToken)
                console.log('return refreshToken-----------'+response.access)
                originalRequest.headers.Authorization = `Bearer ${jwtToken}`
                return service(originalRequest)                
            }).catch((error) => {
                // 刷新token失败，跳转到登录页面
                ElMessage.error('请重新登录！')
                Router.push({name:'login'})
            })

        }
        // 内部错误
        if (error.response.status === 500) {
            let errormsg=error.response.data.msg
            ElMessage.error('服务器内部错误！'+errormsg)
        }
        if (error.response.status === 400)
        {
            ElMessage.error('错误的请求！')
        }
        return Promise.reject(error)
    }
)

export default service
```

在判断`error.response.status === 401`时调用refreshToken重新获取jwttoken进行接口的调用。

------------------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注微信公众号,一起学习、成长！</center>