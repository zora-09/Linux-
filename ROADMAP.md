## 项目简介

项目采用逐步扩展的双机云服务器环境：模块 01～06 主要基于 Ubuntu 26.04 云服务器开展实践；从模块 07（Ansible 自动化运维）开始，新增一台 2 核 4GB 内存的 Rocky Linux 10.2 云服务器，与 Ubuntu 组成控制端和被控端实验架构。通过 Linux 系统管理、安全加固、容器化部署、自动化运维、监控告警和备份恢复等实践，逐步构建综合运维平台。

## 项目环境

### Ubuntu 云服务器

- 操作系统：Ubuntu 26.04.1 LTS
- 用途：模块 01～06 的主要实验主机；模块 07 起作为 Ansible 控制端，并继续承担项目已有服务及后续实验角色
- 规格：约 2 核、2GB 内存

### Rocky Linux 云服务器

- 操作系统：Rocky Linux 10.2
- 用途：从模块 07 开始新增的 Ansible 被控端及后续运维实验节点
- 规格：2 核、4GB 内存

两台云服务器通过内网互通，模块 07 使用 SSH Ed25519 公钥认证建立 Ansible 控制端到被控端的连接。

## 项目实施路线

| 模块 | 实施内容 | 状态 |
|---|---|---|
| 01 | Linux 系统初始化与环境检查 | 已完成 |
| 02 | SSH 远程访问安全加固 | 已完成 |
| 03 | UFW 防火墙与 Fail2ban 安全防护 | 已完成 |
| 04 | Linux 系统安全基线配置 | 已完成 |
| 05 | Docker 环境部署、管理与 Compose 多容器应用部署 | 已完成 |
| 06 | 网站 HTTPS 配置与证书管理 | 已完成 |
| 07 | Ansible 自动化运维 | 已完成 |
| 08 | Prometheus + Grafana 监控告警 | 已完成 |
| 09 | 自动化备份、恢复与故障演练 | 待实施 |

## 模块 05 阶段拆分

| 阶段 | 实施内容 | 状态 |
|---|---|---|
| 1 | Docker 基础与安装 | 已完成 |
| 2 | Docker 镜像管理 | 已完成 |
| 3 | Docker 容器管理 | 已完成 |
| 4 | Docker 数据卷与数据持久化 | 已完成 |
| 5 | Docker 网络管理 | 已完成 |
| 6 | Dockerfile 镜像构建 | 已完成 |
| 7 | Docker Compose 多容器编排 | 已完成 |
| 8 | Docker Swarm 集群编排 | 已完成 |
| 9 | Docker 安全管理 | 已完成 |
| 10 | Docker 综合实战与面试总结 | 已完成 |
| 综合应用部署 | Nginx + Flask + MySQL 多容器应用部署、服务联调与运行验证 | 已完成 |

## 模块 05 阶段 1 验收内容

第一阶段已完成 Docker 基础概念与架构、容器和虚拟机区别、Namespace/Cgroups/镜像分层原理、Docker Engine 安装与服务管理、hello-world 验证、PID Namespace 实验、daemon.json 镜像加速与日志轮转配置、新建容器日志配置验证，以及 Docker Daemon 启动故障排查练习。

详细操作记录见：[阶段 1：Docker 基础与安装](docs/05-docker/01-docker-basics-installation.md)。

## 模块 05 阶段 2 验收内容

第二阶段已完成镜像分层与缓存、Tag/Digest、镜像查看与历史分析、save/load、export/import、镜像清理与空间检查、常见拉取故障排查及五道面试题。

详细记录见：[阶段 2：Docker 镜像管理](docs/05-docker/02-docker-image-management.md)。

## 模块 05 阶段 3 验收内容

第三阶段已完成容器生命周期管理、exec 与 attach、日志查看、on-failure 重启策略、CPU/内存/PIDs 资源限制，以及“启动命令不存在”和“应用运行后异常退出”两类故障模拟与排查，并完成五道阶段面试题。

详细记录见：[阶段 3：Docker 容器管理](docs/05-docker/03-docker-container-management.md)。

