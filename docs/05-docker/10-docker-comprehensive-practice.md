# 阶段 10：Docker 综合实战与面试总结

## 1. 阶段目标

通过一个 Nginx + Flask + Redis 综合项目，将 Docker 镜像、容器、网络、数据卷、Compose 编排、持久化、安全和故障排查知识结合起来。

本阶段主要完成：

- 盘点 Docker Engine 与现有资源，分析综合项目架构。
- 部署并验收 Nginx、Flask、Redis 三个服务。
- 验证 Redis 数据持久化、容器重建、RDB 备份和独立恢复。
- 检查容器网络、服务发现、DNS、端口连通和网络隔离。
- 进行综合故障排查，并整理 Docker 面试知识。

实验环境为 Ubuntu 虚拟机，Docker Engine / Client 版本为 29.9.3。实验过程中 Docker Swarm 处于 active 状态，集群中已有 6 个 Service 正常运行。综合项目实验与现有 Swarm 资源分开管理。

## 2. Docker 环境盘点

### 2.1 查看 Docker 版本与运行状态

```bash
sudo docker version
sudo docker info
sudo systemctl status docker
```

重点检查：

- Client 与 Server 版本。
- Docker Engine 是否正常运行。
- Storage Driver、Cgroup Driver 和安全选项。
- Swarm 当前状态。
- 容器、镜像、网络和数据卷等资源概况。

### 2.2 查看资源清单

```bash
# 容器（包括已停止容器）
sudo docker ps -a

# 镜像
sudo docker image ls

# 网络
sudo docker network ls

# 数据卷
sudo docker volume ls

# Swarm 服务与任务
sudo docker service ls
sudo docker node ls
```

资源盘点的目的是了解主机现状和识别项目依赖，不代表需要清理历史资源。执行 prune、删除网络/卷/镜像或移除 Swarm 服务前，必须先确认资源归属、依赖关系和数据备份。

## 3. 综合项目架构

### 3.1 服务组成

项目目录：

```text
/home/ubuntu/docker-lab/compose-final-project
```

三个核心服务：

| 服务 | 容器名 | 容器端口 | 宿主机发布端口 | 作用 |
|---|---|---:|---:|---|
| Nginx | final-nginx | 80 | 8080 | Web 入口与反向代理 |
| Flask | final-app | 5000 | 未发布 | 处理 HTTP 业务请求 |
| Redis | final-redis | 6379 | 未发布 | 保存页面访问计数 |

访问链路：

```text
客户端
  |
  | http://宿主机:8080
  v
Nginx (80)
  |
  | app:5000
  v
Flask (5000)
  |
  | redis:6379
  v
Redis (6379)
  |
  v
Named Volume：Redis 数据
```

Nginx 只负责接收 HTTP 请求并转发给 Flask；Flask 执行业务逻辑并访问 Redis；Redis 保存页面访问计数。Nginx 不需要直接访问 Redis。

### 3.2 网络划分

项目使用两个 Compose 自定义 bridge 网络：

| 网络 | 子网 | 服务 |
|---|---|---|
| frontend | 172.27.0.0/16 | Nginx、Flask |
| backend | 172.28.0.0/16 | Flask、Redis |

网络地址验收记录：

| 服务 | 网络 | 容器 IP |
|---|---|---|
| Nginx | frontend | 172.27.0.3 |
| Flask | frontend | 172.27.0.2 |
| Flask | backend | 172.28.0.3 |
| Redis | backend | 172.28.0.2 |

Flask 同时加入两个网络，作为前端与后端服务之间的业务连接点。Nginx 不在 backend 网络，不能直接解析或访问 Redis；Redis 也不需要加入 frontend 网络。

容器 IP 可能在重建后变化，应用之间应使用 Compose 服务名通信，不应将动态容器 IP 写死在应用配置中。

### 3.3 挂载与持久化设计

