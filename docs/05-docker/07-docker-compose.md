# 阶段 7：Docker Compose 多容器编排

## 1. 阶段目标

掌握 Docker Compose 的基本使用，能够通过 Compose YAML 文件定义和管理多个关联容器，完成服务网络、数据持久化、启动依赖、环境变量、服务扩容、镜像更新及故障排查。

本阶段使用 Ubuntu 虚拟机进行实验，主要命令均通过 `docker compose` 执行。

## 2. Docker Compose 基础

Docker Compose 用于定义和运行多容器应用。通过 `compose.yaml`（或 `docker-compose.yml`）统一描述服务、网络、数据卷、端口、环境变量和依赖关系。

与逐个使用 `docker run` 相比，Compose 可以将多个服务的配置保存在一个 YAML 文件中，并通过统一命令管理整个应用。

常见结构：

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"

  app:
    image: python:3.12-slim

volumes:
  app-data:

networks:
  frontend:
```

- `services`：定义应用中的服务。
- `volumes`：声明命名卷。
- `networks`：声明网络。
- 每个服务可单独配置镜像、构建、端口、环境变量、挂载、网络和依赖关系。

## 3. 常用命令

| 命令 | 作用 |
|---|---|
| `docker compose config` | 校验并展开 Compose 配置 |
| `docker compose up -d` | 创建并后台启动服务 |
| `docker compose ps` | 查看项目服务状态 |
| `docker compose ps -a` | 查看包括已退出容器在内的服务状态 |
| `docker compose logs` | 查看服务日志 |
| `docker compose logs -f` | 实时跟踪日志 |
| `docker compose logs app` | 查看指定服务日志 |
| `docker compose stop` | 停止服务但保留容器 |
| `docker compose start` | 启动已创建的停止状态容器 |
| `docker compose restart` | 重启服务容器 |
| `docker compose down` | 停止并删除项目容器及默认网络 |
| `docker compose build` | 构建服务镜像 |
| `docker compose up -d --build` | 构建镜像并启动/更新服务 |
| `docker compose exec app sh` | 进入运行中的 app 容器 |

注意：
- 首次部署或需要根据配置创建容器时，通常使用 `up -d`；`start` 只启动已经存在的容器。
- `stop` 不删除容器；`down` 会删除 Compose 管理的容器和默认网络。
- `down` 默认保留命名卷；`down -v` 会删除关联的 Compose 命名卷，可能造成数据丢失。
- `restart` 不会因为修改了 YAML 配置而自动重建容器。配置变更后使用 `docker compose up -d` 使配置生效。

## 4. Compose 服务网络

Compose 默认会为项目创建网络。同一网络中的服务可以通过服务名互相访问，不需要手动维护容器 IP。

实验中使用两个服务验证网络通信：

```yaml
services:
  web:
    image: nginx:alpine
    networks:
      - app-net

  worker:
    image: alpine
    command: ["sh", "-c", "sleep 3600"]
    networks:
      - app-net

networks:
  app-net:
```

进入 web 容器后，通过服务名访问 worker：

```bash
docker compose exec web ping -c 2 worker
```

实验结果：容器能够通过服务名解析并访问同一 Compose 网络中的服务。容器 IP 可能在重建后变化，服务名由 Docker 内部 DNS 解析到当前容器地址。

### 多网络与网络隔离

综合项目中定义两个网络：

```yaml
networks:
  frontend:
  backend:
```

服务网络关系：

| 服务 | frontend | backend |
|---|---:|---:|
| Nginx | 是 | 否 |
| Flask app | 是 | 是 |
| Redis | 否 | 是 |

Nginx 通过 `app:5000` 访问 Flask；Flask 通过 `redis:6379` 访问 Redis。Nginx 与 Redis 不在同一个网络，不能直接通过该网络访问 Redis。

## 5. 数据卷与数据持久化

### Named Volume

命名卷由 Docker 管理，适合数据库或应用持久化数据：

```yaml
services:
  redis:
    image: redis:alpine
    volumes:
      - redis-data:/data

volumes:
  redis-data:
```

实验验证：
1. 在容器内向挂载目录写入测试文件。
2. 执行 `docker compose down` 删除容器。
3. 再次执行 `docker compose up -d`。
4. 重新进入容器读取文件，数据仍然存在。
5. 执行 `docker compose down -v` 后，项目关联命名卷被删除。

### Bind Mount

Bind Mount 将宿主机指定路径挂载到容器：

```yaml
services:
  app:
    image: alpine
    volumes:
      - ./data:/data
