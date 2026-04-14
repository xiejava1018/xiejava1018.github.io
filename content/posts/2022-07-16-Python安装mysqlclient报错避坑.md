---
title: Python3.9环境安装mysqlclient报python setup.py egg_info did not run successfully错避坑
id: 20220716001
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: 7435f815
date: 2022-07-16 11:07:00
---
MySQL是常用的开源数据库，Python环境下django框架连接MySQL数据库用的是mysqlclient库，今天在用pip安装mysqlclient库时报错，特记录一下，避免后续继续踩坑。

# 环境说明：
操作系统：CentOS Linux 7.2
Python版本：Python 3.9.13
pip版本：pip 22.1.2

# 报错信息：
执行`pip3 install mysqlclient==2.1.1` 报错
报错信息如下：

```bash
Using cached http://mirrors.aliyun.com/pypi/packages/50/5f/eac919b88b9df39bbe4a855f136d58f80d191cfea34a3dcf96bf5d8ace0a/mysqlclient-2.1.1.tar.gz (88 kB)
  Preparing metadata (setup.py) ... error
  error: subprocess-exited-with-error
  
  × python setup.py egg_info did not run successfully.
  │ exit code: 1
  ╰─> [16 lines of output]
      /bin/sh: mysql_config: command not found
      /bin/sh: mariadb_config: command not found
      /bin/sh: mysql_config: command not found
      Traceback (most recent call last):
        File "<string>", line 2, in <module>
        File "<pip-setuptools-caller>", line 34, in <module>
        File "/tmp/pip-install-i1nt_asj/mysqlclient_1b92535d58cd440b8797686ac8bc9882/setup.py", line 15, in <module>
          metadata, options = get_config()
        File "/tmp/pip-install-i1nt_asj/mysqlclient_1b92535d58cd440b8797686ac8bc9882/setup_posix.py", line 70, in get_config
          libs = mysql_config("libs")
        File "/tmp/pip-install-i1nt_asj/mysqlclient_1b92535d58cd440b8797686ac8bc9882/setup_posix.py", line 31, in mysql_config
          raise OSError("{} not found".format(_mysql_config_path))
      OSError: mysql_config not found
      mysql_config --version
      mariadb_config --version
      mysql_config --libs
      [end of output]
  
  note: This error originates from a subprocess, and is likely not a problem with pip.
error: metadata-generation-failed

× Encountered error while generating package metadata.
╰─> See above for output.

note: This is an issue with the package mentioned above, not pip.
hint: See above for details.
```
![mysqlclient报错](http://image2.ishareread.com/images/2022/20220716/mysqlclient安装报错.png)


# 避坑：
**从报错信息看是找不到mysql_config** 
通过`whereis mysql_config`命令查看mysql_config
发现mysql_confg没有
**执行`yum install mysql-devel` 安装mysql-devel**
执行whereis mysql_config命令查看mysql_config这时mysql_config有了

```bash
mysql_config: /usr/bin/mysql_config /usr/share/man/man1/mysql_config.1.gz
```

再次执行pip安装命令安装成功！

```bash
pip3 install mysqlclient==2.1.1

Looking in indexes: http://mirrors.aliyun.com/pypi/simple/
Collecting mysqlclient==2.1.1
  Using cached http://mirrors.aliyun.com/pypi/packages/50/5f/eac919b88b9df39bbe4a855f136d58f80d191cfea34a3dcf96bf5d8ace0a/mysqlclient-2.1.1.tar.gz (88 kB)
  Preparing metadata (setup.py) ... done
Using legacy 'setup.py install' for mysqlclient, since package 'wheel' is not installed.
Installing collected packages: mysqlclient
  Running setup.py install for mysqlclient ... done
Successfully installed mysqlclient-2.1.1
```

