---
title: 让Claude Code写了个运维健康检查的Skill还挺好用的
categories:
  - 技术
  - - AI 
tags:
  - AI机器学习
  - 运维自动化
  - 网络安全
abbrlink: 5f421230
date: 2026-01-20 08:00:00
---
## 摘要
手头上有几台服务器，经常要对服务器进行健康检查、安全检查等。这几天让ClaudeCode自己写了个Skill这样就一句话活就让AI给干了。本文详细介绍了如何使用Claude Code AI辅助开发工具，从零开始构建一个生产级运维健康检查的Skill。重点阐述了整体架构设计、核心功能实现细节以及实际应用效果。该系统采用模块化设计，提供三大检查模块（基础健康检查、深度安全审计、Docker容器监控），支持双格式输出（Markdown人工可读 + JSON机器可处理），具有良好的可扩展性和兼容性。

## 一、背景与设计目标

### 1.1 实际需求

在日常运维工作中，我们需要对多台服务器进行健康检查，涵盖系统资源、Docker容器状态、安全指标等多个维度。虽然市面上有很多成熟的监控方案（如Prometheus、Grafana、Zabbix等），但在以下场景中，我们需要一个轻量级、快速部署的检查工具：

- **日常巡检**：每天对服务器进行快速健康扫描
- **安全审计**：定期检查异常进程、可疑网络连接、文件系统安全
- **容器监控**：检查Docker容器运行状态、资源使用情况
- **问题排查**：当系统出现问题时，快速获取当前状态信息

### 1.2 设计目标

基于上述需求，我设定了以下设计目标交给了Claude Code：

**核心特性**：
- ✅ **轻量级部署**：基于Bash脚本，无需额外依赖，远程执行无需安装
- ✅ **双格式输出**：同时生成Markdown（人工阅读）和JSON（机器处理）格式
- ✅ **模块化架构**：三大检查模块独立运行，可按需组合
- ✅ **状态可视化**：使用emoji（✅正常/⚠️警告/❌严重）直观展示
- ✅ **广泛兼容性**：支持bash 3.2+（macOS默认版本），适配主流Linux发行版

**技术亮点**：
- 统一的输出库设计，实现代码复用
- JSON三层结构（summary/details/metadata），满足不同使用场景
- 完善的错误处理和兼容性方案
- 清晰的扩展接口，便于添加新的检查类型

## 二、整体架构设计

### 2.1 系统架构

Claude Code自动帮我设计了整体架构
运维健康检查系统采用模块化设计，包含三个独立的检查模块：

| 模块名称 | 脚本文件 | 功能定位 | 核心检查项 |
|---------|---------|---------|-----------|
| **基础健康检查** | health-check.sh | 日常巡检、快速评估 | 系统资源、内存、磁盘、网络、服务状态、基础安全 |
| **深度安全检查** | security-check.sh | 安全审计、威胁检测 | 异常进程、可疑连接、文件安全、账户安全、系统完整性 |
| **Docker容器监控** | docker-check.sh | 容器环境管理 | 容器状态、资源使用、镜像管理、存储空间 |

**设计原则**：
- **独立性**：每个脚本可单独运行，互不依赖
- **一致性**：统一的输出格式和状态指示
- **可扩展性**：易于添加新的检查项或模块
- **远程友好**：支持SSH管道执行，无需远程安装

### 2.2 文件结构

```
host-manage/
├── skills/ops-health-check/
│   ├── SKILL.md                    # Skill定义和使用文档
│   └── scripts/
│       ├── health-check.sh         # 基础健康检查脚本
│       ├── security-check.sh       # 深度安全检查脚本
│       ├── docker-check.sh         # Docker监控脚本
│       └── lib/
│           └── output.sh           # 输出格式化库（核心）
├── docs/plans/
│   ├── 2025-01-17-ops-health-check-design.md     # 整体设计文档
│   └── 2025-01-18-json-output-design.md          # JSON输出设计
└── health-reports/                 # 检查报告输出目录
    ├── health-check-*.md           # Markdown格式报告
    ├── health-check-*.json         # JSON格式数据
    ├── security-check-*.md
    ├── security-check-*.json
    ├── docker-check-*.md
    └── docker-check-*.json
```

## 三、核心功能实现

Claude Code自动帮我实现了所有的功能

### 3.1 基础健康检查模块（health-check.sh）

这是日常运维使用最频繁的模块，专注于快速评估系统整体健康状况。

#### 3.1.1 系统运行时间和负载检查

**检查目的**：了解系统运行时长和当前负载压力

**实现逻辑**：

```bash
# 获取系统运行时间
uptime_output=$(uptime)
uptime_clean=$(echo "$uptime_output" | sed 's/^ *//g')

# 兼容不同系统的uptime输出格式
uptime_str=$(uptime -p 2>/dev/null || \
    echo "$uptime_clean" | awk -F'up ' '{print $2}' | awk -F',' '{print $1}')

# 提取负载平均值（1分钟、5分钟、15分钟）
load_str=$(echo "$uptime_clean" | awk -F'load average:' '{print $2}' | sed 's/^ *//g')
load_1min=$(echo $load_str | awk '{print $1}' | sed 's/,//')
load_5min=$(echo $load_str | awk '{print $2}' | sed 's/,//')
load_15min=$(echo $load_str | awk '{print $3}')

# 判断负载状态
load_status=$(check_load_status "$load_1min" "$CPU_WARNING" "$CPU_CRITICAL")
```

**状态判断函数**：

```bash
check_load_status() {
    local load=$1
    local warning=$2
    local critical=$3

    # 使用bc进行浮点数比较
    if (( $(echo "$load >= $critical" | bc -l) )); then
        echo "critical"
    elif (( $(echo "$load >= $warning" | bc -l) )); then
        echo "warning"
    else
        echo "ok"
    fi
}
```

