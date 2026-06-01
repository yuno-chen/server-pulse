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
| 容器 | Docker (network_mode: host + 最小 capability) | 访问 /sys、vnstat 数据库（SYS_PTRACE 可选） |

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

def calc_cycle_usage(
    daily_data: list,
    reset_day: int,
    tz_name: str = "UTC",
) -> dict:
    """
    daily_data: vnstat 上报的每日列表 [{"date":"2024-05-15","rx":x,"tx":x}, ...]
    reset_day:  每月几号为周期起点（1~28，DB 层 CHECK 约束保证）
    tz_name:    Agent 上报的 IANA 时区，如 "Asia/Shanghai"
                所有"今天/本月"判断必须用客户端时区，避免 Agent 容器
                与宿主机时区不一致导致的周期错位
    返回本周期内各日的 rx/tx 总和
    """
    from zoneinfo import ZoneInfo
    tz = ZoneInfo(tz_name)
    today = datetime.now(tz).date()

    # 1~28 在任何月份都安全，DB 层 CHECK 约束保证
    if today.day >= reset_day:
        cycle_start = today.replace(day=reset_day)
    else:
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

    # next_reset：当前周期的下一个周期起点，永远是"下个月的 reset_day"
    # reset_day 已被 DB CHECK 约束在 1~28，对任何月份都安全
    next_month_first = (today.replace(day=1) + timedelta(days=32)).replace(day=1)
    next_reset = next_month_first.replace(day=reset_day)

    return {
        "cycle_start":       cycle_start.isoformat(),
        "cycle_end":         (next_reset - timedelta(days=1)).isoformat(),
        "next_reset":        next_reset.isoformat(),
        "days_until_reset":  (next_reset - today).days,
        "total_rx":          total_rx,
        "total_tx":          total_tx,
        "daily_breakdown":   daily_breakdown,
    }
```

### 4.4 上报数据结构（流量部分）

```json
"tz": "Asia/Shanghai",
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

`tz` 是 Agent 启动时从宿主机 `timedatectl` / `/etc/timezone` 读出的 IANA 时区名，
服务端按它来判定"今天"和周期边界。`servers.timezone` 首次上报时自动落库，
后续可手动覆盖（容器宿主机时区不对时可手动修正）。

### 4.5 servers 表流量相关字段（大幅简化）

去掉了原方案中的 `cycle_baseline_*`、`cycle_confirmed_*`、`cycle_prev_*` 等复杂字段，
服务端按需从 vnstat 每日数据实时计算，无需维护状态：

```sql
traffic_reset_day    INTEGER DEFAULT 1
  CHECK (traffic_reset_day BETWEEN 1 AND 28)   -- 限制 1~28，避免 29/30/31 在小月不存在
monitored_interfaces TEXT DEFAULT ''           -- 监控网卡，逗号分隔，空=全部
traffic_limit_gb     INTEGER DEFAULT 0
  CHECK (traffic_limit_gb >= 0)                -- 0=不限
```

> **为什么不支持 29~31？** 2 月没有 29~31 号，跨月周期计算需特殊处理且容易出 bug。
> 99% 的 VPS 服务商账单日在 1~28 区间，限制后语义更清晰。需要"月末"的场景建议用 28 号近似。
> 后续如果业务上确实需要 29~31，可改为"月末 = min(reset_day, 当月最后一天)"逻辑，但 DB 约束先保持 1~28。

### 4.6 Docker 部署注意：vnstat 前置依赖

宿主机需提前安装并启动 vnstat 守护进程：

```bash
# 宿主机执行
apt install vnstat
systemctl enable --now vnstatd
# 首次需等待约 5 分钟让 vnstat 完成网卡初始化
vnstat --iflist   # 确认网卡已被监控
```

#### 容器权限方案（已降权）

Agent 容器**不**使用 `privileged: true`，改为精确的 capability 集合：

```yaml
agent:
  # 不再 privileged: true，仅授予读系统信息的最小 cap
  # SYS_PTRACE 留给 /host/proc 内的进程名读取（可选）
  cap_add:
    - SYS_PTRACE       # /host/proc 内的 cmdline 读取
  cap_drop:
    - ALL               # 先全清再加需要的
  security_opt:
    - no-new-privileges:true
  read_only: true       # 根文件系统只读
  tmpfs:
    - /tmp:size=64m     # 缓冲队列落盘
    - /run:size=8m
  network_mode: host    # 实时网速需要原始 socket 视角（可选，去掉则用 net=bridge 也可）
  pid: host             # 进程数采集用，不采集可去掉 → 进一步降权
  volumes:
    - /var/lib/vnstat:/var/lib/vnstat:ro   # vnstat 数据库（只读）
    - /usr/bin/vnstat:/usr/bin/vnstat:ro   # vnstat 可执行文件
    - /proc:/host/proc:ro
    - /sys:/host/sys:ro
    # 时区文件，方便拿 tz
    - /etc/timezone:/etc/timezone:ro
    - /etc/localtime:/etc/localtime:ro
```

**降权后验证清单**：
- 启动后 vnstat 数据正常读取
- psutil CPU/内存/磁盘/温度均正常
- 如不需要进程数 / TCP 连接数，可一并去掉 `pid: host` 和 `cap_add`，容器权限面缩到最小

#### vnstat 不可用降级

若宿主机 vnstat 未安装 / 数据库损坏 / 进程未运行：
- Agent 捕获异常后 `vnstat` 字段上报 `null`，`vnstat_error` 字段上报错误信息
- 服务端在 Dashboard 显示"流量统计不可用"提示（橙色徽章）
- 不影响 CPU/内存/磁盘/实时网速等其他指标
- 触发 `vnstat_unavailable` 告警（Phase 3 实施）

#### vnstat 数据库完整性

vnstat 数据库为单个二进制文件 (`/var/lib/vnstat/vnstat.db`)，理论上损坏概率低但不为零：
- **监控**：Agent 上报时附带 `vnstat_db_mtime` 和 `vnstat_db_size`，服务端对比上次值
  - 若 mtime 倒退或 size 骤降 → 标记 `vnstat_db_suspect=true`
  - 触发 `vnstat_db_anomaly` 告警，建议管理员备份 `/var/lib/vnstat` 排查
- **修复**：vnstat 自带 `vnstat --rebuilddb`，可在 Agent 容器内 `docker exec` 调用，但**仅在管理员手动确认后**
  - 设计上提供 `POST /api/servers/{id}/rebuild-vnstat` 管理端点，执行后向 Agent 下发命令
- **冷启动**：Agent 启动时若发现 vnstat db 不存在 / 0 字节，自动跳过 vnstat 采集，等待宿主机 `vnstatd` 写入第一条数据后再启用

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

### 5.3 离线判定规则

明确"多久没上报算离线"，所有卡片状态、告警、统计都依赖此定义：

| 状态 | 判定条件 | UI 显示 |
|------|---------|---------|
| `online` | `now - last_seen ≤ 2 × report_interval`（默认 ≤ 20s） | 绿色圆点 |
| `stale` | `2 × interval < now - last_seen ≤ 5 min` | 黄色圆点 + "数据滞后" |
| `offline` | `now - last_seen > 5 min` | 红色圆点 + "离线" |
| `disabled` | `is_active = false` | 灰色 + "已停用" |

- `report_interval` 在 Server 默认 10s，Agent 端可改；阈值随之自动缩放
- `offline` 状态触发 `server_offline` 告警（**Phase 1 即支持**，用 `last_seen` 字段即可实现，无需依赖任何 metrics）
- `stale` 状态**不告警**，但前端提示用户"采集延迟"，避免误报（网络抖动常见）
- 判定在服务端读取时实时计算，`servers.is_active` 仍保留作为管理员手动停用入口

---

## 6. 完整数据模型

### 6.1 servers

