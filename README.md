# Server Operations Platform

基于 Ubuntu 26.04 云服务器构建的综合运维实践项目。

## 1. 项目介绍

本项目以一台 Linux 云服务器为基础，从系统初始化、安全加固开始，逐步构建包含容器化应用部署、自动化运维、监控告警和备份恢复的综合运维平台。

项目主要通过实际操作完成，不仅记录部署过程，还会记录配置文件、验证命令、运行结果以及故障排查过程。

## 2. 技术栈

| 分类 | 技术 |
|---|---|
| 操作系统 | Ubuntu 26.04 |
| 系统管理 | Linux、systemd、APT |
| 安全防护 | SSH、UFW、Fail2ban |
| 容器技术 | Docker、Docker Compose、Docker Swarm |
| Web 服务 | Nginx、Flask |
| 数据库 | MySQL、Redis |
| 自动化运维 | Ansible |
| 监控告警 | Prometheus、Grafana |
| 版本管理 | Git、GitHub/Gitee |

## 3. 项目架构

项目分为以下九个实施模块：

1. 系统初始化与环境检查
2. SSH 远程访问安全加固
3. 防火墙与入侵防护
4. Linux 系统安全基线
5. Docker 环境部署与管理（包含 Docker Compose 多容器应用部署）
6. 网站 HTTPS 配置与证书管理
7. Ansible 自动化运维
8. Prometheus + Grafana 监控告警
9. 自动化备份、恢复与故障演练

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
| 05 | Docker 环境部署、管理与 Compose 多容器应用部署 | 已完成 |
| 06 | 网站 HTTPS 配置与证书管理 | 当前开始 |
| 07 | Ansible 自动化运维 | 待实施 |
| 08 | Prometheus + Grafana 监控告警 | 待实施 |
| 09 | 自动化备份、恢复与故障演练 | 待实施 |

### 模块 05 阶段进度

| 阶段 | 内容 | 状态 |
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
| 10 | Docker 综合项目、持久化验证、故障排查与面试总结 | 已完成 |
| 综合应用部署 | Nginx + Flask + MySQL 多容器应用部署与联调 | 已完成 |

### 模块 05 文档

模块 05 的阶段文档统一存放在 [docs/05-docker/](docs/05-docker/) 目录：

- [阶段 1：Docker 基础与安装](docs/05-docker/01-docker-basics-installation.md)
- [阶段 2：Docker 镜像管理](docs/05-docker/02-docker-image-management.md)
- [阶段 3：Docker 容器管理](docs/05-docker/03-docker-container-management.md)
- [阶段 4：Docker 数据卷与数据持久化](docs/05-docker/04-docker-volumes-persistence.md)
- [阶段 5：Docker 网络管理](docs/05-docker/05-docker-networking.md)
- [阶段 6：Dockerfile 镜像构建](docs/05-docker/06-dockerfile-image-build.md)
- [阶段 7：Docker Compose 多容器编排](docs/05-docker/07-docker-compose.md)
- [阶段 8：Docker Swarm 集群编排](docs/05-docker/08-docker-swarm.md)
- [阶段 9：Docker 安全管理](docs/05-docker/09-docker-security.md)
- [阶段 10：Docker 综合实战与面试总结](docs/05-docker/10-docker-comprehensive-practice.md)

## 6. 项目说明

本项目为个人 Linux 服务器运维实践项目，所有操作记录以实际实施结果为准。涉及服务器地址、账号、密钥、Token 等信息时，提交前必须检查并脱敏。
