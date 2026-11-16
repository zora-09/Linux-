# 阶段 8：Docker Swarm 集群编排

## 1. 学习目标

本阶段学习 Docker Swarm 的集群编排能力，理解 Manager 与 Worker 节点职责，掌握集群初始化、节点加入与维护、Service 部署与扩缩容、Overlay 网络、服务发现、滚动更新、回滚和常见故障排查。

阶段重点：
- 区分单机 Docker 容器管理与 Swarm 集群服务编排。
- 掌握 Swarm 集群初始化和节点加入流程。
- 使用 Service 管理副本，而不是逐个手工创建容器。
- 理解 Routing Mesh、Overlay 网络和内置服务发现。
- 使用 Stack 文件部署多服务应用。
- 完成扩缩容、滚动更新、回滚和节点维护操作。

## 2. Docker Swarm 基础概念

Docker Swarm 是 Docker Engine 内置的集群管理与编排功能。多个 Docker Engine 可以组成一个 Swarm 集群，由 Manager 节点维护集群状态并调度任务，Worker 节点执行任务。

核心对象：

| 对象 | 作用 |
|---|---|
| Node | 集群中的 Docker Engine 主机 |
| Manager | 管理集群状态、调度任务、维护 Raft 状态 |
| Worker | 接收并运行 Manager 分配的任务 |
| Service | 声明式服务，定义镜像、副本数、网络、端口和更新策略 |
| Task | Service 的一个运行任务，通常对应一个容器 |
| Stack | 由 Compose 格式文件定义的一组 Swarm 服务 |
| Overlay Network | 连接不同 Swarm 节点上服务任务的网络 |

### Manager 与 Worker

- Manager 负责集群管理和任务调度，也可以运行工作负载。
- Worker 负责执行分配到本节点的任务，不参与 Raft 管理状态的投票。
- 生产环境通常部署奇数个 Manager，以提高 Raft 仲裁可用性。
- 三个 Manager 可容忍一个 Manager 故障；五个 Manager 可容忍两个 Manager 故障。
- Manager 数量不是越多越好，通常使用 3 或 5 个。

### Swarm 高可用与 Raft

Manager 使用 Raft 共识机制维护一致的集群状态。集群需要超过半数 Manager 可用才能维持管理操作：
- 1 个 Manager：需要 1 个可用。
- 3 个 Manager：需要至少 2 个可用。
- 5 个 Manager：需要至少 3 个可用。

失去仲裁时，已有任务可能继续运行，但集群无法正常完成管理状态变更和调度操作。

## 3. 集群通信端口

节点之间需要允许以下 TCP/UDP 端口通信：

| 端口 | 协议 | 用途 |
|---|---|---|
| 2377 | TCP | Manager 集群管理通信 |
| 7946 | TCP/UDP | 节点发现与通信 |
| 4789 | UDP | Overlay 网络 VXLAN 数据流量 |

注意：
- 这些端口应仅在可信的集群节点网络之间开放。
- 不要将集群管理端口无条件暴露到公网。
- 如果使用 UFW 或云安全组，必须按实际节点私网地址和网络拓扑配置规则。
- Overlay 网络的数据通道通常使用 UDP 4789。

## 4. 初始化 Swarm 集群

### 4.1 检查 Docker

在准备作为 Manager 的节点上执行：

```bash
docker version
docker info
systemctl status docker
```

确认 Docker Engine 正常运行，并确认节点之间网络互通。

### 4.2 初始化 Manager

指定 Manager 节点用于集群通信的可达 IP：

```bash
docker swarm init --advertise-addr <MANAGER_PRIVATE_IP>
```

例如：

```bash
docker swarm init --advertise-addr 192.168.10.11
```

命令成功后会输出 Worker 加入命令及 Token。加入命令包含集群地址和身份 Token，属于敏感信息，不要公开提交到 Git 仓库或聊天记录中。

查看集群状态：

```bash
docker info --format '{{.Swarm.LocalNodeState}}'
docker node ls
```

预期本节点为 Leader，状态为 Ready，Availability 为 Active。

### 4.3 获取加入 Token

在 Manager 节点执行：

```bash
# 获取 Worker 加入命令
docker swarm join-token worker

# 获取 Manager 加入命令
docker swarm join-token manager
```

