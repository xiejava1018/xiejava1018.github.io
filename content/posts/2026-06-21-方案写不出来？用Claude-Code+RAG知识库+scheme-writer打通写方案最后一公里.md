---
title: 方案写不出来？用 Claude Code+RAG知识库+scheme-writer打通写方案最后一公里
tags:
  - Claude-Code
  - Skill
  - RAG
  - 知识库
  - 方案写作
  - 效率工具
categories:
  - AI应用
  - 效率工具
slug: 0f0814f2
date: 2026-06-21T14:00:00+08:00
---

> 写技术方案最痛苦的从来不是"写"本身，而是写之前的"找"——找规范、找制度、找历史方案、找上次评审的修改意见。这篇文章记录我怎么用 Claude Code Skill + WeKnora RAG，把"找资料→写方案→归档"这条链路压缩成一句话的事。

<!-- more -->

## 一、先说痛点：写方案的时间，一半花在找资料上

在企业里写过方案的人应该都有这个感受：

**不是写不出来，是素材找不到。**

运维规范不知道在哪台服务器的哪个文件夹，上次的安全制度修订版还在张三的邮箱里，去年写过的类似方案……忘了存哪儿了。每次接一个"写份 XX 方案"的需求，光"找齐素材"这一步就能花掉半天。

找完之后还不算完：

- **格式不统一**：每个人写的方案结构不同，评审时反复打回；
- **知识不沉淀**：方案写完存在本地，下次同事遇到类似需求，又得从头找；
- **知识库形同虚设**：公司部署了 WeKnora / RAG 平台，但没人主动往里查、往里存，内容越来越旧。

**核心矛盾**：企业知识库里有数据，但"检索 → 写作 → 归档"这条链路太长、太割裂，人不愿意走完整个闭环。

> 如果能把这三步压缩成一句话呢？

---

## 二、解决方案：给 Claude Code 装一个"知识库遥控器"

我写的方案是 `scheme-writer`，一个 **Claude Code Skill**——你可以理解为 Claude Code 的"插件"，让 AI Agent 获得连接企业知识库（WeKnora）的能力。

