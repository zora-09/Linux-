# 阶段 9：Docker 安全管理

## 1. 学习目标

本阶段围绕 Docker 镜像、容器运行权限、敏感数据、配置管理和资源隔离，建立容器安全管理的基础能力。

学习目标：

- 理解容器与宿主机之间的安全边界，以及 Docker 使用的 Linux 内核安全机制。
- 掌握镜像来源、Tag、Digest、漏洞扫描和敏感信息保护。
- 掌握非 root 用户运行容器，以及 Linux Capabilities 的基本管理方式。
- 使用 Docker Secrets 管理密码、令牌和证书等敏感数据。
- 使用 Docker Configs 管理普通应用配置文件。
- 使用只读根文件系统、tmpfs、CPU、内存和 PID 限制加固容器。
- 将多项安全配置组合到一个可正常访问的 Nginx 容器中。

本阶段共分为七个模块：

1. Docker 安全基础
2. 镜像安全
3. 容器权限管理
4. Docker Secrets
5. Docker Configs
6. 容器安全限制
7. 安全综合实验

## 2. Docker 安全基础

### 2.1 容器为什么需要安全加固

Docker 容器并不是一台完整的虚拟机。容器中的进程与宿主机共享 Linux 内核，Docker 通过 Linux 内核提供的隔离和资源控制机制，限制容器进程能够看到和使用的资源。

容器安全需要重点关注：

- 容器是否以 root 身份运行。
- 容器是否拥有不必要的系统管理权限。
- 容器是否可以修改自身根文件系统。
- 容器是否受到 CPU、内存和进程数量限制。
- 镜像是否来自可信来源，是否包含已知漏洞。
- 密码、API Token、证书等敏感数据是否被写入镜像或普通环境变量。
- 容器是否能够访问不必要的宿主机资源。

容器隔离不等于绝对安全。宿主机内核、Docker Engine、镜像、挂载目录和运行权限都可能影响安全边界。

### 2.2 Namespaces：资源隔离

Namespaces 用于隔离进程能够看到的系统资源，使容器拥有相对独立的运行视图。

常见 Namespace：

| 类型 | 隔离内容 |
|---|---|
| PID | 进程 ID 和进程视图 |
| NET | 网络设备、协议栈、端口等 |
| MNT | 挂载点和文件系统视图 |
| UTS | 主机名和域名信息 |
| IPC | 进程间通信资源 |
| USER | 用户和用户组 ID 映射 |
| CGROUP | Cgroup 根目录视图 |

例如，容器内部看到的进程列表通常与宿主机不同，容器也可以拥有独立的网络命名空间。

### 2.3 Cgroups：资源控制

Cgroups（Control Groups）用于限制、统计和隔离进程组的资源使用。

常见资源控制：

- CPU：限制容器可使用的 CPU 时间。
- Memory：限制容器内存使用量。
- PIDs：限制容器可创建的进程数量。
- I/O：控制或统计块设备 I/O。

常见运行参数：

```bash
docker run -d \
  --name resource-demo \
  --memory=128m \
  --cpus=0.5 \
  --pids-limit=100 \
  nginx:alpine
```

查看资源实时使用情况：

```bash
docker stats
docker stats --no-stream resource-demo
```

查看容器资源限制：

```bash
docker inspect resource-demo \
  --format 'Memory={{.HostConfig.Memory}} NanoCpus={{.HostConfig.NanoCpus}} PidsLimit={{.HostConfig.PidsLimit}}'
```

### 2.4 Linux Capabilities：细粒度权限控制

Linux Capabilities 将传统 root 用户拥有的系统管理权限拆分成多个独立能力。容器可以只保留运行应用所需的能力，而不必拥有完整的 root 权限集合。

常用参数：

- `--cap-drop`：删除指定能力。
- `--cap-add`：添加指定能力。
- `--cap-drop ALL`：删除全部默认能力，再按需添加必要能力。

示例：

```bash
# 删除 NET_RAW 能力
docker run -d --name cap-demo \
  --cap-drop NET_RAW nginx:alpine

# 删除全部能力
docker run -d --name cap-minimal \
  --cap-drop ALL nginx:alpine

# 在确有网络管理需求时添加 NET_ADMIN
docker run -d --name net-admin-demo \
  --cap-add NET_ADMIN alpine sleep infinity
```

查看进程的 Capability 信息：

```bash
grep Cap /proc/self/status
```

常见字段：

| 字段 | 含义 |
|---|---|
| CapInh | Inheritable，继承集合 |
| CapPrm | Permitted，允许获得的能力集合 |
| CapEff | Effective，当前生效的能力集合 |
| CapBnd | Bounding，能力边界集合 |
| CapAmb | Ambient，环境能力集合 |

生产环境应遵循最小权限原则，只授予应用实际需要的能力。