```sql
id                   UUID PRIMARY KEY
name                 TEXT NOT NULL
token                TEXT UNIQUE NOT NULL       -- SHA-256
token_rotated_at     DATETIME                   -- 轮换时间，用于 grace period
token_prev_sha256    TEXT                       -- 上一个 token 的 SHA-256（grace 期内并行有效）
token_prev_expires_at DATETIME                  -- 旧 token 失效时间
description          TEXT
os                   TEXT
hostname             TEXT
ip                   TEXT                       -- 上报时自动记录
timezone             TEXT DEFAULT 'UTC'         -- Agent 首次上报时落库，可手动覆盖
created_at           DATETIME
last_seen            DATETIME
is_active            BOOLEAN DEFAULT TRUE

-- 流量（vnstat 方案，字段极简）
traffic_reset_day    INTEGER DEFAULT 1
  CHECK (traffic_reset_day BETWEEN 1 AND 28)
monitored_interfaces TEXT DEFAULT ''
traffic_limit_gb     INTEGER DEFAULT 0
  CHECK (traffic_limit_gb >= 0)
traffic_cycle_cache      JSON          -- 当前周期汇总缓存（见 §14.1.1），5min 刷新
traffic_cycle_cached_at  DATETIME      -- 缓存时间，> 5min 视为陈旧，API 实时算兜底
```

### 6.2 metrics

```sql
id               UUID PRIMARY KEY
server_id        UUID REFERENCES servers(id)
recorded_at      DATETIME NOT NULL
schema_version   INTEGER DEFAULT 1    -- 上报 schema 版本，便于后续字段演进
agent_version    TEXT                 -- Agent 版本号，运维追溯用
missed_count     INTEGER DEFAULT 0    -- 自上次成功上报以来，Agent 本地缓冲丢弃的条数
                                     -- 非零即代表"数据有缺失"

-- CPU
cpu_usage        FLOAT               -- 总使用率 %
cpu_freq         FLOAT               -- 当前频率 MHz
cpu_temp         FLOAT               -- °C，可 NULL
load_avg_1       FLOAT               -- 1 分钟负载
load_avg_5       FLOAT               -- 5 分钟负载
load_avg_15      FLOAT               -- 15 分钟负载

-- 内存
mem_total        BIGINT              -- bytes
mem_used         BIGINT
mem_percent      FLOAT
swap_total       BIGINT
swap_used        BIGINT
swap_percent     FLOAT

-- 磁盘整体概览（保留），明细拆到 server_disks
disks_summary    JSON   -- [{"mount":"/","total":x,"used":x,"percent":x}, ...]
                           -- 仅含容量类指标，不含 I/O 速率

-- 实时网速（psutil，不做周期统计）
net_speed_up     FLOAT               -- bytes/s 上行
net_speed_down   FLOAT               -- bytes/s 下行
net_interfaces   JSON   -- [{"iface":"eth0","speed_up":x,"speed_down":x}, ...]

-- 磁盘 I/O 聚合已下放到 server_disks：read_bs / write_bs / read_iops / write_iops
-- metrics 表不再保留磁盘 I/O 聚合列（避免双源数据不一致）
-- 服务端需要"全盘聚合"时直接从 server_disks GROUP BY recorded_at 算出来

-- 流量（vnstat，每日明细由 Agent 上报，服务端按重置日汇总）
vnstat           JSON   -- 见 §4.4 结构
vnstat_db_mtime  DATETIME             -- 上报时 vnstat.db 的 mtime，检测数据库异常
vnstat_db_size   BIGINT               -- 上报时 vnstat.db 的 size

-- 进程 & 连接（可选）
process_count    INTEGER
tcp_conn_count   INTEGER

-- GPU（可 NULL，多卡 JSON 数组）
gpus             JSON   -- [{"name":"RTX4090","temp":65,"usage":88,
                          --   "mem_used":x,"mem_total":x,"mem_percent":x,
                          --   "vendor":"nvidia"|"amd"}, ...]

-- Uptime
uptime_secs      BIGINT

INDEX (server_id, recorded_at)
INDEX (server_id, recorded_at, schema_version)  -- 用于 schema 演进时的回填
```

**关于 schema 演进的承诺**：
- 只**加字段**不删字段；删除用 `_deprecated` 后缀并保留 N 个版本
- Agent 上报 `schema_version`，服务端读取时按版本兼容解析
- 旧数据新字段为 NULL 是允许的（DB 默认值保证）

### 6.2.1 server_disks（磁盘分盘指标，拆自原 metrics.disks）

**唯一权威源**：磁盘容量 + 磁盘 I/O 速率/IOPS **全部**走这张表。`metrics` 表**不**再保留
`disk_read_bs` / `disk_write_bs` / `disk_read_iops` / `disk_write_iops` 等聚合列，避免双源数据
不一致。需要"全盘聚合"时由服务端 `SELECT ... FROM server_disks WHERE recorded_at = ? GROUP BY recorded_at` 算。

**为什么拆出来**：disks 原本在 metrics JSON 里，多服务器场景下"所有机器的 /var 分区使用率分布"
"哪台机器的 /var > 90%" 这类查询要全表扫 + JSON 解析，性能和易用性都差。

```sql
id              UUID PRIMARY KEY
server_id       UUID REFERENCES servers(id) ON DELETE CASCADE
recorded_at     DATETIME NOT NULL
mount           TEXT NOT NULL          -- 挂载点，如 "/"、"/var"
device          TEXT                   -- 设备名，如 "/dev/sda1"，便于识别
fstype          TEXT                   -- 文件系统类型，ext4/xfs/zfs/...
total_bytes     BIGINT
used_bytes      BIGINT
percent         FLOAT
read_bs         FLOAT                  -- bytes/s
write_bs        FLOAT
read_iops       INTEGER
write_iops      INTEGER

INDEX (server_id, mount, recorded_at)
INDEX (recorded_at)                     -- 全局清理任务用
```

- 与 `metrics` 行同一 `recorded_at`，由 Agent 在同一次采集周期内批量写入
- 服务端查询单盘趋势：`SELECT * FROM server_disks WHERE server_id=? AND mount='/var' AND recorded_at BETWEEN ? AND ?`
- 告警规则可精确到 mount：`alert_rules.metric_path = 'disks[mount=/var].percent'`
- 数据量：10s × 30 机 × 平均 2 盘 × 30 天 = 1,555 万行——在 SQLite/PG 都可承受，PG 上对 `(server_id, mount, recorded_at)` 建 BRIN 进一步压缩

**降采样策略**（见 §14）：`server_disks` 与 `metrics` 走相同的保留+降采样规则，1h/1d 聚合后写入 `server_disks_5m` / `server_disks_1h` 物化表。

### 6.3 traffic_cycles（历史周期存档，归零时写入）

```sql
id           UUID PRIMARY KEY
server_id    UUID REFERENCES servers(id) ON DELETE CASCADE
iface        TEXT NOT NULL DEFAULT ''   -- 网卡名；空字符串 = 监控网卡合并汇总；非空 = 单网卡
                                      -- 见 §14.1.1 缓存策略
cycle_start  DATE NOT NULL
cycle_end    DATE NOT NULL
total_rx     BIGINT              -- 本周期总接收字节
total_tx     BIGINT              -- 本周期总发送字节
total_bytes  BIGINT GENERATED ALWAYS AS (total_rx + total_tx) STORED
created_at   DATETIME

UNIQUE (server_id, iface, cycle_start)   -- 同一周期同一粒度只能归档一次
INDEX (server_id, cycle_start)            -- 合并行（iface=''）的列表查询
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
role         TEXT DEFAULT 'admin'   -- 'admin' | 'viewer'（Phase 4 启用多用户时再细分）
created_at   DATETIME
```

### 6.7 alert_rules

```sql
id            UUID PRIMARY KEY
server_id     UUID REFERENCES servers(id) ON DELETE CASCADE   -- NULL = 全局
monitor_id    UUID REFERENCES service_monitors(id) ON DELETE CASCADE  -- NULL = 非服务探测
metric        TEXT                            -- 见下方"支持告警的指标 Key"
metric_path   TEXT                            -- 可选，精确路径，如 'disks[mount=/var].percent'
threshold     FLOAT
operator      TEXT                            -- '>' | '<' | '>=' | '<=' | '==' | '!='
duration_secs INTEGER DEFAULT 0               -- 持续 N 秒才触发（0=立即触发），抑制抖动
cooldown_secs INTEGER DEFAULT 300             -- 同一规则触发后冷却 N 秒内不重复告警
notify_type   TEXT                            -- 'webhook' | 'telegram' | 'email'
notify_target TEXT                            -- webhook URL / telegram chat_id / email
is_active     BOOLEAN DEFAULT TRUE
created_at    DATETIME

CHECK (operator IN ('>', '<', '>=', '<=', '==', '!='))
CHECK (duration_secs >= 0 AND cooldown_secs >= 0)
```