| 服务 | 挂载类型 | 挂载关系 | 用途 |
|---|---|---|---|
| Nginx | Bind Mount（只读） | 项目 Nginx 配置文件 -> 容器配置文件 | 管理反向代理配置 |
| Flask | 无 | 无 | 应用代码打包在镜像中 |
| Redis | Named Volume | `compose-final-project_redis-data` -> `/data` | 持久化 Redis 数据 |

注意区分：

- Nginx 配置使用 Bind Mount，宿主机配置文件以只读方式挂载。
- Redis 数据使用 Docker Named Volume，卷的生命周期独立于 Redis 容器。
- 后续 RDB 备份恢复实验使用 Bind Mount 将宿主机备份目录只读挂载到临时 Redis 容器；这不改变正式项目使用 Named Volume 的设计。

### 3.4 Compose 配置关键点

项目配置的关键结构如下（省略具体镜像构建细节）：

```yaml
services:
  nginx:
    image: nginx:alpine
    container_name: final-nginx
    ports:
      - "8080:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app
    networks:
      - frontend

  app:
    image: compose-final-app:v1
    container_name: final-app
    environment:
      REDIS_HOST: redis
      REDIS_PORT: "6379"
    networks:
      - frontend
      - backend

  redis:
    image: redis:alpine
    container_name: final-redis
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - backend

volumes:
  redis-data:

networks:
  frontend:
  backend:
```

配置说明：

- Nginx 将宿主机 8080 映射到容器 80。
- Flask 通过环境变量连接 Redis 服务名 `redis` 和端口 6379。
- Flask 同时连接 frontend 与 backend 网络。
- Redis 不发布宿主机端口，只允许同网络中的服务访问。
- Redis 使用命名卷挂载 `/data`，并启用 AOF。
- Redis 配置了 `redis-cli ping` 健康检查。
- Flask 没有配置独立 healthcheck；Nginx 的 `depends_on` 也不等于后端应用已经具备完整的就绪检查。

## 4. 综合部署与运行验收

### 4.1 检查配置并启动

在项目目录执行：

```bash
cd /home/ubuntu/docker-lab/compose-final-project

# 检查并展开 Compose 配置
sudo docker compose config

# 启动服务
sudo docker compose up -d

# 查看服务状态
sudo docker compose ps
```

查看容器详情：

```bash
sudo docker inspect final-nginx final-app final-redis
```

### 4.2 验证 HTTP 访问

```bash
curl -i http://127.0.0.1:8080/
```

验收结果：

- HTTP 状态码：200。
- 页面标题：`Docker Compose Final Project`。
- 页面内容：`Flask application is running!`。
- 页面展示 Redis 维护的 `Page views` 访问计数。

Nginx 作为唯一对宿主机发布 HTTP 端口的服务，Flask 与 Redis 均未发布宿主机端口。

### 4.3 验证 Redis 服务

```bash
sudo docker exec final-redis redis-cli ping
sudo docker exec final-redis redis-cli DBSIZE
sudo docker exec final-redis redis-cli GET page_views
```

实际验证中：

- Redis 返回 `PONG`。
- 数据库中存在 `page_views` 键。
- 页面访问后，`page_views` 数值递增。

### 4.4 检查 Redis 持久化状态

```bash
sudo docker exec final-redis redis-cli INFO persistence
```

重点字段：

| 字段 | 含义 | 验收情况 |
|---|---|---|
| loading | 是否正在加载持久化文件 | 0 |
| rdb_last_bgsave_status | 最近一次 RDB 后台保存结果 | ok |
| aof_enabled | 是否启用 AOF | 1 |
| aof_last_write_status | 最近一次 AOF 写入结果 | ok |

## 5. Redis 持久化与容器重建验证

### 5.1 写入测试数据

```bash
sudo docker exec final-redis redis-cli SET docker_stage10_test persistence_ok
sudo docker exec final-redis redis-cli GET docker_stage10_test
```

预期返回：

```text
OK
persistence_ok
```

### 5.2 重启容器验证