### 2.5 Seccomp：系统调用过滤

Seccomp（Secure Computing Mode）用于限制进程可以使用的 Linux 系统调用，减少应用进程能够触达的内核功能范围。

Docker 默认使用内置 Seccomp 配置文件，对部分系统调用进行限制。

查看 Docker 安全选项：

```bash
docker info --format '{{json .SecurityOptions}}'
```

指定安全选项的形式：

```bash
docker run -d \
  --security-opt seccomp=/path/to/seccomp-profile.json \
  nginx:alpine
```

自定义 Seccomp 配置需要根据应用实际系统调用需求制定，配置过严可能导致应用启动或运行失败。

### 2.6 AppArmor：强制访问控制

AppArmor 是 Linux 安全模块（LSM）之一，可以通过安全配置文件限制程序对文件、目录和系统资源的访问行为。

Docker 在支持 AppArmor 的系统上通常使用默认的 `docker-default` 配置文件。

查看 Docker 安全选项：

```bash
docker info --format '{{json .SecurityOptions}}'
```

查看容器 AppArmor 配置：

```bash
docker inspect <CONTAINER_NAME> \
  --format 'AppArmorProfile={{.AppArmorProfile}}'
```

指定 AppArmor 配置文件：

```bash
docker run -d \
  --security-opt apparmor=<PROFILE_NAME> \
  nginx:alpine
```

AppArmor 配置文件需要在宿主机上预先准备并加载。

### 2.7 三种权限机制的区别

| 机制 | 主要限制对象 | 记忆方式 |
|---|---|---|
| Capabilities | 进程拥有哪些系统管理权限 | 限制拥有什么权限 |
| Seccomp | 进程能够调用哪些系统调用 | 限制能调用什么 |
| AppArmor | 进程能够访问哪些文件和资源 | 限制能访问什么 |

三者可以配合使用，分别从系统权限、系统调用和资源访问行为等角度缩小容器的攻击面。

### 2.8 查看容器安全配置

```bash
docker inspect <CONTAINER_NAME> \
  --format 'User={{.Config.User}}
Privileged={{.HostConfig.Privileged}}
ReadonlyRootfs={{.HostConfig.ReadonlyRootfs}}
CapAdd={{json .HostConfig.CapAdd}}
CapDrop={{json .HostConfig.CapDrop}}
SecurityOpt={{json .HostConfig.SecurityOpt}}
AppArmorProfile={{.AppArmorProfile}}
CgroupnsMode={{.HostConfig.CgroupnsMode}}'
```

注意：

- `docker info` 的 `.SecurityOptions` 用于查看 Docker Engine 支持或启用的安全机制。
- 容器的 `docker inspect` 中不存在同名的 `.SecurityOptions` 字段，应查看 `.AppArmorProfile`、`.HostConfig.SecurityOpt` 等字段。
- `Privileged=false` 表示容器没有使用特权模式。特权模式会显著扩大容器权限，应谨慎使用。

## 3. 镜像安全

### 3.1 使用可信镜像来源

镜像是容器运行的基础，镜像来源和内容会直接影响容器安全。

建议：

- 优先使用官方镜像或可信组织维护的镜像。
- 关注镜像维护状态、版本和更新记录。
- 尽量使用精简基础镜像，减少不必要的软件包。
- 定期更新基础镜像和应用依赖。
- 不使用来源不明的镜像运行生产业务。
- 生产环境应建立镜像版本管理和漏洞扫描流程。

### 3.2 Tag 与 Digest

Tag 是镜像标签，例如：

```bash
docker pull nginx:alpine
```

Tag 便于识别版本，但同一个 Tag 可能在后续被维护者更新，因此它不一定永久指向完全相同的镜像内容。

Digest 是镜像内容摘要，通常以 SHA256 表示，可以唯一标识特定镜像内容。

查看镜像摘要：

```bash
docker image inspect nginx:alpine \
  --format 'ID={{.Id}}
Tags={{json .RepoTags}}
Digests={{json .RepoDigests}}'
```

使用 Digest 拉取或运行镜像：

```bash
docker pull nginx@sha256:<IMAGE_DIGEST>

docker run -d \
  --name digest-demo \
  nginx@sha256:<IMAGE_DIGEST>
```

使用 Digest 可以固定镜像内容，降低 Tag 发生变化导致部署内容不一致的风险。实际使用时应从可信镜像仓库获取完整 Digest。

### 3.3 镜像漏洞扫描

镜像漏洞扫描用于检查镜像中的操作系统软件包、应用依赖和已知安全漏洞。

常见工具：

- Trivy：开源漏洞扫描工具，可扫描镜像、文件系统和代码仓库等。
- Docker Scout：Docker 提供的镜像分析和安全工具，可分析镜像组件及漏洞信息。