### 6.8 alert_history（告警触发与恢复记录）

**为什么单独成表**：原方案只有规则没有记录，触发了什么、什么时候恢复、通知是否送达——全部没落库，
运维无法复盘"为什么没收到告警""昨天那次抖动持续了多久"。

```sql
id              UUID PRIMARY KEY
rule_id         UUID REFERENCES alert_rules(id) ON DELETE SET NULL
server_id       UUID REFERENCES servers(id) ON DELETE SET NULL  -- 冗余，便于按 server 过滤
monitor_id      UUID                                     -- 冗余同上
metric          TEXT                                     -- 冗余，告警触发的指标 key
metric_path     TEXT                                     -- 冗余
threshold       FLOAT                                    -- 冗余
operator        TEXT                                     -- 冗余
triggered_value FLOAT                                    -- 触发时的指标值

state           TEXT NOT NULL  -- 'firing' | 'resolved' | 'silenced' | 'notify_failed'
                                -- firing: 阈值被突破，等待 duration_secs 累计
                                -- resolved: 已恢复（指标回到阈值内）
                                -- silenced: 处于 cooldown，规则不重复通知
                                -- notify_failed: 通知投递失败（连续 3 次）

fired_at        DATETIME NOT NULL                  -- 首次进入 firing 的时间
resolved_at     DATETIME                           -- 进入 resolved 的时间（NULL=仍在 firing）
last_notified_at DATETIME                          -- 最近一次发出通知的时间
notify_attempts INTEGER DEFAULT 0                  -- 累计通知尝试次数
notify_last_error TEXT                             -- 最近一次通知失败的错误信息

INDEX (server_id, fired_at)
INDEX (rule_id, fired_at)
INDEX (state, fired_at)                            -- 用于扫描"仍处于 firing 的告警"
```

**告警状态机**：

```
                  threshold crossed
   ┌──────────────────────────────────┐
   │                                  ▼
[idle] ─────────► [firing] ───► [resolved] ──► [idle]
                   │  │             ▲
                   │  └─cooldown──►[silenced]──┘
                   │
                   └─notify 3次失败──►[notify_failed]
                                       (继续重试，状态保留)
```

- `firing` 状态时，每收到一条新指标就重新评估；若 `now - fired_at >= duration_secs` 才发出通知
- 通知发出后进入 `silenced`（cooldown），期间指标在阈值内/外都不重复通知
- 恢复条件：指标值回到 `threshold ± 5%` 范围内（5% 滞回带，避免阈值线抖动反复触发）
- `notify_failed` 状态保留在历史表中但**不再重试**，需管理员手动干预；避免无 Telegram Token 时刷屏

**支持告警的指标 Key：**

| Key | 说明 | 示例阈值 | 配套 `metric_path` |
|-----|------|---------|--------------------|
| `cpu_usage` | CPU 使用率 % | > 90 | — |
| `cpu_temp` | CPU 温度 °C | > 85 | — |
| `load_avg_5` | 5分钟负载 | > 核心数×0.8 | — |
| `mem_percent` | 内存使用率 % | > 85 | — |
| `swap_percent` | Swap 使用率 % | > 50 | — |
| `disk_percent` | 磁盘使用率 % | > 90 | `disks[mount=/var].percent` 精确到 mount |
| `gpu_temp` | GPU 温度 °C | > 80 | `gpus[name=RTX4090].temp` 精确到卡 |
| `gpu_usage` | GPU 使用率 % | > 95 | 同上 |
| `net_speed_up` | 实时上行 bytes/s | > 900MB/s | `net_interfaces[iface=eth0].speed_up` |
| `net_speed_down` | 实时下行 bytes/s | > 900MB/s | 同上 |
| `cycle_traffic_total_gb` | 本周期总流量 GB | > 1000 | — |
| `service_down` | 服务探测失败 | — | — |
| `ssl_expiry_days` | SSL 证书剩余天数 | < 30 | — |
| `server_offline` | 客户端离线（基于 `last_seen`） | 触发即告警 | — |
| `vnstat_unavailable` | 宿主机 vnstat 不可用 | 触发即告警 | — |
| `vnstat_db_anomaly` | vnstat.db mtime 倒退 / size 骤降 | 触发即告警 | — |
| `data_missed` | Agent 缓冲有丢弃（`missed_count > 0`） | > 0 | — |

无 `metric_path` 的指标作用于"该 server 任何 mount / 任何 GPU"中的最大值；
有 `metric_path` 的指标精确到具体 mount / 卡名 / 网卡。

---

## 7. API 设计

### 7.1 客户端上报

**注册 / 心跳合一**：

```
POST /api/v1/report
Header: X-Agent-Token: <token>
Header: X-Agent-Version: 1.2.3          # Agent 自报版本，用于运维追溯
Content-Encoding: gzip                   # 启用 gzip 压缩，单次 payload ~2-5KB → ~0.5-1KB
Body:
{
  "schema_version": 1,                   // 当前上报格式版本，向后兼容用
  "tz": "Asia/Shanghai",                 // IANA 时区名，Agent 启动时读 /etc/timezone
  "missed_count": 0,                     // 自上次成功上报以来本地缓冲丢弃的条数
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
  "disks_summary": [                     // 仅容量类指标，明细在 server_disks
    {"mount": "/", "total": 107374182400, "used": 53687091200, "percent": 50.0}
  ],
  "disks": [                             // 全量明细，写入 server_disks
    {
      "mount": "/", "device": "/dev/sda1", "fstype": "ext4",
      "total": 107374182400, "used": 53687091200, "percent": 50.0,
      "read_bytes_s": 1048576, "write_bytes_s": 524288,
      "read_iops": 120, "write_iops": 60
    }
  ],
  // 注意：磁盘 I/O 不再在 metrics 顶层重复上报；server_disks 是唯一权威源
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
  "vnstat_error": null,                  // vnstat 异常时填错误信息，vnstat 字段则为 null
  "vnstat_db_mtime": "2024-05-20T03:14:15Z",
  "vnstat_db_size": 32768,
  "process_count": 312,
  "tcp_conn_count": 128,
  "gpus": [
    {"name": "NVIDIA RTX 4090", "vendor": "nvidia",
     "temp": 65, "usage": 88,
     "mem_used": 8589934592, "mem_total": 25769803776, "mem_percent": 33.3}
  ],
  "uptime_secs": 864000,
  "os": "Ubuntu 22.04.3 LTS",
  "hostname": "web-server-01"
}

Response:
{
  "ok": true,
  "server_time": "2024-05-20T03:14:16Z",
  "next_report_in": 10,                  // 服务端可调间隔（如服务端限流时回 30）
  "config": {                            // 服务端可下发热更新配置
    "report_interval": 10,
    "monitored_interfaces": [],
    "skip_process_count": false
  }
}
```

- 首次上报时 `hostname` 跟数据库不一致时**不报错**（重装系统很常见），仅在 Dashboard 提示
- Token 校验：DB 查 `token` SHA-256；若匹配 `token_prev_sha256` 且 `now < token_prev_expires_at`，
  服务端自动把 `servers.token` 更新为新 token 并续约 grace period（让"轮换 token 的服务器"自然过渡）
- `missed_count > 0` 时 Dashboard 在该机器卡片上显示"⚠️ 数据缺失"，并触发 `data_missed` 告警（可关闭）
- 限流：每 Token 1 次/5s 硬限（防失控循环），超限返回 429 + `Retry-After`
- **不支持 register 独立接口**：注册 = 首次成功 report（带 token 即合法；token 决定身份，不依赖 hostname/IP）

### 7.2 流量 API

