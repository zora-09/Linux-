# Server Operations Platform

基于 Ubuntu 26.04 云服务器构建的综合运维实践项目。

## 1. 项目介绍

本项目以一台 Linux 云服务器为基础，
从系统初始化、安全加固开始，逐步构建包含容器化应用部署、
自动化运维、监控告警和备份恢复的综合运维平台。

项目主要通过实际操作完成，不仅记录部署过程，
还会记录配置文件、验证命令、运行结果以及故障排查过程。

## 2. 技术栈

| 分类 | 技术 |
|---|---|
| 操作系统 | Ubuntu 26.04 |
| 系统管理 | Linux、systemd、APT |
| 安全防护 | SSH、UFW、Fail2ban |
| 容器技术 | Docker、Docker Compose |
| Web 服务 | Nginx、Flask |
| 数据库 | MySQL |
| 自动化运维 | Ansible |
| 监控告警 | Prometheus、Grafana |
| 版本管理 | Git、GitHub/Gitee |

## 3. 项目架构

项目分为以下十个实施模块：

1. 系统初始化与环境检查
2. SSH 远程访问安全加固
3. 防火墙与入侵防护
4. Linux 系统安全基线
5. Docker 环境部署与管理
6. Docker Compose 部署 Nginx、Flask、MySQL
7. 网站 HTTPS 配置与证书管理
8. Ansible 自动化运维
9. Prometheus + Grafana 监控告警
10. 自动化备份、恢复与故障演练

## 4. 项目目录

- `docs/`：模块实施记录
- `scripts/`：自动化脚本
- `configs/`：配置文件
- `modules/`：各模块实施文件
- `ROADMAP.md`：项目路线图

## 5. 当前项目进度

| 模块 | 内容 | 状态 |
|---|---|---|
| 01 | Linux 系统初始化 | 已完成 |
| 02 | SSH 安全加固 | 已完成 |
| 03 | UFW + Fail2ban | 已完成 |
| 04 | Linux 安全基线 | 已完成 |
| 05 | Docker 环境部署与管理 | 进行中（阶段 1～2 已完成） |
| 06～10 | 应用部署、HTTPS、自动化、监控、备份 | 待实施 |

### 模块 05 阶段进度

| 阶段 | 内容 | 状态 |
|---|---|---|
| 1 | Docker 基础与安装 | 已完成 |
| 2 | Docker 镜像管理 | 已完成 |
| 3 | Docker 容器管理 | 待开始 |
| 4 | Docker 数据卷与数据持久化 | 待开始 |
| 5 | Docker 网络管理 | 待开始 |
| 6 | Dockerfile 镜像构建 | 待开始 |
| 7 | Docker Compose 多容器编排 | 待开始 |
| 8 | Docker 日常运维与故障排查 | 待开始 |
| 9 | Docker 安全管理 | 待开始 |
| 10 | Docker 综合项目与面试模拟 | 待开始 |

### 模块 05 已完成阶段文档\n\n- [阶段 1：Docker 基础与安装](docs/05-docker-basics-installation.md)\n- [阶段 2：Docker 镜像管理](docs/05-docker-image-management.md)\n\n## 6. 项目说明

本项目为个人 Linux 服务器运维实践项目，
所有操作记录以实际实施结果为准。涉及服务器地址、账号、密钥、
Token 等信息时，提交前必须检查并脱敏。
