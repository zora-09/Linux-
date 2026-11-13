# 模块 05：Docker 镜像管理

> 阶段：第二阶段（Docker 镜像管理）  
> 状态：已完成  
> 实施日期：2026-11-11  
> 实施环境：Ubuntu 26.04.1 LTS（amd64），Docker Engine 29.9.3

## 1. 阶段目标

掌握 Docker 镜像的分层结构、标签和摘要、镜像查看与历史分析、保存加载、导出导入、删除清理及常见故障排查。通过实际实验理解镜像归档与容器文件系统快照的区别。

本阶段不包含 Dockerfile 正式构建实操（阶段 6），也不包含容器生命周期管理（阶段 3）。

## 2. 镜像基础与分层原理

镜像通常由多个只读文件系统层组成。不同镜像可以复用相同的基础层或依赖层，减少重复存储；拉取镜像时，本地已经存在的层通常不需要重复下载。

容器启动后会在镜像只读层之上拥有自己的可写层。容器对文件的修改不会直接改变镜像，也不会影响其他容器。

### 2.1 分层带来的效率

- **存储复用**：多个镜像共享相同层，减少重复保存。
- **下载复用**：本地已有的层无需再次下载，只获取缺失层。
- **构建缓存**：构建步骤及其依赖未变化时，可以复用之前的构建结果。

例如，Dockerfile 先安装稳定依赖，再复制经常变化的应用代码。只修改代码文件时，前面的依赖安装步骤通常可以命中缓存，重新执行 COPY 及其后续受影响步骤。

缓存是否命中不只是简单比较 Dockerfile 行文本；还会受到指令、COPY/ADD 源文件内容及元数据、前序缓存和构建参数等因素影响。具体行为与构建器实现有关。

### 2.2 镜像层与 history 的区别

- `docker image inspect` 中的 `RootFS.Layers` 用于查看镜像文件系统层。
- `docker history` 展示镜像构建历史，其中部分配置指令可能不产生独立文件系统层。
- history 中出现 `<missing>` 不代表镜像层丢失，通常只是该历史项没有可展示的独立镜像 ID。

镜像层数不是固定值，取决于镜像构建过程和内容。

## 3. 常用命令速查

| 命令 | 用途 |
|---|---|
| `docker pull IMAGE[:TAG]` | 拉取镜像 |
| `docker images` / `docker image ls` | 查看本地镜像 |
| `docker image inspect IMAGE` | 查看镜像详细元数据 |
| `docker history IMAGE` | 查看镜像构建历史 |
| `docker tag SOURCE TARGET` | 为镜像增加标签 |
| `docker rmi IMAGE` | 删除镜像标签或镜像 |
| `docker image prune` | 清理悬空镜像 |
| `docker image prune -a` | 清理未被任何容器引用的镜像 |
| `docker system df` | 查看 Docker 磁盘占用 |
| `docker system df -v` | 查看详细空间占用和引用情况 |
| `docker save` / `docker load` | 保存、加载镜像归档 |
| `docker export` / `docker import` | 导出、导入容器文件系统快照 |

常用查看示例：

```bash
sudo docker images
sudo docker image inspect alpine:latest
sudo docker history alpine:latest
sudo docker system df
sudo docker system df -v
```

## 4. Tag 与 Digest

### 4.1 Tag

Tag 是便于识别和管理镜像的标签，例如 `nginx:1.28`、`nginx:latest`。标签可以被重新指向其他镜像，因此同一个标签在不同时间可能对应不同镜像内容。

`latest` 只是一个普通标签名，不代表仓库中最新发布的版本。

### 4.2 Digest

Digest 是基于内容计算的摘要标识，常见形式为：

```text
nginx@sha256:<digest>
```

它用于精确引用镜像内容。生产环境如果只使用可变的 `latest`，不同时间拉取可能得到不同内容，影响部署一致性、故障排查和回滚。生产部署应使用明确版本标签，要求严格固定内容时可使用 Digest。

## 5. 镜像标签实验

实验中为 `alpine:latest` 增加标签：

```bash
sudo docker tag alpine:latest lab-alpine:v1
sudo docker images
```

观察到 `alpine:latest` 和 `lab-alpine:v1` 对应相同镜像 ID。随后删除原标签：