```
-- 本周期流量摘要（来自 traffic_cycle_cache，>5min 走实时算兜底）
GET /api/servers/{id}/traffic/current
Response:
{
  "tz": "Asia/Shanghai",            -- 客户端时区，决定"今天"边界
  "reset_day": 15,
  "limit_gb": 1000,
  "computed_at": "2024-05-20T03:14:15Z",   -- 缓存计算时间
  "rows": [                                -- 数组，按 monitored_interfaces 配置分网卡
    {
      "iface": "",                  -- 空字符串 = 监控全部的合并行
      "cycle_start": "2024-05-15",
      "cycle_end": "2024-06-14",
      "next_reset": "2024-06-15",
      "days_until_reset": 10,
      "total_rx_bytes": 2147483648,
      "total_tx_bytes": 536870912,
      "total_bytes": 2684354560,
      "percent_used": 2.5,
      "daily_breakdown": [...]      -- 每日明细，仅"监控全部"行带；单网卡按需带
    }
  ]
}

-- 历史周期列表（合并行 + 单网卡行混排，由 UI 决定展示）
GET /api/servers/{id}/traffic/history?limit=12&iface=eth0
  -- iface 可选；不传 = 仅合并行；传 = 仅该网卡行

-- 更新流量配置
PATCH /api/servers/{id}/traffic/config
Body: { "reset_day": 15, "monitored_interfaces": ["eth0"], "limit_gb": 1000 }
  -- reset_day 必须 1~28（DB CHECK 约束，>28 会被 400 拒绝）
  -- 切换 monitored_interfaces 后，下次 traffic_cycle_refresh 自动按新粒度缓存
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
# 认证
POST /api/auth/login                            # body: {username, password}
POST /api/auth/refresh                          # body: {refresh_token}
POST /api/auth/logout                           # 撤销 refresh_token
GET  /api/auth/me                               # 当前用户信息

# 服务器管理
GET    /api/servers
POST   /api/servers                             # 创建并返回 token（仅展示一次）
GET    /api/servers/{id}
PATCH  /api/servers/{id}                        # 改 name/desc/timezone/limit/reset_day/is_active
DELETE /api/servers/{id}
POST   /api/servers/{id}/rotate-token
  Body: { "grace_hours": 24 }                   # 默认 24h，0=立即失效
  Response: { "token": "new-...", "expires_at": "...",
              "prev_token_expires_at": "..." }  # 前一个 token 失效时间

# 指标查询
GET /api/servers/{id}/metrics
  ?from=&to=&resolution=raw|5m|1h|1d
  &fields=cpu_usage,load_avg_5,mem_percent,swap_percent,...
  分页：&limit=1000&cursor=<recorded_at>

# 单盘查询（拆表后走这个）
GET /api/servers/{id}/disks
  ?from=&to=&mount=/var&resolution=raw|5m|1h|1d

# 实时推送
WS /ws/metrics                                  # 见下方"WS 鉴权"小节

# 告警
GET    /api/alerts                              # 规则列表
POST   /api/alerts
PATCH  /api/alerts/{id}
DELETE /api/alerts/{id}
GET    /api/alerts/history                      # 历史触发记录（来自 alert_history 表）
  ?from=&to=&state=firing|resolved|silenced|notify_failed
  &server_id=&rule_id=

# vnstat 运维
POST /api/servers/{id}/rebuild-vnstat           # 下发命令给 Agent 执行 vnstat --rebuilddb
```

#### WS 鉴权方案

WebSocket 浏览器无法自定义 header。"**禁止 query string 传 token**" 原则针对的是
**长期凭证**（Access Token / API Key，TTL ≥ 数分钟）——这些一旦写进 Nginx access log
就会被持久化泄露。**短期签名 token**（TTL ≤ 30s，一次性）不在此列：即使被记录也早已过期。

WS 鉴权分两阶段实施：

##### Phase 2（基础版）— 短期签名 token + 第一帧挑战

1. **服务端签发**（用户登录后，前端拉一次）：
   ```
   GET /api/auth/ws-ticket
   Response:
   {
     "ticket": "<base64url payload>.<base64url HMAC-SHA256 sig>",
     "expires_in": 30
   }
   ```
   签发 payload 结构：
   ```json
   {
     "sub": "user_id_or_server_id",   // 区分是用户还是 Agent
     "kind": "user" | "agent",
     "exp": 1716192000,                // 30s 后过期
     "jti": "<uuid>",                  // 防重放，一次性
     "scope": "metrics.read"
   }
   ```
   签名：`HMAC-SHA256(payload, secret=SHORT_LIVED_TICKET_SECRET)`。
   `SHORT_LIVED_TICKET_SECRET` 是独立于 `SECRET_KEY` 的密钥（可从 `SECRET_KEY` 派生）。

2. **浏览器建连**：
   ```
   ws://server/ws/metrics?ticket=<ticket>
   - 服务端立即验证 ticket（验签 + exp + jti 未消费）
   - 通过后立即把 jti 写入一次性集合（5min 后清理），防重放
   - 通过后立即发首帧：{"type":"welcome","server_time":"..."}
   - 然后开始推指标
   - 验证失败：关闭连接（code 4401）
   ```
   30s 内未通过则关闭（code 4401）。

3. **错误码**：
   - 4401：未鉴权 / 签名错 / 过期
   - 4408：ticket 已被使用（jti 重放）
   - 1011：服务端异常

##### Phase 3（加固版）— 扩展首帧挑战

把 Phase 2 的"建连时一次性验证"扩展为"建连后还能重新挑战"：
- 服务端周期性（每 5min）发 `{"type":"reauth_required"}`
- 浏览器用同样的 ticket 端点拉新 ticket 后回 `{"type":"reauth","ticket":"..."}`
- 长期不活跃的连接（> 10min 无数据）被服务端主动关闭
- 旧 Phase 2 客户端的 `?ticket=` 形式保留作为 fallback，标记 `deprecated` 头

这样 Phase 2 的代码可以平滑接 Phase 3 的扩展，不需要重写 WS 层。

##### Agent → Server 反向 WS（未来扩展）

本设计未启用 Agent 走 WS 上报（Agent 仍用 HTTP POST），仅预留：
```
Sec-WebSocket-Protocol: serverpulse-agent-v1, <ticket>
```
第一段子协议是固定标识，第二段是 ticket。**Phase 2 不实现**。

#### 轮换 Token 时的 grace period

`POST /api/servers/{id}/rotate-token` 流程：
1. 生成新 token，返回给管理员
2. 把旧 token 的 SHA-256 写入 `token_prev_sha256`，设 `token_prev_expires_at = now + grace_hours`
3. Agent 用旧 token 上报时，服务端自动把 `servers.token` 升为旧 token 的 SHA-256
   （即"旧 token 首次成功上报后立即升级"，但旧 token 在 `prev_expires_at` 前仍可重试）
4. 管理员可在 Server 端"撤销 grace"按钮：立即清空 `token_prev_*`，旧 token 立刻失效
5. `grace_hours=0` 表示"立即失效"，但**不推荐**——会强制管理员在 Agent 重启前完成轮换

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
report_interval: 10           # 秒，服务端可在响应中热更新

network_interfaces: []        # 留空 = 全部非 lo；指定 ["eth0"]
disk_mounts: []               # 留空 = 自动扫描
gpu_enabled: true
gpu_vendor: "auto"            # 'auto' | 'nvidia' | 'amd' | 'none'
skip_process_count: false     # 进程数采集（高负载机器可关）
skip_tcp_connections: false   # TCP 连接数采集
buffer_max: 200               # 离线时本地缓冲条数（默认 200 ≈ 33 分钟 @ 10s）
                              # 超出按 LRU 丢弃，并累加 missed_count
log_level: INFO
```

### 9.2 采集模块

```
agent/collector/
  cpu.py        psutil.cpu_percent / cpu_freq / sensors_temperatures / getloadavg
  memory.py     psutil.virtual_memory + swap_memory
  disk.py       psutil.disk_partitions + disk_usage + disk_io_counters（差值算速率）
  network.py    psutil.net_io_counters（仅算实时速率）
  vnstat.py     subprocess vnstat --json（流量周期统计）★ 已升级
  gpu.py        pynvml（NVIDIA）/ rocm-smi（AMD），无 GPU 时返回 None ★ 已升级
  process.py    psutil.pids + net_connections（可选）
  uptime.py     psutil.boot_time
  system.py     tz / os / hostname / agent_version  ★ 新增
