# ServerPulse — 项目设计 Plan

> 轻量级多服务器监控探针，服务端 & 客户端均 Docker 部署。

---

## 1. 项目概览

| 项 | 说明 |
|----|------|
| 项目名 | ServerPulse |
| 目标 | 统一收集多台服务器的硬件指标与流量，通过 Web Dashboard 展示 |
| 部署方式 | 服务端 & 客户端均使用 Docker / Docker Compose |
| 认证模型 | 管理员账号 + 每台客户端独立 Token |
| 流量统计 | 由宿主机 **vnstat** 提供，天然支持重启续计，服务端按自定义重置日汇总 |

---

## 2. 整体架构

```
┌──────────────────────────┐     HTTPS / Token      ┌───────────────────────────┐
│   Client Agent (Docker)  │ ──────────────────────> │  Server (API + Web UI)    │
│                          │                         │  (Docker)                 │
│  psutil                  │                         │                           │
│  · CPU 使用率/频率/温度  │                         │  · REST API               │
│  · 内存 + Swap 使用率    │                         │  · WebSocket 实时推送     │
│  · 磁盘占用率 + I/O 速率 │                         │  · JWT 登录               │
│  · 系统负载 load avg     │                         │  · Token 管理             │
│  · 进程数 / TCP 连接数   │                         │  · 流量周期汇总（vnstat） │
│  · 实时网速（psutil）    │                         │  · 服务探测 HTTP/TCP/Ping │
│  · GPU 温度/使用率       │                         │  · SSL 证书监控           │
│  · Uptime / 在线天数     │                         │  · 告警引擎               │
│                          │                         │  · 静态 Web Dashboard     │
│  vnstat（宿主机守护进程）│                         │                           │
│  · 月/日流量（自然月）   │                         └───────────┬───────────────┘
│  · 重启后自动续计        │                                     │
└──────────────────────────┘                         ┌───────────▼───────────────┐
                                                     │  SQLite / PostgreSQL       │
                                                     │  指标历史 + 流量周期归档  │
                                                     └───────────────────────────┘
```

---

## 3. 技术选型

### 3.1 服务端

| 组件 | 推荐技术 | 说明 |
|------|----------|------|
| Web 框架 | **FastAPI** (Python) | 异步、内置 OpenAPI 文档 |
| ORM | **SQLAlchemy 2.0** + Alembic | 迁移管理方便 |
| 数据库 | **SQLite**（默认）/ PostgreSQL（可选） | 小部署用 SQLite 即可 |
| 认证 | **JWT** (python-jose) | Access Token + Refresh Token |
| WebSocket | FastAPI 内置 | 实时推送最新指标 |
| 定时任务 | APScheduler | 清理过期数据、服务探测、归零检查 |
| Web UI | **Vue 3 + Vite** | 单页应用，打包后内嵌服务端 |
| 图表库 | ECharts | 历史趋势 + 流量柱状图 |
| 容器 | Docker + Docker Compose | 一键启动 |

### 3.2 客户端 Agent

| 组件 | 推荐技术 | 说明 |
|------|----------|------|
| 语言 | **Python 3.11+** | 跨平台、依赖丰富 |
| 系统指标 | **psutil** | CPU / 内存 / Swap / 磁盘 / 负载 / 进程 / Uptime |
| 温度 | **psutil.sensors_temperatures()** | Linux 主流主板均支持 |
| 磁盘 I/O | **psutil.disk_io_counters()** | 读写速率，两次差值计算 |
| 实时网速 | **psutil.net_io_counters()** | 仅用于计算 bytes/s，不做周期统计 |
| **流量统计** | **vnstat --json** | 宿主机守护进程，重启续计，按日/月输出 |
| GPU | **pynvml** (NVIDIA) / 可选 AMD | 无 GPU 时优雅跳过 |
| HTTP 上报 | **httpx** | 异步上报，支持重试 |
| 容器 | Docker (network_mode: host + privileged) | 访问 /sys、vnstat 数据库 |

---

## 4. 流量统计设计（vnstat 方案）

### 4.1 为什么用 vnstat

psutil 的 `net_io_counters()` 读取的是**内核启动累计值**，机器重启后清零。
自己在服务端维护基准值来处理重启，逻辑复杂且容易出错（需要"已确认量 + 当前段基准"两段式结构）。