Trivy 常用命令：

```bash
# 扫描镜像
trivy image nginx:alpine

# 仅查看高危和严重漏洞
trivy image --severity HIGH,CRITICAL nginx:alpine

# 输出 JSON 报告
trivy image --format json -o nginx-scan.json nginx:alpine
```

Docker Scout 常见命令：

```bash
docker scout quickview nginx:alpine
docker scout cves nginx:alpine
```

漏洞等级通常包括：

| 等级 | 含义 |
|---|---|
| CRITICAL | 严重 |
| HIGH | 高危 |
| MEDIUM | 中危 |
| LOW | 低危 |
| UNKNOWN | 未知 |

发现漏洞后的处理流程：

1. 确认漏洞对应的软件包和版本。
2. 判断漏洞是否影响当前应用及运行环境。
3. 更新基础镜像或相关依赖。
4. 重新构建镜像。
5. 再次扫描并验证应用功能。
6. 将扫描结果纳入镜像发布记录。

漏洞扫描结果需要结合实际使用场景判断，不能只根据漏洞数量决定是否上线。

### 3.4 避免将敏感信息写入镜像

不要将数据库密码、API Token、私钥等敏感信息写入以下位置：

- Dockerfile 的 `ENV` 指令。
- Dockerfile 的 `ARG` 指令并将其作为构建秘密使用。
- 被 COPY 进镜像的配置文件。
- 镜像构建过程产生的文件层。
- 公开的代码仓库或构建日志。

例如，不推荐：

```dockerfile
ENV DB_PASSWORD=MyPassword123
```

镜像环境变量和构建历史可能暴露相关信息。构建阶段需要秘密时，应使用 BuildKit Secrets；运行阶段需要秘密时，可以使用 Docker Secrets。

### 3.5 BuildKit 构建阶段秘密

BuildKit 支持在构建时临时挂载秘密，避免把秘密直接写入镜像层。

示例：

```bash
docker build \
  --secret id=npm_token,src=./npm_token.txt \
  -t my-app:v1 .
```

Dockerfile 中使用：

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine

RUN --mount=type=secret,id=npm_token \
    TOKEN="$(cat /run/secrets/npm_token)" && \
    echo "完成需要认证的构建操作"
```

BuildKit Secret 仅在对应构建步骤中挂载。不要通过打印命令、调试输出或复制文件的方式再次泄露秘密。

## 4. 容器权限管理

### 4.1 容器 root 用户与宿主机 root 用户

容器内部的 root 用户通常拥有容器命名空间内的高权限，但容器与宿主机共享内核，容器 root 仍然具有安全风险。

风险主要来自：

- 容器内进程拥有过多 Capabilities。
- 使用特权模式运行容器。
- 将宿主机敏感目录或 Docker Socket 挂载进容器。
- 内核或容器运行时存在安全漏洞。
- 容器配置不当导致越过预期隔离边界。

因此，应用容器应尽可能使用普通用户运行。

### 4.2 使用非 root 用户运行容器

运行时指定用户：

```bash
docker run -d \
  --name nonroot-demo \
  --user 1000:1000 \
  nginx:alpine
```

查看容器运行用户：

```bash
docker exec nonroot-demo id
docker inspect nonroot-demo --format '{{.Config.User}}'
```

也可以在 Dockerfile 中指定默认用户：

```dockerfile
FROM alpine
RUN addgroup -S app && adduser -S app -G app
USER app
CMD ["sleep", "infinity"]
```

注意：使用非 root 用户时，应用需要写入的目录必须提前创建，并确保该用户拥有适当的读写权限。否则应用可能因为 Permission denied 无法启动。

### 4.3 非 root Nginx 实验

实验目录：`~/docker-lab/security-demo`

自定义 Nginx 配置文件 `nginx.conf` 的关键内容：

```nginx
pid /tmp/nginx.pid;

events {
    worker_connections 1024;
}

http {
    client_body_temp_path /tmp/client_temp;
    proxy_temp_path /tmp/proxy_temp;
    fastcgi_temp_path /tmp/fastcgi_temp;
    uwsgi_temp_path /tmp/uwsgi_temp;
    scgi_temp_path /tmp/scgi_temp;

    access_log /dev/stdout;
    error_log /dev/stderr;

    server {
        listen 8080;

        location / {
            default_type text/plain;
            return 200 "Nginx running as non-root user!\\n";
        }
    }
}
```

Dockerfile：

```dockerfile
FROM nginx:alpine

RUN mkdir -p /tmp/client_temp \
    /tmp/proxy_temp \
    /tmp/fastcgi_temp \
    /tmp/uwsgi_temp \
    /tmp/scgi_temp \
    && chown -R 1000:1000 /tmp

COPY nginx.conf /etc/nginx/nginx.conf

