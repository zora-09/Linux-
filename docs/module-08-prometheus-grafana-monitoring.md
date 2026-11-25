# 模块08：Prometheus + Grafana 监控告警

## 1. 实验目标与架构

本模块在 Ubuntu 26.04.1 LTS 与 Rocky Linux 10.2 两台云服务器上搭建主机监控平台，实现 Linux 主机指标采集、PromQL 查询、Grafana 可视化、告警规则评估、Alertmanager 接收告警，以及基础故障演练。

### 1.1 实验环境

| 主机 | 操作系统 | 私网地址 | 主要角色 |
|---|---|---|---|
| Ubuntu | Ubuntu 26.04.1 LTS | 10.2.0.15 | Node Exporter、被监控主机 |
| Rocky Linux | Rocky Linux 10.2 | 10.2.4.7 | Prometheus、Grafana、Alertmanager、Node Exporter、被监控主机 |

两台云服务器通过私网通信。Prometheus 部署在 Rocky Linux，定期抓取 Rocky 本机和 Ubuntu 的 Node Exporter 指标。

### 1.2 监控架构

```text
Ubuntu 26.04.1                         Rocky Linux 10.2
+------------------+                  +--------------------------------+
| Node Exporter    |                  | Node Exporter                  |
| TCP 9100         |                  | TCP 9100                       |
+--------+---------+                  +---------------+----------------+
         |                                              |
         | HTTP /metrics                                | HTTP /metrics
         |                                              |
         +--------------------+-------------------------+
                              |
                              v
                    +----------------------+
                    | Prometheus :9091     |
                    | 指标采集 / 存储       |
                    | PromQL / 告警规则     |
                    +----------+-----------+
                               |
                   +-----------+-----------+
                   |                       |
                   v                       v
          +----------------+      +----------------------+
          | Grafana :3000  |      | Alertmanager :9093   |
          | 仪表盘与图表    |      | 告警分组与管理        |
          +----------------+      +----------------------+
```

组件职责：

- **Node Exporter**：采集 Linux 主机 CPU、内存、磁盘、网络等指标，并通过 HTTP `/metrics` 暴露。
- **Prometheus**：采用 Pull 模式定期抓取指标，保存时间序列数据，执行 PromQL 查询和告警规则。
- **Grafana**：连接 Prometheus 数据源，将指标制作成图表和仪表盘。
- **Alertmanager**：接收 Prometheus 发来的告警，负责分组、去重、抑制、静默和通知路由。

## 2. Node Exporter 部署

两台主机均部署 Node Exporter 1.12.1，使用专用系统用户运行。

### 2.1 服务配置

二进制文件：

```text
/usr/local/bin/node_exporter
```

专用用户：

- Rocky Linux：`node_exporter`，不可交互登录。
- Ubuntu：`node_exporter`，不可交互登录。

systemd 单元文件 `/etc/systemd/system/node_exporter.service`：

```ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

加载并启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```

检查监听端口及指标：

```bash
sudo ss -lntp | grep 9100
curl http://127.0.0.1:9100/metrics
```

Node Exporter 默认监听 TCP 9100，指标接口为 `/metrics`。常见指标包括：

- `node_cpu_seconds_total`：CPU 各模式累计运行时间。
- `node_memory_MemAvailable_bytes`：可用内存。
- `node_memory_MemTotal_bytes`：总内存。
- `node_filesystem_avail_bytes`：文件系统可用空间。
- `node_filesystem_size_bytes`：文件系统总容量。
- `node_network_receive_bytes_total`：网络累计接收字节数。
- `node_network_transmit_bytes_total`：网络累计发送字节数。

## 3. Prometheus 部署与配置

### 3.1 安装信息

Prometheus 部署在 Rocky Linux：

- 版本：3.13.5（Linux AMD64）。
- 二进制：`/usr/local/bin/prometheus`、`/usr/local/bin/promtool`。
- 配置目录：`/etc/prometheus`。
- 数据目录：`/var/lib/prometheus`。
- 运行用户：`prometheus`，不可交互登录。
- Web 监听地址：`127.0.0.1:9091`。
- 数据保留时间：7 天。