vnstat 是 Linux 专用流量统计守护进程，解决了所有这些问题：
- 自己持久化每个网卡的流量到 `/var/lib/vnstat/`，重启后**自动续计**
- 原生提供按日、按月、按年的分段统计
- `vnstat --json` 直接输出结构化数据，Agent 一行命令采集完毕
- 宿主机安装一次，容器挂载数据库目录即可读取

### 4.2 Agent 采集 vnstat 数据

```python
import subprocess, json

def collect_vnstat(ifaces=None):
    """
    调用宿主机 vnstat，返回各网卡当月和当日流量。
    ifaces: None = 全部非 lo 网卡，['eth0'] = 指定网卡
    """
    result = subprocess.run(
        ["vnstat", "--json", "2"],   # 2 = 输出最近 2 个月数据
        capture_output=True, text=True
    )
    if result.returncode != 0:
        return None   # vnstat 未安装或无数据，降级处理

    data = json.loads(result.stdout)
    output = []

    for iface in data["interfaces"]:
        name = iface["name"]
        if name == "lo":
            continue
        if ifaces and name not in ifaces:
            continue

        months = iface["traffic"]["month"]
        days   = iface["traffic"]["day"]

        current_month = months[-1] if months else {"rx": 0, "tx": 0}
        today_data    = days[-1]   if days   else {"rx": 0, "tx": 0}

        output.append({
            "iface":     name,
            "month_rx":  current_month["rx"],   # 自然月累计接收字节
            "month_tx":  current_month["tx"],   # 自然月累计发送字节
            "today_rx":  today_data["rx"],       # 今日接收字节
            "today_tx":  today_data["tx"],       # 今日发送字节
            # 每日明细（用于服务端按自定义重置日汇总）
            "daily":  [
                {
                    "date": f"{d['date']['year']}-{d['date']['month']:02d}-{d['date']['day']:02d}",
                    "rx":   d["rx"],
                    "tx":   d["tx"],
                }
                for d in days
            ],
        })

    return output
```

### 4.3 服务端按自定义重置日汇总

vnstat 的月统计是自然月（1号），若用户配置"每月15号归零"，服务端用每日明细加总：

```python
from datetime import date, timedelta

def calc_cycle_usage(daily_data: list, reset_day: int) -> dict:
    """
    daily_data: vnstat 上报的每日列表 [{"date":"2024-05-15","rx":x,"tx":x}, ...]
    reset_day:  每月几号为周期起点（1~28）
    返回本周期内各日的 rx/tx 总和
    """
    today = date.today()

    if today.day >= reset_day:
        cycle_start = today.replace(day=reset_day)
    else:
        # 上个月的 reset_day
        first_of_month = today.replace(day=1)
        prev_month_end = first_of_month - timedelta(days=1)
        cycle_start = prev_month_end.replace(day=reset_day)

    total_rx = total_tx = 0
    daily_breakdown = []

    for d in daily_data:
        day_date = date.fromisoformat(d["date"])
        if day_date >= cycle_start:
            total_rx += d["rx"]
            total_tx += d["tx"]
            daily_breakdown.append(d)

    return {
        "cycle_start":     cycle_start.isoformat(),
        "total_rx":        total_rx,
        "total_tx":        total_tx,
        "daily_breakdown": daily_breakdown,
    }
```

### 4.4 上报数据结构（流量部分）

```json
"vnstat": [
  {
    "iface":    "eth0",
    "month_rx": 5368709120,
    "month_tx": 1073741824,
    "today_rx": 214748364,
    "today_tx": 53687091,
    "daily": [
      {"date": "2024-05-01", "rx": 107374182, "tx": 26843545},
      {"date": "2024-05-02", "rx": 214748364, "tx": 53687091}
    ]
  }
]
```

### 4.5 servers 表流量相关字段（大幅简化）

去掉了原方案中的 `cycle_baseline_*`、`cycle_confirmed_*`、`cycle_prev_*` 等复杂字段，
服务端按需从 vnstat 每日数据实时计算，无需维护状态：

```sql
traffic_reset_day    INTEGER DEFAULT 1    -- 每月几号为周期起点（1~28）
monitored_interfaces TEXT DEFAULT ''      -- 监控网卡，逗号分隔，空=全部
traffic_limit_gb     INTEGER DEFAULT 0    -- 套餐流量上限 GB，0=不限（仅用于进度条）
```