重启 Redis 容器后检查数据：

```bash
sudo docker restart final-redis

sudo docker exec final-redis redis-cli GET docker_stage10_test
sudo docker exec final-redis redis-cli GET page_views
sudo docker exec final-redis redis-cli ping
```

测试数据仍然存在，Redis 返回 PONG，健康状态正常。

### 5.3 强制重建容器验证

在项目目录执行：

```bash
sudo docker compose up -d --no-deps --force-recreate redis
sudo docker compose ps
```

重建后再次检查：

```bash
sudo docker exec final-redis redis-cli GET docker_stage10_test
sudo docker exec final-redis redis-cli GET page_views
sudo docker exec final-redis redis-cli INFO persistence
```

验收结果：

- Redis 容器被重新创建，容器 ID 发生变化。
- `docker_stage10_test` 仍为 `persistence_ok`。
- `page_views` 原有计数仍然保留。
- AOF 仍启用，最近一次写入状态为 ok。

结论：Redis 数据保存在独立的 Named Volume 中，而不是容器可写层。删除或重建容器不会自动删除命名卷中的数据。

## 6. Redis RDB 备份与独立恢复

### 6.1 触发 RDB 后台备份

```bash
sudo docker exec final-redis redis-cli BGSAVE
sudo docker exec final-redis redis-cli INFO persistence
```

检查：

- `rdb_bgsave_in_progress:0`：后台保存已经结束。
- `rdb_last_bgsave_status:ok`：最近一次 RDB 保存成功。

### 6.2 复制备份文件到宿主机

```bash
mkdir -p ~/docker-backups/redis
sudo docker cp final-redis:/data/dump.rdb ~/docker-backups/redis/
```

备份文件：

```text
/home/ubuntu/docker-backups/redis/dump.rdb
```

本次备份记录：

| 项目 | 结果 |
|---|---|
| 文件大小 | 144 bytes |
| 文件格式 | Redis RDB version 0015 |
| SHA256 | `460ad668e756b18fe45d2a0bc1e7ec14a34b1fb7bd4d76deed970ab949e806fe` |

校验命令：

```bash
ls -lh ~/docker-backups/redis/dump.rdb
sha256sum ~/docker-backups/redis/dump.rdb
```

SHA256 可用于确认备份文件在复制、传输前后是否发生变化，但校验和本身不能替代备份恢复测试。

### 6.3 备份文件权限处理

初始备份文件由容器内 root 用户复制到宿主机，文件权限为 root:root、600。临时 Redis 容器以非 root 身份读取只读挂载目录时，出现 `Permission denied`。

检查并按实验需要调整备份文件读取权限：

```bash
sudo chmod 644 ~/docker-backups/redis/dump.rdb
ls -l ~/docker-backups/redis/dump.rdb
```

权限调整应结合备份数据敏感程度和恢复容器运行用户决定。真实业务备份不应为了方便而无条件扩大读取权限，应采用受控目录、专用用户和最小权限策略。

### 6.4 使用临时 Redis 容器验证恢复

将宿主机备份目录以只读 Bind Mount 方式挂载到临时 Redis 容器的 `/data`：

```bash
-v /home/ubuntu/docker-backups/redis:/data:ro
```

这是 Bind Mount：冒号左侧为宿主机路径，右侧为容器路径，`:ro` 表示只读挂载。

恢复验证的核心是让独立 Redis 实例从备份目录加载 RDB 文件，再查询原有数据。实验中临时恢复容器名称为 `redis-restore-test`。

验证命令：

```bash
sudo docker exec redis-restore-test redis-cli GET docker_stage10_test
sudo docker exec redis-restore-test redis-cli GET page_views
```

恢复结果：

```text
docker_stage10_test -> persistence_ok
page_views          -> 7
```

独立恢复实例成功读取两个原有键，说明 RDB 备份文件可用于数据恢复。

注意事项：