配置目录和数据目录由 `prometheus` 用户拥有，并限制目录权限。

### 3.2 Prometheus 配置文件

`/etc/prometheus/prometheus.yml`：

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "/etc/prometheus/rules/*.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["127.0.0.1:9091"]

  - job_name: "node"
    static_configs:
      - targets:
          - "127.0.0.1:9100"
          - "10.2.0.15:9100"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "127.0.0.1:9093"
```

关键配置说明：

- `scrape_interval: 15s`：每 15 秒抓取一次指标。
- `evaluation_interval: 15s`：每 15 秒评估一次告警规则。
- `job_name`：定义采集任务。
- `targets`：定义需要抓取的目标地址。
- `rule_files`：加载告警规则文件。
- `alertmanagers`：指定 Alertmanager 地址。

### 3.3 systemd 服务

`/etc/systemd/system/prometheus.service` 的关键配置：

```ini
[Unit]
Description=Prometheus Monitoring System
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=127.0.0.1:9091 \
  --web.enable-lifecycle \
  --storage.tsdb.retention.time=7d
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

启动与验证：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
curl http://127.0.0.1:9091/-/healthy
```

检查配置：

```bash
promtool check config /etc/prometheus/prometheus.yml
```

配置修改后可使用生命周期接口热加载：

```bash
curl -X POST http://127.0.0.1:9091/-/reload
```

Prometheus Web UI 只监听回环地址，不直接暴露公网。通过本地 SSH 隧道访问：

```bash
ssh -L 9091:127.0.0.1:9091 root@<Rocky公网地址>
```

浏览器访问 `http://127.0.0.1:9091`。

### 3.4 Target 状态验证

通过 Prometheus 页面 Status → Targets，或查询 API：

```bash
curl -s http://127.0.0.1:9091/api/v1/targets
```

部署完成后确认以下三个 Target 均为 `UP`：

| Job | Target | 用途 |
|---|---|---|
| prometheus | 127.0.0.1:9091 | Prometheus 自监控 |
| node | 127.0.0.1:9100 | Rocky Linux 指标 |
| node | 10.2.0.15:9100 | Ubuntu 指标 |

## 4. 双机采集网络与防火墙排障

Prometheus 需要从 Rocky 访问 Ubuntu 的 TCP 9100 端口。排障过程中，虽然两台主机可以互相 ping 通，但 Rocky 执行：

```bash
curl http://10.2.0.15:9100/metrics
```

最初连接超时。

通过 tcpdump 在 Rocky 观察到 TCP SYN 持续发出，但没有收到响应；Ubuntu 侧未观察到预期请求。进一步检查发现 Ubuntu 云服务器安全组缺少允许 Rocky 私网地址访问 TCP 9100 的入站规则。

添加 Ubuntu 云安全组入站规则：

- 协议端口：TCP 9100。
- 来源：Rocky 私网地址 `10.2.4.7/32`。
- 不开放给 `0.0.0.0/0`。

