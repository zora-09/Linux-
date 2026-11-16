# 阶段 6：Dockerfile 镜像构建

## 1. 阶段目标

掌握使用 Dockerfile 构建自定义镜像，理解常用指令、构建上下文、构建缓存和多阶段构建；能够排查文件路径、权限和缓存相关问题，并从镜像体积、安全性和构建效率等方面进行优化。

本阶段基于 Ubuntu 26.04 云服务器和已有的 `lab-alpine:v1` 基础镜像开展实验。

## 2. Dockerfile 基础结构与构建

Dockerfile 是用于描述镜像构建步骤的文本文件。常见指令包括：

| 指令 | 作用 |
|---|---|
| `FROM` | 指定基础镜像，并开启一个构建阶段 |
| `RUN` | 在构建阶段执行命令并生成镜像层 |
| `WORKDIR` | 设置后续指令的工作目录 |
| `COPY` | 从构建上下文复制文件或目录 |
| `ADD` | 复制文件，也支持本地 tar 自动解压等额外功能 |
| `CMD` | 设置容器默认启动命令或参数 |
| `ENTRYPOINT` | 设置容器启动时执行的主要程序 |
| `ARG` | 定义构建阶段可使用的参数 |
| `ENV` | 设置环境变量，并保留在镜像配置中 |
| `USER` | 指定后续构建指令及容器默认进程使用的用户 |
| `EXPOSE` | 声明应用预期监听的端口，不自动发布端口 |
| `LABEL` | 为镜像添加元数据 |

基础示例：

```dockerfile
FROM lab-alpine:v1
LABEL maintainer="server-ops-lab"
LABEL description="myfirst image"
CMD ["echo", "Hello Dockerfile!"]
```

构建和运行：

```bash
sudo docker build -t myfirst-image:v1 .
sudo docker run --rm myfirst-image:v1
```

运行输出：`Hello Dockerfile!`。

## 3. WORKDIR、RUN、COPY 与构建缓存

实验 Dockerfile：

```dockerfile
FROM lab-alpine:v1
LABEL maintainer="server-ops-lab"
LABEL description="myfirst image"
WORKDIR /app
RUN echo "This file was created during image build" > build-info.txt
COPY index.html /app/index.html
CMD ["ls", "-l", "/app"]
```

- `WORKDIR` 设置后续指令的工作目录；目录不存在时会创建。
- `RUN` 在构建镜像时执行，实验中生成了 `build-info.txt`。
- `COPY` 将构建上下文中的 `index.html` 复制到镜像内。

### 构建缓存

Docker 会根据构建步骤及其输入判断是否可以复用之前的构建结果。日志中的 `CACHED` 表示该步骤命中了缓存，不需要重新执行。

实验观察：

1. 首次构建后再次使用相同 Dockerfile 和文件构建，`WORKDIR`、`COPY` 等步骤显示 `CACHED`。
2. 修改 `RUN` 指令内容后，`RUN` 及其后续步骤重新执行。
3. 只修改 `index.html` 内容时，前面的 `WORKDIR`、`RUN` 保持缓存，`COPY` 重新执行。
4. 使用 `cache-debug:v1` 和 `cache-debug:v2` 对比，验证旧镜像仍保留旧文件内容，新镜像包含修改后的内容。

常用参数：

```bash
sudo docker build --no-cache -t cache-debug:v3 .
sudo docker build --pull --no-cache -t cache-debug:v3 .
```

`--no-cache` 禁用构建步骤缓存；`--pull` 尝试拉取更新的基础镜像。两者作用不同。

## 4. CMD 与 ENTRYPOINT

### CMD

为容器提供默认命令或参数。使用 `docker run IMAGE [COMMAND]` 指定新的命令时，会替换镜像的默认 CMD。

```dockerfile
FROM lab-alpine:v1
CMD ["echo", "Hello CMD"]
```

实验中运行 `cmd-demo.v1` 默认输出 `Hello CMD`；在 `docker run` 后指定其他命令时，默认 CMD 被覆盖。

### ENTRYPOINT

指定容器启动时执行的主要程序。使用 exec 格式的 ENTRYPOINT 时，`docker run` 后面的普通参数会传递给 ENTRYPOINT；镜像中的 CMD 常用于提供默认参数。

```dockerfile
FROM lab-alpine:v1
ENTRYPOINT ["echo"]
CMD ["hello entrypoint"]
```

默认运行输出 `hello entrypoint`。运行时追加参数会替换 CMD 的默认参数并传给 ENTRYPOINT。也可以使用 `docker run --entrypoint` 覆盖镜像的 ENTRYPOINT。

## 5. ARG 与 ENV

