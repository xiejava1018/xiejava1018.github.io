---
title: 通过Git Pages+Hexo搭建自己的博客
id: 20200212001
tags:
  - Hexo
categories:
  - [技术,开发]
abbrlink: 79ebd763
date: 2020-02-12 15:41:23
---
# 一、申请并配置Github Pages
## step1 在github上创建一个git库
用github账号登录https://github.com/ ，如没有github账号则申请一个github账号。登录后点击“New repository”新建一个名为username.github.io（username是你的github用户名）如我的是：xiejava1018.github.io ，如果库名以及存在则会报库名已经存在的错误。
![新建库](http://image2.ishareread.com/images/20200212/blogimage/1.png)
## step2 绑定自己的域名（如果没有自己的域名也可以不绑）
访问刚申请的git库，点击Settings
![Settings](http://image2.ishareread.com/images/20200212/blogimage/2.png)
如果库名不是username.github.io（username是你的github用户名）在这里可以修改成username.github.io
![修改库名](http://image2.ishareread.com/images/20200212/blogimage/3.png)
拖到下面可以看到GitHub Pages的信息，如果不绑定自己的域名实际可以通过https://username.github.io/来访问你的站点了。
![站点地址](http://image2.ishareread.com/images/20200212/blogimage/4.png)
如果有申请自己的域名，可以将域名解析到你的GithubPages username.github.io 如我的是xiejava1018.github.io
![解析自定义域名](http://image2.ishareread.com/images/20200212/blogimage/5.png)
在GitHub Pages的自定义域名Custom domain中输入刚解析的域名保存后就可以看到你的站点被发布到你的域名上了，如https://xiejavablog.ishareread.com/ 
![绑定自定义域名](http://image2.ishareread.com/images/20200212/blogimage/6.png)
这时候你就可以用自己的域名来访问GitHub Pages的网站了，不过现在什么都没有，只有个空白页面。这就需要我们借助Hexo这个静态站点生成工具来生我们站点的内容了。

# 二、安装Hexo并生成站点
安装Hexo并生成站点可以参考官方的文档 [https://hexo.io/zh-cn/docs/](https://hexo.io/zh-cn/docs/)  
## 1、安装前的准备
安装 Hexo 相当简单，只需要先安装下列应用程序即可：
Node.js (Node.js 版本需不低于 8.10，建议使用 Node.js 10.0 及以上版本)
Git
## 2、安装Hexo
所有必备的应用程序安装完成后，即可使用 npm 安装 Hexo。
```bash
$ npm install -g hexo-cli
```

安装完了以后可以通过hexo version 查看相应的版本
![版本信息](http://image2.ishareread.com/images/20200212/blogimage/7.png)
## 3、生成站点
安装 Hexo 完成后，执行下列命令，Hexo 将会在指定文件夹中新建所需要的文件。
```bash
$ hexo init <folder>
$ cd <folder>
$ npm install
```

新建完成后，指定文件夹的目录如下：
![目录](http://image2.ishareread.com/images/20200212/blogimage/8.png)
其中_config.yml 文件是网站的配置文件
package.json 是应用程序的信息
scaffolds 
模版文件夹。当新建文章时，Hexo 会根据 scaffold 来建立文件。
Hexo的模板是指在新建的文章文件中默认填充的内容。例如，如果您修改scaffold/post.md中的Front-matter内容，那么每次新建一篇文章时都会包含这个修改。
source
资源文件夹是存放用户资源的地方。除 _posts 文件夹之外，开头命名为 _ (下划线)的文件 / 文件夹和隐藏的文件将会被忽略。Markdown 和 HTML 文件会被解析并放到 public 文件夹，而其他文件会被拷贝过去。
themes
主题 文件夹。Hexo 会根据主题来生成静态页面。
## 4、安装主题
Hexo提供了很多主题，我用的是hexo-theme-next主题，大家可以直接克隆我的主题https://github.com/xiejava1018/hexo-theme-next.git 这里修复了一些bug如乱码问题等。
cd 切换到站点目录下
```bash
$ git clone https://github.com/xiejava1018/hexo-theme-next.git themes/hexo-theme-next
```
也可以用其他git客户端工具将主题拉取到themes目录下
修改_config.yml文件的theme改为

```yaml
theme : hexo-theme-next
```

## 5、写作
可以执行下列命令来创建一篇新文章或者新的页面。

```bash
$ hexo new [layout] <title>
```

可以在命令中指定文章的布局（layout），默认为 post，可以通过修改 _config.yml 中的 default_layout 参数来指定默认布局。
如执行：

```bash
hexo new 2020-02-11-2020-02-11-看完全套149本《书虫》是种什么样的体验
```

执行该命令后就会在响应的站点目录的source\_posts下生成2020-02-11-看完全套149本《书虫》是种什么样的体验.md文件。
![文件](http://image2.ishareread.com/images/20200212/blogimage/9.png)
用任何喜欢的编辑器编辑这个.md文件即可，排版是支持MarkDown的。
## 6、生成和发布
编辑好需要发表的内容后。执行

```bash
$ hexo generate
```

就会生成相应的静态文件。改命令也可以简写成

```bash
$ hexo g
```

执行

```bash
$ hexo server
```

启动服务器。默认情况下，访问网址为： http://localhost:4000/ 就可以通过该地址访问本地的站点。
在本地检查没有问题以后就可以发布到Github Pages上通过互联网上访问了。
首先在配置_config.yml文件配置需要发布的地址。这个地址就是你在github上申请的Github Pages库的git地址
![发布地址配置](http://image2.ishareread.com/images/20200212/blogimage/10.png)
然后就可以通过命令

```bash
$ hexo deploy
```

进行发布了。发布以后就可以通过https://xiejava1018.github.io 或者自定义的域名 https://xiejavablog.ishareread.com 来访问了。需要注意的是，每次重新发布以后，需要重新设置域名绑定才能正确访问，否则会报404的错误。

欢迎大家访问我的BLOG  [https://xiejavablog.ishareread.com/](https://xiejavablog.ishareread.com/)

---
博客文章：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>