- 恢复验证应使用独立实例和独立数据目录，避免覆盖正在运行的正式 Redis 数据。
- 生产恢复前应确认 Redis 版本兼容性、数据目录权限、备份完整性和恢复时间窗口。
- RDB 备份与 AOF 是不同的持久化方式；RDB 适合生成数据快照，AOF 记录写操作日志。实际生产环境可根据数据丢失容忍度组合使用。
- 备份文件应保存到与业务主机故障域不同的位置，并定期进行恢复演练。

## 7. Docker 网络与服务发现排查

### 7.1 查看容器网络归属

```bash
sudo docker inspect final-nginx final-app final-redis \
  --format '{{.Name}} -> {{range $name, $config := .NetworkSettings.Networks}}{{$name}} {{end}}'
```

验收结果：

- Nginx 只连接 frontend。
- Flask 同时连接 frontend 和 backend。
- Redis 只连接 backend。

### 7.2 检查 Flask 到 Redis 的 DNS 解析

```bash
sudo docker exec final-app python -c \
  "import socket; print(socket.gethostbyname('redis'))"
```

实验解析到 backend 网络中的 Redis 地址 `172.28.0.2`。

Docker 自定义网络内置 DNS 可根据服务名解析容器地址。应用应优先使用服务名，而不是将容器 IP 写死。

### 7.3 检查 Flask 到 Redis 的 TCP 连通性

```bash
sudo docker exec final-app python -c \
  "import socket; s=socket.create_connection(('redis',6379),timeout=3); print('Redis TCP连接成功'); s.close()"
```

结果：Redis TCP 6379 端口连接成功。

### 7.4 检查 Nginx 到 Flask 的服务发现

```bash
sudo docker exec final-nginx getent hosts app
```

结果：Nginx 能够解析 frontend 网络中的 Flask 服务名 `app`。

### 7.5 验证网络隔离

```bash
sudo docker exec final-nginx sh -c \
  'getent hosts redis || echo "DNS解析失败：Nginx无法发现Redis"'
```

结果：

```text
DNS解析失败：Nginx无法发现Redis
```

这是预期的隔离结果，不是故障。Nginx 没有加入 backend 网络，因此无法通过该网络发现 Redis。Flask 同时加入两个网络，分别连接 Nginx 和 Redis。

网络隔离的价值：

- 限制服务之间不必要的直接访问。
- 缩小服务暴露面。
- 将入口层与数据层分离。
- 让业务应用承担明确的服务调用职责。

## 8. 容器外网连通性检查

使用独立 Alpine 容器检查默认 bridge 网络的外网访问，不修改综合项目的网络配置。

检查容器网络信息：

```bash
sudo docker inspect network-debug
sudo docker exec network-debug ip route
sudo docker exec network-debug cat /etc/resolv.conf
```

验收记录：

- 容器使用默认 bridge 网络。
- 容器 IP：`172.17.0.13`。
- 默认网关：`172.17.0.1`。
- 默认路由指向 `172.17.0.1`。
- DNS 配置包含 `183.60.83.19` 和 `183.60.82.98`。

### 8.1 外网 IP 连通性

```bash
sudo docker exec network-debug ping -c 4 -W 3 223.5.5.5
```

结果：4 个数据包全部收到，丢包率 0%。

### 8.2 DNS 解析

```bash
sudo docker exec network-debug nslookup www.baidu.com
```

结果：域名解析成功，返回 A/AAAA 记录。

### 8.3 HTTP 与 HTTPS 访问

```bash
sudo docker exec network-debug wget -S -O /dev/null http://www.baidu.com
sudo docker exec network-debug wget -S -O /dev/null https://www.baidu.com
```

结果：HTTP 和 HTTPS 请求均返回 `200 OK`。

结论：本次检查中，默认 bridge 网络的路由、DNS、外网 IP 连通性以及 HTTP/HTTPS 访问均正常。

## 9. 综合故障排查

### 9.1 基础健康检查

