---
title: 【一个人的SOC】(03) Wazuh 接入实战——把第一台主机纳管进你的 SOC
tags:
  - 网络安全
  - Wazuh
  - 主机入侵检测
  - 安全运营
  - AI应用
categories:
  - - 技术
    - 网络安全
slug: f8d2a47c
date: 2026-10-08 11:35:00
description: "从一台裸机到 SOC 收到它的第一条告警：装 agent、跑通 wazuh-collector、Webhook 实时同步的取舍、踩过的 3 个真坑（API Key 不匹配/脚本权限错/${VAR} 密码解析）。本篇正式收编 7 篇旧 Wazuh 教程。"
---

[上一篇](/posts/a9c4e6f2/)我们把 AI-miniSOC 跑起来了——登录进控制台、看到 11 个菜单、概览仪表盘上数字在动。但说实话，**那一刻它还不是一个 SOC**——它只是一个"能用的平台"。

SOC 的真正起点，是**第一台主机的告警流进你的控制台的那一刻**。

> 那一刻你会盯着屏幕，意识到："这台机器刚刚被人扫了 SSH——我是在它真被人扫的时候知道的，不是事后看日志回放。"

本篇接住 [上一篇 (02) 部署完成之后：先做这三件事](/posts/a9c4e6f2/#部署完成之后先做这三件事) 里挖的坑——用四步把第一台主机纳管进你的 SOC。

老规矩，先看路线：

![Wazuh 接入路线图：四步把第一台主机纳管进你的 SOC，每步带过关标准](https://image2.ishareread.com/images/20261008/soc03-01-Wazuh接入路线图.png)

四步、顺利的话 30 分钟、最慢 1 小时。和上一篇一样：**每步都标了"过不过"的标准**，出问题时你永远知道卡在哪一格。

---

## 为什么要先讲 Wazuh？

一个事实：本系列和我的 [Wazuh 系列](https://blog.csdn.net/fullbug/category_7337204.html)都把 Wazuh 当作"主机入侵检测"的事实标准。原因很简单——

- **开源免费 + 商业化质量**：规则库覆盖 90% 的常见威胁（暴力破解、文件完整性、Rootkit、SCA 配置评估……），维护活跃；
- **架构上与 AI-miniSOC 天然契合**：告警结构化、可通过 OpenSearch 暴露给上层做聚合分级；
- **国内有完整中文文档社区**，Wazuh Manager + Agent + Indexer 三件套是经过验证的组合。

所以**第一台纳管的主机，强烈建议从 Wazuh 开始**。其他 HIDS（OSSEC、Suricata 主机的部分）后面单独讲——本篇假设你用 Wazuh。

---

## 第 0 步：基础检查（2 分钟）

开始前确认两件事（上一篇已经完成的事）：

- AI-miniSOC 跑着：浏览器打开 `http://localhost:3006` 能进控制台；
- Wazuh 服务在跑：浏览器打开 `https://<wazuh-server>:443` 能进 Wazuh Dashboard。

如果还没装 Wazuh，参考我的旧教程系列第一篇（[《开源安全管理平台 Wazuh——安装与配置》](https://xiejava.blog.csdn.net/article/details/152279929)），或者直接用 `install.sh` 装基础架构。

> 💡 **这一步为什么单列**：很多新手的"接入失败"其实不是接错，而是**前置环境没就绪**——登录页能进、但 Wazuh 那边 Dashboard 崩着。提前 2 分钟确认，把锅甩不到接入了。

**过关标准**：Wazuh Dashboard 能正常登录（`wazuh-wui` 用户即可）。

---

## 第 1 步：装 agent（5 分钟）

在**要被监控的那台主机**上装 agent，**不是**在 AI-miniSOC 服务端。装之前需要从 Wazuh Manager 拿到一个"注册 key"或直接用 Manager 的 IP/主机名。

```bash
# Linux 主机一行装
curl -o wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.13.0-1_amd64.deb
sudo WAZUH_MANAGER="<wazuh-server-ip>" dpkg -i ./wazuh-agent.deb
sudo systemctl enable --now wazuh-agent
```

```powershell
# Windows
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.13.0-1.msi -OutFile wazuh-agent.msi
.\wazuh-agent.msi /q WAZUH_MANAGER="<wazuh-server-ip>"
Start-Service -Name Wazuh
```

**过关标准**：登录 Wazuh Dashboard，**Server management → Endpoints → Summary** 列表里能看到这台新 agent，状态是 `green`（active）。同时**记下这台 agent 的 Agent ID**（本例是 **025**，AI-miniSOC 同步告警时会用到）：

![Wazuh Dashboard：第 025 号 agent 显示为 active 状态](https://image2.ishareread.com/images/20261009/s1-agent-dashboard.png)

> 🕳️ **坑 #1**：`WAZUH_MANAGER` 写错或者防火墙阻断了 **1514/UDP**（agent 注册端口）—— agent 进程在跑但 Dashboard 上一直是 `Disconnected`。排查：`sudo tail -f /var/ossec/logs/ossec.log`，看是不是一直在 `Trying to register...`。

---

## 第 2 步：让 AI-miniSOC 能看到 Wazuh——取舍一

主机装好了，但这台主机的告警现在还**只**活在 Wazuh Dashboard 里。AI-miniSOC 怎么知道这台主机的存在？告警怎么流过来？

**两种方式**：

![取舍一：Webhook 实时同步 vs 手动全量同步——选哪个？](https://image2.ishareread.com/images/20261008/soc03-02-同步方式取舍.png)

| 维度 | 🟢 方式 A：Webhook 实时同步 | 🔵 方式 B：手动全量同步 |
|---|---|---|
| **触发时机** | 告警一产生，几秒内送达 | 1~5 分钟轮询一次 |
| **配置位置** | **Wazuh 端**（写集成脚本 + ossec.conf） | **AI-miniSOC 端**（wazuh-collector 容器） |
| **复杂度** | 高：要写 Python 集成脚本 + Wazuh 端配 hook + 后端加 API Key 验签 | 低：起个容器，配个 .env 就完事 |
| **Wazuh API 压力** | 几乎无（事件级） | 有（按周期拉全量，agent 多时容易爆速率限制） |
| **对网络的要求** | Wazuh 必须能反向连回 AI-miniSOC | AI-miniSOC 出向拉（01 篇的"拉模型"红利） |
| **告警实时性** | 秒级 | 分钟级 |

**取舍建议（来自我的踩坑）**：

> **先用方式 B 跑起来**（最小阻力，业务跑稳了再升级到方式 A）。
>
> 原因：你刚部署完平台，第一天大概率连三台主机都没有，告警分钟级到达完全够用。等你真正感受到"告警到我手里晚了一分钟"那条痛线，再花一晚上配 Webhook 不迟——那时候你已经知道自己的真实需求了。

这一篇接下来**两个都讲**——B 是默认路径（你应该走的），A 是进阶路径（你想做"实时 SOC"再走）。先讲 B。

### 方式 B（默认路径）：启动 wazuh-collector

回到 [上一篇 (02) 部署的根目录](/posts/a9c4e6f2/)，采集器层是 Docker Compose 起的。Wazuh 采集器是三个 service 之一：

```bash
cd src/collectors
cat docker-compose.yaml | grep -A 5 wazuh-collector   # 确认它在列表里
```

把 Wazuh 接入凭据写到采集器专属 `.env`（**注意：这不是后端那个 .env，是采集器自己的**）：

```bash
# src/collectors/.env
WAZUH_API_URL=https://<wazuh-server>:55000
WAZUH_API_USERNAME=wazuh-wui
WAZUH_PASSWORD=<wazuh-dashboard-密码>
MINISOC_API_KEY=<你在 AI-miniSOC 控制台生成的 Key>
```

启动采集器：

```bash
docker compose up -d wazuh-collector
docker compose logs -f wazuh-collector
```

**过关标准**：日志里看到 `registered to AI-miniSOC backend, agent count: N` 之类的成功消息；同时 `soc_assets` 表里应该出现这台新装 agent 的主机记录（你可以在 AI-miniSOC 控制台"资产管理"里搜到）。

比如 **pve-LXC-wazuh-Ubuntu01** 这台新接进来的主机：

![AI-miniSOC 资产管理页：新接入的 Wazuh agent 主机出现在资产表（风险分 34、在线）](https://image2.ishareread.com/images/20261009/s3-ai-assets.png)

点开它的资产详情，可以看到 **Wazuh Agent ID = 025**——这正是 Wazuh Dashboard 里那个编号，采集器正确地把两端的主机对应起来了：

![AI-miniSOC 资产详情：Wazuh Agent 字段记录了 025](https://image2.ishareread.com/images/20261009/s2-collector-config.png)

再翻到"数据来源"tab，可以看到这行 `来源 = Wazuh / 状态 = 在线`——AI-miniSOC 不仅知道有这台主机，还知道它的数据是从哪个数据源来的：

![AI-miniSOC 数据来源 tab：标明此主机来自 Wazuh 采集器](https://image2.ishareread.com/images/20261009/s5-source-wazuh.png)

> 🕳️ **坑 #2（血的教训）**：8 月 23 日我们栽过一个跟头——`wazuh-collector` 的配置用 `yaml.safe_load` 加载，但它**不展开 `${VAR}` 占位符**。结果密码字面量 `${WAZUH_PASSWORD}` 当值发出去，Wazuh API 返回 401，采集器一直连不上。**修法**：采集器框架加了 `resolve()` 函数，env 优先 → YAML 兜底，占位符没 default 直接抛错（让问题早暴露）。所以——你如果用的不是最新代码、或者自己有定制 yaml，先确认 `${VAR}` 都被解析了。
>
> 🕳️ **坑 #3**：`WAZUH_PASSWORD` 错配也会 401，但**报错信息不一致**——Wazuh 4.x 会返回 `Invalid credentials`，4.7 之前可能只返回 `Unauthorized`。养成习惯：**用 `wazuh-wui` 账号测一次原 API**（`curl -k -u "wazuh-wui:password" https://<wazuh-server>:55000/security/user/authenticate`），通不了就别怪采集器。

### 方式 A（进阶路径）：Webhook 实时同步

当分钟级延迟开始影响你的响应节奏（通常是等保来袭、或者你想做告警时间窗口在 1 分钟内的合规报告时），可以升级到 Webhook。

**先决条件**：Wazuh Server 必须能反向连到 AI-miniSOC（**拉模型架构的红利在这里反过来**——AI-miniSOC 不能躲在 NAT 后面，必须有公网或内网可达地址）。生产部署时，AI-miniSOC 通常在堡垒机/VPC 内部，Wazuh 端能访问即可。

**步骤 1：写 Wazuh 集成脚本**（在 Wazuh Server 上）：

```bash
sudo tee /var/ossec/integrations/custom-minisoc << 'EOF'
#!/usr/bin/env python3
import sys, json, httpx, logging
logging.basicConfig(filename='/var/log/wazuh/integrations.log', level=logging.INFO)

def main():
    alert_file = sys.argv[1]
    api_key = sys.argv[2]
    hook_url = sys.argv[3]
    with open(alert_file) as f:
        alert = json.load(f)
    payload = {
        "agent_id": alert.get('agent', {}).get('id'),
        "agent_name": alert.get('agent', {}).get('name'),
        "rule_id": alert.get('rule', {}).get('id'),
        "level": alert.get('rule', {}).get('level'),
        "alert": alert
    }
    r = httpx.post(hook_url, json=payload,
                   headers={"X-API-Key": api_key}, timeout=5)
    logging.info(f"agent={payload['agent_id']} rule={payload['rule_id']} status={r.status_code}")

if __name__ == "__main__":
    main()
EOF
sudo chmod 750 /var/ossec/integrations/custom-minisoc
sudo chown root:wazuh /var/ossec/integrations/custom-minisoc
```

**步骤 2：配 `ossec.conf`**——在 `<ossec_config>` 里加：

```xml
<integration>
  <name>custom-minisoc</name>
  <hook_url>http://<ai-minisoc-server>:8000/api/v1/webhooks/wazuh</hook_url>
  <api_key><生成的 API Key></api_key>
  <alert_format>json</alert_format>
  <rule_id>504,505,506,510,511,550,553</rule_id>
</integration>
```

**步骤 3：后端 `.env` 接收 webhook**：

```bash
# 后端 .env（src/backend/.env，不是采集器那个）
WAZUH_WEBHOOK_KEY=<步骤 1 的 api_key，必须一致>
WAZUH_WEBHOOK_ALLOWED_IPS=<wazuh-server-ip>
```

**步骤 4：重启 Wazuh manager** 让集成生效：

```bash
sudo systemctl restart wazuh-manager
sudo tail -f /var/log/wazuh/integrations.log   # 看推送有没有失败
```

**过关标准**：故意停掉一台 agent（`sudo systemctl stop wazuh-agent`），**几秒内** AI-miniSOC 告警页出现"agent disconnected"事件。实时性达成。

> 🕳️ **坑 #4**：API Key 字符里有 `&` `=` 这类 URL 保留字符，但 Wazuh 集成脚本通过 `sys.argv` 接收——**不会做 URL 编码**。所以生成 Key 时**避免特殊字符**，或者在集成脚本里 `urllib.parse.quote` 处理。

---

## 第 3 步：验收——确认它真在"你的 SOC"里（10 分钟）

这一步是"用起来"和"看着它能跑"的区别。挑一台最想被"看着"的主机，**人为触发一个安全事件**：

```bash
# 1. SSH 弱口令爆破测试：连续 6 次错密码
for i in 1 2 3 4 5 6; do ssh wronguser@<target-host>; done
# 2. 查 Wazuh Dashboard → Threat Hunting → Events
#    应该看到规则 5720 (Multiple failed logins) 触发
# 3. 查 AI-miniSOC 告警页（如果配了 Webhook，几秒到；手动同步 1~5 分钟）
```

**这就是 02 篇结尾挖的坑**——"找一台你的服务器装 Wazuh agent，看它的日志和告警第一次流进你的控制台"那一刻。

> 💡 **自检小技巧**：如果你暂时没人/没机器做安全事件，**用 `logger` 工具写一条 Wazuh 能识别的日志**（比如 `logger -p auth.warning "test alert from xiejava"`），Wazuh 内部规则会立刻生成事件——保证端到端联通。

**过关标准**：
- ① Wazuh Dashboard 看到触发的事件
- ② AI-miniSOC 告警页出现对应条目（手动同步就等 1~5 分钟，Webhook 秒级）

来自wazuh的告警事件
![AI-miniSOC 告警页：agent 025 的 rootcheck 告警已出现在告警列表](https://image2.ishareread.com/images/20261009/s4-alert-arrives.png)

---

## 踩坑总表（对号入座用）

把这次的坑集中列一遍，接入卡住时直接对表：

| # | 症状 | 根因 | 解法 |
|---|---|---|---|
| 1 | Wazuh Dashboard 上 agent 一直 `Disconnected` | `WAZUH_MANAGER` 配错 / 1514/UDP 端口被挡 | 终端 `tail -f /var/ossec/logs/ossec.log` 排查 |
| 2 | `wazuh-collector` 报 401 认证失败 | `${WAZUH_PASSWORD}` 没被解析 / 密码错 | 终端 `curl` 直测 API；确认 yaml 用 `resolve()` 解析 env |
| 3 | Webhook 集成脚本无输出 | 文件权限错（应 `root:wazuh -rwxr-x---`） | `ls -la /var/ossec/integrations/custom-minisoc` 核对 |
| 4 | Webhook API Key 含特殊字符导致推送失败 | 集成脚本不 URL 编码 | Key 避免 `&` `=` 等；或在脚本里 `urllib.parse.quote` |

这张表以后会持续更新——**你在接入中踩到的新坑，欢迎到仓库提 Issue**。

---

## 收尾：常见问题

**Q：我有 50 台主机，要装 50 次 agent？**

A：是的，每台都得装。但 agent 包可以用 Ansible / SaltStack / 自带配置批量推（你公司如果有 CMDB，配个下发任务就行）。本篇讲的是"接入第一台"——批量分发是运维自动化的事，不在 SOC 的范围。

**Q：Wazuh 太重，我只是想看 SSH 登录失败和文件变化，有更轻的吗？**

A：有。轻量场景可以直接跑 `auditd` + `inotifywait`，把日志推给 Loki（01 篇的"日志分级存储"框架支持）。但 Wazuh 的 SCA 配置评估、CIS 基线检查、Rootkit 检测等是它的护城河，等你的场景明确后再选——**不要为了"轻量"丢掉了"覆盖度"**。

**Q：告警风暴怎么办？Wazuh 默认规则库有几千条规则。**

A：这是告警治理的事。AI-miniSOC 内置了"13/10/7/4"四级告警分级（13=Critical, 10=High, 9=Medium, ≤4=噪音）+ 聚合降噪——所有 Wazuh 告警会先经过 `soc_alerts` 表的预处理，再推送给你。具体在 第 14 篇《告警治理》展开（连载中）。

---

## 写在最后

00 篇回答"为什么做"，01 篇回答"凭什么行"，02 篇回答"怎么跑起来"——这一篇回答**"你亲手接的机器上的告警你能不能看到"**。

回头看这四步：装 agent / 接采集器 / 触发测试 / 验收。**你管的任何 SOC 系统，跑起来之后能不能"用起来"，区别就在这一步。** 告警没流进来，前面三篇的努力只是把一个漂亮的控制台挂在那里。

写完这一篇，我想多说一句：本文的姊妹篇——**我那七篇 Wazuh 旧教程**（侧重单工具：装、规则、FIM、暴力破解……），从今天起正式"收编"进《一个人的SOC》系列——它们负责讲"工具怎么用"，本系列负责讲"工具怎么组成平台"。两边互补，**wazuh 7 篇旧文头部会加导语指向本系列**，老读者从单工具过来、新读者从平台角度再回到单工具深读，都能找到入口。

下一篇 (04)：**用一台 TP-Link 路由器做内网资产发现与上网行为审计**——零成本起步，把家里的设备/办公室的小网络也纳入资产台账。同时也预告 第 09 篇《GLM 统一底座：9 个 AI 消费点如何接进安全运营》——本系列灵魂的第二篇。

老李的第一台主机已经在他的 SOC 里了。你的呢？

---

**【系列导航】**《一个人的SOC》总目录（持续更新）

**【上一篇】**[(02) 从零部署——一小时跑起来的实战与踩坑](/posts/a9c4e6f2/)

**【系列开篇】**[(00) 中小企业为什么需要自己的安全运营中心](/posts/b7e2a9c4/)

**【姊妹篇·收编】** [开源安全管理平台 Wazuh 七篇系列](https://blog.csdn.net/fullbug/category_7337204.html)（从单工具视角讲 Wazuh）

**【项目地址】**[AI-miniSOC - GitHub](https://github.com/xiejava1018/AI-miniSOC) · MIT 开源，欢迎 Star ⭐

https://github.com/xiejava1018/AI-miniSOC

**【交流】** Wazuh 接入卡住了？把 `tail -f /var/log/wazuh/integrations.log` 和 `docker compose logs wazuh-collector` 的输出发到 Issue，我会逐个回复。

**下一篇（预告）**：(04) 用一台 TP-Link 路由器做内网资产发现与上网行为审计（连载中，敬请关注）

---

作者博客：<http://xiejava.ishareread.com/>

<center>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg)

关注：微信公众号，一起学习成长！</center>