USER 1000:1000

EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
```

构建镜像：

```bash
docker build -t nginx-nonroot:v1 .
```

运行容器：

```bash
docker run -d \
  --name nginx-nonroot-demo \
  -p 8081:8080 \
  nginx-nonroot:v1
```

验证：

```bash
curl http://127.0.0.1:8081
docker exec nginx-nonroot-demo id
docker inspect nginx-nonroot-demo \
  --format 'Status={{.State.Status}} User={{.Config.User}} Privileged={{.HostConfig.Privileged}}'
```

实验中使用自定义 Nginx 配置，将 PID 文件和临时目录放在普通用户可写的 `/tmp` 下，使 Nginx 能够以 UID 1000、GID 1000 正常运行。

### 4.4 Linux Capabilities 的最小权限原则

容器运行时可以删除不需要的能力：

```bash
docker run -d \
  --name cap-drop-demo \
  --cap-drop NET_RAW \
  nginx:alpine
```

如果应用不需要任何默认能力，可以删除全部能力：

```bash
docker run -d \
  --name cap-drop-all-demo \
  --cap-drop ALL \
  nginx:alpine
```

只有确认应用确实需要某项能力时，才使用 `--cap-add` 添加对应能力。不要为了方便直接使用 `--privileged`。

## 5. Docker Secrets

### 5.1 Secrets 的用途

Docker Secrets 用于管理密码、API Token、TLS 私钥和其他敏感信息。

适用场景：

- 数据库密码。
- 应用访问外部服务的 Token。
- TLS 证书和私钥。
- 服务之间认证所需的凭据。

Docker Swarm 原生提供 Secrets 管理功能。普通 standalone 容器不能直接使用 Swarm 的 Docker Secrets 管理机制。

### 5.2 Secrets 的安全特点

- Secret 由 Swarm 管理。
- 只有被授权的服务才能访问对应 Secret。
- Secret 在服务容器中以文件形式挂载，默认路径为 `/run/secrets/<SECRET_NAME>`。
- Secret 内容不会通过普通的 `docker secret inspect` 输出。
- Swarm 使用 TLS 保护节点间通信，并对 Raft 日志中的 Secret 数据进行加密保护。
- Secret 创建后不能直接修改内容；需要变更时通常创建新名称的 Secret，再更新服务引用。

### 5.3 创建 Secret

通过标准输入创建，避免把秘密直接作为命令参数：

```bash
printf '%s' 'ChangeThisToARealSecret' | \
  docker secret create db_password -
```

查看 Secret：

```bash
docker secret ls
docker secret inspect db_password
```

Inspect 显示元数据，不显示 Secret 明文内容。

### 5.4 创建授权服务并读取 Secret

```bash
docker service create \
  --name secret-demo \
  --secret db_password \
  nginx:alpine
```

查看服务任务：

```bash
docker service ps secret-demo
```

获取任务容器 ID 后，在容器中读取：

```bash
docker ps --filter name=secret-demo
docker exec <CONTAINER_ID> ls -l /run/secrets/
docker exec <CONTAINER_ID> cat /run/secrets/db_password
```

实验验证中，授权容器能够读取挂载文件，Secret 对象的 inspect 信息不会显示密码内容。

生产环境中不要在终端录屏、日志或公开文档中输出真实 Secret 内容。

### 5.5 更新 Secret

Secret 内容不能原地修改。更新时创建新的 Secret：

```bash
printf '%s' 'NewSecretValue' | \
  docker secret create db_password_v2 -
```

更新服务引用：

```bash
docker service update \
  --secret-rm db_password \
  --secret-add source=db_password_v2,target=db_password \
  secret-demo
```

确认服务更新并验证应用能够读取新 Secret 后，再根据依赖情况删除旧 Secret。

### 5.6 Secrets 常用命令

```bash
# 创建
printf '%s' 'secret-value' | docker secret create app_secret -

# 列出
docker secret ls

# 查看元数据
docker secret inspect app_secret

