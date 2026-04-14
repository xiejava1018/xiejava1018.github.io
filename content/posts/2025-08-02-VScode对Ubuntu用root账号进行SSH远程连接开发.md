---
title: VScode对Ubuntu用root账号进行SSH远程连接开发
id: 20250802001
tags:
  - 运维
categories:
  - - 技术
    - 开发
abbrlink: 9880bf27
date: 2025-08-02 21:19:42
---
由于linux服务器大部分都是基于命令行的操作，缺乏比较方便好用的编辑工具，对于经常在linux服务器上做开发的同学来说直接在服务器上进行开发或配置文件的修改还不是特别的方便。虽然linux上有vi或vim比起图形化的编辑工具体验感还是不是很好。作为程序员常用的宇宙第一的VScode就提供了相应的插件可以通过ssh远程连接到服务器上直接编辑服务器上相应的文件，极大提高了在服务器上开发或修改配置文件的效率和体验感。本文就来介绍通过VScode对Ubuntu用root账号进行SSH远程连接直接在服务器上进行开发。
## 一、安装VScode的ssh扩展插件
Remote-SSH 扩展 允许你将任何带有 SSH 服务器的远程机器用作你的开发环境。这可以在多种情况下极大地简化开发和故障排除：
- 在部署目标操作系统上开发​，或者使用比本地机器更大、更快或更专业化的硬件进行开发。
- 轻松切换不同的远程开发环境，并安全地进行更新​，而无需担心影响本地机器。
- 从多台机器或不同位置访问现有的开发环境。
- 调试运行在其他地方（例如客户现场或云端​）的应用程序。

在VScode的编辑器界面在Extension中搜索ssh排名前三的Remote-SSH、Remote - SSH: Editing Configuration Files、Remote Explorer是微软官方的ssh扩展插件，都安装一下。

![ssh扩展插件](http://image2.ishareread.com/images/2025/20250802/1-ssh-remote.png)


安装好了后就会在左侧的导航看到Remote Exploer

![Remote Exploer](http://image2.ishareread.com/images/2025/20250802/2-remoteExplorer.png)

在SSH中点“+” 后就可以在上面的导航输入框中输入要ssh连接服务器的命令。如:`ssh xiejava@192.168.0.30` 意思就是通过xiejava的用户名ssh登录到192.168.0.30的服务器。

![New Remote](http://image2.ishareread.com/images/2025/20250802/3-NewRemote.png)

输入正确的口令后，表示服务器的图标变亮就意味着已经通过SSH连上了服务器了。

![SSH连上了服务器](http://image2.ishareread.com/images/2025/20250802/4-SSH连上了服务器.png)

连上服务器后就可以通过命令来对服务器进行操作，也可以直接打开服务器上的文件进行操作了。

![直接打开服务器上的文件进行操作](http://image2.ishareread.com/images/2025/20250802/5-直接打开服务器上的文件进行操作.png)

这样就可以直接在服务器上直接编辑文件，直接在命令行中输入命令执行，直接看到执行结果，简直不要太爽。

![直接编辑文件](http://image2.ishareread.com/images/2025/20250802/6-直接编辑文件.png)


一般的教程到这里就结束了。但是有些Linux是默认不能使用root用户登录的如Ubuntu，如果不用root用户登录只能访问到登录用户的文件夹，如我是用xiejava登录的就只能访问到/home/xiejava的目录，只能在这个目录下新建和编辑相应的文件，而且每次打开编辑操作文件都要输入登录密码，作为服务器上的开发来说这显然是不可接受的。所以我们要用root账号进行ssh进行远程连接。

## 二、开启Ubuntu的root账号免密登录
### 1、生成SSH密钥对，并部署到服务器
要使 root 用户成功登录，需先在客户端生成 SSH 密钥对，并将公钥部署到服务器：
1. 客户端生成密钥 (在本地PC执行)：

```bash
ssh-keygen -t ed25519 -C "root_ssh_key"
```

- 按提示选择保存路径（默认 C:\Users\你的用户名\.ssh\id_ed25519）。
- 连续按 3 次回车​，不设密码（实现完全免密）
最后在文件夹中生成SSH密钥对
![生成SSH密钥对](http://image2.ishareread.com/images/2025/20250802/7-生成密钥对.png)

2. 部署公钥到服务器​：
登录 Ubuntu
创建 .ssh 目录并写入公钥

```bash
mkdir -p ~/.ssh
echo "粘贴复制的公钥内容，也就是id_ed25519.pub文件里面的内容" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 2、测试免密登录
在 Windows PowerShell 执行：

```bash
ssh 用户名@Ubuntu_IP
```

- 成功标志​：直接登录 Ubuntu，无需输入密码。
- 失败表现​：提示输入密码或返回 Permission denied。

### 3、启用root用户
使用当前具有 sudo 权限的用户登录系统。执行密码修改命令

```bash
sudo passwd root
```

- 输入当前用户的密码​（用于 sudo 提权）。
- 输入新的 root 密码​（输入时不会显示字符）。
- 再次确认新密码。
禁用root密码登录（增强安全）​

```bash
sudo vim /etc/ssh/sshd_config
```

设置`PermitRootLogin prohibit-password`
![PermitRootLogin prohibit-password](http://image2.ishareread.com/images/2025/20250802/8-PermitRootLogin.png)

重启`systemctl restart ssh`

### 4、找到本地的ssh配置文件并配置密钥登录
在vscode中点击SSH旁边的配置按钮就会出现ssh本地的配置文件，一般是`C:\Users\你的用户名\.ssh\config`

![找到本地的ssh配置文件](http://image2.ishareread.com/images/2025/20250802/9-ssh配置文件.png)

在config文件中配置User root 和IdentityFile 密钥文件的路径。

![配置](http://image2.ishareread.com/images/2025/20250802/10-config配置文件.png)

这样就可以通过root进入到任何目录操作编辑任何文件了。

![root免密登录](http://image2.ishareread.com/images/2025/20250802/11-进入任何目录编辑.png)

最后要提醒大家的是大家在修改任何配置文件的时候要注意备份和确认，因为获得了root权限可以操作编辑任何文件了。

-----------------------------------
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)
<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>