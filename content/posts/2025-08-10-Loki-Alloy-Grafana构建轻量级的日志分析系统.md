---
title: Loki+Alloy+Grafana构建轻量级的日志分析系统
id: 20250810001
tags:
  - 日志分析
categories:
  - - 技术
    - 网络安全
abbrlink: 9804ee9f
date: 2025-08-10 12:50:25
---
在现代运维和开发流程中，日志分析是故障排查、性能优化的核心环节。传统的日志系统（如 ELK Stack）虽功能强大，但资源消耗高、配置复杂，对中小规模环境或边缘设备不够友好。Loki+Alloy+Grafana组合则以 “轻量级、低成本、易部署” 为核心优势，成为替代方案的理想选择。

典型的基于 Loki 的日志记录技术栈由 3 个组件组成：

![Loki](http://image2.ishareread.com/images/2025/20250810/1-LAG架构.png)

- Loki：Grafana Labs 开源的日志聚合系统，借鉴 Prometheus 设计理念，仅索引日志元数据（标签）而非全文，大幅降低存储和计算开销。
- Alloy：代理或客户端，Grafana 官方推荐的新一代数据采集器Alloy（替代 Promtail），支持日志、指标、追踪多信号采集，配置灵活且资源占用低。
- Grafana：强大的可视化平台，原生支持 Loki 数据源，提供丰富的日志查询、过滤和仪表盘功能。

对网络设备、安全设备等的告警日志进行分析是在网络运维和安全运营中经常遇到的场景，我们可以通过Loki+Alloy+Grafana来搭建一套轻量级的日志分析系统来收集各类网络设备、安全设备的日志进行分析。现在就以采集TP-Link路由器的上网行为日志为例来构建这套日志分析系统。

![日志采集分析](http://image2.ishareread.com/images/2025/20250810/2-日志采集存储分析.png)

本文将详细介绍在Ubuntu 24.04 LTS环境中，从零搭建这套日志分析系统的全过程，确保每个步骤可操作、可复现。

环境准备：Ubuntu 24.04 系统配置
系统要求
- 硬件：2 核 CPU、4GB 内存、20GB 磁盘（日志存储根据需求调整）
- 系统：Ubuntu 24.04 LTS（已更新至最新版本）
- 权限：sudo 权限（用于安装软件和配置服务）

根据Grafana官网的建议步骤，先安装Loki，再部署采集代理Alloy,再部署Grafana，通过Grafana来分析日志。

![日志采集步骤](http://image2.ishareread.com/images/2025/20250810/3-LAG日志系统安装步骤.png)

## 部署Loki日志存储系统

Loki 是日志的 “后端存储”，负责接收、存储日志并响应查询请求。采用二进制方式部署，步骤如下：
### 1. 下载Loki二进制文件

通过 GitHub API 获取Loki的二进制安装包：

```bash
# 创建Loki安装目录并下载二进制包  这里安装的是3.5.1 
sudo mkdir -p /opt/loki /etc/loki
wget -qO /opt/loki/loki-linux-amd64.zip "https://github.com/grafana/loki/releases/download/v3.5.1/loki-linux-amd64.zip"

# 解压并设置可执行权限
sudo unzip -d /opt/loki /opt/loki/loki-linux-amd64.zip && rm /opt/loki/loki-linux-amd64.zip
sudo chmod a+x /opt/loki/loki-linux-amd64

# 创建软链接，方便全局调用
sudo ln -s /opt/loki/loki-linux-amd64 /usr/local/bin/loki
```

查看Loki版本 `loki --version`

```bash
loki --version
loki, version 3.5.1 (branch: release-3.5.x, revision: d4e637ce)
  build user:       root@ceaea196ea87
  build date:       2025-05-19T17:06:03Z
  go version:       go1.24.1
  platform:         linux/amd64
  tags:             netgo
```

### 2. 配置 Loki

使用官方提供的本地存储配置文件，创建 Loki 配置目录并下载默认配置文件：

```bash
sudo mkdir -p /etc/loki
sudo wget https://raw.githubusercontent.com/grafana/loki/v3.5.1/cmd/loki/loki-local-config.yaml -O /etc/loki/config.yaml
```

### 3. 创建 Systemd 服务

为 Loki 创建系统服务，确保开机自启和进程管理：

```bash
sudo tee /etc/systemd/system/loki.service <<EOF
Description=Grafana Loki Log Aggregation System
After=network.target 

[Service]
Type=simple
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/config.yaml
Restart=on-failure 
User=root 

[Install]
WantedBy=multi-user.target 
EOF# 重载systemd配置并启动Loki

sudo systemctl daemon-reload
sudo systemctl start loki
sudo systemctl enable loki
# 验证服务状态（应显示active (running)）
sudo systemctl status loki
```

可以使用 `journalctl -u loki -f` 查看 Loki 日志以进行故障排除

### 4.  验证 Loki 运行
通过 HTTP 接口检查 Loki 状态：

```bash
curl http://127.0.0.1:3100/ready
```

预期输出：ready（表示 Loki 已就绪）

![loki ready](http://image2.ishareread.com/images/2025/20250810/4-loki-ready.png)

## 部署配置Alloy
Alloy 是连接日志源与 Loki 的 “桥梁”，负责采集服务器本地日志并发送到 Loki。作为 Grafana Agent 的替代者，Alloy 配置更灵活，支持多信号采集。

### 1. 安装 Alloy
通过 Grafana 官方仓库安装 Alloy（与 Grafana 使用同一仓库，无需重复添加）：

```bash
# 安装Alloy包
sudo apt install -y alloy

# 验证安装
alloy --version
```
![alloy --version](http://image2.ishareread.com/images/2025/20250810/5-alloy-version.png)

### 2. 配置 Alloy 收集日志
Alloy 使用声明式配置文件定义采集规则，默认路径为`/etc/alloy/config.alloy`。我们需配置：
1、 通过syslog采集；2、对日志中的IP进行提取；3、 将日志转发到 Loki。
创建配置文件：

```bash
otelcol.receiver.syslog "local" {
  udp {
    listen_address = "0.0.0.0:514"
    encoding = "GBK"
  }
  output {
    logs = [otelcol.exporter.loki.local.input]
  }
}

otelcol.exporter.loki "local" {
  forward_to = [loki.process.extract_ip.receiver]
}

loki.process "extract_ip" {
  stage.regex {
    expression = "a:(?P<ip>\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3})"
  }
  stage.labels {
    values = {
      ip = "ip",
    }
  }
  forward_to = [loki.write.local.receiver]
}

loki.write "local" {
  endpoint {
    url = "http://localhost:3100/loki/api/v1/push"
  }
}
```

配置说明：

> alloy目前对syslog支持并不是特别好，经过多次尝试，用alloy的syslog组件不能支持中文编码，所以改用通过otelcol的syslog来采集日志。
> listen_address = "0.0.0.0:514" 表示监听所有源的514端口来的upd数据。

### 3. 启动 Alloy 服务

```bash
# 启动Alloy并设置开机自启
sudo systemctl enable --now alloy

# 验证服务状态（应显示active (running)）
sudo systemctl status alloy
```

> 排查提示若启动失败，通过sudo journalctl -u alloy -f查看日志，常见问题：配置文件语法错误、Loki未启动导致连接失败

## 部署Grafana
Grafana 是整个系统的 “前端”，用于日志查询和可视化。通过官方仓库安装可确保自动更新。
### 1. 添加 Grafana 官方仓库
```bash
# 导入Grafana GPG密钥（用于验证包完整性）
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# 添加Grafana OSS仓库（稳定版）
echo 'deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main' | sudo tee /etc/apt/sources.list.d/grafana.list
```

### 2. 安装并启动 Grafana

```bash
# 更新包索引并安装Grafana
sudo apt update && sudo apt install -y grafana

# 启动Grafana服务并设置开机自启
sudo systemctl enable --now grafana-server

# 验证服务状态（应显示active (running)）
sudo systemctl status grafana-server
```
### 3.  验证 Grafana 安装

打开浏览器访问 `http://<服务器IP>:3000`，首次登录使用默认凭据：
- 用户名：admin
- 密码：admin

首次登录会要求修改密码，按提示设置新密码即可。

## 接入数据源
### 1. 在发送端配置采集器的地址
这里我们要接入和分析TP-Link路由器的上网行为日志。所以要在路由器上配置将日志发送到alloy采集节点的服务器。具体的TP-Link路由器的上网行为分析审计的配置参考《[通过TPLink路由器进行用户行为审计实战](http://xiejava.ishareread.com/posts/e87ac72d/)》

![行为审计](http://image2.ishareread.com/images/2025/20250810/6-TPLINK%E9%85%8D%E7%BD%AE.png)

### 2. 在采集端看是否收到数据
通过tcpdump命令查看采集节点是否有日志数据进来

```bash
tcpdump -i ens2 -A port 514
```

![tcpdump](http://image2.ishareread.com/images/2025/20250810/7-tcpdump.png)

这里可以看到有日志数据通过ens2网卡的514端口进来了。

## 用Grafana查询和分析数据
在用Grafana进行查询之前先要配置Grafana的数据源添加loki的数据源

![添加数据源](http://image2.ishareread.com/images/2025/20250810/8-new-data-source.png)

在loki的数据源中设置loki的连接地址

![loki连接](http://image2.ishareread.com/images/2025/20250810/9-配置loki连接地址.png)

设置完成后就可以用Grafana强大的分析查询和可视化来对日志进行分析了。

![日志分析](http://image2.ishareread.com/images/2025/20250810/10-grafana分析展示.png)

通过本文步骤，我们在 Ubuntu 24.04 上构建了 Loki+Alloy+Grafana 日志分析系统：

Loki 负责高效存储日志，仅索引元数据降低开销；
Alloy 轻量采集系统日志，配置灵活易扩展；
Grafana 提供强大的日志查询与可视化能力（支持 LogQL、仪表盘、告警）。

该架构资源占用低（单节点最低 2 核 4GB 内存即可运行），适合中小规模环境、边缘设备或开发测试场景。后续可扩展至多节点 Loki 集群、添加日志告警规则，或集成 Prometheus 监控指标，构建完整的可观测性平台。

-----
博客地址：[http://xiejava.ishareread.com/](http://xiejava.ishareread.com/)

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>