# 模块 05：Docker 基础与安装

> 阶段：第一阶段（Docker 基础与安装）  
> 状态：已完成  
> 实施日期：2026-11-10 至 2026-11-11  
> 实施环境：Ubuntu 26.04.1 LTS（amd64）

## 1. 阶段目标

理解 Docker 的基本概念、整体架构和底层隔离机制；在 Ubuntu 服务器上安装 Docker Engine，完成服务管理、镜像加速、日志配置和基础验证，并能够排查常见的 Docker Daemon 启动问题。

本阶段只覆盖 Docker 基础与安装，不包含后续镜像管理、容器管理、数据卷、网络、Dockerfile、Compose 等阶段内容。

## 2. 实验环境

| 项目 | 实际环境 |
|---|---|
| 操作系统 | Ubuntu 26.04.1 LTS（Resolute） |
| 架构 | x86_64 / amd64 |
| 内核 | 7.0.0-43-generic |
| CPU | 2 vCPU |
| 内存 | 约 1.9 GiB |
| Docker Engine | 29.9.3 |
| Docker CLI | 29.9.3 |
| containerd | 2.4.2 |
| runc | 1.5.2 |
| Docker Compose 插件 | v5.7.0 |
| Buildx 插件 | v0.38.1 |

> 服务器资源有限，后续实验优先使用轻量级镜像，避免无必要的大型镜像构建和高负载操作。

## 3. Docker 基本概念

- **Docker**：容器化平台，用于构建、分发和运行容器化应用。
- **镜像（Image）**：创建容器所使用的只读模板，通常由多个文件层组成。
- **容器（Container）**：镜像运行后的实例，具有独立的运行状态和可写层。
- **仓库（Registry）**：存储和分发镜像的服务，例如 Docker Hub，也可以使用企业内部的 Harbor。

镜像与容器的关系：一个镜像可以创建多个容器。多个容器可以共享镜像的只读层，但每个容器拥有自己的可写层，彼此隔离。

## 4. Docker 整体架构

执行 `docker run nginx` 时，主要组件职责如下：

1. **Docker Client（docker CLI）**：接收用户命令，通过 Docker API 向 Daemon 发起请求。
2. **dockerd（Docker Daemon）**：接收 API 请求，管理镜像、容器、网络和卷等 Docker 资源，并协调底层组件。
3. **Registry**：镜像仓库。本地没有所需镜像时，需要从仓库获取镜像。
4. **containerd**：负责更底层的容器生命周期管理以及镜像传输、存储等工作。
5. **runc**：遵循 OCI Runtime 规范，根据容器配置创建容器进程，并设置相应的 Namespace、Cgroups 等隔离与资源控制。

简化调用关系：

```text
用户执行 docker 命令
        |
        v
Docker Client
        | Docker API
        v
dockerd
        |
        v
containerd
        |
        v
runc
        |
        v
容器进程运行
```

Docker Client 与 Daemon 通常通过 Unix Socket 通信。本机默认 Socket 常见为 `/var/run/docker.sock`（可能链接到 `/run/docker.sock`）。拥有该 Socket 的高权限访问能力通常意味着可以获得宿主机 root 级别的控制能力，因此不能随意开放或授权。

补充：镜像拉取不是简单地由 dockerd 单独完成所有底层工作；dockerd 会协调 containerd 等组件完成镜像相关操作。

## 5. 容器与虚拟机的区别

| 对比项 | Docker 容器 | 虚拟机 |
|---|---|---|
| 内核 | 共享宿主机内核 | 每台虚拟机运行自己的操作系统内核 |
| 隔离基础 | Linux Namespace、Cgroups 等 | Hypervisor 虚拟化硬件并隔离虚拟机 |
| 资源开销 | 通常较小 | 通常较大 |
| 启动过程 | 创建并启动容器进程 | 需要启动虚拟硬件环境和客户机操作系统 |
| 操作系统环境 | 镜像提供用户空间文件和依赖 | 客户机有完整操作系统环境 |

容器通常启动更快，是因为它不需要为每个容器单独启动一套操作系统内核。

例如，CentOS 用户空间镜像可以运行在 Ubuntu Linux 宿主机上，但容器使用的是宿主机 Ubuntu 正在运行的 Linux 内核，而不是 CentOS 内核。需要注意 CPU 架构、内核接口和应用兼容性等条件。

