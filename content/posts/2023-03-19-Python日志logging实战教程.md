---
title: Python日志logging实战教程
id: 20230319001
tags:
  - Python
categories:
  - - 技术
    - 开发
abbrlink: c24f493c
date: 2023-03-19 14:21:31
---
## 一、什么是日志
在[《网络安全之认识日志采集分析审计系统》](http://xiejava.ishareread.com/posts/6a8b36cb/)中我们认识了日志。日志数据的核心就是日志消息或日志，日志消息是计算机系统、设备、软件等在某种刺激下反应生成的东西。

日志数据（log data）就是一条日志消息的内在含义，用来告诉你为什么生成日志消息的信息。日志（log）指用于展示某些事件全貌的日志消息的集合。
## 二、为什么要写日志
日志是对软件执行时所发生事件的一种追踪方式。软件开发人员对他们的代码添加日志调用，借此来指示某事件的发生。一个事件通过一些包含变量数据的描述信息来描述。对于软件系统来说，健全的日志记录是程序调试、故障定位、事件追溯的有效手段。

日志通用的几种类型：
 - 信息（Info）:告诉用户和管理员发生了一些没有风险的事情。
 - 调试（Debug）:在应用程序代码运行时生成调试信息，给软件开发人员提供故障检测和定位问题的帮助。
 - 警告（Warning）:缺少需要的文件、参数、数据，但又不影响系统运行时生成警告。
 - 错误（Error）:传达在计算机系统重出现的各种级别的错误。许多错误消息只能给出为什么出错的起点，要寻找出导致错误发生的根本原因还需要进一步的调查。

## 三、Python日志logging模块实战
在进行Python程序开发时，Python提供了logging模块，能够很好的帮助开发人员很方便的的记录日志信息。

对于简单的日志使用来说日志功能提供了一系列便利的函数。它们是 debug()，info()，warning()，error() 和 critical()。想要决定何时使用日志，请看下表，其中显示了对于每个通用任务集合来说最好的工具。

|你想要执行的任务  |  此任务的最好的工具 |
|--|--|
| 对于命令行或程序的应用，结果显示在控制台。 |  print()|
|在对程序的普通操作发生时提交事件报告(比如：状态监控和错误调查)|logging.info() 函数(当有诊断目的需要详细输出信息时使用 logging.debug() 函数)|
| 提出一个警告信息基于一个特殊的运行时事件 | warnings.warn() 位于代码库中，该事件是可以避免的，需要修改客户端应用以消除告警logging.warning() 不需要修改客户端应用，但是该事件还是需要引起关注|
|对一个特殊的运行时事件报告错误 | 引发异常 |
|报告错误而不引发异常(如在长时间运行中的服务端进程的错误处理) |logging.error(), logging.exception() 或 logging.critical() 分别适用于特定的错误及应用领域 |


日志功能应以所追踪事件级别或严重性而定。各级别适用性如下（以严重性递增）：

| 级别 |何时使用|
|--|--|
|DEBUG  |细节信息，仅当诊断问题时适用。  |
|INFO | 确认程序按预期运行。 |
| WARNING| 表明有已经或即将发生的意外（例如：磁盘空间不足）。程序仍按预期进行。|
| ERROR | 由于严重的问题，程序的某些功能已经不能正常执行|
|CRITICAL|严重的错误，表明程序已不能继续执行 |


默认的级别是 WARNING，意味着只会追踪该级别及以上的事件，除非更改日志配置。

所追踪事件可以以不同形式处理。最简单的方式是输出到控制台。另一种常用的方式是写入磁盘文件。

Python的logging库采用模块化方法，并提供了几类组件：记录器，处理程序，过滤器和格式化程序。

 - 记录器（Logger）：提供应用程序代码直接使用的接口。 
 - 处理器（Handler）：将日志记录（由记录器创建）发送到适当的目的地。
 - 筛选器（Filter）：提供了更细粒度的功能，用于确定要输出的日志记录。 
 -  格式器（Formatter）：程序在最终输出日志记录的内容格式。

logging的工作流程：以记录器Logger为对象，设置合适的处理器Handler，辅助以筛选器Filter、格式器Formatter，设置日志级别以及常用的方法，最终输出理想的日志记录给到指定目标
一个Logger可以包含多个Handler；
每个Handler可以设置自己的Filter和Formatter；
记录器和处理器中的日志事件信息流程如下图所示：
![日志事件信息流程](http://image2.ishareread.com/images/2023/20230319/1_logging流程.png)


接下来我们通过几个简单的应用场景来进行日志记录实战
### 1、最简单的日志记录
**main.py**

```python
import logging

def print_hi(name):
    logging.debug('this is print_hi debug')
    logging.info('this is print_hi info')
    logging.warning('this is print_hi warning')
    logging.error('this is print_hi error')
    logging.critical('this is print_hi critical')
    print(f'Hi print_hi, {name}')

if __name__ == '__main__':
    print_hi('XieJava')
```

结果如下：
![最简单的日志记录结果](http://image2.ishareread.com/images/2023/20230319/2_最简单的日志记录结果.png)

这里体现了两个问题：
1.通过print()在控制台打印的日志，比logging打印的日志提前打印显示，说明日志记录是多线程的。在平时日志调试跟踪的时候注意这一点，print()的信息有时会打印在logging前有时会在logging后。

2.debug和info的日志没有打印出来，说明logging默认的日志级别是waring。
如果要设置改变默认的日志级别可以通过配置来设置日志级别如：level=logging.DEBUG

### 2、设置日志级别

```python
import logging

logging.basicConfig(level=logging.DEBUG) #设置日志级别

def print_hi(name):
    logging.debug('this is print_hi debug')
    logging.info('this is print_hi info')
    logging.warning('this is print_hi warning')
    logging.error('this is print_hi error')
    logging.critical('this is print_hi critical')
    print(f'Hi print_hi, {name}')

if __name__ == '__main__':
    print_hi('XieJava')
```
![设置日志级别结果](http://image2.ishareread.com/images/2023/20230319/3_设置日志级别的结果.png)

这下从DEBUG到CRITICAL级别的都打印出来了。
### 3、设置日志显示格式
默认的日志打印显示的格式是， 日志级别：logger实例名称（默认是root）：日志消息内容
如这里显示的是：

```powershell
DEBUG:root:this is print_hi debug
INFO:root:this is print_hi info
WARNING:root:this is print_hi warning
ERROR:root:this is print_hi error
CRITICAL:root:this is print_hi critical
```

在真实使用的场景下，一般都要显示日志的时间，我们可以通过设置日志显示格式来调整我们需要显示的日志格式和内容。

```python
LOG_FORMAT = "%(asctime)s - %(levelname)s %(name)s %(filename)s [line:%(lineno)d] - %(message)s"
logging.basicConfig(format=LOG_FORMAT,level=logging.DEBUG)
```

这里设置了日志发生时间、日志级别、logger实例名称、日志发生的文件名、日志发生所在的行、日志消息内容。
![设置日志格式](http://image2.ishareread.com/images/2023/20230319/4_设置日志格式.png)

更多的logRecord属性如下：

| 属性名称 | 格式 | 描述  |
|--|--|--|
| args | 此属性不需要用户进行格式化。 | 合并到 msg 以产生 message 的包含参数的元组，或是其中的值将被用于合并的字典（当只有一个参数且其类型为字典时）。  |
|asctime| %(asctime)s| 表示 LogRecord 何时被创建的供人查看时间值。 默认形式为 '2003-07-08 16:49:45,896' （逗号之后的数字为时间的毫秒部分）。|
| created| %(created)f|LogRecord 被创建的时间（即 time.time() 的返回值）。 | 
| exc_info|此属性不需要用户进行格式化。 | 异常元组（例如 sys.exc_info）或者如未发生异常则为 None。|
|filename  | %(filename)s  |  pathname 的文件名部分。|
|funcName|%(funcName)s|函数名包括调用日志记录。|
|levelname|%(levelname)s|消息文本记录级别（'DEBUG'，'INFO'，'WARNING'，'ERROR'，'CRITICAL'）|
| levelno| %(levelno)s|消息数字的记录级别 (DEBUG, INFO, WARNING, ERROR, CRITICAL)|
|lineno|%(lineno)d|发出日志记录调用所在的源行号（如果可用）。|
|message|%(message)s|记入日志的消息，即 msg % args 的结果。 这是在发起调用 Formatter.format() 时设置的。|
|module|%(module)s|模块 (filename 的名称部分)。|
| msecs|%(msecs)d |LogRecord 被创建的时间的毫秒部分。 |
|msg |此属性不需要用户进行格式化。 | 在原始日志记录调用中传入的格式字符串。 与 args 合并以产生 message，或是一个任意对象 (参见 使用任意对象作为消息)。|
|name | %(name)s| 用于记录调用的日志记录器名称。|
| pathname| %(pathname)s |发出日志记录调用的源文件的完整路径名（如果可用）。 |
|process | %(process)d|进程ID（如果可用） |
|processName |  %(processName)s | 进程名（如果可用） |
| relativeCreated | %(relativeCreated)d |以毫秒数表示的 LogRecord 被创建的时间，即相对于 logging 模块被加载时间的差值。|
| stack_info| 此属性不需要用户进行格式化。 | 当前线程中从堆栈底部起向上直到包括日志记录调用并引发创建当前记录堆栈帧创建的堆栈帧信息（如果可用）。|
| thread| %(thread)d | 线程ID（如果可用） |
|threadName | %(threadName)s|线程名（如果可用） |


### 4、记录日志到日志文件
logging默认是显示在控制台，在真实生产环境肯定时需要将日志记录到日志文件的。logging也是可以通过配置很方便的将日志记录到日志文件。

```python
LOG_FORMAT = "%(asctime)s - %(levelname)s %(name)s %(filename)s [line:%(lineno)d] - %(message)s"
logging.basicConfig(filename='log.log',format=LOG_FORMAT,level=logging.DEBUG)
```
![输出到日志文件中](http://image2.ishareread.com/images/2023/20230319/5_日志打印到日志文件.png)
输出到日志文件
![输出到日志文件效果](http://image2.ishareread.com/images/2023/20230319/6_输出到日志文件效果.png)
将日志输出到日志文件，如果是日志量非常大，在实际生产环境经常碰到的是要对日志文件进行分隔，根据日志文件的大小或日期来分割生成多个日志文件。
这里介绍通过日志文件大小分割和通过日期来分割日志文件。
#### 1.通过日志文件大小分割
RotatingFileHandler
日志记录到文件中，且支持指定日志文件大小，备份文件数量

```python
logging.handlers.RotatingFileHandler(filename, mode='a', maxBytes=0, backupCount=0, encoding=None, delay=False)
```

maxBytes：日志文件大小，单位为字节
backupCount：备份文件数量

```python
LOG_FORMAT = "%(asctime)s - %(levelname)s %(name)s %(filename)s [line:%(lineno)d] - %(message)s"
rfh=logging.handlers.RotatingFileHandler(filename='log.log',encoding='UTF-8', maxBytes=1024, backupCount=2)
logging.basicConfig(format=LOG_FORMAT,level=logging.DEBUG,handlers=[rfh])
```

这里设置的是当文件超过1024bytes就会对文件进行分割，备份文件数量为2，得到log.log.1、log.log.2，当log.log.2写满时又回循环写到log.log中。
![根据日志文件大小循环切割生成日志文件](http://image2.ishareread.com/images/2023/20230319/7_根据日志文件大小循环切割生成日志文件.png)

#### 2.通过日期来分割日志文件
TimedRotatingFileHandler
日志记录到文件中，支持按时间间隔来更新日志

```python
logging.handlers.TimedRotatingFileHandler(filename, when='h', interval=1, backupCount=0, encoding=None, delay=False, utc=False, atTime=None)
```

指定的文件会被打开并用作日志记录的流。 对于轮换操作它还会设置文件名前缀。 轮换的发生是基于 when 和 interval 的积。
你可以使用 when 来指定 interval 的类型。 可能的值列表如下。 请注意它们不是大小写敏感的。

| 值 |  间隔类型|如果/如何使用 atTime |
|--|--|---|
| 'S' |  秒| 忽略 |
| 'M' | 分钟 |忽略 |
| 'H'| 小时  |忽略 |
| 'D' | 天 | 忽略 |
|'W0'-'W6'|工作日(0=星期一)|用于计算初始轮换时间|
|'midnight'|如果未指定 atTime 则在午夜执行轮换，否则将使用 atTime。|用于计算初始轮换时间|


```python
import logging
from datetime import time
from logging.handlers import RotatingFileHandler

LOG_FORMAT = "%(asctime)s - %(levelname)s %(name)s %(filename)s [line:%(lineno)d] - %(message)s"
tfh=logging.handlers.TimedRotatingFileHandler('tfh_log.log', when='S', interval=1.5, backupCount=2, encoding='UTF-8', delay=False, utc=False, atTime=time)
rfh=logging.handlers.RotatingFileHandler(filename='log.log',encoding='UTF-8', maxBytes=1024, backupCount=2)
logging.basicConfig(format=LOG_FORMAT,level=logging.DEBUG,handlers=[rfh,tfh])
```

为了演示方便，这里when='S', interval=1.5 即1.5秒循环生成一个日志文件。在实际生产环境一般根据日志量的大小，可以配置成每天生成一个日志文件。
![通过日期来分割日志文件](http://image2.ishareread.com/images/2023/20230319/8_根据日期切割生成日志文件.png)

### 5、既生成日志文件又在控制台打印日志
有时候为了调试方便，还是想在控制台打印日志。能不能既生成日志文件又在控制台打印日志呢？通过配置logging的StreamHandler也是可以做到的。
StreamHandler
StreamHandler 类位于核心 logging 包，它可将日志记录输出发送到数据流例如 sys.stdout, sys.stderr 或任何文件类对象（或者更精确地说，任何支持 write() 和 flush() 方法的对象）。

```python
sh=logging.StreamHandler()
logging.basicConfig(format=LOG_FORMAT,level=logging.DEBUG,handlers=[rfh,tfh,sh])
```
![既生成日志文件又在控制台打印日志](http://image2.ishareread.com/images/2023/20230319/9_既生成日志文件又在控制台打印日志.png)

### 6、多个模块中记录日志
在实际项目使用过程中，一个好的实践是将日志配置的模块封装好成为一个通用的日志模块组件，可以给项目中所有的模块使用。
这里我们将配置好的日志logging从main.py中抽出来形成一个logutils.py的通用模块，其他模块就可以使用了
**logutils.py**

```python
import logging
from datetime import time
from logging.handlers import RotatingFileHandler

LOG_FORMAT = "%(asctime)s - %(levelname)s %(name)s %(filename)s [line:%(lineno)d] - %(message)s"
tfh=logging.handlers.TimedRotatingFileHandler('tfh_log.log', when='S', interval=1.5, backupCount=2, encoding='UTF-8', delay=False, utc=False, atTime=time)
rfh=logging.handlers.RotatingFileHandler(filename='log.log',encoding='UTF-8', maxBytes=1024, backupCount=2)
sh=logging.StreamHandler()
logging.basicConfig(format=LOG_FORMAT,level=logging.DEBUG,handlers=[rfh,tfh,sh])
```
![公共日志模块](http://image2.ishareread.com/images/2023/20230319/10_logutils.png)

如在othermodule.py中使用

```python
from logutils import logging

class TestModule():

    def print_log(self):
        logging.info('this is TestModule.print_log() info')

    @staticmethod
    def print_log_staic():
        logging.info('this is TestModule.print_log_staic info')
```

在main.py中使用

```python
from logutils import logging
from othermodule import TestModule

def print_hi(name):
    logging.debug('this is print_hi debug')
    logging.info('this is print_hi info')
    logging.warning('this is print_hi warning')
    logging.error('this is print_hi error')
    logging.critical('this is print_hi critical')
    print(f'Hi print_hi, {name}')


if __name__ == '__main__':
    print_hi('XieJava')
    TestModule.print_log_staic() #类方法中打印日志
    testModule=TestModule()
    testModule.print_log()  #实例方法中打印日志
```
![其他模块中显示日志](http://image2.ishareread.com/images/2023/20230319/11_其他模块中使用日志.png)

项目工程中所有的模块只要通过 `from logutils import logging` 引入logging就可以使用配置好的logging记录日志了。

### 7、每个不同的模块使用不同的日志记录器记录日志
现在我们在所有的模块中都是用的默认的root记录器来记录的日志，实际上也可以让每个不同的模块使用不同的日志记录器记录日志。
日志事件信息在 LogRecord 实例中的记录器、处理器、过滤器和格式器之间传递。
通过调用 Logger 类（以下称为 loggers ， 记录器）的实例来执行日志记录。 
在命名记录器时使用的一个好习惯是在每个使用日志记录的模块中使用模块级记录器，命名如下:

```python
logger = logging.getLogger(__name__)
```

这意味着记录器名称跟踪包或模块的层次结构，并且直观地从记录器名称显示记录事件的位置。
记录器层次结构的根称为根记录器。 这是函数 debug() 、 info() 、 warning() 、 error() 和 critical() 使用的记录器，它们就是调用了根记录器的同名方法。 函数和方法具有相同的签名。 根记录器的名称在输出中打印为 'root' 。
实际上我们只要通过 `logger = logging.getLogger(__name__)` 给每个模块定义一个记录器就可以了。
**main.py**

```python
from logutils import logging
from othermodule import TestModule
logger=logging.getLogger(__name__)  #定义模块日志记录器

def print_hi(name):
    logger.debug('this is print_hi debug')
    logger.info('this is print_hi info')
    logger.warning('this is print_hi warning')
    logger.error('this is print_hi error')
    logger.critical('this is print_hi critical')
    print(f'Hi print_hi, {name}')


if __name__ == '__main__':
    print_hi('XieJava')
    TestModule.print_log_staic() #类方法中打印日志
    testModule=TestModule()
    testModule.print_log()  #实例方法中打印日志
```

**othermodule.py**

```python
from logutils import logging
logger=logging.getLogger(__name__) #定义模块日志记录器

class TestModule():

    def print_log(self):
        logger.info('this is TestModule.print_log() info')

    @staticmethod
    def print_log_staic():
        logger.info('this is TestModule.print_log_staic info')
```
![每个不同的模块使用不同的日志记录器记录日志](http://image2.ishareread.com/images/2023/20230319/12_每个不同的模块使用不同的日志记录器记录日志.png)
可以看到模块日志记录器打印出来的日志中模块名不再是默认的root，而是各自的模块名。

### 8、通过配置文件配置日志记录器
在实际项目应用的过程中，通常通过配置文件来配置日志记录器的各种配置，这样的好处就是改变日志记录的配置不需要修改代码，直接修改配置文件就可以了。
接下来介绍如何通过配置文件配置logging日志记录器
新建 logging.conf 配置文件，通过如下配置将前面代码中的处理器，日志记录器，通过配置文件的方式配置好。
 **logging.conf** 
 

```python
[loggers]
keys=root,logger01

[logger_root]
level=DEBUG
handlers=sh

[logger_logger01]
level=DEBUG
handlers=sh,tfh,rfh
qualname=logger01
propagate=0

[handlers]
keys=sh,tfh,rfh

[handler_sh]
class=StreamHandler
level=DEBUG
formatter=form01
args=(sys.stderr,)

[handler_tfh]
class=handlers.TimedRotatingFileHandler
level=DEBUG
formatter=form01
args=('tfh_log.log','S',1.5,2,)

[handler_rfh]
class=handlers.RotatingFileHandler
level=DEBUG
formatter=form01
args=('log.log','a',1024,2)

[formatters]
keys=form01

[formatter_form01]
format=%(asctime)s - %(levelname)s %(name)s %(filename)s [line:%(lineno)d] - %(message)s
```

在logutils.py中应用配置文件

```python
import logging.config
logging.config.fileConfig("logging.conf")
```

logutils.py中的代码就异常简单了，应为原来通过代码实现的配置，都写到了logging.conf配置文件中了。
![配置文件实现了日志代码中的功能](http://image2.ishareread.com/images/2023/20230319/12_配置文件实现了代码中一样的功能.png)


这里要注意的是，在应用日志记录器的时候，需要引用配置文件中配置的记录器，如配置文件中配置了root和logger01，在应用的时候可以引用这两个记录器，当然也可以在配置文件中配置更多的记录器。
在mian.py和othermodule.py中应用日志记录器的时候，需要注意记录器用要配置文件中定义的记录器，这里是logger01。

```python
logger=logging.getLogger('logger01')  #定义模块日志记录器
```
![定义模块日志记录器](http://image2.ishareread.com/images/2023/20230319/13_配置文件实现效果引用日志记录器.png)

### 8、日志中中文显示
当日志信息中有中文的时候，在控制台输出会自动的转码，但有时在文件输出的时候会出现乱码。
![中文信息控制台输出没有问题](http://image2.ishareread.com/images/2023/20230319/14_中文信息控制台输出没问题.png)

控制台输出中文，但日志文件中是乱码。
![日志文件中是乱码](http://image2.ishareread.com/images/2023/20230319/15_日志文件中乱码.png)

对照python官方说明文档设置编码，设置处理器的编码为UTF-8

```python
[handler_tfh]
class=handlers.TimedRotatingFileHandler
level=DEBUG
formatter=form01
args=('tfh_log.log','S',1.5,2,'UTF-8')

[handler_rfh]
class=handlers.RotatingFileHandler
level=DEBUG
formatter=form01
args=('log.log','a',1024,2,'UTF-8')
```
![配置文件的参数](http://image2.ishareread.com/images/2023/20230319/16_设置配置文件中的参数.png)
现在重新执行main.py，可以看到日志文件中可以正常显示中文。
![日志文件显示中文](http://image2.ishareread.com/images/2023/20230319/17_日志文件显示中文.png)


至此，我们从一个简单的日志记录实战，一步一步实现了自定义日志格式、写日志文件、抽出公共日志模块让其他模块用、同时写多个日志文件并进行日志文件切割、通过配置文件实现日志参数的定义、解决日志中文显示问题。基本覆盖了真实应用场景日志的使用。

------

博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>