Token 可轮换：

```bash
docker swarm join-token --rotate worker
docker swarm join-token --rotate manager
```

轮换后旧 Token 不再用于新节点加入。不要把真实 Token 写入版本库。

### 4.4 Worker 加入集群

在 Worker 节点执行 Manager 输出的加入命令：

```bash
docker swarm join --token <WORKER_TOKEN> <MANAGER_PRIVATE_IP>:2377
```

回到 Manager：

```bash
docker node ls
```

应看到 Manager 和 Worker 节点均为 Ready。

### 4.5 单机实验说明

只有一台 Linux 虚拟机时，可以先初始化单节点 Swarm，学习 Service、Stack、扩缩容、更新和回滚。单节点无法验证跨主机调度和 Overlay 跨节点数据通信；这些功能需要至少两台可互通的 Docker 主机。

## 5. 节点管理

### 查看节点信息

在 Manager 执行：

```bash
docker node ls
docker node inspect <NODE_ID>
docker node inspect self
```

### 节点 Availability

Availability 常见值：

| 状态 | 含义 |
|---|---|
| Active | 节点可接收新任务 |
| Pause | 不接收新任务，已有任务继续运行 |
| Drain | 不接收新任务，并将现有 Swarm 任务迁移到其他可用节点 |

将节点设为 Drain：

```bash
docker node update --availability drain <NODE_ID>
```

维护完成后恢复：

```bash
docker node update --availability active <NODE_ID>
```

### 节点标签

给节点添加标签：

```bash
docker node update --label-add role=web <NODE_ID>
docker node update --label-add storage=ssd <NODE_ID>
```

查看节点标签：

```bash
docker node inspect <NODE_ID> --pretty
```

服务可以通过 placement constraint 指定运行节点：

```yaml
deploy:
  placement:
    constraints:
      - node.labels.role == web
```

### 节点离开和删除

Worker 主机上主动离开：

```bash
docker swarm leave
```

Manager 上查看节点后删除已离开的节点记录：

```bash
docker node rm <NODE_ID>
```

强制移除节点只应在节点不可恢复或已确认需要清理时使用：

```bash
docker node rm --force <NODE_ID>
```

最后一个 Manager 离开前必须谨慎规划，否则集群管理状态将丢失。

## 6. Service 服务管理

Swarm 使用 Service 声明期望状态，Manager 根据期望副本数调度 Task。

### 6.1 创建 Service

```bash
docker service create \
  --name web \
  --replicas 3 \
  --publish published=8080,target=80 \
  nginx:alpine
```

查看服务：

```bash
docker service ls
docker service ps web
docker service inspect web
docker service logs web
```

说明：
- `--name`：服务名称。
- `--replicas`：期望运行的任务副本数。
- `--publish`：发布服务端口。
- `target`：容器内服务端口。
- `published`：对外发布端口。

### 6.2 扩容与缩容

扩容为 5 个副本：

```bash
docker service scale web=5
docker service ps web
```

缩容为 2 个副本：

```bash
docker service scale web=2
```

也可以更新副本数：

```bash
docker service update --replicas 4 web
```

Swarm 会持续协调实际 Task 数量，使其接近期望副本数。

### 6.3 Replicated 与 Global 模式

- Replicated：运行指定数量的副本，默认模式。
- Global：每个符合调度条件的节点运行一个 Task，适合节点级代理或监控 Agent。

Replicated 示例：

```bash
docker service create --name web --replicas 3 nginx:alpine
```

Global 示例：

```bash
docker service create --name node-agent --mode global nginx:alpine
```

## 7. Routing Mesh 与服务发布

Swarm 默认使用 Routing Mesh。服务端口发布后，集群节点可接收该 published port 的连接，再将流量转发到服务 Task，即使该节点本身没有运行对应 Task。

例如服务发布 `8080:80`，客户端可以访问任意可达 Swarm 节点的 8080 端口，由 Routing Mesh 将流量转发到服务副本。

两种发布模式：

| 模式 | 行为 |
|---|---|
| ingress | 默认 Routing Mesh，集群节点端口均可接收流量 |
| host | 仅在运行该 Task 的节点上发布端口，不经 Routing Mesh |

host 模式示例：