宿主机内核漏洞可能影响容器，严重时可能危及宿主机或其他容器，因此需要及时更新内核，并合理使用非 root 用户、Capabilities、seccomp、AppArmor 等安全措施。

## 6. Docker 底层原理

### 6.1 Namespace：资源隔离

Namespace 为进程提供不同的资源视图，回答“容器能看到什么”。

常见类型：

| Namespace | 作用 |
|---|---|
| PID | 隔离进程编号和进程视图 |
| NET | 隔离网络设备、IP、路由表等网络视图 |
| MNT | 隔离挂载点视图 |
| UTS | 隔离主机名和域名信息 |
| IPC | 隔离进程间通信资源 |
| USER | 隔离用户和用户组 ID 映射 |
| CGROUP | 隔离 Cgroup 根目录视图 |

### 6.2 Cgroups：资源控制

Cgroups（Control Groups）用于限制、统计和隔离进程组的资源使用，回答“容器能使用多少”。

常见控制对象包括 CPU、内存、块设备 IO 和进程数量（PIDs）。例如，可以为容器设置 512MB 内存上限；内存使用超过限制时，可能触发 OOM 处理。

简单区分：

- Namespace：资源隔离，控制“看见什么”。
- Cgroups：资源控制，控制“使用多少”。

### 6.3 UnionFS、镜像分层和可写层

Docker 镜像通常由多个只读层组成。分层可以复用公共文件层，减少重复存储，并提升镜像构建和分发效率。

容器启动后，在镜像只读层之上拥有自己的可写层。容器修改文件时，写入发生在容器自己的可写层，不会直接修改底层镜像，也不会影响使用同一镜像的其他容器。具体读写行为由底层存储驱动实现，常见机制包括 Copy-on-Write（写时复制）。

删除容器时，其可写层默认随容器删除。如果数据需要在容器删除后继续保留，应使用 Volume 或 Bind Mount（后续阶段学习）。

## 7. Docker Engine 组件

- **Docker Engine**：Docker 的核心引擎体系，不等同于单独一个进程。
- **Docker CLI**：用户使用的 `docker` 命令行客户端。
- **dockerd**：Docker Daemon，负责接收 API 请求并管理 Docker 资源。
- **containerd**：容器运行时守护进程，负责底层容器生命周期等工作。
- **runc**：OCI Runtime，负责创建和启动容器进程。
- **containerd-shim**：帮助容器进程与 containerd 管理服务解耦，使容器进程不必与某个长期运行的管理进程绑定。

## 8. Docker 安装过程

### 8.1 添加 Docker 官方 APT 仓库

本次使用 Docker 官方 Ubuntu APT 仓库，仓库配置文件为：

`/etc/apt/sources.list.d/docker.sources`

实际配置：

```text
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: resolute
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
```

APT 仓库签名密钥放置于：

`/etc/apt/keyrings/docker.asc`

实施过程中曾遇到 APT `NO_PUBKEY` 错误。排查发现 `/etc/apt/keyrings` 目录权限为 750，导致 APT 的低权限用户 `_apt` 无法遍历目录。调整目录权限为 755 后，`apt update` 成功。该问题不是公钥文件缺失或密钥过期。

### 8.2 安装 Docker Engine 及插件

安装的软件包：

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 8.3 服务管理

```bash
# 启动 Docker 并设置开机自启
sudo systemctl enable --now docker

# 查看服务状态
sudo systemctl status docker --no-pager

# 查看是否已设置开机自启
sudo systemctl is-enabled docker

# 停止、启动、重启服务（按需使用）
sudo systemctl stop docker
sudo systemctl start docker
sudo systemctl restart docker
```

实际验证结果：

- Docker 服务状态：`active (running)`
- 开机自启：`enabled`
- Docker API 正常监听 `/run/docker.sock`

### 8.4 版本和运行验证

```bash
docker version
sudo docker info
sudo docker run hello-world
```

实际执行 `sudo docker run hello-world` 后，Docker 成功拉取 `hello-world:latest`，创建并运行容器，终端输出：

