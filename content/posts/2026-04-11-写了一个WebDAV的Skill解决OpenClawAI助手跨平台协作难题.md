---
title: 写了一个WebDAV的Skill解决OpenClawAI助手跨平台协作难题
id: 20260411001
tags:
  - AI机器学习
categories:
  - - 技术
    - 开发
abbrlink: cb78a2e5
date: 2026-04-11 21:42:31
---

# 🤔 为什么需要 WebDAV 技能？
# 痛点
在使用 OpenClaw 构建 AI 团队的过程中，我遇到了一个棘手的问题：人机协同跨平台协作断裂，AI agent、本地文件、异地远程访问三者之间缺少桥梁，OpenClawAI 助手生成的东西在本地，我无法拿到，只能让它发邮件或让它放云盘或NAS ，但是AI 助手无法直接访问 NAS 存储。
具体场景包括：
1. 文件中转：AI 生成的文档需要先下载到本地，再手动上传到 NAS，流程繁琐
2. 无法即时访问 NAS 资源：团队共享的模板、素材都在 NAS 上，AI 助手需要我手动搬运文件才能使用
3. 备份自动化困难：想定期把 AI 生成的工作成果自动备份到 NAS，但没有现成方案
4. 跨平台协作断裂：AI agent、本地文件、异地远程访问三者之间缺少桥梁

NAS作为个人的私有数据存储是家庭/团队存储的核心，有没有办法让它将工作成果直接放NAS上呢？这样既保密，空间又不受限制，资料共享也方便。如果能开发一个 OpenClaw 技能，让 AI 助手直接通过 WebDAV 读写 NAS 文件，以上问题就迎刃而解了。

# 解决思路
OpenClaw 的技能（Skill）系统天然支持扩展。WebDAV 又是标准的文件访问协议，几乎所有 NAS 都支持。两者结合，就能实现：
AI 助手 ↔ OpenClaw ↔ WebDAV 技能 ↔ NAS 存储

---
# 🎯 功能设计
基于实际需求，WebDAV 技能需要具备以下核心能力：

|功能    | 说明 |WebDAV 协议方法  |
|---|---|---|
|📂 列出目录|查看 NAS 目录下的文件和子目录|PROPFIND (Depth: 1)|
|📤 文件上传|将本地文件上传到 NAS 指定路径|PUT|
|📥 文件下载|从 NAS 下载文件到本地|GET|
|🗑️ 文件删除|删除 NAS 上的文件或目录|DELETE|
|📁 创建目录|在 NAS 上创建新目录|MKCOL|
|📊 文件属性|查看文件大小、修改时间、类型|PROPFIND (Depth: 0)|
|🔌 连接测试|快速验证 NAS 连接是否正常|PROPFIND (Depth: 0)|

# 关键特性
- 🔄 自动重试：HTTP 500/502/503/504 错误自动重试 3 次，退避策略避免雪崩
- 📈 进度条：上传下载大文件时实时显示进度，不再"干等"
- 🔐 双认证支持：Basic Auth + Digest Auth 自动降级
- 🛡️ SSL 灵活配置：默认开启证书验证，自签名环境可关闭
- 🇨🇳 中文路径友好：自动处理 URL 编码的中文路径
- 🗣️ 双模式调用：支持结构化命令和自然语言两种调用方式
- ⚙️ 首次引导配置：第一次使用自动进入交互式配置向导

---
# 🛠️ 技术实现
项目结构

```bash
webdav/
├── SKILL.md           # 技能描述和使用文档
├── main.py            # 核心代码（客户端 + 命令处理）
├── requirements.txt   # Python 依赖（requests）
├── README.md          # 项目说明
├── .env               # 配置文件（服务器地址、认证信息）
└── .venv/             # Python 虚拟环境
```

核心架构
代码分为三层：
**第一层：配置管理**