```

#### gpu.py：双厂商支持

```python
def collect_gpus() -> list | None:
    if not gpu_enabled:
        return None
    vendor = detect_vendor()    # 优先 NVIDIA（pynvml），回退 AMD（rocm-smi）
    if vendor == "nvidia":
        return collect_nvidia_pynvml()
    elif vendor == "amd":
        return collect_amd_rocm_smi()    # 解析 `rocm-smi --json` 输出
    return None    # 无 GPU 优雅跳过
```

`rocm-smi --json` 在 ROCm 5.4+ 可用；Agent 仅在检测到 `/dev/dri/renderD128` 时尝试拉起。

### 9.3 vnstat.py 降级 + 完整性检测

```python
import os
from datetime import datetime, timezone

def collect_vnstat(ifaces=None):
    try:
        result = subprocess.run(
            ["vnstat", "--json", "2"],
            capture_output=True, text=True, timeout=5
        )
        if result.returncode != 0:
            return None, f"vnstat exit {result.returncode}: {result.stderr.strip()}"
        return parse_vnstat(json.loads(result.stdout), ifaces), None
    except FileNotFoundError:
        return None, "vnstat binary not found"
    except subprocess.TimeoutExpired:
        return None, "vnstat timeout (>5s)"
    except json.JSONDecodeError as e:
        return None, f"vnstat json parse error: {e}"

def collect_vnstat_db_meta():
    """检测数据库完整性，字段写入 metrics.vnstat_db_mtime / vnstat_db_size"""
    path = "/var/lib/vnstat/vnstat.db"
    try:
        st = os.stat(path)
        return {
            "mtime": datetime.fromtimestamp(st.st_mtime, tz=timezone.utc).isoformat(),
            "size":  st.st_size,
        }
    except FileNotFoundError:
        return None
```

### 9.4 system.py：时区获取

```python
def get_timezone() -> str:
    """
    优先读 /etc/timezone（Debian/Ubuntu），回退到 /etc/localtime symlink 解析，
    再回退到 UTC。返回 IANA 名（如 'Asia/Shanghai'），不要返回 'CST' 这种缩写。
    """
    try:
        with open("/etc/timezone") as f:
            return f.read().strip()
    except FileNotFoundError:
        pass
    # 回退：遍历 zoneinfo 目录找匹配
    if os.path.islink("/etc/localtime"):
        target = os.readlink("/etc/localtime")
        if "zoneinfo/" in target:
            return target.split("zoneinfo/", 1)[1]
    return "UTC"
```

### 9.5 上报流程

```
1. 启动：
   a. 读 system.py → 缓存 tz / os / hostname / agent_version
   b. 启动 vnstat 健康检查（mtime 5s 后再次读取，对比 mtime 是否在增长）
      - 若 5s 内 mtime 没变化，警告"vnstatd 可能未在运行"
2. 每 report_interval 秒：
   a. 线程池并行采集（asyncio.to_thread）
      - psutil 调用是同步阻塞，必须放线程池，否则会卡住事件循环
      - 顺序：CPU → 内存 → 磁盘 → 网络 → GPU → 进程 → vnstat
   b. 组装 payload，gzip 压缩
   c. POST /api/v1/report
   d. 成功：
      - 应用响应中的 config（report_interval 等）热更新
      - 清空 missed_count
   e. 失败（网络 / 5xx）：
      - 把当前 payload 推入 deque（maxlen=buffer_max）
      - 递增 missed_count
      - 下个周期先尝试补报 deque，再采集新数据
      - 当 missed_count 触发丢弃时（deque 满），记入日志 + 上报一次特殊指标
3. 优雅退出（SIGTERM）：
   - flush deque 一次（5s 超时），失败则丢弃并记日志
```

---

## 10. Docker 部署方案

### 10.1 服务端 docker-compose.yml（PostgreSQL 优先 + SQLite 玩具模式）

提供两套 profile：**`pg`**（生产/中等规模）+ **`sqlite`**（单机玩具 / 临时试用）。

```yaml
# docker-compose.server.yml
version: "3.9"
x-server-env: &server-env
  # PROFILE 区分 dev/prod（见 §11.2 校验规则）
  PROFILE: ${PROFILE:-prod}
  SECRET_KEY: ${SECRET_KEY:?SECRET_KEY must be set}
  ADMIN_USERNAME: ${ADMIN_USERNAME:-admin}
  ADMIN_PASSWORD: ${ADMIN_PASSWORD:?ADMIN_PASSWORD must be set}
  DATA_RETENTION_DAYS: ${DATA_RETENTION_DAYS:-30}
  DATA_DOWNSAMPLE_5M_DAYS: ${DATA_DOWNSAMPLE_5M_DAYS:-7}
  DATA_DOWNSAMPLE_1H_DAYS: ${DATA_DOWNSAMPLE_1H_DAYS:-30}
  # 通知渠道（任选）
  TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN:-}
  SMTP_HOST: ${SMTP_HOST:-}
  SMTP_PORT: ${SMTP_PORT:-587}
  SMTP_USER: ${SMTP_USER:-}
  SMTP_PASSWORD: ${SMTP_PASSWORD:-}
  SMTP_FROM: ${SMTP_FROM:-}

services:
  postgres:
    image: postgres:16-alpine
    profiles: ["pg"]
    environment:
      POSTGRES_USER: pulse
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set in .env}
      POSTGRES_DB: pulse
    volumes:
      - ./data/pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pulse"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  server:
    image: serverpulse-server:latest
    build: ./server
    ports:
      - "8080:8080"
    volumes:
      - ./data:/app/data    # SQLite 模式或备份目录
    environment:
      <<: *server-env
      # 关键：DB_URL 走 profile 注入
      DB_URL: ${DB_URL:?set to sqlite:///./data/pulse.db (sqlite) or postgresql+asyncpg://pulse:...@postgres/pulse (pg)}
      CORS_ORIGINS: ${CORS_ORIGINS:-}
      LOG_LEVEL: ${LOG_LEVEL:-INFO}
    depends_on:
      postgres:
        condition: service_healthy
        required: false   # sqlite 模式下 postgres 不存在也能启动
    restart: unless-stopped

  # 备份：每天 03:00 把 /app/data 备份到 ./backups/
  # 简易实现，后续可换 restic / borg
  backup:
    image: alpine:3.19
    profiles: ["backup"]
    volumes:
      - ./data:/data:ro
      - ./backups:/backups
    entrypoint: /bin/sh
    command:
      - -c
      - |
        apk add --no-cache sqlite-cli postgresql-client
        while true; do
          ts=$$(date +%Y%m%d-%H%M%S)
          if [ -f /data/pulse.db ]; then
            sqlite3 /data/pulse.db ".backup '/backups/pulse-$$ts.db'"
          fi
          # PG 模式走 pg_dump
          if command -v pg_dump >/dev/null && [ -n "$$PGHOST" ]; then
            PGPASSWORD=$$PGPASSWORD pg_dump -h $$PGHOST -U $$PGUSER $$PGDB \
              > /backups/pulse-pg-$$ts.sql
          fi
          sleep 86400
        done
    environment:
      PGHOST: postgres
      PGUSER: pulse
      PGPASSWORD: ${POSTGRES_PASSWORD:-}
      PGDB: pulse
    restart: unless-stopped
```

**SQLite 模式启动**（单文件数据库，< 5 台机器 / < 7 天数据）：

```bash
echo 'DB_URL=sqlite:///./data/pulse.db' > .env
docker compose --profile sqlite -f docker-compose.server.yml up -d
```

**PostgreSQL 模式启动**（推荐，超过 5 台机器或要长期保留）：

```bash
echo 'DB_URL=postgresql+asyncpg://pulse:STRONG_PW@postgres:5432/pulse' >> .env
echo 'POSTGRES_PASSWORD=STRONG_PW' >> .env
docker compose --profile pg --profile backup -f docker-compose.server.yml up -d
```

### 10.2 客户端 docker-compose.yml

**降权方案**：见 §4.6，此处给出可复制的最终版。

```yaml
# docker-compose.agent.yml
version: "3.9"
services:
  agent:
    image: serverpulse-agent:latest
    build: ./agent
    # 不再 privileged: true
    cap_drop: [ALL]
    cap_add: [SYS_PTRACE]      # /host/proc 内的 cmdline 读取
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp:size=64m
      - /run:size=8m
    network_mode: host
    pid: host                  # 进程数采集用；不采集可去掉 → 进一步降权
    volumes:
      - /var/lib/vnstat:/var/lib/vnstat:ro
      - /usr/bin/vnstat:/usr/bin/vnstat:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      - SERVER_URL=https://pulse.example.com
      - AGENT_TOKEN=your-token-here
      - REPORT_INTERVAL=10
      - GPU_ENABLED=true
      - GPU_VENDOR=auto
      - BUFFER_MAX=200
      - LOG_LEVEL=INFO
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "python", "-c", "import os; assert os.path.exists('/var/lib/vnstat/vnstat.db')"]
      interval: 60s
      timeout: 5s
      retries: 3
      start_period: 30s
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
| Agent Token | 随机 32 字节 hex（`secrets.token_hex(32)`），SHA-256 存库，明文仅展示一次；轮换有 24h grace period（§7.4） |
| 管理员登录 | bcrypt（cost=12）哈希；JWT Access 15min + Refresh 7d；登录限流 5/min/IP（防爆破） |
| JWT 密钥 | `SECRET_KEY` 环境变量注入；启动时校验长度 ≥ 32，否则**拒绝启动** |
| JWT 轮换 | `SECRET_KEY` 变更时，旧 refresh token 立刻失效；access token 仍可用至过期（最多 15min） |
| HTTPS | 强制：所有非 /health 的请求必须走 TLS；HTTP 请求 301 → HTTPS（HSTS 头 1 年） |
| Agent 上报限流 | 每 Token 1次/5秒硬限；超限返回 429 + `Retry-After` |
| 登录限流 | 5 次/分钟/IP，10 次/小时/用户名；超限返回 429 |
| WS 鉴权 | 首条消息挑战模式（§7.4），30s 内未通过关闭 4401；**禁止** token 放 query string |
| CORS | 默认拒绝跨域；如需 Web 分离部署，显式 `CORS_ORIGINS` 白名单 |
| 容器安全 | Agent 已降权（§4.6）：`read_only: true` + 最小 cap + `no-new-privileges` + `/host/*` 只读 |
| vnstat 挂载 | 只读 `:ro`，容器无法篡改宿主机数据 |
| 输入校验 | Pydantic V2 严格校验所有上报字段（`extra='forbid'`） |
| 告警通知 | Telegram Token / SMTP 凭证存服务端环境变量，不落库；通知失败有重试（§14） |
| 数据库 | SQLite 文件权限 0600；PG 用户仅 `pulse` 库 `SELECT/INSERT/UPDATE/DELETE`，无 DDL |
| 审计日志 | 管理员操作（创建/删除 server、轮换 token、改告警规则）写 `audit_log` 表，含 IP/UA/时间 |
| 依赖扫描 | CI 跑 `pip-audit` / `npm audit`，高危 CVE 阻断发布 |