# 删除（必须先确认没有服务继续引用）
docker secret rm app_secret
```

## 6. Docker Configs

### 6.1 Configs 的用途

Docker Configs 用于向 Swarm 服务分发普通配置文件，例如：

- Nginx 配置文件。
- 应用配置文件。
- 非敏感的服务参数。
- 普通文本格式的配置数据。

Configs 与 Secrets 的主要区别：

| 对比项 | Docker Configs | Docker Secrets |
|---|---|---|
| 主要用途 | 普通配置 | 敏感数据 |
| 典型内容 | Nginx 配置、应用参数 | 密码、Token、私钥 |
| 默认挂载位置 | 指定目标路径 | `/run/secrets/` |
| 内容保护 | 不作为秘密加密存储 | Swarm 对 Secret 数据提供加密保护 |
| 修改方式 | 创建新 Config 并更新服务 | 创建新 Secret 并更新服务 |

不要把密码或私钥放入 Configs。

### 6.2 创建 Config

创建配置文件 `nginx.conf`：

```nginx
server {
    listen 8080;

    location / {
        return 200 "Hello from Docker Configs!\\n";
        add_header Content-Type text/plain;
    }
}
```

创建 Config：

```bash
docker config create nginx_config nginx.conf
```

查看：

```bash
docker config ls
docker config inspect nginx_config
```

Config inspect 中可能以 Base64 形式展示配置数据。Base64 是编码方式，不是加密方式，因此 Config 不适合保存敏感信息。

### 6.3 将 Config 挂载到 Swarm 服务

```bash
docker service create \
  --name config-demo \
  --config source=nginx_config,target=/etc/nginx/conf.d/default.conf \
  --publish published=8082,target=8080 \
  nginx:alpine
```

验证：

```bash
docker service ps config-demo
curl http://127.0.0.1:8082
```

实验中服务返回：

```text
Hello from Docker Configs!
```

注意：Nginx 会加载 `/etc/nginx/conf.d/` 下匹配的配置文件。目标路径和文件名应与镜像中的实际配置加载规则一致。

### 6.4 更新 Config

Docker Config 创建后不能直接修改内容。通常创建新版本，再更新服务挂载关系。

创建新版配置文件 `nginx-v2.conf`，然后创建新 Config：

```bash
docker config create nginx_config_v2 nginx-v2.conf
```

更新现有服务：

```bash
docker service update \
  --config-rm nginx_config \
  --config-add source=nginx_config_v2,target=/etc/nginx/conf.d/default.conf \
  config-demo
```

验证：

```bash
curl http://127.0.0.1:8082
docker service ps config-demo
```

实验中服务返回：

```text
Hello from Docker Configs V2!
```

也可以使用新 Config 创建另一个服务，便于对比不同配置版本。

### 6.5 Configs 常用命令

```bash
# 创建
docker config create <CONFIG_NAME> <FILE>

# 列出
docker config ls

# 查看元数据
docker config inspect <CONFIG_NAME>

# 删除（必须先确认没有服务继续引用）
docker config rm <CONFIG_NAME>
```

删除服务或更新服务配置之前，应确认 Config 不再被其他服务引用。

## 7. 容器安全限制

### 7.1 只读根文件系统

默认情况下，容器可以在自身可写层写入文件。使用 `--read-only` 可以将容器根文件系统设为只读，降低攻击者修改容器文件和持久化恶意文件的风险。

运行示例：

```bash
docker run -d \
  --name readonly-demo \
  --read-only \
  nginx:alpine
```

部分应用启动时需要写入 PID、缓存或临时文件。只读根文件系统可能导致应用启动失败，需要将必要的临时目录单独挂载为可写文件系统。

### 7.2 tmpfs 临时文件系统

tmpfs 将临时文件存放在内存支持的文件系统中，容器停止后其中的数据不会作为容器可写层持久保存。

常见用途：

- PID 文件。
- 临时缓存。
- 应用运行期间需要写入的临时目录。

示例：

```bash
docker run -d \
  --name readonly-demo \
  --read-only \
  --tmpfs /var/cache/nginx \
  --tmpfs /var/run \
  -p 8084:80 \
  nginx:alpine
```

验证：

```bash
# 根目录写入应失败
docker exec readonly-demo touch /test-file

# tmpfs 目录允许写入
docker exec readonly-demo touch /var/cache/nginx/test-file

# 查看挂载
docker inspect readonly-demo \
  --format 'ReadonlyRootfs={{.HostConfig.ReadonlyRootfs}} Tmpfs={{json .HostConfig.Tmpfs}}'
```

实验中，Nginx 初次使用只读根文件系统启动失败，日志提示无法创建 `/var/cache/nginx/client_temp`。为 Nginx 需要写入的运行目录提供 tmpfs 后，容器正常启动；根目录写入失败，tmpfs 目录写入成功。

### 7.3 CPU 限制

使用 `--cpus` 限制容器可使用的 CPU 资源。

```bash
docker run -d \
  --name cpu-limit-demo \
  --cpus=0.5 \
  nginx:alpine
```

`--cpus=0.5` 表示容器最多使用约半个 CPU 核心的计算配额，并不表示固定绑定到某一个 CPU 核心。

### 7.4 内存限制

使用 `--memory` 限制容器内存：

```bash
docker run -d \
  --name memory-limit-demo \
  --memory=128m \
  nginx:alpine