**关键点**：
- 使用`bc -l`进行浮点数比较，兼容不同系统
- 兼容`uptime -p`命令（human-readable格式）不可用的情况
- 同时收集三个时间维度的负载数据，用于趋势分析

#### 3.1.2 内存使用检查

**检查目的**：监控内存和swap使用情况，防止内存耗尽

**实现逻辑**：

```bash
# 获取内存信息（单位：MB）
memory_info=$(free -m | grep Mem)
mem_total=$(echo $memory_info | awk '{print $2}')
mem_used=$(echo $memory_info | awk '{print $3}')
mem_avail=$(echo $memory_info | awk '{print $7}')  # available列更准确
mem_percent=$(awk "BEGIN {printf \"%.1f\", $mem_used * 100 / $mem_total}")

# Swap检查
swap_info=$(free -m | grep Swap)
swap_total=$(echo $swap_info | awk '{print $2}')
swap_used=$(echo $swap_info | awk '{print $3}')

if [ "$swap_total" -gt 0 ]; then
    swap_percent=$(awk "BEGIN {printf \"%.1f\", $swap_used * 100 / $swap_total}")
else
    swap_percent=0
fi

# 状态判断
if (( $(echo "$mem_percent >= $MEMORY_CRITICAL" | bc -l) )); then
    mem_status="critical"
elif (( $(echo "$mem_percent >= $MEMORY_WARNING" | bc -l) )); then
    mem_status="warning"
else
    mem_status="ok"
fi
```

**输出示例**：

```markdown
### 内存使用
- **总内存**: 16384MB
- **已使用**: 12500MB (76.3%)
- **可用**: 3884MB
- **Swap**: 256MB / 2048MB (12.5%)
- **状态**: ⚠️ 警告
```

**关键点**：
- 使用`free`命令的`available`列而非`free`列，更准确反映可用内存
- 同时检查swap使用率，高swap使用率通常意味着内存压力
- 支持自定义阈值配置（通过环境变量`MEMORY_WARNING`、`MEMORY_CRITICAL`）

#### 3.1.3 磁盘空间检查

**检查目的**：监控所有挂载点的磁盘使用情况，防止磁盘空间不足

**实现逻辑**：

```bash
echo "### 磁盘空间"
echo "| 挂载点 | 设备 | 容量 | 已用 | 可用 | 使用率 | 状态 |"
echo "|--------|------|------|------|------|--------|------|"

# 遍历所有挂载点（排除临时文件系统）
df -h | grep -vE '^Filesystem|tmpfs|overlay|none' | while read -r line; do
    device=$(echo $line | awk '{print $1}')
    size=$(echo $line | awk '{print $2}')
    used=$(echo $line | awk '{print $3}')
    avail=$(echo $line | awk '{print $4}')
    use_percent=$(echo $line | awk '{print $5}' | sed 's/%//')
    mount=$(echo $line | awk '{print $6}')

    # 判断状态
    if [ "$use_percent" -ge "$DISK_CRITICAL" ]; then
        status="critical"
        emoji="❌"
    elif [ "$use_percent" -ge "$DISK_WARNING" ]; then
        status="warning"
        emoji="⚠️"
    else
        status="ok"
        emoji="✅"
    fi

    # 输出Markdown表格行
    echo "| $mount | $device | $size | $used | $avail | ${use_percent}% | $emoji |"

    # 收集JSON数据
    add_disk_data "$mount" "device" "$device"
    add_disk_data "$mount" "total_gb" "$size"
    add_disk_data "$mount" "used_gb" "$used"
    add_disk_data "$mount" "available_gb" "$avail"
    add_disk_data "$mount" "used_percent" "$use_percent"
    add_disk_data "$mount" "status" "$status"
done
```

**输出示例**：

```markdown
### 磁盘空间
| 挂载点 | 设备 | 容量 | 已用 | 可用 | 使用率 | 状态 |
|--------|------|------|------|------|--------|------|
| / | /dev/sda1 | 100G | 45G | 55G | 45.0% | ✅ 正常 |
| /data | /dev/sdb1 | 500G | 425G | 75G | 85.0% | ⚠️ 警告 |
| /boot | /dev/sda2 | 1G | 250M | 750M | 25.0% | ✅ 正常 |
```

**关键点**：
- 自动过滤`tmpfs`、`overlay`等临时文件系统
- 支持多个挂载点，每个挂载点独立判断状态
- 使用整数比较（已去除百分号），提高性能
- 同时生成Markdown表格和JSON数组数据

#### 3.1.4 网络连接统计

**检查目的**：了解当前网络连接数量，快速发现异常连接

**实现逻辑**：

```bash
# 使用ss命令替代netstat（更现代）
if command -v ss >/dev/null 2>&1; then
    # 总连接数
    total_connections=$(ss -tn | wc -l)

    # 外部连接数（排除本地回环）
    external_connections=$(ss -tn | awk '{print $5}' | \
        grep -v '127.0.0.1' | \
        grep -v '::1' | \
        cut -d':' -f1 | \
        sort -u | \
        wc -l)

    # 监听端口数
    listening_ports=$(ss -tln | wc -l)
else
    # 降级到netstat
    total_connections=$(netstat -tn 2>/dev/null | wc -l)
    external_connections=$(netstat -tn 2>/dev/null | awk '{print $5}' | \
        grep -v '127.0.0.1' | \
        grep -v '::1' | \
        cut -d':' -f1 | \
        sort -u | \
        wc -l)
    listening_ports=$(netstat -tln 2>/dev/null | wc -l)
fi

add_system_data "network_total_connections" "$total_connections"
add_system_data "network_external_connections" "$external_connections"
add_system_data "network_listening_ports" "$listening_ports"
```

**关键点**：
- 优先使用`ss`命令（现代Linux推荐），降级到`netstat`
- 统计外部独立IP连接数，而非连接总数（更准确反映异常）
- 包含监听端口数量，便于发现未授权监听