```bash
sudo docker rmi alpine:latest
sudo docker images
```

结果：删除的是指定标签，`lab-alpine:v1` 仍然存在。这说明多个标签可以指向同一个镜像内容，删除一个标签不等于立即删除全部镜像数据。

## 6. save/load：镜像归档迁移

`docker save` 保存镜像及其层和元数据；`docker load` 从归档恢复镜像，适合联网服务器向离线服务器迁移镜像。

实验命令：

```bash
# 保存镜像
sudo docker save -o ~/docker-lab/alpine-v1.tar lab-alpine:v1

# 删除最后一个标签后，从归档恢复
sudo docker rmi -f lab-alpine:v1
sudo docker load -i ~/docker-lab/alpine-v1.tar

# 验证
sudo docker images
sudo docker image inspect lab-alpine:v1
```

实际结果：归档文件约 3.8 MB；执行 load 后恢复 `lab-alpine:v1`，镜像 ID 与保存前一致（`sha256:f59abe656b692d901e184be3445e084786602108599d2a51d229c0f5a48c71bf`）。

离线迁移示例：

```bash
# 联网服务器
sudo docker save -o nginx-1.28.tar nginx:1.28

# 将 tar 文件传输到离线服务器后
sudo docker load -i nginx-1.28.tar
sudo docker images
sudo docker image inspect nginx:1.28
```

注意：`docker save` 生成的是镜像归档文件；使用上述命令时不代表一定经过 gzip 压缩。需要压缩时可以另行使用 gzip 管道或压缩工具。

## 7. export/import：容器文件系统快照

`docker export` 导出容器当前的合并文件系统快照，包含镜像原有文件以及容器可写层中的文件；`docker import` 将该文件系统快照导入为一个新镜像。

实验过程：

```bash
# 创建测试容器并在容器可写层写入文件
sudo docker run -d --name export-demo lab-alpine:v1 sleep 600
sudo docker exec export-demo sh -c 'echo "docker export test" > /tmp/export-test'

# 导出容器文件系统
sudo docker export -o ~/docker-lab/alpine-container.tar export-demo

# 导入为新镜像
sudo docker import ~/docker-lab/alpine-container.tar lab-alpine:imported

# 验证导入后的文件
sudo docker run --rm lab-alpine:imported sh -c 'cat /tmp/export-test'
```

实验中读取到文件内容：

```text
docker export test
```

导入后的镜像可以保留导出快照中的文件，但不会保留原镜像的分层历史和镜像配置（例如原来的 CMD）；导入后可按需要重新指定配置。

### 7.1 两组命令的区别

| 对比项 | save / load | export / import |
|---|---|---|
| 操作对象 | 镜像 | 容器 |
| 保存内容 | 镜像层、元数据、标签等 | 容器合并后的文件系统快照 |
| 原镜像分层历史 | 保留 | 不保留 |
| 原镜像配置 | 保留 | 不完整保留，需重新设置 |
| 常见用途 | 镜像备份、离线迁移 | 导出容器文件系统快照 |

重要：挂载卷中的数据不属于容器自身的合并文件系统快照，不能依赖 `docker export` 备份挂载卷数据。需要备份卷数据时，应使用对应的数据备份方案（数据卷将在后续阶段学习）。

生产环境常规镜像迁移优先使用 `docker save` / `docker load`。不要把 export/import 当作 save/load 的等价替代。

## 8. 镜像删除与空间清理

### 8.1 rmi

`docker rmi IMAGE` 用于删除指定镜像标签或镜像引用。若多个标签指向同一镜像，删除其中一个标签后，其他标签仍可使用该镜像。

如果容器仍引用镜像，删除操作可能因冲突而无法完成。不要习惯性使用 `-f` 强制删除。

### 8.2 prune

- `docker image prune` 默认清理悬空镜像（dangling images）。
- `docker image prune -a` 清理未被任何容器引用的镜像。
- **停止状态的容器仍然引用镜像**，所以只要容器对象还存在，不能把它的镜像当作无引用镜像清理。

清理前先检查：

```bash
sudo docker system df
sudo docker system df -v
sudo docker ps -a
sudo docker images
```

确认镜像和容器引用关系后，再选择清理范围。不要在不了解影响时直接执行 `docker image prune -a`。