```dockerfile
FROM lab-alpine:v1
ARG APP_VERSION=1.0
ENV APP_ENV=production
RUN echo "build version: ${APP_VERSION}" > /build-version.txt
CMD ["sh", "-c", "echo APP_ENV=$APP_ENV;echo APP_VERSION=$APP_VERSION;cat /build-version.txt"]
```

构建参数示例：

```bash
sudo docker build --build-arg APP_VERSION=2.0 -t args-env-demo:v2 .
```

运行时覆盖环境变量：

```bash
sudo docker run --rm -e APP_ENV=testing args-env-demo:v2
```

| 对比 | ARG | ENV |
|---|---|---|
| 主要用途 | 构建阶段参数 | 环境变量配置 |
| 可用范围 | 声明后在构建阶段使用 | 构建阶段及容器运行阶段 |
| 是否保留为镜像运行环境变量 | 否 | 是 |
| 常见传值方式 | `docker build --build-arg` | `docker run -e` |

实验中构建时将 `APP_VERSION` 设置为 2.0，版本值被写入构建生成的文件；运行时直接读取 `APP_VERSION` 为空，因为 ARG 不会自动成为运行时环境变量。ENV 的 `APP_ENV` 则可以在运行时读取和覆盖。

不要把密码、Token 等秘密直接写入 ARG 或 ENV；应使用适合的秘密管理机制，例如 Docker Secrets（在适用的编排环境中）。

## 6. USER、WORKDIR 与 EXPOSE

实验 Dockerfile：

```dockerfile
FROM lab-alpine:v1
RUN addgroup -S appgroup && adduser -S -G appgroup appuser
WORKDIR /app
COPY index.html /app/index.html
USER appuser
EXPOSE 8080
CMD ["sh", "-c", "id && ls -l /app && sleep 3600"]
```

构建后通过 `docker inspect` 验证：

- `Config.User` 为 `appuser`。
- `Config.WorkingDir` 为 `/app`。
- `Config.ExposedPorts` 包含 `8080/tcp`。

`USER` 用于避免应用默认以 root 身份运行，降低进程权限；但需要确保应用目录和挂载目录的 UID/GID、属主及权限满足应用需要。

`EXPOSE` 只是声明容器预期监听的端口，不会自动创建宿主机端口映射。真正发布端口需要运行容器时使用 `-p HOST_PORT:CONTAINER_PORT`，并确保容器内应用确实监听该端口。

## 7. COPY 与 ADD

普通文件复制优先使用 COPY：

```dockerfile
COPY copy-demo/ /app/copy-demo
```

ADD 除了复制文件，还支持自动解压本地 tar 归档：

```dockerfile
ADD add-demo.tar.gz /app/
```

实验验证本地 `add-demo.tar.gz` 被 ADD 自动解压，镜像中的目标目录出现归档内的 `test.txt`。COPY 对 tar 文件只进行普通复制，不自动解压。ADD 也支持远程 URL 下载，但远程 URL 下载不会自动解压。

一般优先使用 COPY，使行为清晰、可预测；只有确实需要 ADD 的额外功能时再使用 ADD。

## 8. .dockerignore 与构建上下文

构建上下文（Build Context）是执行 `docker build` 时提供给构建过程的文件集合。命令最后的路径指定构建上下文：

```bash
sudo docker build -f Dockerfile.copy -t cache-debug:v1 .
```

其中 `.` 表示当前目录作为构建上下文。COPY/ADD 的源路径相对于构建上下文，而不是相对于 Dockerfile 所在目录。

实验中的 `.dockerignore`：

```text
.git
logs/
*.tar.gz
node_modules/
```

被排除的文件不会进入构建上下文，因此不能再通过 COPY/ADD 引用。实验中 COPY 被忽略的 `add-demo.tar.gz` 时，BuildKit 提示源文件被 `.dockerignore` 排除；注释掉对应规则后重新构建成功，容器中也验证了文件存在。

`.dockerignore` 的作用包括缩小构建上下文、减少不必要的文件传输、避免无关文件进入镜像，并降低误打包敏感文件的风险。

## 9. 多阶段构建

实验 Dockerfile：

```dockerfile
FROM lab-alpine:v1 AS builder
WORKDIR /build
RUN echo "Generated in build stage" > app.txt

FROM lab-alpine:v1
WORKDIR /app
COPY --from=builder /build/app.txt .
CMD ["cat", "/app/app.txt"]
```

第一个 FROM 开启 builder 阶段；第二个 FROM 开启新的最终阶段。第二阶段不会自动包含第一阶段的文件，只有通过 `COPY --from=builder` 指定复制的文件才进入最终镜像。

实验运行输出 `Generated in build stage`；检查最终镜像时，`/app/app.txt` 存在，而 `/build` 不存在。