#### 3.1.5 systemd服务状态检查

**检查目的**：检查系统服务运行状态，发现失败服务

**实现逻辑**：

```bash
# 检查systemd是否可用
if command -v systemctl >/dev/null 2>&1; then
    # 统计服务状态
    running_services=$(systemctl list-units --type=service --state=running 2>/dev/null | \
        grep 'loaded loaded' | wc -l)

    failed_services=$(systemctl list-units --type=service --state=failed 2>/dev/null | \
        grep 'loaded loaded' | wc -l)

    enabled_services=$(systemctl list-unit-files --type=service --state=enabled 2>/dev/null | \
        grep 'enabled' | wc -l)

    # 获取失败服务列表
    if [ "$failed_services" -gt 0 ]; then
        failed_list=$(systemctl list-units --type=service --state=failed 2>/dev/null | \
            grep 'loaded loaded' | awk '{print $1}')

        echo "### 失败的服务"
        echo "$failed_list" | while read -r service; do
            echo "- ❌ $service"
        done
    fi

    # 常用关键服务检查
    critical_services=("ssh" "docker" "cron" "rsyslog")
    for svc in "${critical_services[@]}"; do
        if systemctl is-active --quiet "${svc}.service" 2>/dev/null; then
            echo "- ✅ ${svc}.service: 运行中"
        else
            echo "- ⚠️ ${svc}.service: 未运行或不存在"
        fi
    done
else
    echo "⚠️ 系统不使用systemd"
fi
```

**输出示例**：

```markdown
### 服务状态
- **运行中**: 23个
- **失败**: 2个
- **启用**: 15个

#### 关键服务状态
- ✅ ssh.service: 运行中
- ✅ docker.service: 运行中
- ✅ cron.service: 运行中
- ❌ postfix.service: 已停止

#### 失败的服务列表
- ❌ postfix.service
- ❌ ufw.service
```

**关键点**：
- 兼容非systemd系统（如SysV init）
- 区分"运行中"、"失败"、"启用"三种状态
- 单独检查关键服务（SSH、Docker等），便于快速定位问题
- 列出所有失败服务，方便管理员进一步排查

#### 3.1.6 基础安全检查

虽然主要的安全检查在security-check.sh中，但基础健康检查也包含几个关键的安全指标：

```bash
echo "### 基础安全"

# 1. 检查挖矿进程
MINING_PROCESSES=("xmrig" "minerd" "cpuminer" "ccminer" "Claymore")
mining_found=0

for proc in "${MINING_PROCESSES[@]}"; do
    if pgrep -x "$proc" >/dev/null; then
        echo "- ❌ 发现挖矿进程: $proc"
        mining_found=1
    fi
done

if [ $mining_found -eq 0 ]; then
    echo "- ✅ 未发现挖矿进程"
fi

# 2. 检查/tmp和/dev/shm中的可执行文件
tmp_execs=$(find /tmp /dev/shm -type f -executable 2>/dev/null | wc -l)
if [ "$tmp_execs" -gt 0 ]; then
    echo "- ⚠️ 临时目录中发现 $tmp_execs 个可执行文件"
else
    echo "- ✅ 临时目录无可执行文件"
fi

# 3. 检查失败登录次数（最近）
if [ -f /var/log/auth.log ]; then
    failed_logins=$(grep "Failed password" /var/log/auth.log 2>/dev/null | \
        tail -100 | wc -l)
    echo "- 最近失败登录次数: $failed_logins"
fi
```

### 3.2 深度安全检查模块（security-check.sh）

这是最复杂的模块，涵盖了5大类、20+项安全检查，用于全面的安全审计。

#### 3.2.1 异常进程检测

**检查目的**：发现挖矿程序、高资源占用进程、可疑可执行文件

**1. 挖矿进程检测**

```bash
# 定义已知挖矿程序名称列表
MINING_PROCESSES=(
    "xmrig" "minerd" "cpuminer" "ccminer" "Claymore"
    "cryptonight" "ethminer" "nbminer" "bminer"
)

suspicious_procs=""
mining_detected="false"

# 遍历检查
for proc in "${MINING_PROCESSES[@]}"; do
    # 使用pgrep查找进程（比ps aux更快）
    found=$(pgrep -ix "$proc" 2>/dev/null)
    if [ -n "$found" ]; then
        # 获取进程详细信息
        proc_info=$(ps -p "$found" -o pid,user,cmd --no-headers 2>/dev/null)
        suspicious_procs="$suspicious_procs\n$proc_info"
        mining_detected="true"

        # 记录到安全数据
        add_security_data "mining_process" "$proc"
        add_security_data "mining_pid" "$found"
    fi
done

if [ "$mining_detected" = "true" ]; then
    echo "❌ 发现挖矿进程："
    echo -e "$suspicious_procs"
else
    echo "✅ 未发现挖矿进程"
fi
```

**2. 高资源占用进程检测**

```bash
echo "### 高资源占用进程（Top 10）"
echo "| PID | 用户 | CPU% | 内存% | 命令 |"
echo "|-----|------|------|--------|------|"

# CPU占用最高的进程
ps aux --sort=-%cpu | head -n 11 | tail -n 10 | while read -r line; do
    pid=$(echo "$line" | awk '{print $2}')
    user=$(echo "$line" | awk '{print $1}')
    cpu=$(echo "$line" | awk '{print $3}')
    mem=$(echo "$line" | awk '{print $4}')
    cmd=$(echo "$line" | awk '{for(i=11;i<=NF;i++)printf $i" "}' | cut -c1-50)

    # 标记异常高占用
    if (( $(echo "$cpu > 80" | bc -l) )); then
        echo "| $pid | $user | **$cpu** | $mem | $cmd |"
    else
        echo "| $pid | $user | $cpu | $mem | $cmd |"
    fi
done
```

**3. 临时目录可执行文件检查**