```bash
# Compose 服务状态
sudo docker compose ps

# HTTP 状态码
curl -s -o /dev/null -w 'HTTP状态码: %{http_code}\n' http://127.0.0.1:8080/

# Redis 连通性
sudo docker exec final-redis redis-cli ping

# 最近日志
sudo docker logs --tail 50 final-nginx 2>&1
sudo docker logs --tail 50 final-app 2>&1
sudo docker logs --tail 50 final-redis 2>&1
```

实际检查结果：

- 三个核心容器均处于运行状态。
- Redis 健康检查为 healthy。
- HTTP 访问返回 200。
- Redis 返回 PONG。
- Nginx 日志曾出现历史 502：`connect() failed (113: Host is unreachable)`，之后访问恢复为 200。
- Flask 日志提示当前使用 Flask development server。该服务适合学习和开发验证，生产部署应使用 Gunicorn 等 WSGI Server。
- Redis 日志提示当前未设置认证及网络限制。实验中 Redis 没有发布宿主机端口，且仅连接 backend 网络；生产环境仍应根据访问边界配置认证、网络策略和访问控制。

### 9.2 典型排查命令

```bash
# 容器状态（包含已退出容器）
sudo docker ps -a

# 容器详细状态、退出码、网络和挂载
sudo docker inspect <CONTAINER_NAME>

# 查看日志
sudo docker logs --tail 100 <CONTAINER_NAME>
sudo docker logs -f <CONTAINER_NAME>

# 查看端口映射
sudo docker port <CONTAINER_NAME>

# 查看网络
sudo docker network ls
sudo docker network inspect <NETWORK_NAME>

# 查看实时资源
sudo docker stats --no-stream

# 查看容器内部进程
sudo docker top <CONTAINER_NAME>

# 进入容器
sudo docker exec -it <CONTAINER_NAME> sh
```

### 9.3 网站无法访问的分层排查流程

1. **确认容器状态**：使用 `docker compose ps`、`docker ps -a` 检查 Nginx、应用和数据库容器是否运行，是否存在反复重启或退出。
2. **查看日志**：分别查看入口代理、应用和数据库日志，先寻找明确错误信息。
3. **检查入口端口**：核对宿主机端口映射、容器监听端口和应用监听地址。
4. **检查网络与 DNS**：确认相关容器加入正确网络，服务名解析正常。
5. **检查 TCP 连通性**：从调用方容器测试目标服务端口是否可连接。
6. **检查应用健康状态**：确认应用进程正常、健康检查通过，并能处理本机请求。
7. **检查数据库依赖**：确认数据库服务运行、端口可达、认证信息和连接参数正确。
8. **修复并复测**：根据证据修复对应配置或服务，再从用户访问入口验证 HTTP 状态码和页面功能。

常见命令组合：

```bash
sudo docker compose ps
sudo docker compose logs --tail=100 nginx app redis
curl -v http://127.0.0.1:8080/
sudo docker inspect final-nginx final-app final-redis
sudo docker network inspect compose-final-project_frontend
sudo docker network inspect compose-final-project_backend
sudo docker exec final-app getent hosts redis
sudo docker exec final-app sh -c 'nc -zv redis 6379'
sudo docker exec final-redis redis-cli ping
```

其中 `nc` 需要镜像中存在该工具；若没有，可使用 Python socket 等方式检查 TCP 连接。

## 10. 生产部署优化点

以下是基于实验观察到的改进方向，不代表已经在本次实验中实施：

| 项目 | 当前实验配置 | 生产环境建议 |
|---|---|---|
| Flask 运行方式 | Flask development server | 使用 Gunicorn 等 WSGI Server |
| Flask 健康检查 | 未配置独立 healthcheck | 配置能反映应用就绪状态的健康检查 |
| Redis 认证 | 未配置密码认证 | 按业务边界配置认证和访问控制 |
| Redis 网络 | 仅连接 backend，不发布宿主机端口 | 保持数据层网络隔离，限制可访问服务 |
| 日志 | 使用 Docker logs 查看 | 配置日志轮转、集中采集和保留策略 |
| 备份 | RDB 复制到宿主机目录并恢复验证 | 异机/异地域保存、自动化备份、定期恢复演练 |
| 镜像 | 使用实验镜像 Tag | 固定版本或 Digest，扫描漏洞并维护更新流程 |
| 资源限制 | 综合项目未为所有服务配置限制 | 根据监控数据设置 CPU、内存和进程限制 |