### 11.1 审计日志表

```sql
CREATE TABLE audit_log (
  id          UUID PRIMARY KEY,
  user_id     UUID REFERENCES users(id),
  action      TEXT NOT NULL,    -- 'server.create' | 'token.rotate' | 'alert.update' | ...
  target_id   UUID,             -- 对象 ID
  target_type TEXT,             -- 'server' | 'alert' | 'user' | ...
  ip          TEXT,
  user_agent  TEXT,
  detail      JSON,             -- 改动前后的 diff
  created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX audit_log_created_at_idx ON audit_log (created_at);
CREATE INDEX audit_log_user_id_idx ON audit_log (user_id);
```

保留 90 天后由 `data_retention` 任务清理。

### 11.2 启动时安全校验

服务端启动时若以下条件不满足，**直接退出**（而非警告）：

引入 `PROFILE` 环境变量区分部署场景（`dev` / `prod`），校验规则按 profile 走：

```bash
PROFILE=dev     # 本地开发，宽松（SQLite / 弱密码也允许，但打 WARNING）
PROFILE=prod    # 生产环境，严格（任一不满足即退出）
# 不设置 = 默认 prod（fail-safe：宁严勿宽）
```

| 校验项 | dev | prod |
|--------|-----|------|
| `SECRET_KEY` 长度 ≥ 32 且不在已知弱密钥列表 | WARNING | **拒绝启动** |
| `ADMIN_PASSWORD` 长度 ≥ 12 且非默认值 | WARNING | **拒绝启动** |
| `DB_URL` 可连接（启动时 `SELECT 1`） | 必须 | 必须 |
| `DB_URL` 必须是 `postgresql+asyncpg://` 开头 | WARNING | **拒绝启动**（不允许 SQLite） |
| `CORS_ORIGINS = '*'` 且鉴权用 Cookie | WARNING | **拒绝启动**（CSRF 风险） |
| `DEBUG=true` | 允许 | **拒绝启动** |
| `/docs` `/redoc` OpenAPI 文档 | 允许 | **拒绝启动**（prod 默认关闭） |

dev 与 prod 共用规则（如 DB 可达、Token 格式）始终执行。dev 模式仅放宽上述"硬性安全策略"，方便本地试用。

> **`SELECT 1` 与 `DB_URL` 格式校验不重复**：前者是**运行时网络可达**（DNS 解析、TCP 握手、认证、DB 存在），
> 后者是**配置格式**（scheme 头、驱动前缀）。Docker Compose 服务启动顺序不保证，
> "自然暴露"意味着第一个真实请求才会崩——沉默失败几秒再崩，比启动时直接退出难排查。
> 两条规则分别覆盖两个独立的失败面，缺一不可。

---

## 12. 开发阶段划分

### Phase 1 — MVP（约 3 周）
- [ ] 服务端：FastAPI + Alembic + 上报接口 + JWT 登录（首启安全校验 §11.2）
- [ ] 服务端：PostgreSQL 默认（profile=pg）/ SQLite 玩具模式（profile=sqlite）
- [ ] 服务端：数据模型（§6，含 `server_disks` 拆表 + `alert_history`）
- [ ] 客户端：CPU（含负载）/ 内存+Swap / 磁盘+I/O（含分盘）/ 实时网速 / Uptime / 时区
- [ ] Agent：`asyncio.to_thread` 并行采集 + 本地缓冲 + `missed_count` + gzip
- [ ] vnstat 集成：Agent 采集（含 DB 完整性检测）+ 服务端按重置日汇总（含时区）
- [ ] Web UI：登录 + 总览 + 单机详情（基础图表 + 流量卡片 + 数据缺失徽章）
- [ ] Docker Compose：`--profile pg` 和 `--profile sqlite` 两套
- [ ] Agent 容器降权方案（§4.6）已落地
- [ ] Alembic 迁移脚本就绪
- [ ] `data_retention` 定时清理任务

### Phase 2 — 服务监控 + GPU + WebSocket 基础（约 2 周）
- [ ] **WebSocket 实时推送（基础版）**：用简单 query string 鉴权（Phase 1 已用 token，此处复用 Agent token 思路但走 Cookie/Header 转换），单进程 broadcast，先把"实时数据能 push"这件事做掉
- [ ] 服务端探测引擎：HTTP / TCP / Ping
- [ ] SSL 证书到期检查
- [ ] 服务监控页（可用率时间轴 + 延迟图）
- [ ] GPU 支持：**NVIDIA（pynvml）+ AMD（rocm-smi）** 一起做（pynvml 写好再扩展 AMD）
- [ ] 告警引擎：`alert_history` 状态机 + duration_secs + cooldown + 滞回带
- [ ] 告警通知：Telegram Bot + Webhook（带重试，§14）

> **关于 WS 提前的理由**：Phase 1 全部用 10s 轮询，CPU 使用率/网速这类"瞬时高敏感"指标用户体验差。Phase 2 基础版可只鉴权用户（让 login 后的 Cookie 直接升级为 WS token），不引入"首条消息挑战"——后者（§7.4）放到 Phase 3。

### Phase 3 — 完善（约 1.5 周）
- [ ] WebSocket 鉴权加固：首条消息挑战模式（§7.4），替换 Phase 2 的简易方案
- [ ] 数据降采样：5m 聚合、1h 聚合（§14）
- [ ] 告警历史 UI（筛选 / 静默 / 解决标记）
- [ ] 管理后台：流量配置 + 告警规则 UI + Token 轮换 grace
- [ ] 进程数 / TCP 连接数（可选采集，配置开关）
- [ ] IP 归属地（MaxMind GeoLite2 自下载）