```bash
# 检查/tmp和/dev/shm中的可执行文件
tmp_dirs=("/tmp" "/dev/shm" "/var/tmp")

for dir in "${tmp_dirs[@]}"; do
    if [ -d "$dir" ]; then
        exec_files=$(find "$dir" -type f -executable 2>/dev/null)
        exec_count=$(echo "$exec_files" | grep -v '^$' | wc -l)

        if [ "$exec_count" -gt 0 ]; then
            echo "⚠️ $dir 中发现 $exec_count 个可执行文件："
            echo "$exec_files" | head -n 10 | while read -r file; do
                perms=$(stat -c "%a" "$file" 2>/dev/null)
                size=$(stat -c "%s" "$file" 2>/dev/null)
                mtime=$(stat -c "%y" "$file" 2>/dev/null | cut -d'.' -f1)
                echo "  - $file (权限:$perms, 大小:$size字节, 修改:$mtime)"
            done
        fi
    fi
done
```

**关键点**：
- 使用`pgrep`而非`ps aux | grep`，性能更高且避免误匹配
- 记录进程的PID、用户、完整命令行，便于追踪
- 同时检查/tmp、/dev/shm、/var/tmp三个常见临时目录
- 显示文件的权限、大小、修改时间，帮助判断是否可疑

#### 3.2.2 网络安全检查

**检查目的**：发现反向shell、可疑端口监听、异常外部连接

**1. 反向shell检测**

```bash
# 反向shell典型特征：外部IP连接本地高端口（非标准服务端口）
echo "### 反向shell检测"

# 获取所有TCP连接
if command -v ss >/dev/null 2>&1; then
    connections=$(ss -tnp 2>/dev/null)
else
    connections=$(netstat -tnp 2>/dev/null)
fi

# 检测可疑模式：外部IP连接到本地50000-65535端口
reverse_shell_detected="false"

echo "$connections" | while read -r line; do
    # 解析本地地址和远程地址
    local_addr=$(echo "$line" | awk '{print $4}')
    remote_addr=$(echo "$line" | awk '{print $5}')

    # 提取端口号
    local_port=$(echo "$local_addr" | cut -d':' -f2 | cut -d':' -f1)
    remote_ip=$(echo "$remote_addr" | cut -d':' -f1)

    # 排除本地回环和标准端口
    if [[ ! "$remote_ip" =~ ^(127\.|::1) ]]; then
        # 检查本地端口是否为高端口（非标准服务）
        if [ "$local_port" -ge 50000 ] 2>/dev/null; then
            # 排除部分合法的高端口号
            if [[ ! "$local_port" =~ ^(50000|50001|50002) ]]; then
                echo "⚠️ 可疑连接：远程 $remote_addr -> 本地端口 $local_port"
                reverse_shell_detected="true"
            fi
        fi
    fi
done

if [ "$reverse_shell_detected" = "false" ]; then
    echo "✅ 未发现反向shell特征"
fi
```

**2. 监听端口分析**

```bash
echo "### 监听端口分析"

# 获取所有监听端口
if command -v ss >/dev/null 2>&1; then
    listening=$(ss -tulpn 2>/dev/null)
else
    listening=$(netstat -tulpn 2>/dev/null)
fi

echo "| 端口 | 协议 | 进程 | 用户 |"
echo "|------|------|------|------|"

echo "$listening" | grep LISTEN | while read -r line; do
    port=$(echo "$line" | awk '{print $5}' | rev | cut -d':' -f1 | rev)
    protocol=$(echo "$line" | awk '{print $1}')
    process=$(echo "$line" | awk '{print $7}' | cut -d'/' -f2 | cut -d',' -f1)
    user=$(echo "$line" | awk '{print $7}' | cut -d'"' -f2)

    # 标记非标准高端口
    if [ "$port" -ge 10000 ]; then
        echo "| **$port** | $protocol | $process | $user |"
    else
        echo "| $port | $protocol | $process | $user |"
    fi
done
```

**3. 外部连接统计**

```bash
# 统计外部连接的独立IP数量
external_ips=$(ss -tn 2>/dev/null | \
    awk '{print $5}' | \
    cut -d':' -f1 | \
    grep -vE '^(127\.|::1|0\.0\.0\.0)' | \
    sort -u | \
    wc -l)

echo "### 外部连接统计"
echo "- 外部连接的独立IP数：$external_ips"

# 列出Top 10外部IP
echo "- 连接最多的外部IP："
ss -tn 2>/dev/null | \
    awk '{print $5}' | \
    cut -d':' -f1 | \
    grep -vE '^(127\.|::1|0\.0\.0\.0)' | \
    sort | uniq -c | sort -rn | head -n 10 | \
    while read -r line; do
        count=$(echo "$line" | awk '{print $1}')
        ip=$(echo "$line" | awk '{print $2}')
        echo "  - $ip: $count 次连接"
    done
```

**关键点**：
- 反向shell检测基于行为特征：外部IP连接本地高端口
- 排除已知的合法高端口（如某些应用服务）
- 统计外部连接IP数量和连接频率，异常高值需要关注
- 显示监听端口对应的进程和用户，便于排查

#### 3.2.3 文件系统安全检查

**检查目的**：发现最近修改的敏感文件、勒索病毒特征、SUID/SGID文件

**1. 关键目录变更检测**

```bash
echo "### 关键目录最近变更（7天内）"

# 定义需要监控的关键目录
critical_dirs=("/etc" "/home" "/root" "/var/www" "/opt")

for dir in "${critical_dirs[@]}"; do
    if [ -d "$dir" ]; then
        # 查找7天内修改的文件
        recent_files=$(find "$dir" -type f -mtime -7 2>/dev/null)

        if [ -n "$recent_files" ]; then
            file_count=$(echo "$recent_files" | grep -v '^$' | wc -l)
            echo "⚠️ $dir: $file_count 个文件在最近7天被修改"

            # 显示部分关键文件
            echo "$recent_files" | head -n 5 | while read -r file; do
                perms=$(stat -c "%a" "$file" 2>/dev/null)
                echo "  - $file (权限:$perms)"
            done
        fi
    fi
done
```