```

实验中，宿主机创建 `./data/test.txt`，容器内可以读取；在容器内修改文件后，宿主机对应文件内容也同步变化。

| 对比 | Named Volume | Bind Mount |
|---|---|---|
| 路径管理 | Docker 管理 | 用户指定宿主机路径 |
| 常见用途 | 数据库、持久化业务数据 | 配置文件、代码目录 |
| 容器删除后 | 默认保留 | 宿主机源文件仍存在 |
| 删除风险 | `down -v` 可删除关联命名卷 | 删除容器不会自动删除宿主机源目录 |

## 6. 服务依赖与健康检查

`depends_on` 用于声明服务启动依赖；`healthcheck` 用于检查容器内应用是否达到健康状态。

仅使用短语法 `depends_on` 通常只能控制依赖服务的启动顺序，不能保证应用已经准备就绪。

实验使用健康检查控制 app 启动：

```yaml
services:
  app:
    image: alpine
    depends_on:
      database:
        condition: service_healthy
    command: ["sh", "-c", "echo app started; sleep 3600"]

  database:
    image: alpine
    command: ["sh", "-c", "sleep 15; touch /tmp/db-ready; sleep 3600"]
    healthcheck:
      test: ["CMD-SHELL", "test -f /tmp/db-ready"]
      interval: 2s
      timeout: 1s
      retries: 10
```

实验结果：database 启动约 15 秒后健康检查通过，随后 app 才启动。

常见健康检查配置项：
- `test`：执行健康检查的命令。
- `interval`：检查间隔。
- `timeout`：单次检查超时时间。
- `retries`：连续失败次数阈值。
- `start_period`：应用启动宽限期。

`service_healthy` 可用于等待依赖服务通过健康检查。健康检查本身不是应用故障后的自动修复机制。

## 7. 环境变量与配置管理

### environment

直接在 Compose 文件中定义容器环境变量：

```yaml
services:
  app:
    image: alpine
    environment:
      APP_ENV: production
      APP_PORT: "8080"
      DB_HOST: database
```

### .env 变量插值

在 Compose 项目目录中创建 `.env`：

```dotenv
APP_ENV=development
APP_PORT=9000
```

Compose YAML 引用变量：

```yaml
services:
  app:
    image: alpine
    environment:
      APP_ENV: ${APP_ENV}
      APP_PORT: ${APP_PORT}
```

实验中容器读取到 `APP_ENV=development` 和 `APP_PORT=9000`。

注意：项目目录中的 `.env` 主要用于 Compose 文件变量插值，不代表其中所有变量会自动进入容器；需要通过 `environment` 或 `env_file` 显式传入。

### env_file

`env_file` 可将指定文件中的变量传入容器：

```yaml
services:
  app:
    image: alpine
    env_file:
      - app.env
```

配置文件中的密码、Token 等敏感信息不应提交到 Git 仓库。`.env` 文件不是加密存储；生产环境应使用适当的密钥管理方案。

## 8. 服务扩容与更新

### 扩容

实验使用 Nginx 服务：

```bash
docker compose up -d --scale web=3
docker compose ps
```

实验成功创建三个实例：`web-1`、`web-2`、`web-3`。

缩容：

```bash
docker compose up -d --scale web=2
```

注意：
- 扩容服务不要设置固定的 `container_name`，多个实例需要唯一名称。
- 不要让多个实例绑定同一个宿主机 IP 和端口，例如都发布 `8080:80`，否则会发生端口冲突。
- 需要统一对外入口时，可在前面增加反向代理或负载均衡服务。
- Compose 的 `--scale` 创建多个实例，不代表自动提供完整的负载均衡能力。

### 镜像更新与副本数

修改服务镜像版本后，可拉取镜像并重新部署：

```bash
docker compose pull
docker compose up -d
```

实验中曾将 Nginx 镜像从 `nginx:alpine` 修改为 `nginx:1.27-alpine`。未显式指定 `--scale` 时，Compose 按配置中的服务副本数收敛，原先手动扩展的三个实例可能恢复为配置定义的一个实例。需要保留两个实例时，再执行：

```bash
docker compose up -d --scale web=2
```

## 9. 故障排查实验

### 9.1 Nginx 配置语法错误

故障配置中的启动命令缺少分号，Nginx 日志提示：

```text
unexpected end of parameter, expecting ";"
```

排查命令：

```bash
docker compose ps -a
docker compose logs nginx
```

修正配置后使用正确的前台启动命令：

```yaml
command: ["nginx", "-g", "daemon off;"]
```

重新启动服务，并通过 `curl` 验证 HTTP 返回正常。

### 9.2 端口映射错误

将宿主机端口由 `8080:80` 改为 `8081:80` 后，访问旧端口失败，新端口返回 HTTP 200。

排查顺序：
1. 使用 `docker compose ps` 查看端口映射。
2. 检查容器内服务是否正常监听。
3. 在宿主机访问实际映射端口。
4. 如果本机访问正常但远端访问失败，再检查防火墙、安全组和网络策略。

### 9.3 Flask 服务停止导致 Nginx 502

综合项目中执行：

```bash
docker compose stop app
curl -I http://localhost:8080
```

Nginx 返回 `502 Bad Gateway`。原因是 Nginx 仍在运行，但上游 Flask 服务已停止。

恢复服务：

```bash
docker compose start app
docker compose ps
curl -I http://localhost:8080
```

恢复后 HTTP 返回 `200 OK`。

### 9.4 通用故障排查命令

```bash
# 检查配置
docker compose config

# 查看所有容器，包括已退出容器
docker compose ps -a

# 查看指定服务日志
docker compose logs --tail=100 app
docker compose logs -f app