### 4.6 Docker 部署注意：vnstat 前置依赖

宿主机需提前安装并启动 vnstat 守护进程：

```bash
# 宿主机执行
apt install vnstat
systemctl enable --now vnstatd
# 首次需等待约 5 分钟让 vnstat 完成网卡初始化
vnstat --iflist   # 确认网卡已被监控
```

Agent 容器挂载宿主机的 vnstat 数据库和 vnstat 二进制：

```yaml
agent:
  privileged: true
  network_mode: host
  volumes:
    - /var/lib/vnstat:/var/lib/vnstat:ro   # vnstat 数据库（只读）
    - /usr/bin/vnstat:/usr/bin/vnstat:ro   # vnstat 可执行文件
    - /proc:/host/proc:ro
    - /sys:/host/sys:ro
```

若宿主机 vnstat 不可用，Agent 降级为仅上报实时网速，流量统计显示"不可用"，不影响其他指标。

---

## 5. 补充指标（对比审查后新增）

### 5.1 新增采集项

| 指标 | 采集方法 | 说明 |
|------|----------|------|
| 系统负载 load avg | `psutil.getloadavg()` | 1min / 5min / 15min，CPU 压力诊断必需 |
| Swap 使用率 | `psutil.swap_memory()` | 内存告警前奏，VPS 用户尤其重要 |
| 磁盘 I/O 速率 | `psutil.disk_io_counters()` | 读写 bytes/s + IOPS，诊断 I/O 瓶颈 |
| 进程数 | `len(psutil.pids())` | 可选，配置跳过 |
| TCP 连接数 | `len(psutil.net_connections())` | 可选，高连接数机器建议跳过 |

### 5.2 服务端新增：主动服务探测

服务端独立模块，不依赖 Agent，定期主动探测目标：

```
services_monitors 表：
  id, name, target, type(http/tcp/ping), interval_secs,
  expected_status, ssl_check, is_active, created_at

service_results 表：
  id, monitor_id, checked_at, is_up, latency_ms,
  ssl_expiry_days, status_code, error_msg
```

探测类型：
- **HTTP/HTTPS**：检查状态码 + 响应时间，HTTPS 顺带检查 SSL 证书到期天数，30天内告警
- **TCP**：检查端口是否可达
- **ICMP Ping**：检查主机可达性 + 延迟

---

## 6. 完整数据模型

### 6.1 servers

```sql
id                   UUID PRIMARY KEY
name                 TEXT NOT NULL
token                TEXT UNIQUE NOT NULL       -- SHA-256
description          TEXT
os                   TEXT
hostname             TEXT
ip                   TEXT                       -- 上报时自动记录
created_at           DATETIME
last_seen            DATETIME
is_active            BOOLEAN DEFAULT TRUE

-- 流量（vnstat 方案，字段极简）
traffic_reset_day    INTEGER DEFAULT 1
monitored_interfaces TEXT DEFAULT ''
traffic_limit_gb     INTEGER DEFAULT 0
```

### 6.2 metrics

```sql
id             UUID PRIMARY KEY
server_id      UUID REFERENCES servers(id)
recorded_at    DATETIME NOT NULL

-- CPU
cpu_usage      FLOAT               -- 总使用率 %
cpu_freq       FLOAT               -- 当前频率 MHz
cpu_temp       FLOAT               -- °C，可 NULL
load_avg_1     FLOAT               -- 1 分钟负载
load_avg_5     FLOAT               -- 5 分钟负载
load_avg_15    FLOAT               -- 15 分钟负载

-- 内存
mem_total      BIGINT              -- bytes
mem_used       BIGINT
mem_percent    FLOAT
swap_total     BIGINT
swap_used      BIGINT
swap_percent   FLOAT

-- 磁盘（多挂载点）
disks          JSON   -- [{"mount":"/","total":x,"used":x,"free":x,"percent":x,
                      --   "read_bytes_s":x,"write_bytes_s":x,"read_iops":x,"write_iops":x}, ...]

-- 实时网速（psutil，不做周期统计）
net_speed_up   FLOAT               -- bytes/s 上行
net_speed_down FLOAT               -- bytes/s 下行
net_interfaces JSON   -- [{"iface":"eth0","speed_up":x,"speed_down":x}, ...]

-- 流量（vnstat，每日明细由 Agent 上报，服务端按重置日汇总）
vnstat         JSON   -- 见 §4.4 结构

-- 进程 & 连接（可选）
process_count  INTEGER
tcp_conn_count INTEGER

-- GPU（可 NULL）
gpus           JSON   -- [{"name":"RTX4090","temp":65,"usage":88,
                      --   "mem_used":x,"mem_total":x,"mem_percent":x}, ...]

-- Uptime
uptime_secs    BIGINT

INDEX (server_id, recorded_at)
```