```bash
docker service create \
  --name web-host \
  --publish mode=host,published=8081,target=80 \
  --replicas 2 \
  nginx:alpine
```

host 模式下，同一节点不能让多个 Task 占用相同的宿主机端口，因此副本调度和端口设计需要特别注意。

## 8. Overlay 网络与服务发现

### 8.1 创建 Overlay 网络

在 Manager 执行：

```bash
docker network create \
  --driver overlay \
  --attachable \
  app-overlay
```

- Overlay 网络用于连接不同 Swarm 节点上的容器和服务。
- `--attachable` 允许普通容器手动加入该网络，便于调试。
- 服务加入同一 Overlay 网络后，可通过服务名访问彼此。
- Overlay 网络依赖节点间的网络连通性及所需端口。

查看网络：

```bash
docker network ls
docker network inspect app-overlay
```

### 8.2 使用 Overlay 网络部署服务

```bash
docker service create \
  --name web \
  --network app-overlay \
  --replicas 3 \
  nginx:alpine
```

另一个服务加入同一网络后，可以通过服务 DNS 名称访问目标服务。

Swarm 的服务发现通过内置 DNS 提供服务名解析；VIP 模式下，服务名通常解析到服务 VIP，再由 Swarm 将连接分发到后端 Task。DNSRR 模式则返回任务地址列表，客户端需要自行处理负载分配。

### 8.3 VIP 与 DNSRR

```bash
docker service create \
  --name api \
  --network app-overlay \
  --endpoint-mode vip \
  --replicas 3 \
  nginx:alpine
```

DNSRR 示例：

```bash
docker service create \
  --name api-dnsrr \
  --network app-overlay \
  --endpoint-mode dnsrr \
  --replicas 3 \
  nginx:alpine
```

- VIP：使用虚拟 IP，由 Swarm 进行服务负载分发。
- DNSRR：DNS 返回任务地址，通常由客户端或外部负载均衡器负责分发。
- DNSRR 与某些端口发布模式组合时存在限制，部署前应检查服务配置要求。

## 9. 使用 Stack 部署多服务应用

Stack 使用 Compose 格式的 YAML 文件描述一组 Swarm 服务，通过 `docker stack deploy` 部署。Stack 文件中的 `deploy` 配置用于描述副本、更新策略、重启策略和调度约束。

示例 `stack.yml`：

```yaml
services:
  web:
    image: nginx:1.27-alpine
    ports:
      - target: 80
        published: 8080
        protocol: tcp
        mode: ingress
    networks:
      - frontend
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 5s
      restart_policy:
        condition: on-failure

  redis:
    image: redis:7-alpine
    networks:
      - backend
    volumes:
      - redis-data:/data
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay

volumes:
  redis-data:
```

部署：

```bash
docker stack deploy -c stack.yml demo
```

查看：

```bash
docker stack ls
docker stack services demo
docker stack ps demo
docker service ls
```

删除 Stack：

```bash
docker stack rm demo
```

注意：
- Stack 部署时会根据文件创建服务、网络和相关资源。
- Stack 文件应使用适用于 Swarm 的配置；并非所有 Compose 字段都被 Swarm 部署支持。
- Stack 部署使用的镜像需要能被各节点获取。多节点环境应使用所有节点均可访问的镜像仓库，或确保镜像已分发。
- 使用外部网络、外部卷或持久化数据时，需明确其生命周期和清理方式。
- Stack 删除会删除 Stack 管理的服务和网络等资源；命名卷的数据生命周期应单独确认，不能把删除 Stack 当作数据备份手段。

## 10. 滚动更新与回滚

### 10.1 更新镜像

```bash
docker service update --image nginx:1.27-alpine web
```

查看更新进度：

```bash
docker service ps web
docker service inspect web
```

### 10.2 控制滚动更新策略

```bash
docker service update \
  --update-parallelism 1 \
  --update-delay 10s \
  --update-order start-first \
  web
```

常见配置：
- parallelism：每批更新的 Task 数量。
- delay：批次之间的等待时间。
- order：start-first 先启动新 Task，再停止旧 Task；stop-first 先停止旧 Task，再启动新 Task。
- failure_action：更新失败时的处理策略，如 pause 或 rollback。
- monitor：观察新 Task 的时间窗口。