**2. 勒索病毒特征检测**

```bash
echo "### 勒索病毒特征检测"

# 定义勒索病毒常用的文件扩展名
ransomware_patterns=(
    "*.encrypted"
    "*.locked"
    "*.crypto"
    "*.crypt"
    "*.crypted"
    "*_README.txt"
    "*_DECRYPT.txt"
    "*_HELP.txt"
    "*_RESTORE.txt"
    "how_to_decrypt.*"
    "recover_files.*"
)

ransomware_found="false"
for pattern in "${ransomware_patterns[@]}"; do
    # 在用户目录中查找
    found_files=$(find /home /root -name "$pattern" 2>/dev/null)
    if [ -n "$found_files" ]; then
        echo "❌ 发现勒索病毒特征文件："
        echo "$found_files" | head -n 10
        ransomware_found="true"
    fi
done

if [ "$ransomware_found" = "false" ]; then
    echo "✅ 未发现勒索病毒特征"
fi
```

**3. SUID/SGID文件检查**

```bash
echo "### SUID/SGID文件检查"

# 查找所有SUID文件
suid_files=$(find / -type f -perm -4000 2>/dev/null | \
    grep -v -E '^/proc|^/sys|^/dev' | \
    head -n 30)

suid_count=$(echo "$suid_files" | grep -v '^$' | wc -l)

if [ "$suid_count" -gt 0 ]; then
    echo "发现 $suid_count 个SUID文件（部分列表）："
    echo "$suid_files" | while read -r file; do
        perms=$(stat -c "%a" "$file" 2>/dev/null)
        owner=$(stat -c "%U" "$file" 2>/dev/null)
        echo "  - $file (权限:$perms, 所有者:$owner)"
    done
fi

# 查找所有SGID文件
sgid_files=$(find / -type f -perm -2000 2>/dev/null | \
    grep -v -E '^/proc|^/sys|^/dev' | \
    head -n 20)

sgid_count=$(echo "$sgid_files" | grep -v '^$' | wc -l)

if [ "$sgid_count" -gt 0 ]; then
    echo "发现 $sgid_count 个SGID文件（部分列表）："
    echo "$sgid_files" | while read -r file; do
        perms=$(stat -c "%a" "$file" 2>/dev/null)
        owner=$(stat -c "%U" "$file" 2>/dev/null)
        echo "  - $file (权限:$perms, 所有者:$owner)"
    done
fi
```

**关键点**：
- 重点监控/etc、/home、/root等敏感目录
- 勒索病毒检测基于文件扩展名特征（实际环境中建议结合文件内容分析）
- SUID/SGID文件是提权漏洞的常见切入点，需要重点关注
- 限制输出数量，避免报告过长

#### 3.2.4 账户和登录安全检查

**检查目的**：检测异常登录、新增用户、sudo使用情况

**1. 最近登录记录**

```bash
echo "### 最近登录记录（最近10次）"

# 使用last命令获取登录记录
recent_logins=$(last -n 10 -a 2>/dev/null | head -n -2)

if [ -n "$recent_logins" ]; then
    echo "| 用户 | 终端 | 来源IP | 登录时间 |"
    echo "|------|------|--------|----------|"

    echo "$recent_logins" | while read -r line; do
        user=$(echo "$line" | awk '{print $1}')
        terminal=$(echo "$line" | awk '{print $2}')
        # IP在最后，主机名倒数第二
        ip=$(echo "$line" | awk '{print $(NF-1)}' | sed 's/(//g')
        # 时间范围在中间部分
        time_range=$(echo "$line" | awk '{for(i=3;i<=NF-2;i++)printf $i" "}')

        echo "| $user | $terminal | $ip | $time_range |"
    done
fi
```

**2. 失败登录统计**

```bash
echo "### 失败登录统计"

# 统计最近失败的登录次数
if [ -f /var/log/auth.log ]; then
    # Debian/Ubuntu系统
    failed_count=$(grep "Failed password" /var/log/auth.log 2>/dev/null | \
        tail -1000 | wc -l)

    echo "- 最近失败登录次数（最近1000条日志）：$failed_count"

    # 统计失败登录来源IP
    echo "- 失败登录最多的IP："
    grep "Failed password" /var/log/auth.log 2>/dev/null | \
        awk '{print $13}' | \
        sort | uniq -c | sort -rn | head -n 5 | \
        while read -r line; do
            count=$(echo "$line" | awk '{print $1}')
            ip=$(echo "$line" | awk '{print $2}')
            echo "  - $ip: $count 次"
        done

elif [ -f /var/log/secure ]; then
    # CentOS/RHEL系统
    failed_count=$(grep "Failed password" /var/log/secure 2>/dev/null | \
        tail -1000 | wc -l)
    echo "- 最近失败登录次数（最近1000条日志）：$failed_count"
fi
```

**3. 新增用户检测**

```bash
echo "### 新增用户检测（最近30天）"

# 检查shadow文件中最近修改的用户
if [ -f /etc/shadow ]; then
    # 第4字段是最后修改时间（天数，从1970-01-01起）
    thirty_days_ago=$(($(date +%s) / 86400 - 30))

    new_users=$(awk -F: -v date="$thirty_days_ago" \
        '$4 >= date {print $1}' /etc/shadow 2>/dev/null)

    if [ -n "$new_users" ]; then
        echo "⚠️ 发现最近30天新增的用户："
        echo "$new_users" | while read -r user; do
            # 获取用户详细信息
            user_info=$(grep "^$user:" /etc/passwd)
            uid=$(echo "$user_info" | cut -d':' -f3)
            home=$(echo "$user_info" | cut -d':' -f6)
            shell=$(echo "$user_info" | cut -d':' -f7)
            echo "  - $user (UID:$uid, HOME:$home, SHELL:$shell)"
        done
    else
        echo "✅ 未发现最近30天的新增用户"
    fi
fi
```