### 6.3 traffic_cycles（历史周期存档，归零时写入）

```sql
id           UUID PRIMARY KEY
server_id    UUID REFERENCES servers(id)
cycle_start  DATE NOT NULL
cycle_end    DATE NOT NULL
total_rx     BIGINT              -- 本周期总接收字节
total_tx     BIGINT              -- 本周期总发送字节
created_at   DATETIME
```

### 6.4 service_monitors（服务探测配置）

```sql
id               UUID PRIMARY KEY
name             TEXT NOT NULL
target           TEXT NOT NULL        -- URL / IP:port / domain
type             TEXT NOT NULL        -- 'http' | 'tcp' | 'ping'
interval_secs    INTEGER DEFAULT 60
expected_status  INTEGER              -- HTTP 期望状态码，NULL=任意2xx
ssl_check        BOOLEAN DEFAULT TRUE -- HTTPS 时检查证书
is_active        BOOLEAN DEFAULT TRUE
created_at       DATETIME
```

### 6.5 service_results

```sql
id               UUID PRIMARY KEY
monitor_id       UUID REFERENCES service_monitors(id)
checked_at       DATETIME NOT NULL
is_up            BOOLEAN
latency_ms       FLOAT
ssl_expiry_days  INTEGER              -- NULL = 非 HTTPS
status_code      INTEGER
error_msg        TEXT

INDEX (monitor_id, checked_at)
```

### 6.6 users

```sql
id           UUID PRIMARY KEY
username     TEXT UNIQUE NOT NULL
password     TEXT NOT NULL          -- bcrypt
role         TEXT DEFAULT 'admin'
created_at   DATETIME
```

### 6.7 alert_rules

```sql
id            UUID PRIMARY KEY
server_id     UUID REFERENCES servers(id)   -- NULL = 全局
monitor_id    UUID REFERENCES service_monitors(id)  -- NULL = 非服务探测
metric        TEXT
threshold     FLOAT
operator      TEXT    -- '>' | '<' | '>='
notify_type   TEXT    -- 'webhook' | 'telegram' | 'email'
notify_target TEXT    -- webhook URL / telegram chat_id / email
is_active     BOOLEAN DEFAULT TRUE
```

**支持告警的指标 Key：**

| Key | 说明 | 示例阈值 |
|-----|------|---------|
| `cpu_usage` | CPU 使用率 % | > 90 |
| `cpu_temp` | CPU 温度 °C | > 85 |
| `load_avg_5` | 5分钟负载 | > 核心数×0.8 |
| `mem_percent` | 内存使用率 % | > 85 |
| `swap_percent` | Swap 使用率 % | > 50 |
| `disk_percent` | 磁盘使用率 % | > 90 |
| `gpu_temp` | GPU 温度 °C | > 80 |
| `net_speed_up` | 实时上行 bytes/s | > 900MB/s |
| `cycle_traffic_total_gb` | 本周期总流量 GB | > 1000 |
| `service_down` | 服务探测失败 | — |
| `ssl_expiry_days` | SSL 证书剩余天数 | < 30 |

---

## 7. API 设计

### 7.1 客户端上报

