---
title: Supervisor离线安装及使用
id: 20200518001
tags:
  - 运维
categories:
  - - 技术
    - 开发
abbrlink: d670c9b8
date: 2020-05-18 11:35:36
---
Supervisor是用Python开发的一套通用的进程管理程序，能将一个普通的命令行进程变为后台daemon，并监控进程状态，异常退出时能自动重启。它是通过fork/exec的方式把这些被管理的进程当作supervisor的子进程来启动，这样只要在supervisor的配置文件中，把要管理的进程的可执行文件的路径写进去即可。也实现当子进程挂掉的时候，父进程可以准确获取子进程挂掉的信息的，可以选择是否自己启动和报警

supervisor的安装有多种方式
配置好yum源后，可以直接安装
```bash
yum install supervisor
```
Debian/Ubuntu可通过apt安装
```bash
apt-get install supervisor
```
pip安装
```bash
pip install supervisor
```
easy_install安装
```bash
easy_install 
```
这几种安装方式都需要在线联网。但大部分的生产环境都是离线环境，是封闭的网络没有办法在线安装。

这里整理了Supervisor的离线安装包和安装脚本，可以进行离线安装并能指定安装目录。
# 一、整理Supervisor安装需要的工具和依赖包
包括有：
setuptools
elementtree
meld3
supervisor

# 二、编写离线安装脚本
整体思路：依次解压并安装Supervisor所需要的工具和依赖包，将Supervisor的配置文件的默认安装目录路径替换成制定的目录路径

```bash
vi install_supervisor.sh
```

```bash
#!/usr/bin/env bash
function Install_Supervisor()
{
    #Install supervisord
    tar -zxvf setuptools-24.0.2.tar.gz 2>&1 >/dev/null
    cd setuptools-24.0.2/
    python setup.py install >/dev/null 2>&1
    cd ..
    easy_install elementtree-1.2.7-20070827-preview.zip >/dev/null 2>&1
    easy_install meld3-0.6.5.tar.gz 2>/dev/null 2>&1
    easy_install supervisor-3.3.0.tar.gz >/dev/null 2>&1
    mkdir -p ${INSTALL_DIR}/etc/
    mkdir -p ${INSTALL_DIR}/tmp/
    mkdir -p ${INSTALL_DIR}/logs/
    cp etc/supervisord.conf ${INSTALL_DIR}/etc/
    sed -i "s#__install_dir__#${INSTALL_DIR}#g" ${INSTALL_DIR}/etc/supervisord.conf
    sed -i "s#__user__#${USER}#g" ${INSTALL_DIR}/etc/supervisord.conf
    ln -s /usr/bin/supervisorctl ${INSTALL_DIR}/commandctl
    cp run_supervisor.sh ${INSTALL_DIR}/
    sed -i "s#__install_dir__#${INSTALL_DIR}#g" ${INSTALL_DIR}/run_supervisor.sh
    chmod +x ${INSTALL_DIR}/run_supervisor.sh
}

USER='root'
if [ ! -n "$1" ]; then
   INSTALL_DIR='/app/supervisor'
else
   INSTALL_DIR=$1
fi
Install_Supervisor
```

安装脚本中默认的安装路径是/app/supervisor，可以根据实际情况进行调整。

另外整理了一个run_supervisor.sh的脚本，在安装以后根据安装目录来生成这个启动脚本。

```bash
#!/usr/bin/env bash
cd __install_dir__
if [ ! -d "tmp" ];then
  mkdir tmp
else
  echo "tmp文件夹已经存在"
fi
if [ ! -d "logs" ];then
  mkdir logs
else
  echo "logs文件夹已经存在"
fi
/usr/bin/supervisord -c __install_dir__/etc/supervisord.conf
echo "supervisord 已执行。"
```