```

容器超过内存限制时，可能触发 OOM 处理。生产环境应根据应用实际使用情况设置合理限制，并结合监控观察峰值。

### 7.5 PID 数量限制

使用 `--pids-limit` 限制容器可创建的进程和线程任务数量：

```bash
docker run -d \
  --name pids-limit-demo \
  --pids-limit=100 \
  nginx:alpine
```

PID 限制有助于降低进程数量失控或 fork bomb 对宿主机造成的影响。

### 7.6 禁止权限提升

`--security-opt no-new-privileges` 用于禁止进程通过 setuid、setgid 等机制获得额外权限。

```bash
docker run -d \
  --name no-privileges-demo \
  --security-opt no-new-privileges \
  nginx:alpine
```

查看配置：

```bash
docker inspect no-privileges-demo \
  --format '{{json .HostConfig.SecurityOpt}}'
```

预期包含：

```text
["no-new-privileges"]
```

### 7.7 资源使用与限制检查

```bash
docker stats --no-stream <CONTAINER_NAME>

docker inspect <CONTAINER_NAME> \
  --format 'ReadonlyRootfs={{.HostConfig.ReadonlyRootfs}}
Memory={{.HostConfig.Memory}}
NanoCpus={{.HostConfig.NanoCpus}}
PidsLimit={{.HostConfig.PidsLimit}}
SecurityOpt={{json .HostConfig.SecurityOpt}}'
```

注意区分资源限制值与实时资源使用量：`docker inspect` 用于查看配置，`docker stats` 用于观察当前资源使用情况。

## 8. 安全综合实验：运行受限权限的 Nginx

### 8.1 实验目标

在单台 Ubuntu Docker 主机上运行一个 Nginx 容器，同时应用多项安全措施：

- 使用普通用户 UID 1000、GID 1000。
- 将根文件系统设置为只读。
- 通过 tmpfs 提供必要的临时写入目录。
- 删除全部 Linux Capabilities。
- 禁止进程获得额外权限。
- 限制内存、CPU 和 PID 数量。
- 对外发布 HTTP 端口并验证服务正常。

### 8.2 启动综合安全容器

使用前面构建的 `nginx-nonroot:v1` 镜像：

```bash
docker run -d \
  --name secure-nginx \
  --user 1000:1000 \
  --read-only \
  --tmpfs /tmp:uid=1000,gid=1000,mode=1777 \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --memory=128m \
  --cpus=0.5 \
  --pids-limit=100 \
  -p 8085:8080 \
  nginx-nonroot:v1
```

参数说明：

| 参数 | 作用 |
|---|---|
| `--user 1000:1000` | 以 UID 1000、GID 1000 运行 |
| `--read-only` | 根文件系统只读 |
| `--tmpfs /tmp:...` | 为临时文件提供可写 tmpfs |
| `--cap-drop ALL` | 删除全部默认 Capabilities |
| `--security-opt no-new-privileges` | 禁止权限提升 |
| `--memory=128m` | 内存限制 128 MiB |
| `--cpus=0.5` | CPU 配额限制为 0.5 核 |
| `--pids-limit=100` | PID 数量上限为 100 |
| `-p 8085:8080` | 宿主机 8085 映射到容器 8080 |

### 8.3 综合验收

```bash
# 1. 测试 HTTP 服务
curl http://127.0.0.1:8085

# 2. 检查运行用户
docker exec secure-nginx id

# 3. 检查安全配置
docker inspect secure-nginx \
  --format 'User={{.Config.User}}
ReadonlyRootfs={{.HostConfig.ReadonlyRootfs}}
CapDrop={{json .HostConfig.CapDrop}}
SecurityOpt={{json .HostConfig.SecurityOpt}}
Memory={{.HostConfig.Memory}}
NanoCpus={{.HostConfig.NanoCpus}}
PidsLimit={{.HostConfig.PidsLimit}}'