```bash
def load_config():
    """加载配置：环境变量 > .env 文件"""
    config = {}
    env_file = SKILL_DIR / ".env"
    if env_file.exists():
        with open(env_file, encoding="utf-8") as f:
            for line in f:
                if line.strip() and not line.startswith("#") and "=" in line:
                    key, value = line.split("=", 1)
                    os.environ.setdefault(key.strip(), value.strip())
    ...
```

配置加载优先级：构造函数参数 > 环境变量 > .env 文件 > 交互式向导。
**第二层：WebDAV 客户端**

```bash
class WebDAVClient:
    def __init__(self, server, username, password, verify_ssl):
        # 构建带重试的 session
        self.session = requests.Session()
        retry = Retry(total=3, backoff_factor=0.5,
                      status_forcelist=[500, 502, 503, 504])
        adapter = HTTPAdapter(max_retries=retry)
        self.session.mount("http://", adapter)
        self.session.mount("https://", adapter)
```

每个操作（list/upload/download/delete/mkdir）对应一个 WebDAV HTTP 方法。
**第三层：命令处理**
支持两种调用模式：

```bash
# 结构化命令
"upload local=/tmp/report.pdf remote=openclaw_sharedoc/report.pdf"

# 自然语言命令
"上传 /tmp/report.pdf 到NAS openclaw_sharedoc/"
```

通过正则表达式匹配自然语言，转为结构化调用。
上传下载的进度条实现

```bash
def upload_file(self, local_path, remote_path, progress=True):
    file_size = os.path.getsize(local_path)
    def gen():
        with open(local_path, "rb") as f:
            uploaded = 0
            chunk_size = 64 * 1024  # 64KB 分块
            while True:
                chunk = f.read(chunk_size)
                if not chunk:
                    break
                uploaded += len(chunk)
                yield chunk
                if progress and file_size > 0:
                    pct = min(uploaded / file_size * 100, 100)
                    bar = "█" * int(pct // 5) + "░" * (20 - int(pct // 5))
                    sys.stdout.write(f"\r  上传 {filename}: [{bar}] {pct:.1f}%")
```

使用 Python generator 实现流式上传，64KB 分块读写，兼顾内存效率和进度展示。