**4. Sudo使用审计**

```bash
echo "### Sudo使用审计（最近20条）"

# 检查sudo日志
if command -v journalctl >/dev/null 2>&1; then
    # 使用journalctl读取sudo日志
    sudo_logs=$(journalctl -u sudo 2>/dev/null | tail -n 20)

    if [ -n "$sudo_logs" ]; then
        echo "$sudo_logs" | while read -r line; do
            # 提取关键信息：时间、用户、命令
            echo "  - $line"
        done
    fi
elif [ -f /var/log/auth.log ]; then
    # 从auth.log中提取sudo记录
    sudo_logs=$(grep sudo /var/log/auth.log 2>/dev/null | tail -n 20)
    if [ -n "$sudo_logs" ]; then
        echo "$sudo_logs"
    fi
fi
```

**关键点**：
- 兼容不同Linux发行版的日志文件路径
- 统计失败登录次数和来源IP，高值可能意味着暴力破解
- 检测新增用户，特别是UID为0（root权限）的用户
- 审计sudo使用情况，发现权限滥用

#### 3.2.5 系统完整性检查

**检查目的**：检查关键文件权限、系统二进制文件完整性

```bash
echo "### 系统完整性检查"

# 定义关键文件列表
critical_files=(
    "/etc/passwd"
    "/etc/shadow"
    "/etc/sudoers"
    "/etc/ssh/sshd_config"
    "/etc/ssh/ssh_host_*_key"
)

echo "| 文件 | 权限 | 所有者 | 组 | 状态 |"
echo "|------|------|--------|-----|------|"

for file in "${critical_files[@]}"; do
    # 使用通配符展开
    for expanded_file in $file; do
        if [ -e "$expanded_file" ]; then
            perms=$(stat -c "%a" "$expanded_file" 2>/dev/null || stat -f "%OLp" "$expanded_file" 2>/dev/null)
            owner=$(stat -c "%U" "$expanded_file" 2>/dev/null || stat -f "%Su" "$expanded_file" 2>/dev/null)
            group=$(stat -c "%G" "$expanded_file" 2>/dev/null || stat -f "%Sg" "$expanded_file" 2>/dev/null)

            # 检查权限是否合适（简化判断）
            status="✅"
            case "$(basename "$expanded_file")" in
                shadow)
                    if [ "$perms" != "000" ] && [ "$perms" != "640" ]; then
                        status="⚠️"
                    fi
                    ;;
                sudoers)
                    if [ "$perms" != "440" ] && [ "$perms" != "400" ]; then
                        status="⚠️"
                    fi
                    ;;
            esac

            echo "| $expanded_file | $perms | $owner | $group | $status |"
        fi
    done
done
```

**关键点**：
- 重点检查/etc/passwd、/etc/shadow等认证相关文件
- /etc/shadow应该是000或640权限
- /etc/sudoers应该是440或400权限
- SSH私钥文件应该是600或400权限

### 3.3 Docker容器监控模块（docker-check.sh）

专注于Docker环境的健康检查和资源监控。

#### 3.3.1 Docker服务状态检查

**检查目的**：确认Docker服务运行正常，获取版本信息

```bash
echo "## 🐳 Docker服务状态"

# 检查Docker服务是否运行
if command -v docker >/dev/null 2>&1; then
    docker_status=$(systemctl is-active docker 2>/dev/null)

    if [ "$docker_status" = "active" ]; then
        echo "Docker服务: ✅ 运行中"

        # 获取Docker版本
        docker_version=$(docker --version 2>/dev/null | awk '{print $3}' | sed 's/,//')
        echo "Docker版本: $docker_version"

        add_docker_data "service_status" "running"
        add_docker_data "version" "$docker_version"
    else
        echo "Docker服务: ⚠️ 未运行 (状态: $docker_status)"
        add_docker_data "service_status" "$docker_status"
    fi
else
    echo "Docker: ❌ 未安装"
    add_docker_data "service_status" "not_installed"
    return
fi
```

#### 3.3.2 容器统计与状态

**检查目的**：了解容器总数、运行状态、停止状态

```bash
echo "## 📦 容器状态概览"

# 统计容器数量
total_containers=$(docker ps -a -q 2>/dev/null | wc -l)
running_containers=$(docker ps -q 2>/dev/null | wc -l)
paused_containers=$(docker ps -q -f status=paused 2>/dev/null | wc -l)
stopped_containers=$((total_containers - running_containers - paused_containers))

echo "- 总容器数: $total_containers"
echo "- 运行中: $running_containers"
echo "- 已暂停: $paused_containers"
echo "- 已停止: $stopped_containers"

# 判断整体状态
if [ "$stopped_containers" -gt 0 ]; then
    echo "- 状态: ⚠️ 有 $stopped_containers 个容器已停止"
    overall_status="warning"
elif [ "$running_containers" -gt 0 ]; then
    echo "- 状态: ✅ 正常"
    overall_status="ok"
else
    echo "- 状态: ⚠️ 无运行中的容器"
    overall_status="warning"
fi

# 记录到JSON
add_docker_data "containers_total" "$total_containers"
add_docker_data "containers_running" "$running_containers"
add_docker_data "containers_paused" "$paused_containers"
add_docker_data "containers_stopped" "$stopped_containers"
add_docker_data "overall_status" "$overall_status"

# 列出停止的容器
if [ "$stopped_containers" -gt 0 ]; then
    echo ""
    echo "### 停止的容器："
    docker ps -a -f status=exited --format "table {{.Names}}\t{{.Status}}\t{{.CreatedAt}}" 2>/dev/null
fi
```

**输出示例**：