### Phase 4 — 增强（按需）
- [ ] 服务器分组 / 标签
- [ ] 公开状态页 `/status`（无需登录 + 自定义 SLA 通知）
- [ ] 多用户 / 只读权限（拆分 `users.role`，RBAC 最小化）
- [ ] 移动端响应式
- [ ] Docker Hub / GHCR 镜像发布
- [ ] 国际化（i18n）
- [ ] 单盘 / 单 GPU / 单网卡独立告警规则 UI

---

## 13. 快速上手

```bash
# 0. 宿主机（被监控机器）安装 vnstat
apt install vnstat && systemctl enable --now vnstatd
# 等待约 5 分钟初始化，验证：
vnstat -i eth0

# 1. 启动服务端（PostgreSQL 模式，推荐）
cp .env.example .env
# 编辑 .env，**必填**：
#   SECRET_KEY=$(openssl rand -hex 32)
#   ADMIN_PASSWORD=<强密码，≥12 字符>
#   POSTGRES_PASSWORD=<DB 强密码>
#   DB_URL=postgresql+asyncpg://pulse:POSTGRES_PASSWORD@postgres:5432/pulse
docker compose --profile pg --profile backup \
  -f docker-compose.server.yml up -d

# 或者 SQLite 玩具模式（单文件，仅 <5 台 / <7 天）
#   DB_URL=sqlite:///./data/pulse.db
# docker compose --profile sqlite -f docker-compose.server.yml up -d

# 2. 打开 https://your-server:8080（Nginx/Caddy 反代见部署文档）
#    管理后台 → 新建服务器 → 复制 Token
#    设置流量重置日（仅 1~28 合法，见 §4.5）

# 3. 被监控机器启动 Agent（容器已降权，见 §4.6）
SERVER_URL=https://your-server:8080 \
AGENT_TOKEN=paste-token-here \
docker compose -f docker-compose.agent.yml up -d

# 4. 刷新 Dashboard，即可看到实时指标、流量统计、服务探测
```

**首次启动后必做**：
- 修改默认 admin 密码
- 配置 HTTPS（Nginx/Caddy 反代 + Let's Encrypt）
- 设置 `data_retention` 任务为每天 03:00 运行（默认已开）
- 检查 `pg_dump` / `sqlite3 .backup` 备份是否成功写入 `./backups/`

---

## 14. 运营设计

把"项目跑起来后日常会遇到的事"集中放在这一节：数据保留、降采样、通知重试、schema 迁移、灾难恢复、容量规划。

### 14.1 数据保留与降采样

**问题**：默认 10s 上报 → 单机 8,640 行/天，30 台 × 30 天 = **777 万行 metrics** + 同样量级的 `server_disks`。
SQLite/PG 都可承受，但 UI 查询超过 90 天会很慢；告警历史的回溯也不该穿透 7 年细粒度数据。

**三层保留**（由 `data_retention` APScheduler 任务执行，每天 03:30 跑）：

| 表 | 原始（raw）| 5min 聚合 | 1h 聚合 | 永久归档 |
|----|------------|-----------|---------|----------|
| `metrics` | 7 天 | 30 天 | 1 年 | `metrics_archive`（可选，PG `pg_partman`） |
| `server_disks` | 7 天 | 30 天 | 1 年 | 同上 |
| `service_results` | 30 天 | 1 年 | — | — |
| `alert_history` | 1 年 | — | — | — |
| `audit_log` | 90 天 | — | — | — |
| `traffic_cycles` | 永久 | — | — | 永久 |

**5min 聚合 SQL（示意）**：

```sql
INSERT INTO metrics_5m (server_id, recorded_at, cpu_usage, mem_percent, ...)
SELECT
  server_id,
  to_timestamp(floor(extract(epoch from recorded_at) / 300) * 300) AT TIME ZONE 'UTC' AS bucket,
  avg(cpu_usage),
  avg(mem_percent),
  ...
FROM metrics
WHERE recorded_at < now() - interval '7 days'
  AND recorded_at >= now() - interval '30 days'
GROUP BY server_id, bucket
ON CONFLICT (server_id, bucket) DO UPDATE
  SET cpu_usage = EXCLUDED.cpu_usage, ...;
```

- SQLite 模式用 `strftime('%s', recorded_at) / 300` 算 bucket
- 聚合后**删除**已聚合的 raw 行（节省空间）
- API 查询 `resolution=5m|1h|1d` 时由应用层根据 `from/to` 选对应表
- `metrics_5m` / `metrics_1h` 表结构与 `metrics` 相同，仅保留聚合列

#### 14.1.1 vnstat 周期汇总的预计算缓存

**问题**：`/api/servers/{id}/traffic/current` 每次请求都要从 `metrics.vnstat.daily[]` 数组里
遍历加总，假设一个 server 一年 vnstat 日数据 = 365 条，每条遍历是 O(n)，
多用户并发访问 Dashboard 时这个路径会重复算。

**解决方案**：每 5 分钟（与 Agent 上报同频）跑一次 `traffic_cycle_refresh` 任务，
把每台 server 的"当前周期汇总"算好缓存到 `servers` 表的冗余字段：

```sql
-- servers 表新增（不破坏原 schema）
traffic_cycle_cache  JSON   -- 缓存 calc_cycle_usage 的完整返回
traffic_cycle_cached_at DATETIME
```

任务流程：

```python
@scheduler.scheduled_job("interval", minutes=5, id="traffic_cycle_refresh")
async def refresh_traffic_cycles():
    for server in db.query(Server).filter(is_active=True).all():
        # 取最近一次上报的 metrics.vnstat
        latest = db.query(Metrics).filter(
            server_id=server.id
        ).order_by(Metrics.recorded_at.desc()).first()

        if not latest or not latest.vnstat:
            continue

        # 服务端按 reset_day + tz 算当前周期
        cache = calc_cycle_usage(
            daily_data=merge_vnstat_daily(latest.vnstat),  # 多网卡合并或拆分，看配置
            reset_day=server.traffic_reset_day,
            tz_name=server.timezone or "UTC",
        )
        server.traffic_cycle_cache = cache
        server.traffic_cycle_cached_at = datetime.utcnow()
    db.commit()
```

API 层逻辑：

```
GET /api/servers/{id}/traffic/current
1. 读 servers.traffic_cycle_cache
2. 若 cached_at 距 now < 5min → 直接返回
3. 否则实时算（保证不返回 > 5min 陈旧数据，兼做降级）
4. 实时算后回填 cache（懒刷新）
```

`merge_vnstat_daily` 行为：若 `monitored_interfaces` 为空则合并所有网卡（求和），
否则仅合并 `monitored_interfaces` 里列出的网卡。Phase 1 简单实现：直接对所有网卡求和。

##### 按网卡粒度缓存

`traffic_cycle_cache` 缓存粒度由 `monitored_interfaces` 决定，**按需拆分，避免冗余**：

| `monitored_interfaces` 配置 | 缓存条目数 | `iface` 字段 | 说明 |
|------------------------------|-----------|--------------|------|
| 空（监控全部） | 1 | `""` | 所有非 lo 网卡流量求和 |
| `['eth0']` | 1 | `'eth0'` | 仅 eth0 |
| `['eth0', 'eth1']` | 2 | `'eth0'`, `'eth1'` | 独立缓存两个网卡 |

`traffic_cycle_cache` JSON 结构：

```json
{
  "rows": [
    {"iface": "",      "total_rx": ..., "total_tx": ..., "percent_used": ...},
    {"iface": "eth0",  "total_rx": ..., "total_tx": ..., "percent_used": ...}
  ],
  "computed_at": "2024-05-20T03:14:15Z"
}
```

任务伪代码：

