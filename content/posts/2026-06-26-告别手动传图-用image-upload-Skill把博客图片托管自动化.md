---
title: 告别手动传图：用一个 image-upload Skill 把博客图片托管自动化
id: 20260626001
tags:
  - AI机器学习
categories:
  - 技术
slug: image-upload-skill
date: 2026-06-26 09:20:00
---

> 发博客最折腾的从来不是"写文字"，而是"弄图片"——逐张传图床、手动把 `![[图]]` 一处处改成图床 URL、漏改错改、重传一遍就在图床堆出一堆重复文件。这篇文章记录我怎么用一个 Claude Code Skill **`image-upload`**，把"扫描本地图片引用 → 按内容哈希去重上传 → 原地改写链接"这条链路压成一句话的事；配置自包含（不再绑死 PicGo 图形界面），重跑幂等不重复传，七牛为主、GitHub 为备。
<!-- more -->

# 🤔 痛点：每次发博客，图片都是体力活

我用 Obsidian 写博客，再通过 Hugo 发布。文章里插图很自然地写成 Obsidian 的本地引用：

```
![[scheme-writer-architecture.png]]
![[scheme-writer-architecture.png|架构图]]
```

但 Hugo 不认 `![[...]]` 这种 wiki 链接，它要的是标准 Markdown 的 URL：

```
![](http://image2.ishareread.com/images/.../scheme-writer-architecture.png)
```

于是每发一篇文章，都要重复一套体力活：

1. 把 `assets/blog/` 里的截图，**逐张**上传到图床（七牛云）；
2. 拿到 URL，**手动**把文章里的 `![[图]]` 一处处改成 `![](图床URL)`；
3. 漏改、改错、重名覆盖是常态；
4. 重新传一次？记不清哪张传过，只能再传一遍——图床里堆满重复文件。

以前这套流程靠 PicGo 的图形界面点来点去，后来我干脆让 AI 每次帮我传。但这还是「每次都要叮嘱一遍」的临时活儿，没有沉淀成能力。而且配置散落在 PicGo 里，万一哪天 PicGo 不装了，连密钥都找不回来。

> 📷 ：Obsidian 里一篇带 `![[...]]` 本地图片引用的博文草稿截图。