# 4. 检查实时资源使用
docker stats --no-stream secure-nginx
```

本次实验验收结果：

HTTP 返回：

```text
Nginx running as non-root user!
```

容器用户：

```text
uid=1000 gid=1000 groups=1000
```

安全配置：

```text
User=1000:1000
ReadonlyRootfs=true
CapDrop=["ALL"]
SecurityOpt=["no-new-privileges"]
Memory=134217728
NanoCpus=500000000
PidsLimit=100
```

资源使用情况：

```text
CPU：0.00%
MEM：2.566MiB / 128MiB
PIDS：2
```

验收说明：

- HTTP 请求成功，说明服务在安全限制下仍能正常工作。
- UID/GID 为 1000，说明 Nginx 没有以 root 身份运行。
- ReadonlyRootfs 为 true，说明容器根文件系统只读。
- CapDrop 为 ALL，说明已删除全部默认 Capabilities。
- SecurityOpt 包含 no-new-privileges，说明已配置禁止权限提升。
- Memory、NanoCpus、PidsLimit 与设定值一致，说明资源限制配置生效。

## 9. 常见问题与排障记录

### 9.1 非 root Nginx 启动失败

现象：

```text
Permission denied
```

原因：

- Nginx 默认可能需要写入缓存目录、PID 文件或临时文件。
- 普通用户没有这些目录的写入权限。

处理思路：

1. 查阅容器日志，确认具体失败路径。
2. 在镜像构建阶段创建应用需要的目录。
3. 使用 `chown` 将目录所有权交给运行用户。
4. 修改应用配置，将 PID 和临时目录指向可写位置。
5. 使用 `USER` 指定非 root 用户后重新构建镜像。

### 9.2 只读根文件系统导致容器启动失败

现象：

```text
Read-only file system
```

原因：

应用启动过程中需要在根文件系统写入 PID、缓存或临时文件。

处理思路：

- 先从日志确认具体写入目录。
- 只为必要的临时目录提供 tmpfs。
- 不要为了让应用启动而直接取消所有只读保护。
- 验证根目录仍不可写，必要的临时目录可以写入。

### 9.3 Docker Config 内容更新后服务未变化

排查：

```bash
docker config ls
docker service inspect <SERVICE_NAME> --pretty
docker service ps <SERVICE_NAME>
docker service logs <SERVICE_NAME>
```

Config 创建后不能直接修改内容。应创建新 Config，并通过 `docker service update --config-rm` 和 `--config-add` 更新服务挂载。

### 9.4 Docker Secret 无法删除

原因通常是仍有 Swarm 服务引用该 Secret。

排查：

```bash
docker service ls
docker service inspect <SERVICE_NAME> --pretty
docker secret inspect <SECRET_NAME>
```

先更新或删除引用该 Secret 的服务，再删除 Secret 对象。

### 9.5 端口访问异常

排查：

```bash
docker ps
docker port <CONTAINER_NAME>
docker inspect <CONTAINER_NAME> --format '{{json .NetworkSettings.Ports}}'
docker logs <CONTAINER_NAME>
curl -v http://127.0.0.1:<PUBLISHED_PORT>
```

确认容器内部监听端口、宿主机映射端口和应用监听地址一致。

## 10. 常用安全命令汇总

### 镜像安全

```bash
# 查看镜像摘要
docker image inspect nginx:alpine \
  --format 'ID={{.Id}} Digests={{json .RepoDigests}}'

# 扫描镜像（需要安装 Trivy）
trivy image nginx:alpine

# Docker Scout 分析
docker scout quickview nginx:alpine
docker scout cves nginx:alpine
```

### 容器权限

```bash
# 指定普通用户
docker run --user 1000:1000 nginx:alpine

# 删除能力
docker run --cap-drop ALL nginx:alpine

# 指定安全选项
docker run --security-opt no-new-privileges nginx:alpine

# 查看容器用户和权限配置
docker inspect <CONTAINER_NAME> \
  --format 'User={{.Config.User}} Privileged={{.HostConfig.Privileged}} CapDrop={{json .HostConfig.CapDrop}}'
```

### Secrets 与 Configs

```bash
# Secrets
docker secret ls
docker secret inspect <SECRET_NAME>

# Configs
docker config ls
docker config inspect <CONFIG_NAME>
```

### 只读和资源限制

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp \
  --memory=128m \
  --cpus=0.5 \
  --pids-limit=100 \
  nginx:alpine

# 查看资源使用
docker stats --no-stream

# 查看资源限制
docker inspect <CONTAINER_NAME> \
  --format 'Memory={{.HostConfig.Memory}} NanoCpus={{.HostConfig.NanoCpus}} PidsLimit={{.HostConfig.PidsLimit}}'
```

## 11. 阶段学习检查清单

- [ ] 理解容器与宿主机共享 Linux 内核的特点和安全边界。
- [ ] 理解 Namespaces 的隔离作用。
- [ ] 理解 Cgroups 的资源控制作用。
- [ ] 理解 Linux Capabilities 及 cap-add、cap-drop。
- [ ] 理解 Seccomp 的系统调用过滤机制。
- [ ] 理解 AppArmor 的强制访问控制机制。
- [ ] 能够查看 Docker Engine 和容器的安全配置。
- [ ] 理解镜像来源可信度、Tag 和 Digest 的区别。
- [ ] 理解 Trivy、Docker Scout 的镜像漏洞扫描用途。
- [ ] 掌握镜像和构建过程中的敏感信息保护原则。
- [ ] 理解 BuildKit Secrets 的构建阶段秘密挂载方式。
- [ ] 掌握使用非 root 用户运行容器。
- [ ] 理解 Docker Secrets 的用途、挂载和版本更新方式。
- [ ] 理解 Docker Configs 的用途、挂载和版本更新方式。
- [ ] 掌握只读根文件系统和 tmpfs 的配合使用。
- [ ] 掌握 CPU、内存和 PID 限制。
- [ ] 理解 no-new-privileges 的作用。
- [ ] 能够组合多项安全参数部署并验收安全容器。