这个skill我已经开源，放到github上：[https://github.com/xiejava1018/myopenclaw-skills/blob/master/scheme-writer](https://github.com/xiejava1018/myopenclaw-skills/blob/master/scheme-writer/SKILL.md)

一句话就能启动完整流程：

```
帮我参考知识库中的资料写一份《风险监测的管理制度》方案，写完归档到《02_管理制度》
```

AI 自动完成全部工作：

![scheme-writer工作流时序图](http://image2.ishareread.com/images/blog/scheme-writer-workflow-sequence.png)

整个过程，只说了**一句话**。AI 帮你做了三件事：

| 步骤 | AI做了什么 | 用户需要做什么 |
|------|----------|---------------|
| **检索** | 从 WeKnora 语义检索相关文档片段 | 什么都不用做 |
| **写作** | 基于真实素材生成结构化方案 | 确认大纲，审核终稿 |
| **归档** | 方案自动上传回知识库 | 确认一下就完事 |

> 📌 插图：AI 通过 scheme-writer 的skill 生成的《风险监测的管理制度》目录结构截图 

![风险监测管理制度目录结构](http://image2.ishareread.com/images/blog/scheme-writer-risk-doc-toc.png)

> 📌 插图：scheme-writer 写完方案后归档到 企业知识库（WeKnora）形成知识的截图

![方案归档到WeKnora知识库](http://image2.ishareread.com/images/blog/scheme-writer-archive-to-weknora.png)

可以看到通过scheme-writer到知识库中找资料进行编排方案文档，比人去写方案要全面、严谨、效率更好，还可以很直观的看到编写的文档参考资料都来自哪里，使编写的方案更权威。

---

## 三、架构长什么样

`scheme-writer` 的核心是让 Claude Code 通过一组 Python 脚本与 WeKnora 的 REST API 对接，实现语义检索、文档上传、知识库管理等功能。

![scheme-writer三层架构](http://image2.ishareread.com/images/blog/scheme-writer-architecture.png)

**三层架构**：

- **对话层**：用户在 Claude Code 里自然语言输入，AI 理解意图后调用脚本；
- **脚本层**：`kb_search`（检索）、`kb_list`（列库）、`kb_docs`（文档清单）、`kb_upload`（上传）、`init`（配置），各司其职；
- **API 层**：通过统一的 `kb_http` 客户端与 WeKnora 通信，错误归一化、认证统一处理。

对于用户来说，这三层是透明的——你只需要说话，AI 帮你调库。

---

## 四、多来源怎么办

现实世界里，知识库往往不止一个：运维一个空间、安全一个空间、开发一个空间，各自独立部署，各自独立认证。

`scheme-writer` v1.3 支持**同时连接多个来源**，一句话就能跨库检索：

```
从运维空间的《技术规范库》和安全部的《漏洞库》一起查 K8s 相关风险
```

AI 会从两个不同部门的知识库同时检索，合并结果后综合分析：

![多来源跨库检索架构](http://image2.ishareread.com/images/blog/scheme-writer-multi-source.png)

**部分失败不致命**：如果某个来源临时挂了（网络抖动 / Key 过期），其他来源的结果照常返回，不会因为一个部门的服务抖动导致整个方案写不出来。

---

## 五、四个实际使用场景

### 场景 1：一句话写方案（最常用）

```
请参考xiejava的个人知识库中方案库的内容，帮我规划一个LLM wiki知识库的方案。写完归档到《方案库》
```

AI 自动：检索 → 写大纲 → 你确认 → 展开完整方案 → 确认归档 → 上传。

![场景1：生成LLM wiki方案](http://image2.ishareread.com/images/blog/scheme-writer-scene1-draft.png)

写完后，会和你确认要不要归档到WeKnora。

![场景1：确认归档到WeKnora](http://image2.ishareread.com/images/blog/scheme-writer-scene1-confirm-archive.png)

scheme-writer 自动完成上传入知识库

![场景1：自动上传知识库](http://image2.ishareread.com/images/blog/scheme-writer-scene1-upload-done.png)

然后就可以在WeKnora知识库中看到并且可以在后续直接查询和引用了。

![场景1：WeKnora中查看方案](http://image2.ishareread.com/images/blog/scheme-writer-scene1-view-in-kb.png)

### 场景 2：跨部门知识融合

安全部门的漏洞库和运维部门的技术规范库是两个独立空间，但写安全运营方案时两边都得查。一条指令搞定：

```
从安全部门的《漏洞库》和运维空间的《技术规范库》一起查漏洞修复流程，写一份漏洞管理方案
```

可以看到scheme-writer 从多个知识库空间及网上搜集参考资料形成方案文档。

![场景2：跨部门多库检索](http://image2.ishareread.com/images/blog/scheme-writer-scene2-multi-search.png)

在方案中列出了资料多个来源的出处，包括参考的行业标准的等。

![场景2：方案列出资料来源](http://image2.ishareread.com/images/blog/scheme-writer-scene2-sources.png)

最后看归档到知识库中的效果

![场景2：归档到知识库效果](http://image2.ishareread.com/images/blog/scheme-writer-scene2-archive.png)

### 场景 3：知识库盘点

新接手一个项目，不知道知识库里都存了什么：

```
列一下所有来源的库，看看都有些什么
```

AI 执行 `kb_list.py --all`，列出全部来源的所有知识库，包括文档数、chunk 数、连通状态。

![场景3：知识库盘点列表](http://image2.ishareread.com/images/blog/scheme-writer-scene3-inventory.png)

### 场景 4：诊断"为什么搜不到"

知识库里有文档，但检索返回空？可以通过scheme-writer 对知识库进行健康检查 AI 自动调 `kb_docs.py` 检查每个文档的 `parse_status`，定位是"文档还在解析中"还是"向量化索引有问题"。

```
请对 xiejava的个人知识库 进行健康检查
```

![场景4：知识库健康检查](http://image2.ishareread.com/images/blog/scheme-writer-scene4-healthcheck.png)

---

## 六、效果对比

| 维度 | 传统方式 | scheme-writer |
|------|----------|---------------|
| **找资料** | 手动翻多个系统，30min+ | 一句话语义检索，10s |
| **写方案** | 从空白页开始，2-4h | 基于真实素材生成，含审核 30min |
| **归档** | 手动上传，经常忘 | AI 提示归档，确认即完成 |
| **格式** | 各写各的 | 统一 8 章节骨架 |
| **知识复用** | 下次重新找 | 已归档，下次检索命中 |

---

## 七、快速上手

### 前置条件

- 已安装 [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- 已部署 [WeKnora](https://github.com/Tencent/WeKnora) 实例（参见我的上一篇文章）
- Python 3.10+

### 安装

```bash
cp -r scheme-writer ~/.claude/skills/
```

安装完成后就可以在claude code中直接使用这个scheme-writer的skill技能

![安装：Claude Code中加载skill](http://image2.ishareread.com/images/blog/scheme-writer-skill-loaded.png)


### 首次使用

直接在 Claude Code 里说：

```
帮我写一份 XX 方案
```

Claude 会自动检测配置状态，**在对话内引导你完成首次来源配置**——全程不需要切终端，像填表一样一步步完成。

![首次使用：对话内配置引导](http://image2.ishareread.com/images/blog/scheme-writer-init-guide.png)

### 配置多来源

![配置多来源](http://image2.ishareread.com/images/blog/scheme-writer-multi-source-config.png)

```
加一个来源：安全部门的知识库
```

Claude 会引导输入名称、URL、API Key，然后自动探测连通性、发现可用知识库、设置别名。
![添加新来源输入](http://image2.ishareread.com/images/blog/scheme-writer-add-source.png)

这个URL和API Key是WeKnora的知识库空间的API 和 API Key ，具体可以在WeKnora的知识库空间的API信息中找到。

![WeKnora知识库空间API信息](http://image2.ishareread.com/images/blog/scheme-writer-weknora-api-info.png)


![WeKnora API Key位置](http://image2.ishareread.com/images/blog/scheme-writer-api-key-location.png)

---

## 八、写在最后

`scheme-writer` 解决的不是"AI 会不会写方案"的问题——Claude 本身就能写方案。它解决的是**最后一公里的断点**：

1. **素材在哪？** → 语义检索，AI 帮你找；
2. **格式怎么统一？** → 内置方案骨架，保证一致；
3. **写完怎么沉淀？** → 一键归档，知识自增长。

> 好的知识管理不是让人适应系统，而是让系统适应人的工作流。给 Claude Code 装一个"知识库遥控器"，你会发现——方案写起来快了，知识库也终于"活"了。

这个skill我已经开源，放到github上：[https://github.com/xiejava1018/myopenclaw-skills/blob/master/scheme-writer](https://github.com/xiejava1018/myopenclaw-skills/blob/master/scheme-writer/SKILL.md)  欢迎大家使用并提建议。

---

**相关阅读**：

- 上期：[腾讯开源 WeKnora 知识库部署实战（含踩坑排查）](https://xiejava.ishareread.com/posts/a3f7c9e2/)
- 上期：[让AI帮你养知识库LLM-Wiki与Obsidian和Claude的实践](http://xiejava.ishareread.com/posts/c5d8a1b3/)
- [WeKnora — 腾讯开源 LLM 知识框架](https://github.com/Tencent/WeKnora)
- [Claude Code Skills 文档](https://docs.anthropic.com/en/docs/claude-code)

如果你也在用 Claude Code，又正好有个想用起来的知识库，不妨试试这个思路。有问题欢迎留言交流。