```
POST /api/v1/report
Header: X-Agent-Token: <token>
Body:
{
  "cpu_usage": 42.3,
  "cpu_freq": 3600,
  "cpu_temp": 58.2,
  "load_avg_1": 1.2,
  "load_avg_5": 0.9,
  "load_avg_15": 0.7,
  "mem_total": 17179869184,
  "mem_used": 9126805504,
  "mem_percent": 53.1,
  "swap_total": 4294967296,
  "swap_used": 536870912,
  "swap_percent": 12.5,
  "disks": [
    {
      "mount": "/", "total": 107374182400, "used": 53687091200,
      "free": 53687091200, "percent": 50.0,
      "read_bytes_s": 1048576, "write_bytes_s": 524288,
      "read_iops": 120, "write_iops": 60
    }
  ],
  "net_speed_up": 1048576,
  "net_speed_down": 5242880,
  "net_interfaces": [
    {"iface": "eth0", "speed_up": 1048576, "speed_down": 5242880}
  ],
  "vnstat": [
    {
      "iface": "eth0",
      "month_rx": 5368709120, "month_tx": 1073741824,
      "today_rx": 214748364,  "today_tx": 53687091,
      "daily": [
        {"date": "2024-05-01", "rx": 107374182, "tx": 26843545}
      ]
    }
  ],
  "process_count": 312,
  "tcp_conn_count": 128,
  "gpus": [
    {"name": "NVIDIA RTX 4090", "temp": 65, "usage": 88,
     "mem_used": 8589934592, "mem_total": 25769803776, "mem_percent": 33.3}
  ],
  "uptime_secs": 864000,
  "os": "Ubuntu 22.04.3 LTS",
  "hostname": "web-server-01"
}
```

### 7.2 流量 API

```
-- 本周期流量摘要（服务端实时从 vnstat 每日数据汇总）
GET /api/servers/{id}/traffic/current
Response:
{
  "cycle_start": "2024-05-15",
  "reset_day": 15,
  "next_reset": "2024-06-15",
  "days_until_reset": 10,
  "total_rx_bytes": 2147483648,
  "total_tx_bytes": 536870912,
  "total_bytes": 2684354560,
  "limit_gb": 1000,
  "percent_used": 2.5,
  "daily_breakdown": [...]
}

-- 历史周期列表
GET /api/servers/{id}/traffic/history?limit=12

-- 更新流量配置
PATCH /api/servers/{id}/traffic/config
Body: { "reset_day": 15, "monitored_interfaces": ["eth0"], "limit_gb": 1000 }
```

### 7.3 服务探测 API

```
GET    /api/monitors                    所有探测配置
POST   /api/monitors                    新建探测
PATCH  /api/monitors/{id}
DELETE /api/monitors/{id}
GET    /api/monitors/{id}/history       探测历史（延迟、状态）
GET    /api/monitors/{id}/ssl           SSL 证书信息
```

### 7.4 其他 API

```
POST /api/auth/login | /refresh | /logout

GET    /api/servers
POST   /api/servers
GET    /api/servers/{id}
PATCH  /api/servers/{id}
DELETE /api/servers/{id}
POST   /api/servers/{id}/rotate-token

GET /api/servers/{id}/metrics
  ?from=&to=&resolution=raw|5m|1h|1d
  &fields=cpu_usage,load_avg_5,mem_percent,swap_percent,...

WS /ws/metrics

GET/POST/PATCH/DELETE /api/alerts
GET /api/alerts/history
```

---

## 8. Web Dashboard 页面设计

### 8.1 总览页 `/`
每台服务器一张卡片：
- 服务器名 + 在线/离线状态 + IP 归属地（可选）
- CPU 使用率 · 温度 · **5分钟负载**
- 内存使用率 + **Swap 使用率**
- 磁盘总使用率
- 本周期流量：↑/↓ 已用 + 距重置天数（有上限则显示进度条）
- 实时网速 ↑/↓
- GPU 信息（有则显示）
- 在线天数

### 8.2 单机详情页 `/server/:id`

**顶部概览栏**：服务器名、OS、Hostname、IP、Uptime、本周期流量卡片

**实时指标区（WebSocket）**
- CPU：使用率环形图 + 频率 + 温度 + 负载（1/5/15min）
- 内存：used / cached / free 堆叠 + Swap 进度条
- 磁盘：各分区占用进度条 + **读写速率 + IOPS**
- 网络：实时网速仪表盘
- GPU：使用率 + 显存 + 温度（有则显示）
- 进程数 / TCP 连接数（有则显示）

**历史图表区（1h / 6h / 24h / 7d / 30d）**
- CPU 使用率 & 温度 & 负载折线图
- 内存 & Swap 使用率折线图
- 磁盘使用率折线图（各分区）
- **磁盘 I/O 读写速率折线图**
- 网速历史折线图（上行 / 下行）
- 本周期内每日流量柱状图（vnstat 每日数据）
- GPU 温度 & 使用率折线图（有则显示）

**历史周期流量**：过去 N 个周期总量表格