## 11. Docker 面试知识复盘

### 11.1 Docker 架构与运行流程

**Docker 架构：**

- Docker Client：接收用户的 Docker CLI/API 请求。
- Docker Daemon（dockerd）：管理镜像、容器、网络、卷等 Docker 对象。
- containerd：负责容器生命周期管理等底层工作。
- runc：依据 OCI Runtime Specification 创建和运行容器进程。
- OCI：开放容器标准，包括镜像规范和运行时规范等。

**执行 `docker run` 的基本流程：**

1. Docker Client 将请求发送给 Docker Daemon。
2. Docker 检查本地镜像；不存在时按配置从 Registry 拉取。
3. dockerd 协调 containerd 准备容器生命周期。
4. containerd 调用 OCI 兼容运行时（通常为 runc）创建容器进程。
5. 容器进程启动并按配置运行。

Docker CLI 本身不是容器运行时；`docker run` 是创建并启动容器的命令。

### 11.2 容器与虚拟机

| 对比 | 容器 | 虚拟机 |
|---|---|---|
| 内核 | 与宿主机共享内核 | 通常运行独立 Guest OS 内核 |
| 隔离方式 | Namespace、Cgroups、Capabilities 等 | Hypervisor 提供硬件虚拟化 |
| 启动速度 | 通常较快 | 通常需要启动 Guest OS |
| 资源开销 | 相对较小 | 通常较大 |

容器不是完整虚拟机，隔离边界与虚拟机不同。

### 11.3 Namespace 与 Cgroups

- Namespace：隔离进程看到的资源视图，例如 PID、网络、挂载点、主机名和用户 ID。
- Cgroups：限制、统计和控制进程组资源，例如 CPU、内存、PIDs 和 I/O。

记忆：Namespace 主要解决“能看到什么”，Cgroups 主要解决“能使用多少”。

### 11.4 容器无法连接 Redis

建议按顺序检查：

```bash
sudo docker ps -a
sudo docker logs <APP_CONTAINER>
sudo docker inspect <APP_CONTAINER>
sudo docker network inspect <NETWORK_NAME>
sudo docker exec <APP_CONTAINER> getent hosts redis
sudo docker exec <APP_CONTAINER> nc -zv redis 6379
sudo docker exec <REDIS_CONTAINER> redis-cli ping
```

重点检查容器状态、网络归属、服务名 DNS、目标端口、Redis 监听和认证配置。

### 11.5 Volume 与容器生命周期

容器可写层属于容器自身，容器删除后不会作为独立持久化存储保留。Named Volume 生命周期独立于容器，容器删除后卷通常仍存在，可以挂载到新容器继续使用。

- `docker rm` 默认不会删除关联命名卷。
- `docker compose down` 默认保留命名卷。
- `docker compose down -v` 会删除 Compose 管理的命名卷，操作前必须确认数据用途和备份情况。

### 11.6 Docker Compose

Compose 使用 YAML 定义多个服务、网络、卷、环境变量和依赖关系，并通过统一命令管理应用生命周期。

常用命令：

```bash
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs -f
sudo docker compose exec app sh
sudo docker compose stop
sudo docker compose start
sudo docker compose down
```

`depends_on` 用于声明服务依赖和启动顺序；配合支持的健康条件可以等待依赖服务通过健康检查。健康检查需要真实反映服务是否可用。

### 11.7 Dockerfile 的 COPY 与 ADD