## 模块 05 阶段 4 验收内容

第四阶段已完成 Volume、Bind Mount、tmpfs 原理与实验，验证 Volume 生命周期、只读挂载、宿主机与容器文件共享、UID/GID 权限影响；完成 Volume 备份到宿主机、归档检查及恢复到新 Volume 的实验，并完成 Mounts 信息检查、数据不可见、权限不足和数据生命周期故障排查及面试复盘。

详细记录见：[阶段 4：Docker 数据卷与数据持久化](docs/05-docker/04-docker-volumes-persistence.md)。

## 模块 05 阶段 5 验收内容

第五阶段已完成 Docker 默认 bridge 与自定义 bridge 网络对比、自定义网络创建与容器接入、容器名 DNS 解析、容器多网络连接和路由查看；验证端口映射、host/none 网络模式及网络命名空间；使用 nc、ss/netstat 检查 TCP 监听和连通性，并模拟网络断开、端口监听退出等故障后恢复。另了解 Overlay 网络用途。

详细记录见：[阶段 5：Docker 网络管理](docs/05-docker/05-docker-networking.md)。

## 模块 05 阶段 6 验收内容

第六阶段已完成 Dockerfile 常用指令与自定义镜像构建，掌握构建缓存、CMD/ENTRYPOINT、ARG/ENV、USER/EXPOSE、COPY/ADD、.dockerignore 与构建上下文、多阶段构建；完成 COPY 源路径错误、普通用户目录写权限不足、构建缓存失效等故障模拟与排查，并完成镜像体积优化及九道面试题复盘。

详细记录见：[阶段 6：Dockerfile 镜像构建](docs/05-docker/06-dockerfile-image-build.md)。

## 模块 05 阶段 7 验收内容

第七阶段已完成 Docker Compose YAML 配置、常用命令、服务网络与服务名 DNS、Named Volume 与 Bind Mount、depends_on 与 healthcheck、环境变量与 .env、服务扩缩容、镜像更新和故障排查；完成 Nginx + Flask + Redis 综合项目，验证反向代理、Redis 数据持久化、服务停止后的 502 故障及恢复。

详细记录见：[阶段 7：Docker Compose 多容器编排](docs/05-docker/07-docker-compose.md)。

## 模块 05 阶段 8 验收内容

第八阶段已学习 Docker Swarm 集群编排，涵盖 Swarm 架构、Manager/Worker、Raft 仲裁、集群初始化与节点管理、Service/Task、扩缩容、Routing Mesh、Overlay 网络、服务发现、Stack 部署、滚动更新、回滚和故障排查；并在单机环境中使用 Swarm 状态与服务进行管理和检查。

详细记录见：[阶段 8：Docker Swarm 集群编排](docs/05-docker/08-docker-swarm.md)。

## 模块 05 阶段 9 验收内容

第九阶段已完成 Docker 安全基础、镜像安全、容器权限管理、Secrets 与 Configs、只读根文件系统、tmpfs、Capabilities、Seccomp、AppArmor 和资源限制等知识学习；完成非 root Nginx 综合安全实验，验证只读根文件系统、删除 Capabilities、禁止权限提升以及 CPU、内存和 PID 限制。

详细记录见：[阶段 9：Docker 安全管理](docs/05-docker/09-docker-security.md)。

## 模块 05 阶段 10 验收内容

第十阶段已完成 Nginx + Flask + Redis 综合项目架构分析与 Compose 运行验收；验证 Redis 命名卷在容器重启和强制重建后的数据保留；执行 RDB 备份、SHA256 校验和独立 Redis 容器恢复验证；检查 frontend/backend 网络隔离、服务名 DNS、TCP 连通性及默认 bridge 网络外网访问；完成综合故障排查和 Docker 面试知识复盘。

详细记录见：[阶段 10：Docker 综合实战与面试总结](docs/05-docker/10-docker-comprehensive-practice.md)。

## Docker Compose 综合应用部署验收