### 8.3 服务监控页 `/monitors`
- 所有探测项列表：名称、类型、状态（在线/离线）、延迟、SSL到期天数
- 30天可用率时间轴（仿 Uptime Kuma 风格）
- 点击进入单项历史延迟折线图

### 8.4 管理后台 `/admin`

**服务器管理**
- 新增 → 生成 Token（仅展示一次）
- 编辑：名称、备注、流量重置日、监控网卡、套餐上限
- 重新生成 Token / 删除

**服务探测管理**
- 新增 / 编辑 / 删除探测目标
- SSL 告警配置

**告警规则**：指标阈值 + 通知渠道（Telegram / Webhook / 邮件）

**用户管理**：修改密码（预留多用户）

### 8.5 设置页 `/settings`
- 数据保留天数
- 服务探测全局间隔
- 系统版本

---

## 9. 客户端 Agent 设计

### 9.1 配置（环境变量 / config.yaml）

```yaml
server_url: "https://pulse.example.com"
token: "your-agent-token"
report_interval: 10           # 秒

network_interfaces: []        # 留空 = 全部非 lo；指定 ["eth0"]
disk_mounts: []               # 留空 = 自动扫描
gpu_enabled: true
skip_process_count: false     # 进程数采集（高负载机器可关）
skip_tcp_connections: false   # TCP 连接数采集
```

### 9.2 采集模块

```
agent/collector/
  cpu.py        psutil.cpu_percent / cpu_freq / sensors_temperatures / getloadavg
  memory.py     psutil.virtual_memory + swap_memory
  disk.py       psutil.disk_partitions + disk_usage + disk_io_counters（差值算速率）
  network.py    psutil.net_io_counters（仅算实时速率）
  vnstat.py     subprocess vnstat --json（流量周期统计）★ 新增
  gpu.py        pynvml（NVIDIA），无 GPU 时返回 None
  process.py    psutil.pids + net_connections（可选）
  uptime.py     psutil.boot_time
```

### 9.3 vnstat.py 降级处理

```python
def collect_vnstat(ifaces=None):
    try:
        result = subprocess.run(
            ["vnstat", "--json", "2"],
            capture_output=True, text=True, timeout=5
        )
        if result.returncode != 0:
            return None   # 降级：流量数据不可用
        return parse_vnstat(json.loads(result.stdout), ifaces)
    except (FileNotFoundError, subprocess.TimeoutExpired):
        return None       # vnstat 未安装，静默跳过
```

### 9.4 上报流程

```
1. 启动时 POST /api/v1/register（OS、Hostname）
2. 每 report_interval 秒：
   a. 并行采集各模块（asyncio.gather）
   b. POST /api/v1/report
   c. 失败则本地 deque 缓冲（最多 100 条），网络恢复后补报
```

---

## 10. Docker 部署方案

### 10.1 服务端 docker-compose.yml

```yaml
version: "3.9"
services:
  server:
    image: serverpulse-server:latest
    build: ./server
    ports:
      - "8080:8080"
    volumes:
      - ./data:/app/data
    environment:
      - SECRET_KEY=change-me
      - DB_URL=sqlite:////app/data/pulse.db
      - ADMIN_USERNAME=admin
      - ADMIN_PASSWORD=change-me
      - DATA_RETENTION_DAYS=30
    restart: unless-stopped
```

### 10.2 客户端 docker-compose.yml

```yaml
version: "3.9"
services:
  agent:
    image: serverpulse-agent:latest
    build: ./agent
    network_mode: host
    pid: host
    privileged: true
    volumes:
      - /var/lib/vnstat:/var/lib/vnstat:ro   # vnstat 数据库（只读）
      - /usr/bin/vnstat:/usr/bin/vnstat:ro   # vnstat 可执行文件
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
    environment:
      - SERVER_URL=https://pulse.example.com
      - AGENT_TOKEN=your-token-here
      - REPORT_INTERVAL=10
      - GPU_ENABLED=true
      - NETWORK_INTERFACES=                  # 留空=全部
    restart: unless-stopped
```

### 10.3 目录结构