### 10.3 回滚

```bash
docker service update --rollback web
```

查看回滚结果：

```bash
docker service ps web
docker service inspect web
```

回滚恢复到服务上一次配置，不代表回滚应用数据库结构或外部数据。数据库变更需要单独设计兼容性和恢复方案。

## 11. 健康检查、重启策略与资源限制

Service 可以配置重启策略、资源限制和健康检查。容器健康检查用于反映应用健康状态；Swarm 根据任务状态和重启策略管理失败任务。

Stack YAML 示例：

```yaml
services:
  app:
    image: nginx:alpine
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "-", "http://127.0.0.1/"]
      interval: 10s
      timeout: 3s
      retries: 3
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: "0.50"
          memory: 256M
        reservations:
          cpus: "0.10"
          memory: 64M
```

- limits：限制任务可使用的最大资源。
- reservations：调度时声明预留资源。
- restart_policy：定义任务退出后的重启条件和次数。
- 健康检查命令必须存在于镜像中；示例使用 wget，若镜像不包含该工具需更换检查方式。

## 12. 故障排查

### 12.1 节点无法加入集群

检查：
```bash
docker info
systemctl status docker
ip addr
ip route
ping <MANAGER_PRIVATE_IP>
```

排查方向：
- Manager IP 是否选择正确，是否为 Worker 可达地址。
- TCP 2377 是否可达。
- 节点之间 TCP/UDP 7946 和 UDP 4789 是否按需放通。
- Token 是否正确、是否过期或已轮换。
- Docker Engine 是否运行正常。
- 主机名、时间和网络路由是否存在异常。

### 12.2 节点状态 Down 或 Unreachable

```bash
docker node ls
docker node inspect <NODE_ID> --pretty
docker info
journalctl -u docker --since "30 minutes ago"
```

检查节点 Docker 服务、主机网络、资源压力、防火墙和 Manager 可达性。

### 12.3 Service 副本不足

```bash
docker service ls
docker service ps <SERVICE_NAME> --no-trunc
docker service inspect <SERVICE_NAME> --pretty
```

常见原因：
- 节点资源不足，无法满足 CPU/内存 reservations。
- placement constraint 没有符合条件的节点。
- 镜像拉取失败或镜像仓库认证失败。
- 端口冲突或挂载路径在目标节点不存在。
- 容器启动命令或应用配置错误。

### 12.4 Overlay 网络通信失败

```bash
docker network ls
docker network inspect <OVERLAY_NETWORK>
docker service ps <SERVICE_NAME>
```

检查：
- 相关服务是否连接同一个 Overlay 网络。
- 节点间 UDP 4789、TCP/UDP 7946 是否可达。
- 是否存在 MTU、路由、防火墙或安全组问题。
- 服务 DNS 名称是否正确。
- 服务 Task 是否正常运行。

### 12.5 服务更新卡住

```bash
docker service ps <SERVICE_NAME> --no-trunc
docker service inspect <SERVICE_NAME> --pretty
docker service logs <SERVICE_NAME>
```

检查新镜像是否可拉取、健康检查是否失败、资源是否足够、更新策略是否等待以及应用启动日志。

### 12.6 常用排障命令汇总

```bash
# 集群状态
docker info
docker node ls

# 节点详情
docker node inspect <NODE_ID> --pretty

# 服务列表和任务
docker service ls
docker service ps <SERVICE_NAME> --no-trunc

# 服务详情与日志
docker service inspect <SERVICE_NAME> --pretty
docker service logs <SERVICE_NAME>

# 网络
docker network ls
docker network inspect <NETWORK_NAME>

# Docker 服务日志
journalctl -u docker -f
```

## 13. 与 Docker Compose 的区别

| 对比项 | Docker Compose | Docker Swarm |
|---|---|---|
| 主要用途 | 定义和管理多容器应用 | 多节点集群编排 |
| 管理范围 | 通常为单台 Docker 主机 | 多台 Docker 主机组成的集群 |
| 部署对象 | Container/Service | Swarm Service/Task |
| 扩容 | 在单机上增加服务实例 | 在集群节点间调度副本 |
| 网络 | Bridge 等单机网络常见 | 支持 Overlay 跨节点网络 |
| 高可用调度 | 不提供集群级调度 | Manager 协调 Task 调度和期望状态 |
| 部署命令 | docker compose up | docker stack deploy |