---
# 📦 准备工作与安装部署
## 	一、准备工作
前提是你的NAS 要开通配置WebDAV的文件共享协议，以下以飞牛NAS为例来配置WebDAV。
1.打开系统设置 > 2.找到文件共享协议 > 3.切换到WebDAV > 4.开启WebDAV  > 5.设置可文件夹的范围（也就是设置共享目录）
![\[图片\]](http://image2.ishareread.com/images/2026/20260411/1-%E9%A3%9E%E7%89%9Bnas%E7%9A%84WEBDAV%E9%85%8D%E7%BD%AE.png)

配置好WebDAV服务后，可以配置ddns，让外网可以访问，如果不配就只能通过fnos提供的FN Connect来外网访问了。
![\[图片\]](http://image2.ishareread.com/images/2026/20260411/2-%E9%A3%9E%E7%89%9Bnas%E9%85%8D%E7%BD%AEDDNS.png)

配置好了WebDAV服务后就可以通过任意的终端来访问WebDAV的共享目录了。
ubuntu的WebDAV客户端的配置：
![\[图片\]](http://image2.ishareread.com/images/2026/20260411/3-ubuntu%E9%85%8D%E7%BD%AEwebdav.png)
![在这里插入图片描述](http://image2.ishareread.com/images/2026/20260411/4-ubuntu%E7%9C%8B%E5%88%B0webdav%E7%9A%84%E5%85%B1%E4%BA%AB%E7%9B%AE%E5%BD%95.png)

MAC OS的WebDAV客户端的配置：
![\[图片\]](http://image2.ishareread.com/images/2026/20260411/5-mac%E9%85%8D%E7%BD%AEwebdav%E8%BF%9E%E6%8E%A5.png)
![\[图片\]](http://image2.ishareread.com/images/2026/20260411/6-mac%E9%85%8D%E7%BD%AEwebdav%E7%9A%84%E5%9C%B0%E5%9D%80.png)

![\[图片\]](http://image2.ishareread.com/images/2026/20260411/7-mac%E7%9C%8B%E5%88%B0webdav%E7%9A%84%E5%85%B1%E4%BA%AB%E7%9B%AE%E5%BD%95.png)

## 二、安装部署
1. 获取技能代码

```bash
# 从 GitHub 克隆
cd /tmp
git clone https://github.com/xiejava1018/myopenclaw-skills.git

# 复制到 OpenClaw 技能目录
mkdir -p ~/.openclaw/skills
cp -r /tmp/myopenclaw-skills/webdav ~/.openclaw/skills/webdav
```

2. 安装 Python 依赖

```bash
# 安装 venv 支持（Ubuntu/Debian）
sudo apt install python3.12-venv

# 创建虚拟环境并安装依赖
cd ~/.openclaw/skills/webdav
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

3. 配置 NAS 连接
创建 .env 配置文件：

```bash
cat > ~/.openclaw/skills/webdav/.env << 'EOF'
WEBDAV_SERVER=https://192.168.0.18:5006
WEBDAV_USERNAME=your_username
WEBDAV_PASSWORD=your_password
WEBDAV_SSL_VERIFY=false
EOF
```

4. 测试连接

```bash
cd ~/.openclaw/skills/webdav
WEBDAV_SERVER=https://192.168.0.18:5006 \
WEBDAV_USERNAME=your_username \
WEBDAV_PASSWORD=your_password \
WEBDAV_SSL_VERIFY=false \
.venv/bin/python3 main.py test
```

输出：

```bash
✅ WebDAV 连接正常
   服务器: https://192.168.0.18:5006
   用户名: xiejava
   HTTP 状态: 207
   SSL 验证: 关闭
```

对于OpenClaw或Claude Code来说根本就不要这么麻烦，直接将skill的github地址[https://github.com/xiejava1018/myopenclaw-skills/tree/master/webdav](https://github.com/xiejava1018/myopenclaw-skills/tree/master/webdav) 发给它让它自己装就好了。

![在这里插入图片描述](http://image2.ishareread.com/images/2026/20260411/8-openclaw%E5%AE%89%E8%A3%85webdav%E6%8A%80%E8%83%BD.png)
在安装过程中会询问配置NAS的WebDAV地址、用户名和密码等连接信息，你可以告诉OpenClaw让它自动配置，当然你也可以不告诉它，自己手动在`.env`文件里去配置。
配置好后可以测试一下，让它列出NAS的Webdav共享的目录。
![\[图片\]](http://image2.ishareread.com/images/2026/20260411/9-openclaw%E9%85%8D%E7%BD%AE%E5%92%8C%E6%B5%8B%E8%AF%95webdav%E6%8A%80%E8%83%BD.png)


---
# 📖 使用指南
 安装完成后就可以在OpenClaw或ClaudeCode中通过自然语言让AI agent直接它通过WebDAV操作NAS的共享目录了。
 
  **自然语言使用**
WebDAV 技能支持中文自然语言指令：
通过中文自然语言就可以告诉AI agent 操作NAS的webDAV的目录，写文件等。
**列出NAS共享目录
列出NAS目录 openclaw_sharedoc
上传 /tmp/file.txt 到NAS openclaw_sharedoc/
下载NAS文件 openclaw_sharedoc/file.txt 到 /tmp/
删除NAS文件 openclaw_sharedoc/file.txt
创建NAS目录 openclaw_sharedoc/new_folder
查看NAS文件 openclaw_sharedoc/file.txt 的信息
测试NAS连接**


这样通过NAS的webDAV服务， AI agent 、本地电脑、异地的电脑，都可以通过webDAV服务共享文件。解决了AI团队跨平台协作的问题。
如：有个文件要处理，可以将这个文件放到webDAV共享目录，让openclaw或claude code处理，处理完后实时可以通过访问webDAV共享目录看到处理后的成果;要openclaw去网上收集材料整理文档后，让它将成果文档放到webDAV共享目录，你就可以实时看到它的成果等等。

---
# 🌟 实战案例

**案例 1：AI 生成文档自动备份到 NAS，作为团队共享**
每天AI agent的工作成果自动保存到 NAS，不仅仅在AI agent的本地工作目录，实现了离线备份与团队共享：
**效果**：AI 助手生成的工作都自动归档到 NAS，再也不怕数据丢失，实现了离线备份与团队共享。

**案例 2：NAS 素材文件快速分发**
团队在 NAS 上共享了模板文件，AI 助手可以直接读取：
**效果**：AI 助手直接从 NAS 拉取素材，无需人工中转。

**案例 3：定期清理 NAS 过期文件**
配合 OpenClaw 的定时任务（HEARTBEAT.md），实现定期清理：
**效果**：NAS 存储空间自动管理，不需要手动清理。

# 实战效果
我的OpenClaw是部署在云服务器上面的，我让OpenClaw写了篇博客文章， 让它形成.md的文档放到NAS的webDAV知识库的共享目录，OpenClaw自动调用了WebDAV的skill上传到了NAS的共享目录。
![\[图片\]](http://image2.ishareread.com/images/2026/20260411/10-webdav%E5%AE%9E%E6%88%98%E5%86%99%E5%85%A5nas.png)

我本地的Ubuntu机器，就可以通过Trae直接访问NAS上的文档进行修改完善。
![\[图片\]](http://image2.ishareread.com/images/2026/20260411/11-webdav%E5%AE%9E%E6%88%98%E9%80%9A%E8%BF%87trae%E8%AE%BF%E9%97%AEwebdav%E7%9A%84%E6%96%87%E6%A1%A3.png)

直接打开NAS上的文档进行编辑完善。
![\[图片\]](http://image2.ishareread.com/images/2026/20260411/12-webdav%E5%AE%9E%E6%88%98%E9%80%9A%E8%BF%87trae%E6%89%93%E5%BC%80nas%E5%85%B1%E4%BA%AB%E6%96%87%E4%BB%B6.png)
这样极大的提高了和AI agent人、机协同的效率。

---
📊 兼容性
WebDAV 是开放标准协议，该技能兼容所有支持 WebDAV 的设备和服务：

|设备/服务| 兼容性|
|---|---|
|群晖 (Synology)|✅|
|威联通 (QNAP)|✅|
|飞牛云 (fnOS)|✅ |
|Nextcloud| ✅ |
|ownCloud |✅ |
|RAIDrive | ✅ |
|macOS Server| ✅ |
|IIS WebDAV | ✅ |

---
# 💡 总结与展望
WebDAV 技能解决了 AI 助手与 NAS 存储之间的鸿沟，让文件管理变得自然流畅。核心价值：
1. 消除文件中转：AI 助手直接读写 NAS，省去手动搬运
2. 统一存储管理：NAS 成为 AI 团队的共享文件中心
3. 自动化备份：AI 生成的内容可以自动归档到 NAS
4. 开放兼容：基于标准 WebDAV 协议，兼容所有主流 NAS

未来可以继续增强的方向：
- 支持文件搜索（按名称/时间/类型）
- 增加同步功能（双向同步指定目录）
- 集成到 OpenClaw 定时任务，实现全自动备份
- 支持多 WebDAV 服务器配置切换



📝 技能开源地址：[https://github.com/xiejava1018/myopenclaw-skills/tree/master/webdav](https://github.com/xiejava1018/myopenclaw-skills/tree/master/webdav)   
直接让OpenClaw或Claude Code安装后就可以使用。

---


作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)
<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>