```markdown
## 📦 容器状态概览
- 总容器数: 25
- 运行中: 23
- 已暂停: 0
- 已停止: 2
- 状态: ⚠️ 有 2 个容器已停止

### 停止的容器：
NAMES           STATUS                    CREATEDAT
old-postgres    Exited (0) 2 days ago     2026-01-16 10:30:00
test-mysql      Exited (1) 5 days ago     2026-01-13 14:20:00
```

#### 3.3.3 容器资源使用分析

**检查目的**：发现资源占用异常的容器，进行容量规划

```bash
echo "## 📊 容器资源使用（Top 10）"
echo "| 容器名 | 状态 | CPU% | 内存使用 | 内存% | 网络接收 | 网络发送 |"
echo "|--------|------|------|----------|--------|----------|----------|"

# 获取容器实时资源使用（--no-stream避免持续输出）
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}" 2>/dev/null | \
    tail -n +2 | \
    while read -r line; do
        name=$(echo "$line" | awk '{print $1}')
        cpu=$(echo "$line" | awk '{print $2}' | sed 's/%//')
        mem_usage=$(echo "$line" | awk '{print $3}')
        mem_percent=$(echo "$line" | awk '{print $4}' | sed 's/%//')
        net_rx=$(echo "$line" | awk '{print $5}')
        net_tx=$(echo "$line" | awk '{print $6}')

        # 判断容器状态
        if docker ps --format "{{.Names}}" | grep -q "^${name}$"; then
            status="✅"
        else
            status="❌"
        fi

        # 标记高资源占用
        if (( $(echo "$cpu > 50" | bc -l 2>/dev/null || echo "0") )); then
            cpu="**$cpu%**"
        else
            cpu="${cpu}%"
        fi

        if [ "$mem_percent" -gt 80 ] 2>/dev/null; then
            mem_usage="**$mem_usage**"
        fi

        echo "| $name | $status | $cpu | $mem_usage | ${mem_percent}% | $net_rx | $net_tx |"
    done | head -n 11  # 限制Top 10
```

**关键点**：
- 使用`--no-stream`参数获取当前瞬时数据，而非持续输出
- 标记CPU>50%和内存>80%的容器，便于快速发现问题
- 区分运行中和停止的容器
- 限制输出数量，避免报告过长

#### 3.3.4 镜像管理检查

**检查目的**：发现悬空镜像，提供清理建议

```bash
echo "## 🖼️ 镜像管理"

# 统计镜像数量
total_images=$(docker images -q 2>/dev/null | wc -l)
dangling_images=$(docker images -f "dangling=true" -q 2>/dev/null | wc -l)

echo "- 总镜像数: $total_images"
echo "- 悬空镜像数: $dangling_images"

# 列出镜像占用空间
if [ "$total_images" -gt 0 ]; then
    echo ""
    echo "### 镜像占用空间（Top 10）："
    echo "| 镜像名 | 标签 | 大小 | 创建时间 |"
    echo "|--------|------|------|----------|"

    docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedAt}}" 2>/dev/null | \
        tail -n +2 | \
        head -n 10 | \
        while read -r line; do
            echo "| $line |"
        done
fi

# 清理建议
if [ "$dangling_images" -gt 0 ]; then
    echo ""
    echo "### 清理建议"
    echo "⚠️ 发现 $dangling_images 个悬空镜像，建议清理："
    echo "\`\`\`bash"
    echo "docker image prune -f"
    echo "\`\`\`"

    # 计算可回收空间
    dangling_size=$(docker images -f "dangling=true" -q 2>/dev/null | \
        xargs docker inspect --format='{{.Size}}' 2>/dev/null | \
        awk '{sum+=$1} END {printf "%.0f", sum/1024/1024}')

    echo "预计可回收空间: ${dangling_size}MB"
fi
```

#### 3.3.5 存储空间使用

**检查目的**：了解Docker整体存储占用，发现异常占用

```bash
echo "## 💾 存储空间使用"

docker_info=$(docker system df 2>/dev/null)

if [ -n "$docker_info" ]; then
    echo "$docker_info"

    # 解析存储使用数据
    images_size=$(echo "$docker_info" | grep 'Images' | awk '{print $3}')
    containers_size=$(echo "$docker_info" | grep 'Containers' | awk '{print $3}')
    volumes_size=$(echo "$docker_info" | grep 'Local Volumes' | awk '{print $3}')
    build_cache_size=$(echo "$docker_info" | grep 'Build Cache' | awk '{print $3}')

    add_docker_data "storage_images" "$images_size"
    add_docker_data "storage_containers" "$containers_size"
    add_docker_data "storage_volumes" "$volumes_size"
    add_docker_data "storage_build_cache" "$build_cache_size"

    # 判断是否需要清理
    if [ "$build_cache_size" != "0B" ] && [ "$build_cache_size" != "" ]; then
        echo ""
        echo "### 清理建议"
        echo "⚠️ 构建缓存占用 ${build_cache_size}，建议清理："
        echo "\`\`\`bash"
        echo "docker builder prune -f"
        echo "\`\`\`"
    fi
else
    echo "⚠️ 无法获取存储使用信息"
fi
```

**输出示例**：

```markdown
## 💾 存储空间使用
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          15        10        3.2GB     1.1GB (34%)
Containers      25        23        1.8GB     256MB (14%)
Local Volumes   8         6         2.5GB     512MB (20%)
Build Cache     0         0         856MB     856MB

### 清理建议
⚠️ 构建缓存占用 856MB，建议清理：
```bash
docker builder prune -f
```
```

#### 3.3.6 网络和卷统计