```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

该结果证明 Docker CLI 能够连接 Daemon，镜像可以拉取，容器可以正常创建和执行。

## 9. daemon.json 配置

Docker Daemon 常用配置文件：

`/etc/docker/daemon.json`

本次保留了已配置的镜像加速器，并增加默认日志驱动和日志轮转选项。当前配置如下：

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

配置说明：

| 配置项 | 值 | 作用 |
|---|---|---|
| registry-mirrors | 多个镜像加速地址 | 为 Docker Hub 镜像拉取配置镜像加速入口 |
| log-driver | json-file | 使用 JSON 文件记录容器标准输出和错误输出 |
| log-opts.max-size | 10m | 单个日志文件达到指定大小后触发轮转 |
| log-opts.max-file | 3 | 最多保留 3 个日志文件 |

配置检查和应用：

```bash
# 修改前备份
sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.bak

# 检查 JSON 语法
sudo python3 -m json.tool /etc/docker/daemon.json

# 应用配置
sudo systemctl restart docker

# 检查服务状态和日志驱动
sudo systemctl is-active docker
sudo docker info
```

本次通过 `python3 -m json.tool` 检查 JSON 格式；重启 Docker 后，`docker info` 显示日志驱动为 `json-file`，三个镜像加速器仍然存在。

### 9.1 验证容器实际日志配置

新建测试容器后，通过 inspect 查看实际日志配置：

```bash
sudo docker run -d --name log-demo alpine:latest sleep 600
sudo docker inspect log-demo --format '{{json .HostConfig.LogConfig}}'
```

实际输出与预期一致：

```json
{"Type":"json-file","Config":{"max-file":"3","max-size":"10m"}}
```

说明新建容器已经采用指定的日志驱动和轮转参数。

注意：

- 这里验证的是新建容器实际采用了日志配置，并没有通过大量日志输出实际触发轮转。
- Daemon 的默认日志配置主要应用于之后创建的容器；已经存在的容器不会因为修改 daemon.json 就自动更新其日志配置，通常需要重建容器。
- 修改 daemon.json 前应备份，确认 JSON 格式正确后再重启 Docker。
- 当前配置中的镜像加速器是本次实验实际使用的配置，不代表所有镜像仓库都一定能通过这些地址加速。

## 10. PID Namespace 实验

### 10.1 启动常驻测试容器

```bash
sudo docker run -d --name ns-demo alpine:latest sleep 600
sudo docker ps
sudo docker top ns-demo
```

实际观察到容器中的 `sleep 600` 进程在宿主机进程视角下的 PID 为 `654344`。

### 10.2 对比容器和宿主机看到的 PID

```bash
# 容器内部 PID 1 的 NSpid
sudo docker exec ns-demo sh -c "grep '^NSpid:' /proc/1/status"

# 宿主机进程对应的 NSpid
sudo grep '^NSpid:' /proc/654344/status
```

实际结果：

```text
容器内部：NSpid:  1
宿主机：  NSpid:  654344  1
```

同一个进程在不同 PID Namespace 中可以拥有不同的 PID。宿主机看到的 PID 是 654344，容器内部看到的 PID 是 1。

### 10.3 对比 PID Namespace inode

```bash
sudo docker exec ns-demo readlink /proc/1/ns/pid
sudo readlink /proc/1/ns/pid
```

实际结果：

```text
容器：pid:[4026532286]
宿主机：pid:[4026531836]
```

两个 inode 不同，说明容器与宿主机处于不同的 PID Namespace。

结论：不同容器可以同时拥有 PID 为 1 的进程，因为每个容器有独立的 PID Namespace 视图；它们仍然共享宿主机 Linux 内核。

## 11. Docker 启动故障排查

Docker 服务启动异常时，先定位原因，不要直接卸载重装。

### 11.1 常用排查命令

```bash
# 查看服务状态
sudo systemctl status docker --no-pager

# 查看本次系统启动以来 Docker 服务日志
sudo journalctl -u docker -b --no-pager -n 50