### 8.3 本次实验清理结果

实验完成后清理了导入测试镜像 `lab-alpine:imported`。清理后的空间统计为：

```text
Images:        2 total, 2 active, 13.08MB
Containers:    4 total, 0 active, 24.58kB
Local Volumes: 0
```

当时保留的镜像为 `hello-world:latest` 和 `lab-alpine:v1`。四个停止状态容器仍然引用这两个镜像，因此它们不属于可由 `prune -a` 清理的无引用镜像。

## 9. 常见镜像拉取故障

| 报错或现象 | 常见原因 | 排查方向 |
|---|---|---|
| `manifest unknown` / `manifest for ... not found` | 指定标签或 manifest 不存在 | 检查镜像名、标签是否拼写正确，确认仓库是否提供该标签 |
| `unauthorized` | 未认证或无仓库访问权限 | 检查登录状态、凭据和仓库权限 |
| `no such host` | DNS 解析失败 | 检查域名解析和 DNS 配置 |
| `TLS handshake timeout` / timeout | 网络连接超时 | 检查网络、代理、镜像仓库连通性 |
| `no space left on device` | 磁盘空间不足 | 使用 `df -h`、`docker system df` 检查空间 |
| `no matching manifest` | 镜像没有当前 CPU 架构对应的 manifest | 检查镜像支持的操作系统和架构 |

排障时先确认镜像名、标签和架构，再检查网络、认证和磁盘空间；不要一遇到拉取失败就盲目删除镜像或重装 Docker。

## 10. 阶段面试题与参考回答

### 1. Docker 镜像为什么采用分层结构？

镜像分层可以复用公共文件层，减少重复存储；拉取时已有层不需要重复下载；构建时未变化的步骤可以复用缓存，提高构建效率。

### 2. docker save 和 docker export 有什么区别？

save 操作对象是镜像，会保存镜像层、元数据和标签，配合 load 适合离线迁移。export 操作对象是容器，导出的是容器合并后的文件系统快照，不保留原镜像分层历史和完整配置。

### 3. docker rmi 和 docker image prune -a 有什么区别？停止容器会影响清理吗？

rmi 是主动删除指定镜像引用；prune -a 是批量清理没有被任何容器引用的镜像。停止状态的容器仍然引用镜像，因此该镜像不会被 prune -a 清理。

### 4. Tag 和 Digest 有什么区别？为什么生产环境不建议只用 latest？

Tag 便于识别，但可以被重新指向其他镜像；Digest 是基于内容的摘要标识，可精确引用镜像内容。只使用 latest 可能导致不同时间部署得到不同镜像，影响版本一致性和回滚。

### 5. 如何把联网服务器上的 Nginx 镜像迁移到离线服务器？

联网服务器执行 `docker save -o nginx-1.28.tar nginx:1.28`，将归档传到离线服务器后执行 `docker load -i nginx-1.28.tar`，再用 `docker images` 和 `docker image inspect nginx:1.28` 验证。

## 11. 阶段验收清单

- [x] 理解镜像分层、公共层复用和构建缓存
- [x] 理解 Tag 可变、Digest 用于精确引用
- [x] 掌握 pull、images、inspect、history、tag 等命令
- [x] 观察 RootFS.Layers 与 docker history 的区别
- [x] 实际完成镜像 tag、删除标签和镜像恢复
- [x] 使用 save/load 完成镜像归档与恢复
- [x] 使用 export/import 完成容器文件系统快照导出与导入
- [x] 理解停止状态容器仍然引用镜像
- [x] 使用 system df / system df -v 检查空间与引用情况
- [x] 理解常见镜像拉取错误及排查方向
- [x] 完成五道阶段面试题

## 12. 阶段总结与后续计划

第二阶段完成了 Docker 镜像管理的核心原理、命令、归档迁移、清理和故障排查。实验验证了镜像标签可以共享同一镜像 ID、save/load 可以恢复原镜像，以及 export/import 可以保留容器快照中的文件但不保留原镜像历史和完整配置。

下一阶段进入 **阶段 3：Docker 容器管理**，学习容器生命周期、启动停止、前后台运行、exec 与 attach、日志查看、资源限制和容器启动故障排查。本阶段不提前开展数据卷、网络、Dockerfile 或 Compose 实操。