```python
@scheduler.scheduled_job("interval", minutes=5, id="traffic_cycle_refresh")
async def refresh_traffic_cycles():
    for server in db.query(Server).filter(is_active=True).all():
        latest = db.query(Metrics).filter(server_id=server.id) \
                   .order_by(Metrics.recorded_at.desc()).first()
        if not latest or not latest.vnstat:
            continue

        ifaces = parse_ifaces(server.monitored_interfaces)  # [] or ['eth0', ...]
        rows = []
        for iface in (ifaces or [""]):    # 配了具体网卡按网卡；空列表 = 合并
            daily = pick_daily(latest.vnstat, iface)        # "" 合并所有，'eth0' 取 eth0
            cycle = calc_cycle_usage(daily, server.traffic_reset_day, server.timezone)
            rows.append({
                "iface": iface,
                "total_rx": cycle["total_rx"],
                "total_tx": cycle["total_tx"],
                "percent_used": calc_percent(cycle, server.traffic_limit_gb),
                "days_until_reset": cycle["days_until_reset"],
                "next_reset": cycle["next_reset"],
            })
        server.traffic_cycle_cache = {
            "rows": rows,
            "computed_at": datetime.utcnow().isoformat(),
        }
        server.traffic_cycle_cached_at = datetime.utcnow()
    db.commit()
```

`traffic_cycles` 表（§6.3）保持"归零时归档"语义不变，归档时按 `iface` 列分别写入：
- 配了具体网卡 → 每网卡一行
- 监控全部 → `iface=''` 合并行

UI 层根据 `monitored_interfaces` 配置渲染：
- 配置了具体网卡 → 显示**单网卡卡片**（或汇总表）
- 监控全部 → 显示**合并卡片**（单卡图标 + 汇总数字）

`traffic_cycles` 表（§6.3）保持"归零时归档"语义不变，**预计算 cache 不影响归档**。
缓存可视为"当前周期的实时视图"，归档表是"历史周期的累计快照"，两者职责清晰。

**降采样任务失败处理**：
- 失败时**不删除** raw，重试 3 次
- 仍失败 → 写 `data_retention_errors` 表，UI 提示"数据保留任务异常"
- 绝不因清理任务失败导致数据丢失

### 14.2 通知重试

**问题**：Telegram Bot API / Webhook / SMTP 都可能瞬时不可用（网络抖动、限流、SMTP 鉴权失败）。

**重试策略**（在 `alert_history` 状态机里）：

| 失败类型 | 重试次数 | 退避 | 终态 |
|---------|---------|------|------|
| 网络超时 / 连接拒绝 | 5 | 指数 1s/2s/4s/8s/16s | `notify_failed` |
| HTTP 4xx（除 429）| 0 | — | `notify_failed`（错误信息落库） |
| HTTP 429（限流）| 3 | 读 `Retry-After` 头 | `notify_failed` |
| HTTP 5xx | 5 | 同上 | `notify_failed` |

**去重**：同一 `alert_history.id` 在重试期间不重发新通知，仅重试失败的这一次。
**冷却**：`cooldown_secs` 是"成功通知后多久内不重复"，跟重试无关。
**死信**：连续 3 条 `notify_failed` → 触发 `notify_dead_letter` 系统告警（管理员邮箱兜底）。

### 14.3 Schema 迁移

- 任何表结构变更必须新增 Alembic 迁移脚本，**不可手动 `ALTER TABLE`**
- 迁移脚本必须：
  - **向后兼容**（旧 Agent 仍能上报，旧数据能读）
  - 大表加列用 `ALTER TABLE ... ADD COLUMN ... DEFAULT NULL`（PG 11+ 不锁表）
  - 删除列前先 rename 为 `_deprecated_*`，**保留 ≥ 2 个版本**再物理 DROP
- 升级流程：`docker compose pull && docker compose up -d` → 容器内自动跑 `alembic upgrade head`
- 回滚：保留上一版本镜像，`alembic downgrade -1` 显式执行

### 14.4 灾难恢复

| 场景 | 恢复方式 | RPO | RTO |
|------|---------|-----|-----|
| 数据库文件损坏（SQLite） | 从 `./backups/pulse-YYYYMMDD-HHMMSS.db` 恢复 | 24h | 10min |
| PG 数据误删 | 从 `./backups/pulse-pg-*.sql` 恢复 | 24h | 30min |
| 服务端整机丢失 | 重建容器 + 恢复 backup | 24h | 1h |
| Agent 失联 | 重新生成 Token + 启动 Agent | 0 | 5min |
| `SECRET_KEY` 泄露 | 改 env + 重启；旧 JWT 自然过期 | 0 | 5min（access 15min 内全部失效） |
| vnstat.db 损坏 | 容器内 `vnstat --rebuilddb`（管理员手动确认） | 0 | 10min |
| 误删 `servers` 行 | 备份恢复（**注意**：同时恢复 metric 关联数据） | 24h | 30min |

**备份校验**（每日 04:00 跑）：
- SQLite：`.backup` 后 `PRAGMA integrity_check` 必须返回 `ok`
- PG：`pg_dump` 后 `pg_restore --list` 校验归档可解析
- 失败 → 触发 `backup_failed` 告警

### 14.5 容量规划

| 规模 | servers | DB 推荐 | 磁盘 | 内存 | 网络 |
|------|---------|---------|------|------|------|
| 玩具 | 1-3 | SQLite | 1 GB/月 | 256 MB | <1 Mbps |
| 小型 | 3-15 | SQLite / PG | 5 GB/月 | 512 MB | <5 Mbps |
| 中型 | 15-50 | PG（必）| 30 GB/月 | 1 GB | <20 Mbps |
| 大型 | 50-200 | PG + pg_partman 分区 | 200 GB/月 | 2 GB | <100 Mbps |

计算依据：raw 7 天 + 5m 30 天 + 1h 365 天 ≈ 1.2 GB/服务器/年（PG 压缩后），中型规模按 30 台 × 1.2 GB ≈ 36 GB/年，5 年 200 GB。

**SQLite 极限**：单库 < 50 GB、单表 < 1 亿行、并发写 < 100/s。超过任意一条切 PG。

### 14.6 监控的监控

谁来监控 ServerPulse 自身？

- **外部探针**：用户用 ServerPulse 自己监控一个最小目标（如 `/health`），构成自监控
- **健康检查端点**：
  - `GET /health` → 200 OK（liveness，只检查进程）
  - `GET /ready` → 200/503（readiness，检查 DB 可达 + 上次清理任务成功）
- **/health 不限流、不鉴权**，用于 LB / K8s readinessProbe
- **关键告警**：`backup_failed` / `data_retention_errors` / `notify_dead_letter` 这三个系统级告警**必须**配置外部通知渠道（用户邮箱），否则 ServerPulse 自身挂了没人知道

### 14.7 公网部署注意

- 服务端**不直接暴露 8080** 到公网，必须经 Nginx/Caddy + HTTPS
- 反代配置需保留：
  - `client_max_body_size 10m`（上报 payload 上限）
  - WebSocket 升级头（`Upgrade` / `Connection`）
  - `/docs` / `/redoc` 仅在 `DEBUG=true` 时开放
- Agent → Server 走 HTTPS 时，证书用 Let's Encrypt，**不要**自签（Agent 默认不信任）
- 若服务端有 IP 白名单需求（Nginx 层 `allow` / `deny`），Agent 公网 IP 需固定或经中转

---

*ServerPulse — 轻量、自托管、无外部依赖的服务器监控探针。*

---

## 附录 A — 修订记录

| 版本 | 日期 | 主要变更 |
|------|------|---------|
| v0.1 | 2024-05 | 初稿，vnstat + 单体 SQLite |
| v0.2 | 2026-06-01 | 全面优化：容器降权 / 时区 / reset_day 约束 / server_disks 拆表 / alert_history 状态机 / WS 挑战鉴权 / Token grace / PostgreSQL 优先 / 通知重试 / 降采样策略 / 灾难恢复 / 容量规划 |
| v0.2.1 | 2026-06-01 | 修 calc_cycle_usage next_reset bug（reset_day>1 时错算到下下个月）；移除 metrics 顶层磁盘 I/O 聚合列（避免与 server_disks 双源）；§11.2 引入 PROFILE 变量；WS 基础版提前到 Phase 2；新增 §14.1.1 vnstat 预计算缓存 |
| v0.2.2 | 2026-06-01 | WS 鉴权改为短期签名 token（HMAC-SHA256 + jti + TTL ≤30s，Phase 2 基础 + Phase 3 扩展）；traffic_cycles 加 `iface` 列；§14.1.1 按 `monitored_interfaces` 动态缓存（空配置=合并行，否则各卡独立行）；§11.2 补 `SELECT 1` 与 URL 格式校验不重复的论证 |