# 检查 daemon.json 是否为合法 JSON
sudo python3 -m json.tool /etc/docker/daemon.json
```

### 11.2 排查思路

1. 使用 `systemctl status docker` 判断服务是否处于 active、failed 等状态。
2. 使用 `journalctl` 找到 Daemon 启动失败的具体错误。
3. 如果提示配置解析失败，检查 `daemon.json` 的括号、逗号、引号和字段格式。
4. 修复配置后再次运行 JSON 检查。
5. 重启 Docker，检查服务状态和 `docker info`，必要时再验证容器运行情况。

故障示例：

```text
failed to start daemon:
unable to configure the Docker daemon with file
/etc/docker/daemon.json:
invalid character '}' looking for beginning of object key string
```

判断：错误指向 `daemon.json` 的 JSON 语法问题。应先备份并修复配置，而不是直接卸载重装 Docker。

### 11.3 本次日志中的非致命提示

本次 Docker 启动日志中出现过 nftables 清理规则时提示对应表不存在，以及 fs-verity 能力不受支持的提示。随后日志显示：

```text
Loading containers: done.
Daemon has completed initialization
API listen on /run/docker.sock
```

且 systemd 显示服务为 `active (running)`。因此这些提示没有阻止本次 Daemon 完成启动。排障时应结合后续初始化结果和服务状态判断，不能只看到日志中出现 error 或 warning 就认定服务启动失败。

## 12. 面试问题与参考回答

### 1. Docker 是什么？解决了什么问题？

Docker 是容器化平台，可以将应用程序及其依赖打包成镜像，再通过镜像创建容器运行应用，减少不同环境之间的依赖和配置差异。

### 2. 镜像和容器有什么区别？

镜像是创建容器的只读模板，容器是镜像运行后的实例。一个镜像可以创建多个容器，每个容器拥有自己的运行状态和可写层。

### 3. Docker 和虚拟机有什么区别？

容器共享宿主机内核，主要通过 Namespace 和 Cgroups 实现隔离与资源控制；虚拟机通过 Hypervisor 虚拟化硬件，每台虚拟机有独立的操作系统内核。容器通常资源开销更小、启动更快。

### 4. Docker 整体架构是什么？

Docker Client 通过 Docker API 向 dockerd 发送请求；dockerd 管理 Docker 资源并协调 containerd；containerd 负责更底层的容器生命周期管理；runc 遵循 OCI Runtime 规范创建和启动容器进程。

### 5. Namespace 和 Cgroups 分别有什么作用？

Namespace 负责资源隔离，决定容器能看到什么；Cgroups 负责资源限制和统计，控制容器能使用多少 CPU、内存、IO 等资源。

### 6. 容器为什么可以在 Ubuntu 宿主机上运行 CentOS 用户空间？

容器镜像提供 CentOS 用户空间文件和依赖，但容器共享宿主机 Ubuntu 的 Linux 内核。需要满足架构和内核接口等兼容条件。

### 7. 宿主机内核漏洞会不会影响容器？

有可能。容器共享宿主机内核，内核漏洞可能被容器利用并影响宿主机或其他容器。应及时更新内核，并采用非 root 用户、Capabilities、seccomp、AppArmor 等措施降低风险。

### 8. Docker 镜像为什么采用分层结构？

镜像分层可以复用公共文件层，减少重复存储，并提高构建和分发效率。容器在镜像只读层之上拥有独立可写层，容器之间的修改互不影响。

## 13. 阶段验收清单

- [x] 理解 Docker、镜像、容器和 Registry 的基本概念
- [x] 理解 Docker Client、dockerd、containerd、runc 的职责
- [x] 能解释容器与虚拟机的主要区别
- [x] 理解 Namespace、Cgroups 和镜像分层的基本原理
- [x] 在 Ubuntu 26.04 上安装 Docker Engine 和相关插件
- [x] 验证 Docker 服务运行状态和开机自启
- [x] 成功运行 hello-world
- [x] 完成 PID Namespace 的 PID 和 inode 对比实验
- [x] 配置镜像加速器和 json-file 日志轮转参数
- [x] 验证新建容器实际采用日志轮转配置
- [x] 掌握 Docker Daemon 启动失败的基础排查流程
- [x] 完成五道核心面试题练习

## 14. 阶段总结与后续计划

第一阶段已完成 Docker 基础原理、安装、服务管理、基础实验、Daemon 配置和启动故障排查。

下一阶段为 **阶段 2：Docker 镜像管理**，将学习镜像分层细节、镜像拉取与查看、标签管理、历史层分析、镜像保存与加载、导出与导入、删除和清理等内容。

本阶段不提前开展第二阶段实验。