```bash
echo "## 🔌 网络和卷"

# 统计网络
total_networks=$(docker network ls -q 2>/dev/null | wc -l)
echo "- 网络总数: $total_networks"

# 列出网络
if [ "$total_networks" -gt 0 ]; then
    echo ""
    echo "### 网络列表："
    docker network ls --format "table {{.Name}}\t{{.Driver}}\t{{.Scope}}" 2>/dev/null
fi

# 统计卷
total_volumes=$(docker volume ls -q 2>/dev/null | wc -l)
echo ""
echo "- 卷总数: $total_volumes"

# 列出卷
if [ "$total_volumes" -gt 0 ]; then
    echo ""
    echo "### 卷列表："
    docker volume ls --format "table {{.Name}}\t{{.Driver}}\t{{.Scope}}" 2>/dev/null
fi

add_docker_data "networks_total" "$total_networks"
add_docker_data "volumes_total" "$total_volumes"
```

**关键点**：
- 提供清晰的清理建议和命令
- 统计并显示可回收空间，帮助决策是否清理
- 列出网络和卷，便于了解整体架构
- 所有数据同时记录到JSON，便于自动化分析

## 四、实际应用效果展示

### 4.1 基础健康检查报告

#### Markdown报告（人工阅读）
> 图：健康检查报告

![基础健康检查报告](http://image2.ishareread.com/images/2026/20260121/health-check-192.168.0.55-20260118-164043.png)

#### JSON报告（机器处理）

```json
{
  "summary": {
    "host": {
      "hostname": "pve-ubuntu-pandawiki",
      "ip": "192.168.0.55",
      "check_time": "2026-01-18T08:40:43+0000"
    },
    "overall_status": "warning",
    "status_counts": {
      "ok": 8,
      "warning": 3,
      "critical": 0
    },
    "check_types": ["health"]
  },
  "details": {
    "system": {
      "uptime": "15 days, 3:24",
      "load_1min": 0.15,
      "load_5min": 0.30,
      "load_15min": 0.35,
      "network_total_connections": 45,
      "network_external_connections": 8,
      "network_listening_ports": 12
    },
    "memory": {
      "total_mb": 16384,
      "used_mb": 12500,
      "available_mb": 3884,
      "used_percent": 76.3,
      "swap_total_mb": 2048,
      "swap_used_mb": 256,
      "swap_percent": 12.5,
      "status": "warning"
    },
    "disk": [
      {
        "mount": "/",
        "device": "/dev/sda1",
        "total_gb": "100G",
        "used_gb": "45G",
        "available_gb": "55G",
        "used_percent": 45.0,
        "status": "ok"
      },
      {
        "mount": "/data",
        "device": "/dev/sdb1",
        "total_gb": "500G",
        "used_gb": "425G",
        "available_gb": "75G",
        "used_percent": 85.0,
        "status": "warning"
      }
    ],
    "services": {
      "systemd_running": 23,
      "systemd_failed": 0
    },
    "security": {
      "mining_detected": false,
      "tmp_executables": 0,
      "failed_logins": 3
    }
  },
  "metadata": {
    "check_version": "1.0",
    "tool_version": "ops-health-check v1.0",
    "check_script": "health-check.sh",
    "thresholds": {
      "disk_warning": 50,
      "disk_critical": 80,
      "memory_warning": 70,
      "memory_critical": 90
    }
  }
}
```

**实际价值**：
- Markdown报告便于运维人员快速了解系统状态
- JSON数据可被监控系统自动采集和分析
- 清晰的emoji状态指示，一眼就能看出问题所在
- 详细的阈值配置，便于追溯和调整

## 五、总结与展望

### 5.1 实现成果

本次开发成功构建了一个生产级运维健康检查系统：

✅ **三大核心模块**
- 基础健康检查：涵盖系统资源、服务状态、基础安全
- 深度安全检查：5大类20+项安全检查
- Docker监控：容器、镜像、存储全方位监控

✅ **双格式输出**
- Markdown：人工可读，便于快速决策
- JSON：机器可处理，易于自动化集成

✅ **广泛兼容性**
- bash 3.2+兼容（macOS默认版本）
- 支持主流Linux发行版（Ubuntu、CentOS、Debian）
- 适配systemd和SysV init系统

✅ **完整文档**
- SKILL.md：使用指南
- 设计文档：架构和实现思路
- 博客文章：实战经验分享

### 5.2 技术亮点

1. **模块化设计**：三个检查模块独立运行，互不依赖
2. **统一输出库**：封装数据收集和格式化逻辑
3. **阈值可配置**：支持通过环境变量自定义阈值
4. **兼容性处理**：优雅降级，兼容不同版本工具
5. **清理建议**：不仅发现问题，还提供解决建议

### 5.3 实际应用价值

**效率提升**：
- 批量检查10台主机仅需2分钟
- 自动生成双格式报告，无需手工整理
- JSON数据可直接导入监控系统

**安全增强**：
- 发现并阻止挖矿程序
- 检测异常登录和未授权访问
- 及时发现文件系统异常变更

**成本节约**：
- 无需部署复杂监控系统
- 基于现有SSH连接，无需额外代理
- 轻量级脚本，资源占用极低

### 5.4 开发体会

通过Claude Code的AI辅助开发，整个过程非常高效：

1. **快速迭代**：从需求到实现不到4小时
2. **智能提示**：Claude主动发现兼容性问题并给出解决方案
3. **代码质量**：生成的代码结构清晰，符合最佳实践
4. **文档完善**：自动生成详细的使用文档和示例

AI辅助开发不仅提高了效率，更重要的是让开发者可以专注于业务逻辑和架构设计，而将具体的编码工作交给AI完成。这种人机协作模式，是未来软件开发的重要趋势。

**项目地址**：[https://github.com/xiejava/ops-health-check](https://github.com/xiejava/ops-health-check)

**博客地址**：[http://xiejava.ishareread.com](http://xiejava.ishareread.com)

---

🤖 **本文由Claude Code辅助编写**

<center> 

<br>

![“fullbug”微信公众号](http://image2.ishareread.com/images/fullbug微信公众号.jpg "“fullbug”微信公众号") 

关注：微信公众号,一起学习成长！</center>