Rocky firewalld 同时添加仅允许 Ubuntu 私网地址访问 9100 的规则：

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.2.0.15/32" port port="9100" protocol="tcp" accept'
sudo firewall-cmd --reload
```

Ubuntu UFW 已有仅允许 Rocky 来源访问 9100 的规则：

```text
9100/tcp ALLOW IN 10.2.4.7
```

再次从 Rocky 执行 curl 后成功获取指标。

排障结论：

1. 主机防火墙与云安全组是两层独立的访问控制。
2. ping 使用 ICMP，不能证明 TCP 端口可访问。
3. `tcpdump` 可辅助判断 SYN 是否发出、是否到达以及是否有响应。
4. 监控端口只允许可信私网来源访问，避免将 Node Exporter 暴露到公网。

## 5. PromQL 指标查询

PromQL 是 Prometheus 的查询语言。Node Exporter 中不少指标是持续递增的 Counter，例如 CPU 累计时间和网络累计字节数。需要使用 `rate()` 计算指定时间窗口内的平均每秒速率。

### 5.1 查看原始 CPU 指标

```promql
node_cpu_seconds_total
```

通过 `mode` 标签区分 idle、user、system、iowait 等 CPU 状态。该指标是累计时间，不是直接的 CPU 使用率。

### 5.2 CPU 使用率

```promql
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))
```

计算逻辑：

1. `rate(...[5m])` 计算最近 5 分钟内 CPU idle 累计时间的平均每秒速率。
2. `avg by (instance)` 对同一主机各 CPU 核心求平均。
3. `1 - idle` 得到非空闲比例。
4. 乘以 100 转换为百分比。

### 5.3 内存使用率

```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```

使用 MemAvailable 而不是简单用 MemFree，避免把 Linux 用于缓存且可回收的内存误判为不可用。

### 5.4 根分区使用率

```promql
100 * (
  1 -
  node_filesystem_avail_bytes{mountpoint="/"}
  /
  node_filesystem_size_bytes{mountpoint="/"}
)
```

### 5.5 网络接收速率

```promql
rate(node_network_receive_bytes_total{device="eth0"}[5m]) * 8 / 1024 / 1024
```

将字节每秒转换为 Mbit/s：乘以 8 转换为 bit，再除以 1024²。

### 5.6 网络发送速率

```promql
rate(node_network_transmit_bytes_total{device="eth0"}[5m]) * 8 / 1024 / 1024
```

### 5.7 常见标签匹配

| 运算符 | 含义 |
|---|---|
| `=` | 标签值精确匹配 |
| `!=` | 标签值不匹配 |
| `=~` | 正则表达式匹配 |
| `!~` | 正则表达式不匹配 |

### 5.8 实际查询结果记录

PromQL 实验中，两台主机均成功返回 CPU、内存、磁盘和网络指标。某次查询结果记录如下，数值会随主机负载变化：

| 指标 | Rocky Linux | Ubuntu |
|---|---:|---:|
| 内存使用率 | 18.71% | 42.40% |
| 根分区使用率 | 8.66% | 26.17% |
| 网络接收速率 | 0.02053 Mbit/s | 0.01075 Mbit/s |
| 网络发送速率 | 0.00890 Mbit/s | 0.02543 Mbit/s |

CPU、网络等动态指标应结合查询时间窗口理解，不应把某次采样值当作固定性能数据。

## 6. Grafana 可视化

### 6.1 安装与监听

Grafana OSS 13.3.1 部署在 Rocky Linux。服务通过 systemd 管理，Web 监听地址设置为：

```ini
[server]
http_addr = 127.0.0.1
http_port = 3000
```

检查服务和端口：

```bash
sudo systemctl status grafana-server
sudo ss -lntp | grep 3000
```

确认监听地址为 `127.0.0.1:3000`。

通过 SSH 隧道访问：

```bash
ssh -L 3000:127.0.0.1:3000 root@<Rocky公网地址>
```

浏览器访问 `http://127.0.0.1:3000`。首次登录后修改默认密码。

### 6.2 添加 Prometheus 数据源

在 Grafana 中添加 Prometheus 数据源，URL 设置为：

```text
http://127.0.0.1:9091
```

执行 Save & test，确认数据源连接成功。

### 6.3 Linux 双机监控仪表盘

创建仪表盘“Linux 双机监控”，包含以下五张 Time series 图表：

| 图表 | PromQL 指标 | 单位 |
|---|---|---|
| CPU 使用率 | CPU idle 计算表达式 | Percent (0-100) |
| 内存使用率 | MemAvailable / MemTotal | Percent (0-100) |
| 根分区使用率 | 根分区 avail / size | Percent (0-100) |
| 网络接收速率 | receive bytes rate | Mbits/sec |
| 网络发送速率 | transmit bytes rate | Mbits/sec |

仪表盘时间范围设置为 Last 15 minutes，自动刷新间隔设置为 30s。通过 `instance` 标签区分 Rocky 与 Ubuntu 两台主机。

## 7. Alertmanager 与告警规则

### 7.1 Alertmanager 部署

Alertmanager 部署在 Rocky Linux：