![](http://image2.ishareread.com/images/20260626/Pasted-image-20260626094434.png)


# 🎯 我想要什么样的传图方式

需求其实很朴素：

- **一条命令搞定**：别让我一张张点、一处处改；
- **AI 能直接调**：在 Claude Code 里说一句"把这篇的图传一下"就完事；
- **七牛为主、GitHub 为备**：七牛国内快，GitHub 免费可回溯；
- **配置不依赖 PicGo**：skill 要自洽，PicGo 装不装都能跑；
- **别重复传**：同一张图传过就别再传，URL 还能找回。

顺着这几个需求，我写了一个 Claude Code Skill：**`image-upload`**。这篇文章就讲它解决什么问题、怎么装、怎么用。

# 🧩 方案：image-upload skill 的架构

先看全貌：

![image-upload skill 架构图](http://image2.ishareread.com/images/20260626/arch.png)

整个 skill 是分层的，自上而下五层：

| 层 | 模块 | 职责 |
|----|------|------|
| 入口 | `main.py` | CLI 派发：`upload` / `migrate` / `init` / `list` |
| 编排 | `rewriter.py` / `manifest.py` / `keys.py` / `config.py` | 解析改写、去重账本、key 命名、配置加载 |
| 抽象 | `Uploader` 基类 | 对外只暴露一个接口 `upload(local_path, key) → URL` |
| 实现 | `QiniuUploader` / `GithubUploader` | 七牛 SDK / GitHub Contents API |
| 图床 | 七牛云 / GitHub 仓库 | 实际存储 |

**最关键的设计是「Provider 抽象」**：上层（`migrate` 命令）只认 `upload()` 这个接口，根本不在乎底下是七牛还是 GitHub。所以"多一个图床后端"只是多写一个 `Uploader` 子类，编排逻辑一行不改。这就是后面能轻松加上 GitHub 的原因。

配置全部在 skill 自己的 `.env` 里（自包含），`config.py` 加载并校验。**PicGo 只在首次 `init` 时当一次"密钥搬运工"**，搬完即可卸载，skill 此后完全独立。

# 🔧 安装

```bash
# 1. 拿到 skill（仓库里或同步到 ~/.claude/skills/image-upload/）
cd image-upload

# 2. 建虚拟环境、装依赖（含测试是 requirements-dev.txt）
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

# 3. 配置（二选一）
cp .env.example .env && $EDITOR .env          # 手填
# 或者——如果你机器上还有 PicGo，一行导入七牛配置：
.venv/bin/python main.py init
```

`init` 会读 PicGo 的 `data.json`，把里面的 `picBed.qiniu`（accessKey/secretKey/bucket/domain/area）一次性写进 `.env`：

```
[done] 已从 PicGo 导入七牛配置到 .../image-upload/.env
       bucket=ishareblog, domain=http://image2.ishareread.com, area=z2
       PicGo 此后可卸载，skill 已自包含。
```


# 🚀 使用：三个真实场景

## 场景一：单张上传（`upload` 原语）

最原始的能力——给我一个图片文件，还你一个图床 URL：

```bash
.venv/bin/python main.py upload assets/blog/foo.png --json
```

```json
[
  {
    "file": "/abs/path/foo.png",
    "url": "http://image2.ishareread.com/images/20260626/foo.png",
    "status": "uploaded",
    "backend": "qiniu"
  }
]
```

`--json` 输出结构化结果，方便脚本/AI 组合调用；不加就是一行 `本地路径 -> URL`。

这个原语的价值在于**可组合**：任何需要"文件→URL"的流程都能调它，不限于博客。

![](http://image2.ishareread.com/images/20260626/Pasted-image-20260626095304.png)

上传成功后直接返回可以用的外链URL

![](http://image2.ishareread.com/images/20260626/Pasted-image-20260626095417.png)
## 场景二：整篇博客迁移（`migrate`）——真正省事的命令

这才是解决"体力活"的核心。给一篇文章，它自动扫描所有本地图片引用、逐张上传、原地改写链接。

先 `--dry-run` 预览，不传不改：

```bash
.venv/bin/python main.py migrate notes/blog/posts/scheme-writer-guide.md \
    --dry-run --vault-root ~/my-knowledge-base
```

```
[dry-run] 将上传 scheme-writer-architecture.png -> qiniu:images/scheme-writer-guide/scheme-writer-architecture.png
[dry-run] 将上传 scheme-writer-workflow.png     -> qiniu:images/scheme-writer-guide/scheme-writer-workflow.png
[dry-run] 计划上传 2，manifest命中 0，已是URL跳过 1，缺失 0（未改写文件）
```

确认无误，去掉 `--dry-run` 真跑：

```bash
.venv/bin/python main.py migrate notes/blog/posts/scheme-writer-guide.md \
    --vault-root ~/my-knowledge-base
```

```
[done] 改写 2 处，上传 2，manifest命中 0，已是URL跳过 1，缺失 0
```

文章里的链接，**原地**从这样：

```
![[scheme-writer-architecture.png]]
![[scheme-writer-architecture.png|架构图]]
![已有图](https://image2.ishareread.com/old.png)
```

变成了这样：

```
![](http://image2.ishareread.com/images/scheme-writer-guide/scheme-writer-architecture.png)
![架构图](http://image2.ishareread.com/images/scheme-writer-guide/scheme-writer-architecture.png)
![已有图](https://image2.ishareread.com/old.png)
```

注意三件事都对了：

1. `![[x.png]]` → `![](URL)`，无 alt；
2. `![[x.png|架构图]]` → `![架构图](URL)`，**alt 文本保留了**；
3. 已经是 URL 的那行**原样不动**——不会重复上传。

整个 `migrate` 的工作流：

![migrate 命令工作流](http://image2.ishareread.com/images/20260626/flow.png)


## 场景三：切换到 GitHub 后端

同一篇文章，想改用 GitHub 当图床？加个 `--backend github`：

```bash
.venv/bin/python main.py migrate notes/blog/posts/scheme-writer-guide.md \
    --backend github --vault-root ~/my-knowledge-base
```

```
[done] 改写 2 处，上传 2，manifest命中 0，已是URL跳过 1，缺失 0
```

返回的 URL 变成 `https://raw.githubusercontent.com/<owner>/<repo>/main/images/...`。而且——**七牛那次的记录还在**，同一张图同时挂着两个后端的 URL，随时能切回来。

为什么 GitHub 默认走 `raw.githubusercontent.com`？因为博客默认后端仍是七牛（国内快），GitHub 是"免费/可回溯/给非国内读者"的备用。真要 GitHub 图也跑国内流量，把 `.env` 里的 `GH_DOMAIN` 指向一个 Cloudflare Worker 反代域名就行，**零代码改动**。

# 🧠 几个关键的设计决定

**1. 内容哈希去重，而不是按文件名**

manifest 按文件的 **sha256 内容哈希** 做索引，不是按文件名或路径。

- 同一张截图在多处引用 → 只传一次，省图床空间；
- 图片被重新编辑过（内容变了）→ 自动识别为新文件，重传新版本；
- 文件改名/移动 → 不影响，哈希一样还是命中。

这是"重跑不重复传、URL 可回溯"的基础。manifest 同时分桶记七牛和 GitHub 两条 URL，所以后端切换不重传。

**2. 配置自包含，不绑死 PicGo**

`.env` 存所有参数（七牛 + GitHub）。PicGo 仅在 `init` 时当一次密钥搬运工。这是有意的：我正在逐步脱离 PicGo 的图形界面，skill 不能因为 PicGo 不在就瘫掉。改密钥？在 `.env` 改就行，PicGo 不再是必经之路。

**3. 链接改写按 offset，不用字符串替换**

文章里同一张图引用多次时，朴素的 `str.replace` 会误伤。skill 记录每个引用在原文的精确偏移，**从后往前**逐个替换，重复引用也不会错位。

**4. 文件名自动净化，URL 永远安全**

Obsidian 粘贴的截图默认叫 `Pasted image 20260626094343.png` 这种带空格的名字，直接当 key 会让图床 URL 也带空格、贴进 Markdown 就断链。skill 在生成 key 时对文件名做净化：空格变 `-`、去掉 `()`/`#`/`?`/`&` 等 URL 不安全字符、折叠多余连字符，**中文保留**。于是 `Pasted image x.png` → `Pasted-image-x.png`，`图 1.png` → `图-1.png`——命名再随意也不会产出断链 URL。

**5. YAGNI：没做的事**

- 不做图形界面（Claude Code 就是界面）；
- 不做图片压缩/格式转换（需要时再加 `--optimize`）；
- 不做多用户鉴权（本地个人 skill）。

# ✅ 效果与小结

博客写完后截图都是本地引用，以前需要一张张图片上传，然后手动改链接。
![](http://image2.ishareread.com/images/20260626/Pasted-image-20260626101038.png)

这套东西落地后，发博客的图片流程从"逐张上传 + 手动改链接"压缩成了一条命令在Obsidian中对着Claudin说：

- 请将‘告别手动传图-用image-upload-Skill把博客图片托管自动化’博文中的图片上传到图床并更新链接

写完文章直接跑，链接自动改写好，Hugo 那边刷新就能看到图文并茂的成品。重跑安全（幂等），后端可切（七牛/GitHub），配置自洽（不依赖 PicGo）。

![](http://image2.ishareread.com/images/20260626/Pasted-image-20260626102644.png)

> 📷 **[截图]**：发布到 Hugo 后，浏览器里渲染出的图文博客效果（展示一篇有有截图的完整文章）。

![](http://image2.ishareread.com/images/20260626/Pasted-image-20260626103539.png)

**一个有点元的小事**：你正在看的这篇文章里的两张架构图（上面的架构图和 migrate 流程图），就是用这个 skill 自己传到图床的——`upload /tmp/arch.png /tmp/flow.png` 一行搞定，URL 拿来直接贴。自己造的轮子，自己先用起来。

如果你也在用 Obsidian + Hugo 写博客、又被图片托管折腾过，可以照着这个思路做一个。完整的代码和设计文档我都放在了 [myopenclaw-skills](https://github.com/xiejava1018/myopenclaw-skills) 仓库的 `image-upload/` 目录，下载后就可以直接用，设计文档在 `docs/plans/2026-06-25-image-upload-skill-design.md`，欢迎自取。

---
作者博客：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)
<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>