```
serverpulse/
├── server/
│   ├── Dockerfile
│   ├── app/
│   │   ├── main.py
│   │   ├── api/
│   │   │   ├── auth.py
│   │   │   ├── servers.py
│   │   │   ├── metrics.py
│   │   │   ├── traffic.py        ← vnstat 周期汇总
│   │   │   ├── monitors.py       ← 服务探测 ★ 新增
│   │   │   └── alerts.py
│   │   ├── models/
│   │   ├── schemas/
│   │   ├── services/
│   │   │   ├── probe.py          ← HTTP/TCP/Ping 探测引擎 ★ 新增
│   │   │   ├── ssl_check.py      ← SSL 证书检查 ★ 新增
│   │   │   ├── alert_engine.py
│   │   │   └── data_retention.py
│   │   └── ws/
│   │       └── hub.py
│   └── frontend/
│       └── src/views/
│           ├── Login.vue
│           ├── Overview.vue
│           ├── ServerDetail.vue
│           ├── Monitors.vue      ★ 新增
│           └── Admin.vue
│
├── agent/
│   ├── Dockerfile
│   ├── main.py
│   └── collector/
│       ├── cpu.py
│       ├── memory.py
│       ├── disk.py
│       ├── network.py
│       ├── vnstat.py             ★ 新增
│       ├── gpu.py
│       ├── process.py            ★ 新增
│       └── uptime.py
│
├── docker-compose.server.yml
├── docker-compose.agent.yml
└── README.md
```

---

## 11. 安全设计

| 项目 | 方案 |
|------|------|
| Agent Token | 随机 32 字节 hex，SHA-256 存库，明文仅展示一次 |
| 管理员登录 | bcrypt 哈希，JWT（Access 15min + Refresh 7d） |
| HTTPS | 前置 Nginx/Traefik 层终止 TLS |
| Rate Limit | 上报接口每 Token 最多 1次/5秒 |
| vnstat 挂载 | 只读挂载 `:ro`，容器无法篡改宿主机数据 |
| 输入校验 | Pydantic V2 严格校验所有上报字段 |
| 告警通知 | Telegram Token / SMTP 凭证存服务端环境变量，不落库 |

---

## 12. 开发阶段划分

### Phase 1 — MVP（约 2-3 周）
- [ ] 服务端：FastAPI + SQLite + 上报接口 + JWT 登录
- [ ] 客户端：CPU（含负载）/ 内存+Swap / 磁盘+I/O / 实时网速 / Uptime
- [ ] vnstat 集成：Agent 采集 + 服务端按重置日汇总
- [ ] Web UI：登录 + 总览 + 单机详情（基础图表 + 流量卡片）
- [ ] Docker Compose 完整跑通

### Phase 2 — 服务监控（约 1 周）
- [ ] 服务端探测引擎：HTTP / TCP / Ping
- [ ] SSL 证书到期检查
- [ ] 服务监控页（可用率时间轴 + 延迟图）
- [ ] 告警：Telegram Bot + Webhook

### Phase 3 — 完善（约 1 周）
- [ ] GPU 支持（NVIDIA pynvml）
- [ ] WebSocket 实时推送替代轮询
- [ ] 数据保留策略自动清理 + 历史数据降采样
- [ ] 管理后台：流量配置 + 告警规则 UI

### Phase 4 — 增强（可选）
- [ ] 进程数 / TCP 连接数（可选采集）
- [ ] IP 归属地显示（MaxMind GeoLite2）
- [ ] 服务器分组 / 标签
- [ ] 公开状态页 `/status`（无需登录）
- [ ] 多用户 / 只读权限
- [ ] AMD GPU 支持
- [ ] 移动端响应式
- [ ] Docker Hub 镜像发布

---

## 13. 快速上手

```bash
# 宿主机（被监控机器）安装 vnstat
apt install vnstat && systemctl enable --now vnstatd
# 等待约 5 分钟初始化，验证：
vnstat -i eth0

# 1. 启动服务端
cp .env.example .env   # 配置 SECRET_KEY / 管理员密码
docker compose -f docker-compose.server.yml up -d

# 2. 打开 http://your-server:8080
#    管理后台 → 新建服务器 → 复制 Token
#    设置流量重置日（如每月 1 日）

# 3. 被监控机器启动 Agent
SERVER_URL=http://your-server:8080 \
AGENT_TOKEN=paste-token-here \
docker compose -f docker-compose.agent.yml up -d

# 4. 刷新 Dashboard，即可看到实时指标、流量统计、服务探测
```

---

*ServerPulse — 轻量、自托管、无外部依赖的服务器监控探针。*