Compose 适合单机多容器应用编排；Swarm 适合多节点服务编排。两者使用相似的 YAML 服务定义方式，但 Swarm 的部署配置和支持字段需要单独确认。

## 14. 阶段学习检查清单

- [ ] 理解 Swarm、Node、Manager、Worker、Service、Task、Stack。
- [ ] 理解 Raft 仲裁与 Manager 高可用数量原则。
- [ ] 掌握 Swarm 初始化、Token 获取和节点加入。
- [ ] 掌握节点查看、标签、Drain、Active 和节点移除。
- [ ] 掌握 Service 创建、查看、扩容和缩容。
- [ ] 理解 Replicated 与 Global 模式。
- [ ] 理解 Routing Mesh、ingress 与 host 发布模式。
- [ ] 掌握 Overlay 网络和服务名发现。
- [ ] 理解 VIP 与 DNSRR。
- [ ] 掌握 Stack YAML 部署和删除。
- [ ] 掌握滚动更新策略与回滚。
- [ ] 理解健康检查、重启策略和资源限制。
- [ ] 能够排查节点加入、服务副本不足、Overlay 网络和更新失败。

## 15. 阶段面试题

### 1. Docker Swarm 是什么？解决什么问题？

参考回答：Docker Swarm 是 Docker 内置的集群管理和编排工具，可以把多台 Docker 主机组成集群，通过 Manager 统一管理服务和调度任务，减少逐台主机手动部署和维护容器的工作。

### 2. Manager 和 Worker 有什么区别？

参考回答：Manager 负责维护集群状态、调度任务和管理节点；Worker 主要负责运行 Manager 分配的任务。Manager 也可以运行工作负载，但生产环境通常会根据高可用需求规划多个 Manager。

### 3. 为什么 Manager 通常使用奇数个？

参考回答：Manager 使用 Raft 机制维护一致性，需要多数节点形成仲裁。三个 Manager 可以容忍一个故障，五个可以容忍两个故障，因此常见部署会选择 3 或 5 个。

### 4. Service 和 Container 有什么区别？

参考回答：Container 是实际运行的容器实例；Service 是 Swarm 中的声明式服务对象，定义镜像、副本数、网络和发布端口等，Swarm 会根据 Service 的期望状态创建和维护 Task，每个 Task 通常对应一个容器。

### 5. docker service scale 的作用是什么？

参考回答：调整 Service 的期望副本数。例如 `docker service scale web=5` 将 web 服务调整为五个副本，Swarm 会根据节点资源和调度条件分配任务。

### 6. Overlay 网络解决什么问题？

参考回答：Overlay 网络可以把不同 Swarm 节点上的容器连接到同一个逻辑网络，使跨主机服务能够通过服务名通信，不需要应用直接维护各节点容器 IP。

### 7. Routing Mesh 是什么？

参考回答：Routing Mesh 是 Swarm 的服务端口转发机制。使用 ingress 发布端口时，集群节点都可以接收该端口的请求，再把流量转发到实际运行服务 Task 的节点。

### 8. docker stack deploy 有什么作用？

参考回答：它根据 Compose 格式的 YAML 文件，在 Swarm 集群中部署一组相关服务，可以统一定义副本、网络、端口、更新策略和资源限制等。

### 9. Swarm 如何进行滚动更新和回滚？

参考回答：可以通过 `docker service update` 更新镜像，并配置并行更新数量、批次间隔和更新顺序。如果更新失败或新版本异常，可以使用 `docker service update --rollback` 回到上一次服务配置。

### 10. Service 副本数不足时如何排查？

参考回答：先使用 `docker service ls` 和 `docker service ps --no-trunc` 查看期望副本与任务失败原因，再检查节点状态、资源预留、调度约束、镜像拉取、挂载路径和应用日志。

## 16. 阶段总结

完成本阶段后，应能够解释 Swarm 集群架构，完成基础集群部署，管理节点和 Service，使用 Overlay 网络连接跨主机服务，通过 Stack 部署多服务应用，并完成扩缩容、滚动更新、回滚和基础故障排查。

阶段 8 的主题为：**Docker Swarm 集群编排**。