- COPY：将构建上下文中的文件或目录复制到镜像。
- ADD：除复制外，还支持自动解压本地 tar 归档，并具有额外的来源处理行为。
- 常规文件复制优先使用 COPY，使 Dockerfile 意图更明确。

### 11.8 构建缓存优化

将变化较少的依赖清单放在前面复制并安装依赖，再复制经常变化的应用代码：

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
```

这样仅修改应用代码时，依赖安装层通常可以复用缓存。

### 11.9 Docker 网络模式

- Bridge：默认常用模式，容器加入桥接网络，可通过端口映射向宿主机发布服务。
- Host：容器共享宿主机网络命名空间，网络隔离较少，常规 `-p` 端口映射不生效。
- None：不配置常规外部网络连接，适用于不需要网络的容器场景。
- 自定义 Bridge：支持容器间服务名 DNS 解析，并便于按服务划分网络。

### 11.10 Swarm Manager 与 Worker

- Manager：维护集群状态、调度任务和维护 Raft 状态；默认也可以运行工作负载。
- Worker：执行 Manager 分配的 Task，不参与 Raft 管理状态投票。
- Raft：用于 Manager 之间维护一致的集群状态。

生产环境常见 3 或 5 个 Manager，以便在部分节点故障时仍有多数派仲裁。

**Worker 节点宕机：** Manager 检测到节点不可用后，会尝试在其他可用节点重新调度 Task，以恢复 Service 的期望副本数。它不是把原容器原样迁移到另一台主机；如果其他节点资源不足或约束不满足，副本无法立即恢复。

### 11.11 容器启动后立即退出

排查：

```bash
sudo docker ps -a
sudo docker logs <CONTAINER_NAME>
sudo docker inspect <CONTAINER_NAME> --format '{{.State.ExitCode}} {{.State.Error}}'
sudo docker inspect <CONTAINER_NAME> --format '{{json .Config.Entrypoint}} {{json .Config.Cmd}}'
```

检查启动命令、ENTRYPOINT/CMD、环境变量、挂载路径、文件权限和应用日志。容器主进程退出后，容器通常也会退出。

### 11.12 Docker 安全加固

常见措施：

- 使用非 root 用户。
- 删除不需要的 Capabilities。
- 使用只读根文件系统，并为必要临时目录提供 tmpfs。
- 配置 Seccomp 系统调用过滤和 AppArmor 访问控制。
- 使用 `no-new-privileges` 禁止进程通过 setuid/setgid 等方式获得额外权限。
- 设置 CPU、内存和 PID 限制。
- 使用可信镜像，定期漏洞扫描。
- 使用 Secrets 管理敏感数据，避免将秘密写入镜像。

区别：

| 机制 | 作用 |
|---|---|
| Capabilities | 控制进程拥有哪些细粒度系统权限 |
| Seccomp | 过滤进程可使用的系统调用 |
| AppArmor | 限制进程对文件和系统资源的访问行为 |
| Cgroups | 限制和统计资源使用 |

### 11.13 综合项目介绍

面试回答示例：

> 我做过一个基于 Docker Compose 的 Nginx、Flask 和 Redis 综合部署项目。Nginx 作为入口，通过反向代理把请求转发给 Flask；Flask 负责处理业务逻辑，并连接 Redis 保存页面访问次数。
>
> 网络上划分了 frontend 和 backend 两个网络，Nginx 和 Flask 在 frontend，Flask 和 Redis 在 backend，Flask 同时连接两个网络。Redis 使用命名卷保存数据。
>
> 后续我验证了 Redis 容器重启和强制重建后的数据保留，也将 RDB 文件复制到宿主机，并使用独立 Redis 容器完成恢复验证。

### 11.14 Nginx 为什么不直接连接 Redis

Nginx 在项目中是 HTTP 入口和反向代理，Flask 才是业务逻辑层。让 Flask 访问 Redis，可以保持职责分离，并通过 frontend/backend 网络限制数据层的直接访问范围。

### 11.15 Flask 正常但 Nginx 返回 502

排查方向：

1. 查看 Nginx 错误日志，确认上游连接失败原因。
2. 检查 Flask 容器是否运行、是否监听正确端口。
3. 检查 Nginx 和 Flask 是否连接同一网络。
4. 从 Nginx 容器解析 `app` 服务名。
5. 测试 Nginx 到 Flask 的 TCP 端口连通性。
6. 检查 Nginx upstream/proxy_pass 配置和 Flask 日志。

502 通常表示代理无法从上游获得有效响应，需要进一步根据日志定位，不能仅凭状态码认定是网络问题。

### 11.16 容器 CPU 或内存占用过高

```bash
sudo docker stats
sudo docker top <CONTAINER_NAME>
top
```

- `docker stats`：查看容器实时 CPU、内存、网络和 I/O 使用。
- `docker top`：查看容器内进程。
- `top`：查看宿主机整体资源压力。

根据业务需求评估资源限制，例如：

```bash
sudo docker update --cpus 1 --memory 512m <CONTAINER_NAME>
```

调整限制前应确认应用资源需求，避免限制过低导致服务异常。

### 11.17 镜像体积优化

- 使用适合业务的精简基础镜像。
- 使用多阶段构建，将编译环境与运行环境分离。
- 使用 `.dockerignore` 减少构建上下文。
- 合理安排 Dockerfile 指令顺序，复用缓存。
- 清理构建依赖和临时文件。
- 避免在镜像中保存不需要的工具、缓存和敏感信息。

### 11.18 Compose Web 应用无法访问的排查流程

从入口向后端逐层排查：

1. 检查容器状态和退出原因。
2. 查看 Nginx、应用、数据库日志。
3. 检查宿主机端口映射和容器内部监听。
4. 检查容器网络、DNS 服务发现和 TCP 连通性。
5. 检查应用自身健康状态。
6. 检查应用与数据库的连接。
7. 修复后从用户访问入口重新验证。

## 12. 阶段验收清单

- [x] 检查 Docker Engine、Swarm 状态及现有资源。
- [x] 梳理 Nginx、Flask、Redis 服务架构。
- [x] 验证 Compose 服务状态和 HTTP 访问。
- [x] 验证 Redis PING、访问计数和持久化状态。
- [x] 验证 Redis 容器重启后的数据保留。
- [x] 验证 Redis 强制重建后的数据保留。
- [x] 执行 RDB 后台备份并复制到宿主机。
- [x] 校验备份文件格式、大小和 SHA256。
- [x] 使用独立 Redis 容器完成 RDB 恢复验证。
- [x] 检查 Nginx、Flask、Redis 的网络归属。
- [x] 验证服务名 DNS 解析与 TCP 连通性。
- [x] 验证 frontend/backend 网络隔离。
- [x] 验证默认 bridge 网络外网、DNS、HTTP/HTTPS 访问。
- [x] 完成综合故障排查和面试知识复盘。

## 13. 阶段总结

本阶段通过 Nginx + Flask + Redis 综合项目，将 Compose 编排、服务发现、网络隔离、数据持久化和故障排查结合起来。

关键结论：

1. 容器与数据卷的生命周期不同，数据库数据应保存在独立持久化存储中。
2. Named Volume 适合管理 Redis 数据；Bind Mount 适合挂载宿主机指定配置或备份目录。
3. Compose 自定义网络支持服务名 DNS，服务应使用服务名而不是固定容器 IP 通信。
4. 多网络设计可以将入口层和数据层隔离，减少不必要的访问路径。
5. 备份是否有效，最终需要通过独立恢复验证，而不只是确认文件存在。
6. 故障排查应从用户访问入口开始，结合状态、日志、端口、网络、DNS、应用和数据库逐层定位。
7. 容器安全需要遵循最小权限原则，并结合非 root、Capabilities、Seccomp、AppArmor、只读文件系统和资源限制等机制。

阶段 10 主题：**Docker 综合实战与面试总结**。