已在 Docker 学习模块中完成 Nginx + Flask + MySQL 多容器应用部署与联调，涉及 Flask 应用容器化、MySQL 数据库服务、Compose 服务编排、容器网络通信、Nginx 反向代理和数据持久化验证。该综合应用部署作为模块 05 的扩展实战内容，不再单独编号为模块。

## 模块 06 验收内容

第六模块已完成当前学习范围内的 HTTPS/TLS 原理学习，以及 OpenSSL 私钥生成、CSR 生成与查看、自签名证书生成、证书信息和 SAN 检查、证书与私钥公钥匹配验证、自签名证书验证。公共 CA 申请、Certbot 自动续期、Nginx HTTPS 部署和真实站点故障演练未在本轮执行，相关原理、配置示例和排查方法已整理为知识参考。

详细记录见：[模块 06：网站 HTTPS 配置与证书管理](docs/module-06-https-certificates.md)。


## 模块 07 验收内容

第七模块已完成 Ansible 自动化运维实践，采用 Ubuntu 26.04.1 LTS 作为控制端、Rocky Linux 10.2 作为被控端，通过 SSH Ed25519 密钥认证连接。完成 Ansible Core 安装、Inventory 与 ansible.cfg 配置、Ad-Hoc 批量管理，以及软件包、服务、防火墙、用户、目录和文件管理。

Playbook 工程化部分完成变量与命令行变量覆盖、Facts、when、loop、Handlers、Jinja2 Templates 和 Roles；安全与复杂任务部分完成 Ansible Vault、become、Tags、ignore_errors、failed_when、block/rescue/always，以及 limit、serial、forks、run_once、throttle 等执行控制知识与实验。

综合项目使用 Common 和 Nginx 两个 Role，通过统一入口 `playbooks/site.yml` 完成 Rocky Linux 运维目录初始化和 Nginx 部署。完成 Nginx 配置冲突排查、Check Mode、Diff Mode、幂等性验证、日志查看及软件包错误排查。综合 Playbook 再次执行时达到 `changed=0`、`failed=0`。

详细记录见：[模块 07：Ansible 自动化运维](docs/module-07-ansible-automation.md)。


## 模块 08 验收内容

第八模块已完成 Prometheus + Grafana 双机主机监控实践。采用 Rocky Linux 10.2 部署 Prometheus 3.13.5、Grafana OSS 13.3.1 和 Alertmanager 0.35.0；Ubuntu 26.04.1 与 Rocky Linux 均部署 Node Exporter 1.12.1。通过 Prometheus Pull 模式采集两台主机的 CPU、内存、磁盘和网络指标，完成 PromQL 查询及 Grafana 双机仪表盘配置。

告警部分完成 Prometheus 与 Alertmanager 对接，配置 CPU 使用率和根分区使用率告警规则，并通过临时调整阈值验证 Pending、Firing 和 Alertmanager 告警接收。尚未配置邮件、企业微信、钉钉或外部 Webhook 等真实通知接收端。

综合演练完成 Node Exporter 停止导致 Target Down 的检测与恢复、CPU 压力下的进程定位和指标观察、磁盘告警阈值模拟，以及结合 uptime、free、df、ss、curl 等命令的综合 Web 访问排查。CPU 压力测试未触发正式 80% 阈值告警；磁盘告警通过临时降低阈值模拟，未实际填满磁盘。

详细记录见：[模块 08：Prometheus + Grafana 监控告警](docs/module-08-prometheus-grafana-monitoring.md)。

## 实施原则

1. 每个模块按照“原理讲解 → 命令学习 → 动手实验 → 故障排查 → 面试问题 → 文档记录”的顺序推进。
2. 按照既定模块和阶段顺序实施。
3. 每个阶段完成配置后进行功能验证和阶段验收。
4. 记录实际执行的命令和关键配置，不虚构实施结果。
5. 保留关键验证结果和故障处理过程。
6. 涉及密码、密钥、Token、内网地址等信息时必须脱敏。
7. 每完成一个模块或阶段，及时更新项目文档和进度，并提交 Git。