- 版本：0.35.0。
- 二进制：`/usr/local/bin/alertmanager`、`/usr/local/bin/amtool`。
- 配置目录：`/etc/alertmanager`。
- 数据目录：`/var/lib/alertmanager`。
- 运行用户：`alertmanager`，不可交互登录。
- Web/API 监听：`127.0.0.1:9093`。

配置文件 `/etc/alertmanager/alertmanager.yml`：

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: "default-receiver"
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h

receivers:
  - name: 'default-receiver'
```

检查配置并启动：

```bash
amtool check-config /etc/alertmanager/alertmanager.yml
sudo systemctl enable --now alertmanager
sudo systemctl status alertmanager
curl http://127.0.0.1:9093/-/healthy
```

Alertmanager Web 页面通过 SSH 隧道访问：

```bash
ssh -L 9093:127.0.0.1:9093 root@<Rocky公网地址>
```

Prometheus 通过 `alerting.alertmanagers` 配置连接 Alertmanager，并通过 API 验证：

```bash
curl -s http://127.0.0.1:9091/api/v1/alertmanagers
```

### 7.2 告警规则

规则文件：`/etc/prometheus/rules/node-alerts.yml`。

```yaml
groups:
  - name: node-alerts
    rules:
      - alert: CPUUSAGE
        expr: 100*(1-avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m]))) > 80
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "服务器cpu使用率过高"
          description: "服务器{{ $labels.instance }} cpu 使用率持续超过80%,当前值为{{ $value }}%"

      - alert: diskusage
        expr: 100*(1-(node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) > 80
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "服务器根分区空间不足"
          description: "服务器{{ $labels.instance }}根分区使用率超过80%,当前值为{{ $value }}%"
```

规则含义：

- CPU 使用率超过 80%，并持续 1 分钟，触发 `CPUUSAGE`。
- 根分区使用率超过 80%，并持续 1 分钟，触发 `diskusage`。
- `severity: warning` 为告警级别标签。
- `summary` 和 `description` 用于描述告警内容。

检查规则并热加载：

```bash
promtool check rules /etc/prometheus/rules/node-alerts.yml
curl -X POST http://127.0.0.1:9091/-/reload
```

通过 API 查看规则状态：

```bash
curl -s http://127.0.0.1:9091/api/v1/rules
```

### 7.3 告警状态

- **Inactive**：告警表达式当前不满足。
- **Pending**：告警条件已满足，但尚未达到 `for` 指定的持续时间。
- **Firing**：告警条件持续满足，正式触发并发送给 Alertmanager。

### 7.4 告警测试与验证

CPU 告警测试时，临时将阈值调整为 2%，验证 Prometheus 告警从 Pending 转为 Firing，并通过 Alertmanager API 确认收到两台主机的告警。测试后恢复正式阈值 80%。

磁盘告警测试时，临时将根分区使用率阈值调整为 10%。当两台主机的实际根分区使用率分别约为 13.25% 和 26.31% 时，告警进入 Firing，Alertmanager API 收到两条 active 告警。测试后将阈值恢复为 80%，重新加载规则，并确认告警解除。

检查告警 API：

```bash
curl -s http://127.0.0.1:9091/api/v1/alerts
curl -s http://127.0.0.1:9093/api/v2/alerts
```

**告警通知范围说明：** 本模块已完成 Prometheus 规则评估以及 Alertmanager 告警接收和状态验证；尚未配置邮件、企业微信、钉钉或外部 Webhook 等实际通知接收端。因此不将外部消息通知描述为已实现功能。

## 8. 综合故障演练

### 8.1 Node Exporter 停止导致 Target Down

在 Rocky Linux 停止 Node Exporter：

```bash
sudo systemctl stop node_exporter
```

通过 Prometheus Targets 页面或 API 检查状态：

```bash
curl -s http://127.0.0.1:9091/api/v1/targets
```

Rocky 的 `127.0.0.1:9100` Target 状态变为 `down`，错误信息为连接被拒绝；Ubuntu 与 Prometheus 自身 Target 仍为 `up`。

恢复服务：

```bash
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

再次检查 Targets，确认三个目标均恢复为 `up`。

排查思路：

1. 在 Prometheus Targets 页面定位 Down 的 Job/Instance。
2. 登录目标主机检查 `systemctl status node_exporter`。
3. 检查端口监听：`ss -lntp | grep 9100`。
4. 使用 `curl http://目标地址:9100/metrics` 测试接口。
5. 如果连接失败，再检查主机防火墙、云安全组和网络路径。

### 8.2 CPU 压力与指标观察

环境未安装 `stress-ng`，使用以下命令启动一个短时 CPU 压力进程：

```bash
timeout 70s yes > /dev/null &
```

通过 `top` 观察到 `yes` 进程占用一个 CPU 核心，整机 CPU 使用率随之上升。Prometheus 的 CPU 使用率查询也出现变化。由于查询使用 5 分钟 `rate()` 窗口，短时压力会被时间窗口平滑，且正式告警阈值为 80%，本次压力测试没有触发正式 CPU 告警。

进程在 70 秒后由 `timeout` 自动结束，随后检查 CPU 空闲比例恢复正常。

本实验验证了 CPU 进程定位、指标变化和压力结束后的资源恢复；CPU 告警链路的触发验证由前述临时降低阈值的测试完成。

### 8.3 磁盘空间告警模拟

通过 PromQL 检查根分区容量使用率，并将告警阈值临时从 80% 调整为 10%，验证告警进入 Firing，Alertmanager 收到 active 告警。测试后恢复正式 80% 阈值。

本次没有实际写入大量数据或填满磁盘，属于通过临时阈值调整进行的告警模拟，不应描述为真实磁盘写满演练。

### 8.4 综合 Web 访问排查

模拟用户反馈网站访问缓慢，按系统资源、网络连接、服务监听和 HTTP 响应的顺序检查 Rocky Linux。

| 检查项 | 命令 | 实际观察 |
|---|---|---|
| 系统负载 | `uptime` | Load average 为 0.12、0.05、0.02，2 核主机负载较低 |
| 内存 | `free -h` | MemAvailable 约 2.5 GiB，内存可用 |
| 磁盘容量 | `df -hT` | 根分区约 14% 使用率 |
| inode | `df -ih` | 根分区 inode 使用率约 1% |
| TCP 连接 | `ss -s` | 26 个 ESTABLISHED，未见明显连接堆积 |
| 监听端口 | `ss -lntp` | Nginx 监听 80、8081、8082 |
| HTTP 状态 | `curl -I http://127.0.0.1/` | 返回 HTTP 200 OK |
| HTTP 耗时 | `curl -w ...` | 本机请求总耗时约 0.000512 秒 |

本机 HTTP 测试结果：

```text
HTTP状态码: 200
DNS耗时: 0.000022s
连接耗时: 0.000231s
首字节时间: 0.000480s
总耗时: 0.000512s
```

综合判断：本次检查没有发现 Rocky 主机明显的 CPU、内存、磁盘、inode、TCP 连接或 Nginx 本机响应瓶颈。

限制：测试请求访问的是 `127.0.0.1`，只验证本机到 Nginx 的链路，没有覆盖外部客户端到服务器的完整网络路径。因此不能据此断定外部用户访问一定正常。若真实场景仍然缓慢，应继续从客户端发起请求，结合 DNS、路由、网络时延、云安全组、Nginx 访问日志和上游应用响应时间排查。

## 9. 常用运维命令汇总

| 用途 | 命令 |
|---|---|
| 查看服务状态 | `systemctl status prometheus` |
| 查看 Node Exporter 状态 | `systemctl status node_exporter` |
| 查看 Grafana 状态 | `systemctl status grafana-server` |
| 查看 Alertmanager 状态 | `systemctl status alertmanager` |
| 查看监听端口 | `ss -lntp` |
| 检查 Prometheus 配置 | `promtool check config /etc/prometheus/prometheus.yml` |
| 检查告警规则 | `promtool check rules /etc/prometheus/rules/node-alerts.yml` |
| 热加载 Prometheus | `curl -X POST http://127.0.0.1:9091/-/reload` |
| 查看 Targets | `curl -s http://127.0.0.1:9091/api/v1/targets` |
| 查看告警 | `curl -s http://127.0.0.1:9091/api/v1/alerts` |
| 查看 Alertmanager 告警 | `curl -s http://127.0.0.1:9093/api/v2/alerts` |
| 查看主机指标 | `curl http://127.0.0.1:9100/metrics` |

## 10. 阶段验收与面试复盘

### 10.1 完成内容

| 阶段 | 内容 | 状态 |
|---|---|---|
| 1 | 监控基础与架构 | 已完成 |
| 2 | Prometheus 部署与入门 | 已完成 |
| 3 | Node Exporter 部署 | 已完成 |
| 4 | PromQL 指标查询 | 已完成 |
| 5 | Grafana 数据源与仪表盘 | 已完成 |
| 6 | Alertmanager 与告警规则 | 已完成 |
| 7 | 综合项目与故障演练 | 已完成 |

### 10.2 核心知识点

1. Prometheus 通过 Pull 模式定期抓取 Exporter 暴露的指标。
2. Node Exporter 负责采集 Linux 主机指标，不负责长期存储和图表展示。
3. PromQL 的 `rate()` 用于计算 Counter 在指定时间窗口内的平均每秒速率。
4. Grafana 连接 Prometheus 数据源并展示监控图表。
5. Prometheus 负责告警规则评估，Alertmanager 负责接收和管理告警。
6. Pending 表示告警条件满足但持续时间尚未达到；Firing 表示告警正式触发。
7. 排查 Target Down 时，应检查服务状态、监听端口、指标接口、防火墙和云安全组。
8. 排查网站慢时，需要将系统资源、网络连接、服务监听、HTTP 响应时间和应用日志结合起来分析。

### 10.3 面试题

**1. Prometheus 和 Grafana 有什么区别？**

Prometheus 负责采集、存储指标并提供 PromQL 查询；Grafana 负责连接数据源，将指标展示成仪表盘和图表。

**2. Node Exporter 的作用是什么？**

Node Exporter 采集 Linux 主机的 CPU、内存、磁盘、网络等指标，通过 HTTP 的 `/metrics` 接口暴露给 Prometheus。

**3. Prometheus 为什么采用 Pull 模式？**

Prometheus 主动定期抓取目标指标，便于集中管理采集目标、采集周期和 Target 健康状态。

**4. CPU 使用率为什么需要使用 rate()？**

`node_cpu_seconds_total` 是持续递增的累计时间。使用 `rate()` 计算一段时间内的每秒速率，再根据 idle 比例计算 CPU 使用率。

**5. Target 显示 DOWN 如何排查？**

先在 Prometheus Targets 页面定位具体目标，再登录目标机检查 Exporter 服务、监听端口和 `/metrics` 接口；如果服务本身正常，则继续检查主机防火墙、云安全组和网络路径。

**6. Pending 和 Firing 有什么区别？**

Pending 表示告警表达式已经满足，但还没有持续达到 `for` 时间；Firing 表示持续时间达到要求，告警正式触发。

**7. Prometheus 与 Alertmanager 如何分工？**

Prometheus 根据规则计算告警并发送给 Alertmanager；Alertmanager 负责告警分组、去重、抑制、静默和通知路由。

**8. 为什么 ping 通了，Node Exporter 仍然可能访问失败？**

ping 测试的是 ICMP 连通性，而 Node Exporter 使用 TCP 9100。云安全组或主机防火墙可能只允许 ICMP 或未开放 TCP 9100，因此需要结合 curl、ss 和 tcpdump 排查。

## 11. 模块总结与后续扩展

模块08完成了双机 Linux 主机监控平台搭建，实现 Node Exporter 指标采集、Prometheus 时序数据存储与 PromQL 查询、Grafana 双机监控仪表盘，以及 Prometheus 告警规则和 Alertmanager 告警接收验证。

本模块已完成基础告警链路验证，但尚未接入邮件、企业微信、钉钉或外部 Webhook 通知。CPU 压力测试未触发正式 80% 阈值告警；磁盘告警通过临时降低阈值模拟，没有实际填满磁盘。上述边界均按实际实验结果记录。

后续可扩展真实通知接收端、更多业务指标、服务可用性探测、告警分级与静默策略，以及监控数据备份和恢复。