# 三、将所有的安装包脚本等打成离线安装包
```bash
tar -czvf  supervisor_install_pack.tar.gz  supervisor 
```
已打好的离线安装包下载 https://545c.com/file/21165215-443895501
[城通网盘下载](https://545c.com/file/21165215-443895501)   https://545c.com/file/21165215-443895501
[CSDN下载](https://download.csdn.net/download/fullbug/12434225) https://download.csdn.net/download/fullbug/12434225

# 四、离线安装包使用
下载离线安包，解压

```bash
tar -zxvf supervisor_install_pack.tar.gz
```

解压后看到supervisor的目录，在supervisor的目录中找到install_supervisor.sh的脚本
![supervisor离线包安装目录](http://image2.ishareread.com/images/2020/20200518/1.png)
执行install_supervisor.sh的脚本，默认安装/app/supervisor下，可以带参数指定安装目录。也可以修改install_supervisor.sh脚本自定义默认的安装路径。
执行完安装脚本，就可以在/app/supervisor目录下看到有相应的启动脚本、配置文件目录、日志目录及临时文件目录。
![supervisor运行目录](http://image2.ishareread.com/images/2020/20200518/2.png)
执行run_supervisor.sh就可以启动supervisor

```bash
./run_supervisor.sh
```

# 五、验证和使用supervisor
ps -ef|grep supervisor  查看supervisor是否已经启动

![查看supervisor是否已经启动](http://image2.ishareread.com/images/2020/20200518/3.png)

通过web界面的9001看web界面控制台http://127.0.0.1:9001

![web界面控制台](http://image2.ishareread.com/images/2020/20200518/4.png)

- supervisord
运行 Supervisor 时会启动一个进程 supervisord，它负责启动所管理的进程，并将所管理的进程作为自己的子进程来启动，而且可以在所管理的进程出现崩溃时自动重启。
supervisord -v 查看supervisor版本号

- supervisorctl
是命令行管理工具，可以用来执行 stop、start、restart 等命令，来对这些子进程进行管理。
supervisor是所有进程的父进程，管理着启动的子进展，supervisor以子进程的PID来管理子进程，当子进程异常退出时supervisor可以收到相应的信号量。

**supervisor常用管理命令**
supervisorctl restart < application name> ;重启指定应用
supervisorctl stop < application name> ;停止指定应用
supervisorctl start < application name> ;启动指定应用
supervisorctl restart all ;重启所有应用
supervisorctl stop all ;停止所有应用
supervisorctl start all ;启动所有应用

# 六、配置文件说明
## supervisor.conf配置文件

```bash
[unix_http_server]
file=/tmp/supervisor.sock   ;UNIX socket 文件，supervisorctl 会使用
;chmod=0700                 ;socket文件的mode，默认是0700
;chown=nobody:nogroup       ;socket文件的owner，格式：uid:gid
 
;[inet_http_server]         ;HTTP服务器，提供web管理界面
;port=127.0.0.1:9001        ;Web管理后台运行的IP和端口，如果开放到公网，需要注意安全性
;username=user              ;登录管理后台的用户名
;password=123               ;登录管理后台的密码
 
[supervisord]
logfile=/tmp/supervisord.log ;日志文件，默认是 $CWD/supervisord.log
logfile_maxbytes=50MB        ;日志文件大小，超出会rotate，默认 50MB，如果设成0，表示不限制大小
logfile_backups=10           ;日志文件保留备份数量默认10，设为0表示不备份
loglevel=info                ;日志级别，默认info，其它: debug,warn,trace
pidfile=/tmp/supervisord.pid ;pid 文件
nodaemon=false               ;是否在前台启动，默认是false，即以 daemon 的方式启动
minfds=1024                  ;可以打开的文件描述符的最小值，默认 1024。注意托管ES进程，这里要进行调整至65535
minprocs=200                 ;可以打开的进程数的最小值，默认 200。注意托管ES进程，这里要进行调整至4096
 
[supervisorctl]
serverurl=unix:///tmp/supervisor.sock ;通过UNIX socket连接supervisord，路径与unix_http_server部分的file一致
;serverurl=http://127.0.0.1:9001 ; 通过HTTP的方式连接supervisord
 
; [program:xx]是被管理的进程配置参数，xx是进程的名称
[program:xx]
command=/opt/apache-tomcat-8.0.35/bin/catalina.sh run  ; 程序启动命令
autostart=true       ; 在supervisord启动的时候也自动启动
startsecs=10         ; 启动10秒后没有异常退出，就表示进程正常启动了，默认为1秒
autorestart=true     ; 程序退出后自动重启,可选值：[unexpected,true,false]，默认为unexpected，表示进程意外杀死后才重启
startretries=3       ; 启动失败自动重试次数，默认是3
user=tomcat          ; 用哪个用户启动进程，默认是root
priority=999         ; 进程启动优先级，默认999，值小的优先启动
redirect_stderr=true ; 把stderr重定向到stdout，默认false
stdout_logfile_maxbytes=20MB  ; stdout 日志文件大小，默认50MB
stdout_logfile_backups = 20   ; stdout 日志文件备份数，默认是10
; stdout 日志文件，需要注意当指定目录不存在时无法正常启动，所以需要手动创建目录（supervisord 会自动创建日志文件）
stdout_logfile=/opt/apache-tomcat-8.0.35/logs/catalina.out
stopasgroup=false     ;默认为false,进程被杀死时，是否向这个进程组发送stop信号，包括子进程
killasgroup=false     ;默认为false，向进程组发送kill信号，包括子进程
 
;包含其它配置文件
[include]
files = supervisord.d/*.ini    ;默认放在安装目录的supervisord.d目录下，可以指定一个或多个以.ini结束的配置文件。
```

## 子进程配置文件
需要给托管的子进程配置相应的配置文件，每个进程的配置文件都可以单独分拆也可以把相关的脚本放一起。目录及文件后缀可以在
supervisor.conf配置文件中进行自定义。见supervisor.conf的

```bash
[include]
files = supervisord.d/*.ini  #目录路径及文件后缀名都可以自定义。
```

logstash.ini 样例说明：

```bash
#项目名
[program:logstash-test]
#脚本目录
directory=/app/elk/logstash-7.6.0
#脚本执行命令
command=/app/elk/logstash-7.6.0/bin/logstash -f /app/elk/logstash-7.6.0/bin/test-pipeline.conf
#进程数
numprocs=1
#supervisor启动的时候是否随着同时启动，默认True
autostart=true
#当程序exit的时候，这个program不会自动重启,默认unexpected，设置子进程挂掉后自动重启的情况，有三个选项，false,unexpected和true。如果为false的时候，无论什么情况下，都不会被重新启动，如果为unexpected，只有当进程的退出码不在下面的exitcodes里面定义的
autorestart=false
#这个选项是子进程启动多少秒之后，此时状态如果是running，则我们认为启动成功了。默认值为1
startsecs=1
#脚本运行的用户身份 
user = root
#把stderr重定向到stdout，默认 false
redirect_stderr = true
#stdout日志文件大小，默认 50MB
stdout_logfile_maxbytes = 10M
#stdout日志文件备份数
stdout_logfile_backups = 10
#日志输出 
stderr_logfile=/app/elk/logstash-7.6.0/logs/logstash_test_error.log
stdout_logfile=/app/elk/logstash-7.6.0/logs/logstash_test_out.log

```

------
作者博客:[http://xiejava.ishareread.com](http://xiejava.ishareread.com)
<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>