## 12. 阶段面试题

### 1. Docker 容器是如何实现隔离的？

参考回答：Docker 主要利用 Linux 内核提供的 Namespaces 和 Cgroups 等机制。Namespaces 用于隔离进程、网络和挂载点等资源视图，Cgroups 用于限制和统计 CPU、内存等资源。

### 2. 容器中的 root 和宿主机 root 一样吗？

参考回答：不完全一样。容器通过 Namespace 等机制隔离了资源视图，容器内 root 通常只在容器环境中拥有较高权限。但容器仍共享宿主机内核，如果使用特权模式、挂载敏感目录或存在内核漏洞，仍可能影响宿主机，因此应尽量使用非 root 用户和最小权限。

### 3. Linux Capabilities、Seccomp 和 AppArmor 有什么区别？

参考回答：Capabilities 将 root 权限拆分成多个细粒度能力；Seccomp 用于限制进程可以使用的系统调用；AppArmor 用于限制进程能够访问的文件和系统资源。它们可以配合使用，提高容器安全性。

### 4. 为什么建议使用非 root 用户运行容器？

参考回答：如果应用存在漏洞，攻击者获得的权限会受到普通用户身份的限制，能够降低容器内权限被滥用的风险。实际使用时还要确保应用所需目录对该用户可写。

### 5. Docker Secrets 和 Configs 有什么区别？

参考回答：Secrets 用于保存密码、Token 和证书等敏感信息；Configs 用于保存 Nginx 配置、应用参数等普通配置文件。两者都可以由 Swarm 服务挂载使用，但不能把 Configs 当作加密的秘密存储。

### 6. Docker Secret 为什么不能直接修改？

参考回答：Secret 创建后内容不可原地修改。更新时一般创建一个新的 Secret，再通过 Service Update 将服务切换到新 Secret，确认服务正常后再清理旧版本。

### 7. Docker Configs 中的 Base64 是加密吗？

参考回答：不是。Base64 只是编码方式，可以轻易还原原始内容，因此 Configs 适合普通配置，不适合保存密码或私钥。

### 8. Docker 镜像 Tag 和 Digest 有什么区别？

参考回答：Tag 是便于识别的镜像标签，可能随着维护更新而指向不同内容；Digest 是镜像内容的摘要，可以固定具体镜像内容。生产部署可以使用 Digest 提高版本一致性和可复现性。

### 9. 如何检查 Docker 镜像是否存在漏洞？

参考回答：可以使用 Trivy 或 Docker Scout 扫描镜像，检查操作系统软件包和应用依赖中的已知 CVE，再根据漏洞等级和业务影响更新基础镜像或依赖，并重新构建和扫描。

### 10. `--read-only` 有什么作用？

参考回答：它会把容器根文件系统设置为只读，减少容器运行过程中修改文件的机会。如果应用需要临时写入，可以使用 tmpfs 为必要目录提供临时可写空间。

### 11. `--memory`、`--cpus` 和 `--pids-limit` 有什么作用？

参考回答：分别用于限制容器内存、CPU 使用配额和进程数量，避免单个容器资源使用失控影响宿主机和其他业务。

### 12. `no-new-privileges` 有什么作用？

参考回答：它可以阻止进程通过 setuid、setgid 等机制获得额外权限，减少权限提升风险。

### 13. 如何对一个 Nginx 容器进行安全加固？

参考回答：我会先使用非 root 用户运行，再根据应用需求设置只读根文件系统和 tmpfs，删除不必要的 Capabilities，开启 no-new-privileges，并设置 CPU、内存和 PID 限制。同时选择可信镜像、定期扫描漏洞，避免把密码等敏感信息写进镜像。

## 13. 阶段总结

Docker 安全管理的核心是最小权限、隔离、资源限制、可信镜像和敏感信息保护。

实际部署时，可以从以下几个方面检查容器：

1. 镜像是否可信，是否经过漏洞扫描。
2. 容器是否使用普通用户运行。
3. 是否删除不必要的 Capabilities。
4. 是否启用只读根文件系统，并为必要临时目录提供 tmpfs。
5. 是否设置 CPU、内存和 PID 限制。
6. 是否禁止不必要的权限提升。
7. 敏感数据是否通过 Secrets 等机制管理。
8. 普通配置是否通过 Configs 等方式分发。

通过本阶段学习，能够理解 Docker 的基础安全机制，管理镜像和容器权限，使用 Secrets 与 Configs，并组合多项安全措施部署受限权限的应用容器。

阶段 9 的主题为：**Docker 安全管理**。