# 查看容器环境变量
docker compose exec app env

# 查看 Docker 网络
docker network ls
docker network inspect <network_name>

# 进入容器检查
docker compose exec app sh

# 重新构建镜像并部署
docker compose up -d --build app
```

建议按顺序排查：先检查配置和容器状态，再查看日志，接着检查环境变量、网络、端口和挂载，定位原因后修复并验证。不要在未定位原因时直接删除整个项目。

## 10. 综合项目：Nginx + Flask + Redis

### 10.1 项目结构

项目目录为 `~/docker-lab/compose-final-project`：

```text
compose-final-project/
├── app/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
├── data/
├── nginx/
│   └── default.conf
└── compose.yaml
```

### 10.2 Flask 应用

`app/requirements.txt`：

```text
flask
redis
```

`app/app.py`：

```python
import os
import redis
from flask import Flask

app = Flask(__name__)

REDIS_HOST = os.getenv("REDIS_HOST", "redis")
REDIS_PORT = int(os.getenv("REDIS_PORT", "6379"))

r = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, decode_responses=True)

@app.route("/")
def index():
    count = r.incr("page_views")
    return f"""
    <h1>Docker Compose Final Project</h1>
    <p>Flask application is running!</p>
    <p>Page views: {count}</p>
    """

@app.route("/health")
def health():
    return "OK", 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### 10.3 Flask Dockerfile

`app/Dockerfile`：

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

构建镜像：

```bash
docker build -t compose-final-app:v1 ./app
```

### 10.4 Nginx 反向代理配置

`nginx/default.conf`：

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://app:5000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Nginx 使用 Compose 服务名 `app` 访问 Flask，两个服务需要连接到同一个网络。

### 10.5 compose.yaml

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
    depends_on:
      redis:
        condition: service_healthy
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

### 10.6 部署与验证

```bash
# 检查 Compose 配置
docker compose config

# 后台启动
docker compose up -d

# 查看服务状态
docker compose ps

# 查看服务日志
docker compose logs

# 访问应用
curl http://localhost:8080

# 检查健康接口
curl http://localhost:8080/health
```

验证结果：
- Nginx 通过宿主机 8080 端口对外提供访问。
- Nginx 将请求转发到 Flask 的 5000 端口。
- Flask 通过 Redis 服务名连接 Redis 的 6379 端口。
- Redis 使用命名卷 `redis-data` 持久化数据。
- Redis 健康检查通过后，Compose 才启动 Flask。
- Flask 不直接发布宿主机端口，Redis 也不直接发布宿主机端口。

### 10.7 数据持久化验证

访问首页多次，Redis 中的 `page_views` 计数递增。执行：

```bash
docker compose down
docker compose up -d
curl http://localhost:8080
```

重新部署后计数继续递增，说明 Redis 数据保存在命名卷中。不要使用 `down -v` 进行该验证，否则会删除命名卷。

### 10.8 服务故障与恢复验证

停止 Flask：

```bash
docker compose stop app
curl -I http://localhost:8080
```

访问返回 `502 Bad Gateway`。恢复 Flask：

```bash
docker compose start app
docker compose ps
curl -I http://localhost:8080
```

恢复后返回 `HTTP/1.1 200 OK`。

说明：本实验使用 Flask 自带开发服务器，适用于学习验证。生产部署应使用 Gunicorn 等 WSGI Server，并配置适当的进程数、日志和健康检查。

## 11. 阶段面试复盘

本阶段练习的核心问题：

1. Docker Compose 是什么？与逐个使用 `docker run` 有什么区别？
2. 如何启动、停止、查看状态、查看日志和删除 Compose 服务？
3. Compose 中服务如何通过服务名通信？内部 DNS 有什么作用？
4. Flask 无法连接 Redis 时，如何检查状态、环境变量、网络、DNS、端口和日志？
5. `depends_on` 和 `healthcheck` 有什么区别？为什么仅配置启动依赖不够？
6. Named Volume 和 Bind Mount 有什么区别？
7. 服务扩容时为什么不应设置固定 `container_name` 或相同宿主机端口映射？
8. `stop`、`down` 和 `restart` 有什么区别？
9. 修改 Dockerfile 或应用代码后，如何重新构建镜像并使变更生效？
10. Nginx、Flask、Redis 项目中 Flask 退出时，如何按顺序排查和恢复？

## 12. 阶段验收

- [x] 理解 Compose 用途及 YAML 基本结构
- [x] 掌握 Compose 常用命令
- [x] 完成服务网络与服务名 DNS 通信实验
- [x] 完成 Named Volume 与 Bind Mount 持久化实验
- [x] 完成 depends_on 与 healthcheck 实验
- [x] 完成 environment 与 .env 变量配置实验
- [x] 完成服务扩容、缩容与镜像更新实验
- [x] 完成 Nginx 配置错误和端口映射故障排查
- [x] 完成 Nginx + Flask + Redis 综合项目
- [x] 完成数据持久化、服务停止和恢复验证
- [x] 完成阶段面试问答复盘

阶段 7 已完成。下一阶段为：**阶段 8：Docker 日常运维与故障排查**。