多阶段构建常用于先在完整构建环境中编译应用，再把编译产物复制到较精简的运行镜像中，从而减少最终镜像体积、无关依赖和攻击面。本实验是文件生成示例，并非完整应用编译流程。

## 10. 构建故障排查实验

### 10.1 COPY 源文件路径错误

故障 Dockerfile 使用了不存在的源路径：

```dockerfile
COPY copy-demo/not-found.txt /app/
```

构建报错：

```text
"/copy-demo/not-found.txt": not found
```

排查时执行 `ls copy-demo/`，发现真实文件是 `test.txt`。将 COPY 修改为：

```dockerfile
COPY copy-demo/test.txt /app/
```

重新构建成功，并通过运行容器读取 `/app/test.txt` 验证文件内容。

排查重点：源文件是否存在、COPY 路径是否正确、构建上下文是否正确，以及文件是否被 .dockerignore 排除。

### 10.2 普通用户没有目录写权限

故障 Dockerfile 的关键部分：

```dockerfile
RUN chown root:root /app && chmod 755 /app
USER appuser
CMD ["sh", "-c", "touch /app/test.txt"]
```

运行时报错：

```text
touch: /app/test.txt: Permission denied
```

原因是 `/app` 属主为 root，权限 755，appuser 对该目录没有写权限。修复方式：

```dockerfile
RUN chown appuser:appuser /app && chmod 755 /app
USER appuser
```

重新构建后，默认 CMD 执行成功。该实验发生在容器运行阶段，而不是镜像构建阶段。

不要为了绕过权限问题直接使用 `chmod 777`；应根据应用实际需要设置正确的属主、属组和最小权限。

### 10.3 构建缓存与文件修改

基准构建后修改 `copy-demo/test.txt` 内容，再次构建时观察到：

- `WORKDIR` 仍显示 CACHED。
- `COPY` 重新执行。
- 旧镜像 `cache-debug:v1` 保留旧文件内容。
- 新镜像 `cache-debug:v2` 包含修改后的内容。

这验证了构建缓存按步骤及输入判断是否可复用；源文件变化会使相关 COPY 步骤缓存失效。排查缓存异常时可使用 `--no-cache`，但应先确认 Dockerfile、上下文和源文件本身没有问题。

## 11. 镜像构建优化要点

1. 根据应用需求选择合适的基础镜像，不盲目追求最小镜像。
2. 合理安排 Dockerfile 指令顺序，把相对稳定的依赖步骤放在前面，把经常变化的应用文件放在后面，以提高缓存复用率。
3. 使用 .dockerignore 排除版本控制目录、日志、依赖缓存和无关文件。
4. 使用多阶段构建分离编译环境与运行环境。
5. 使用 USER 运行应用，配合正确的目录和挂载权限。
6. 清理构建阶段不需要保留的临时文件；关注镜像内容和攻击面，而不只看镜像大小。

本阶段使用 `docker image ls` 查看镜像。基于同一个 Alpine 基础镜像构建的多个练习镜像大小接近（显示约 13MB DISK USAGE、约 3.86MB CONTENT SIZE），因为增加的内容主要是少量文本文件和配置，且共享基础镜像层。

## 12. 面试复盘

本阶段完成以下问题的练习：

1. 镜像体积过大时，如何检查和优化？
2. CMD 与 ENTRYPOINT 有什么区别？
3. ARG 与 ENV 有什么区别？
4. USER 指令有什么作用，为什么不建议默认使用 root？
5. EXPOSE 有什么作用，为什么不等于端口发布？
6. COPY 与 ADD 有什么区别，为什么优先使用 COPY？
7. .dockerignore 有什么作用，COPY 找不到文件时如何排查？
8. 什么是多阶段构建，有什么好处？
9. 镜像构建失败时如何按步骤排查？

排查问题时先查看构建日志，定位失败指令，再检查 Dockerfile、基础镜像、构建上下文、COPY/ADD 路径、.dockerignore 和权限。需要区分构建阶段错误与容器运行阶段错误；怀疑缓存问题时再使用 `--no-cache` 验证。

## 13. 阶段验收

- [x] 使用 Dockerfile 构建自定义镜像
- [x] 掌握 FROM、RUN、WORKDIR、COPY、ADD、CMD、ENTRYPOINT、ARG、ENV、USER、EXPOSE、LABEL
- [x] 理解构建缓存及缓存失效
- [x] 理解构建上下文与 .dockerignore
- [x] 完成 ADD 本地 tar 自动解压实验
- [x] 完成多阶段构建实验
- [x] 完成 COPY 路径、普通用户权限和缓存相关故障排查
- [x] 完成阶段面试问题复盘

阶段 6 已完成。下一阶段为：**阶段 7：Docker Compose 多容器编排**。
