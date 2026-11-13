# Docker 数据卷与数据持久化

> 模块：05 Docker 环境部署与管理  
> 阶段：04 Docker 数据卷与数据持久化  
> 实验环境：Ubuntu 26.04.1 LTS、Docker Engine 29.9.3、镜像 `lab-alpine:v1`

## 1. 阶段目标

理解容器可写层与持久化存储的区别，掌握 Docker Volume、Bind Mount、tmpfs 的使用方式，能够完成数据备份与恢复，并能排查挂载路径、数据可见性和 Linux 文件权限问题。

## 2. 核心原理

### 2.1 容器可写层

容器启动后会在镜像只读层之上拥有自己的可写层。写入容器文件系统、但没有写入挂载存储的数据，通常位于该容器的可写层中。容器删除后，其可写层随之删除，因此不适合保存需要长期保留的数据。

### 2.2 Volume

Volume 由 Docker 创建和管理，数据存储位置由 Docker 管理。容器通过挂载点访问 Volume；容器内部挂载路径由启动容器时指定，并不是 Volume 固定绑定的路径。

例如：

```bash
docker run -v mysql-data-demo:/var/lib/mysql ...
```

表示把名为 `mysql-data-demo` 的 Volume 挂载到容器的 `/var/lib/mysql`。同一个 Volume 也可以在另一个容器中挂载到 `/data`：

```bash
docker run -v mysql-data-demo:/data:ro ...
```

两个容器访问的是同一份 Volume 数据，只是容器内部的访问路径不同。

常用命令：

```bash
sudo docker volume create my-data
sudo docker volume ls
sudo docker volume inspect my-data
sudo docker volume rm my-data
```

Volume 的生命周期独立于普通容器。删除容器通常不会删除命名 Volume；删除 Volume 前应确认数据是否仍需要，并完成必要备份和恢复验证。

### 2.3 Bind Mount

Bind Mount 将宿主机指定的文件或目录映射到容器内部。宿主机路径由用户指定，适合共享配置文件、代码、日志或需要由宿主机直接管理的文件。

推荐使用显式的 `--mount`：

```bash
sudo docker run --rm \
  --mount type=bind,src=/home/ubuntu/docker-lab/bind-demo,dst=/data \
  lab-alpine:v1 ls -l /data
```

与 Volume 不同，Bind Mount 的源路径是宿主机指定路径。使用 `--mount type=bind` 时，源路径不存在通常会报错；短语法 `-v /host/path:/container/path` 在源路径不存在时通常会自动创建目录，容易因路径拼写错误而创建出空目录。

### 2.4 tmpfs

tmpfs 将临时文件系统挂载到容器中，适合临时缓存或不需要作为普通持久化数据保存的临时文件。容器停止或挂载被移除后，数据不会作为持久化数据保留。

```bash
sudo docker run --rm --tmpfs /tmp:rw,size=16m lab-alpine:v1 \
  sh -c 'echo "temporary data" > /tmp/demo.txt && cat /tmp/demo.txt'
```

### 2.5 三种挂载方式对比

| 类型 | 存储位置管理 | 容器删除后 | 常见用途 |
|---|---|---|---|
| Volume | Docker 管理 | 数据通常保留，除非显式删除卷 | 数据库、业务持久化数据 |
| Bind Mount | 用户指定宿主机路径 | 宿主机文件仍保留 | 配置、代码、日志共享 |
| tmpfs | 临时内存文件系统 | 不作为持久化数据保留 | 临时文件、缓存 |

## 3. 挂载语法与 inspect 检查

### 3.1 Volume 与 Bind Mount 语法

```bash
# Volume：卷名:容器内路径
docker run -v my-volume:/data IMAGE

# Volume 只读
docker run -v my-volume:/data:ro IMAGE

# Bind Mount：宿主机路径:容器内路径
docker run -v /host/path:/data IMAGE

# 显式挂载 Volume
docker run --mount type=volume,src=my-volume,dst=/data IMAGE

# 显式只读 Bind Mount
docker run --mount type=bind,src=/host/path,dst=/data,readonly IMAGE
```

### 3.2 查看 Mounts

```bash
sudo docker inspect CONTAINER
sudo docker inspect CONTAINER --format '{{json .Mounts}}'
sudo docker inspect CONTAINER --format '{{json .Mounts}}' | python3 -m json.tool
```

重点字段：

| 字段 | 含义 |
|---|---|
| `Type` | 挂载类型，如 `volume` 或 `bind` |
| `Name` | Volume 名称（Bind Mount 通常没有该字段） |
| `Source` | 宿主机数据源路径 |
| `Destination` | 容器内部挂载路径 |
| `RW` | 是否可读写，`false` 表示只读 |
| `Driver` | Volume 驱动信息（如适用） |

示例结构（字段值仅用于说明，不代表本机某个容器的实时输出）：

```json
{
  "Type": "volume",
  "Name": "mysql-data-demo",
  "Source": "/var/lib/docker/volumes/mysql-data-demo/_data",
  "Destination": "/var/lib/mysql",
  "Driver": "local",
  "RW": true
}
```

注意区分：`Source` 是宿主机来源位置，`Destination` 是容器内部挂载点。Volume 可以在不同容器中挂载到不同的 Destination。

## 4. 实验记录

### 4.1 Volume 持久化与容器生命周期

创建卷并写入文件：

```bash
sudo docker volume create volume-demo-data

sudo docker run --name volume-writer \
  -v volume-demo-data:/data lab-alpine:v1 \
  sh -c "echo 'persistent data from volume' > /data/demo.txt && cat /data/demo.txt"
```

验证过程：在宿主机检查 Docker 管理的卷挂载点，确认 `demo.txt` 存在；删除写入容器后，卷仍存在且文件内容仍可读取；再用新容器挂载同一个卷读取成功。这验证了命名 Volume 与普通容器生命周期相互独立。

查看卷挂载点：

```bash
sudo docker volume inspect volume-demo-data
```

本机 Docker Root Dir 为 `/var/lib/docker`，该卷的 Mountpoint 为：

`/var/lib/docker/volumes/volume-demo-data/_data`

这是宿主机上 Docker 管理的存储位置；容器内部挂载路径由 `-v volume-demo-data:/data` 中的 `/data` 指定。日常管理应通过 Docker 和挂载卷的容器操作，不建议直接手工修改 Docker 内部目录。

### 4.2 只读 Volume

```bash
sudo docker run --rm --name volume-read \
  -v volume-demo-data:/data:ro lab-alpine:v1 \
  sh -c "echo 'hello' > /data/test.txt"
```

实验结果：出现 `Read-only file system`，符合预期。只读挂载可降低容器误修改数据的风险。

### 4.3 Bind Mount 双向共享

宿主机准备目录和文件：

```bash
mkdir -p ~/docker-lab/bind-demo
echo "hello from ubuntu host" > ~/docker-lab/bind-demo/host.txt
```

容器读取宿主机文件：

```bash
sudo docker run --rm \
  --mount type=bind,src=/home/ubuntu/docker-lab/bind-demo,dst=/data \
  lab-alpine:v1 cat /data/host.txt
```

容器写入文件：

```bash
sudo docker run --rm \
  --mount type=bind,src=/home/ubuntu/docker-lab/bind-demo,dst=/data \
  lab-alpine:v1 \
  sh -c 'echo "hello from container" > /data/container.txt'
```

验证结果：宿主机在 `~/docker-lab/bind-demo/container.txt` 中读取到 `hello from container`。这验证了宿主机和容器通过 Bind Mount 共享同一目录。

### 4.4 tmpfs

执行：

```bash
sudo docker run --rm --name tmpfs-demo \
  --tmpfs /tmp:rw,size=16m lab-alpine:v1 \
  sh -c 'echo "temporary data" > /tmp/demo.txt && cat /tmp/demo.txt'
```

实验输出为 `temporary data`，临时挂载读写正常。

### 4.5 Volume 备份与恢复

本实验使用 Alpine 临时容器和文本文件模拟业务数据，不是实际 MySQL 数据库的在线物理备份。

创建模拟数据卷并写入两份数据：

```bash
sudo docker volume create mysql-data-demo

sudo docker run --rm \
  -v mysql-data-demo:/var/lib/mysql \
  lab-alpine:v1 \
  sh -c 'echo "database user data" > /var/lib/mysql/users.txt && echo "database order data" > /var/lib/mysql/orders.txt'
```

验证同一个 Volume 可以挂载到不同容器路径：

```bash
sudo docker run --rm \
  -v mysql-data-demo:/data:ro \
  lab-alpine:v1 ls -l /data

sudo docker run --rm \
  -v mysql-data-demo:/var/lib/mysql:ro \
  lab-alpine:v1 ls -l /var/lib/mysql
```

两次查看均看到 `users.txt` 和 `orders.txt`。说明卷名相同，容器内部挂载路径可以不同。

创建宿主机备份目录：

```bash
mkdir -p ~/docker-lab/mysql-backup
```

备份时，临时容器同时挂载原始 Volume 和宿主机备份目录：

```bash
sudo docker run --rm \
  --mount type=volume,src=mysql-data-demo,dst=/data,readonly \
  --mount type=bind,src=/home/ubuntu/docker-lab/mysql-backup,dst=/backup \
  lab-alpine:v1 \
  tar czf /backup/mysql-data-demo.tar.gz -C /data .
```

其中：
- Volume 挂载到容器的 `/data`，供 tar 读取原始数据。
- 宿主机目录 Bind Mount 到容器的 `/backup`，供 tar 写入归档文件。
- `tar czf` 创建 gzip 压缩归档；`-C /data .` 表示切换到 `/data` 后打包其中内容。

实际验证：宿主机目录中生成 `mysql-data-demo.tar.gz`，大小约 174 bytes；使用 `tar -tzvf` 检查，归档中包含 `users.txt` 和 `orders.txt`。

```bash
ls -lh ~/docker-lab/mysql-backup/
sudo tar -tzvf ~/docker-lab/mysql-backup/mysql-data-demo.tar.gz
```

恢复到新 Volume：

```bash
sudo docker volume create mysql-data-restore

sudo docker run --rm \
  --mount type=volume,src=mysql-data-restore,dst=/restore \
  --mount type=bind,src=/home/ubuntu/docker-lab/mysql-backup,dst=/backup,readonly \
  lab-alpine:v1 \
  tar xzf /backup/mysql-data-demo.tar.gz -C /restore
```

验证恢复卷中的文件：

```bash
sudo docker run --rm \
  -v mysql-data-restore:/restore \
  lab-alpine:v1 ls -l /restore
```

实际输出包含 `orders.txt`（20 bytes）和 `users.txt`（19 bytes）。文件内容曾在前一个恢复卷实验中通过 `cat` 验证；本次 `mysql-data-restore` 的已记录验证结果为文件列表和大小匹配。

### 4.6 UID/GID 与 Bind Mount 权限

容器进程访问 Bind Mount 时，Linux 会根据进程 UID/GID 和宿主机文件系统权限进行检查。

实验中使用 UID/GID 1000 创建文件：

```bash
sudo docker run --rm --user 1000:1000 \
  --mount type=bind,src=/home/ubuntu/docker-lab/bind-demo,dst=/data \
  lab-alpine:v1 \
  sh -c 'id && echo "written by uid 1000" > /data/uid-test.txt'
```

宿主机检查显示 `uid-test.txt` 的属主为 `1000:1000`。

再使用 UID/GID 2000 写入同一目录：

```bash
sudo docker run --rm --user 2000:2000 \
  --mount type=bind,src=/home/ubuntu/docker-lab/bind-demo,dst=/data \
  lab-alpine:v1 \
  sh -c 'id && echo "written by uid 2000" > /data/uid2000-test.txt'
```

实验结果：进程身份为 `uid=2000 gid=2000`，写入时报 `Permission denied`。原因是宿主机目录属主为 UID 1000，目录权限没有授予其他用户写权限。

处理方式应按实际需求选择：
- 将目录属主调整为容器运行 UID/GID（确认不会影响宿主机其他服务）。
- 使用共享组并设置合适的组权限，例如 `chgrp` 配合 `chmod 775`。
- 避免把 `chmod 777` 当作默认解决方案，遵循最小权限原则。

## 5. 故障排查清单

### 5.1 容器内看不到 Volume 数据

1. `docker volume ls`：确认卷存在。
2. `docker volume inspect 卷名`：确认卷名称和驱动等信息。
3. `docker inspect 容器名`：检查 `Mounts` 中的 `Type`、`Name`、`Source`、`Destination`、`RW`。
4. 确认应用实际使用的数据目录与容器内挂载点一致。
5. 检查挂载的卷名是否正确，以及卷中目标文件是否存在。

挂载成功不代表应用数据一定在该目录；如果挂载路径与应用真实数据目录不一致，可能看到空目录或遮蔽镜像原目录中的文件。

### 5.2 Bind Mount 报 Permission denied

1. 用 `docker inspect` 检查挂载类型、源路径、目标路径和读写状态。
2. 检查容器运行用户：`docker inspect --format '{{.Config.User}}' 容器名`。
3. 在宿主机检查源目录及文件权限：`ls -ld /host/path`、`ls -l /host/path`、`stat /host/path`。
4. 对比容器进程 UID/GID 与宿主机目录属主、属组和权限。
5. 根据业务需要调整属主或共享组权限，不要直接扩大为所有用户可写。

### 5.3 容器删除后数据不见

检查数据是否写入容器可写层，还是写入了正确挂载的 Volume/Bind Mount。需要持久化的数据应写入挂载点对应目录。删除容器后，命名 Volume 通常仍然保留。

### 5.4 删除容器后 Volume 仍存在

这是命名 Volume 的正常行为。删除容器不等于删除命名 Volume。确认数据不再需要并完成备份后，才使用：

```bash
sudo docker volume rm 卷名
```

清理未被容器引用的卷可使用 `docker volume prune`，但执行前必须检查影响范围，避免误删仍需保留的数据。

## 6. 面试复盘

### Q1：Volume、Bind Mount 和 tmpfs 有什么区别？

- Volume：Docker 管理存储位置，生命周期独立于容器，适合数据库和业务持久化数据。
- Bind Mount：用户指定宿主机目录或文件，适合配置、代码和日志共享。
- tmpfs：临时文件系统，不作为普通持久化存储，适合临时文件和缓存。

### Q2：部署 MySQL 时为什么通常选择 Volume？

Volume 由 Docker 管理，生命周期独立于容器，删除 MySQL 容器后数据仍可保留；也可以挂载到其他容器用于备份或恢复。生产环境还应规划备份、一致性和恢复验证。

### Q3：如何备份和恢复 Docker Volume？

备份时启动临时容器，同时挂载原始 Volume 和宿主机备份目录，用 tar 将 Volume 内容打包到宿主机目录。恢复时创建新 Volume，将备份目录和新 Volume 挂载到临时容器，再将归档解压到新 Volume，最后验证文件和内容。

### Q4：容器无法读取挂载目录，报 Permission denied，如何排查？

先用 `docker inspect` 检查 Mounts 的 Type、Source、Destination 和 RW；再检查容器运行用户 UID/GID，以及宿主机文件或目录的属主、属组和权限；根据实际情况调整属主或共享组权限，遵循最小权限原则。

### Q5：容器删除后数据为什么消失？如何避免？

如果数据写入容器可写层，删除容器时可写层也会被删除。需要持久化的数据应写入 Volume 或 Bind Mount 对应的挂载目录。

### Q6：删除容器后 Volume 为什么还存在？如何删除？

命名 Volume 的生命周期独立于容器。确认数据不再需要并完成备份后，可以使用 `docker volume rm 卷名` 删除指定 Volume；清理未使用卷时要谨慎使用 `docker volume prune`。

## 7. 生产环境注意事项

1. 不要把直接打包正在运行的数据库数据目录当作一定一致、一定可恢复的备份。MySQL/PostgreSQL 等数据库应优先使用数据库原生备份工具、数据库支持的热备方案，或经过验证的一致性快照流程。
2. 备份成功不等于恢复成功。应定期在独立 Volume 或测试环境执行恢复，并检查数据内容和应用可用性。
3. 对不需要写入的挂载使用只读模式，降低误操作风险。
4. 按最小权限原则配置容器 UID/GID、宿主机目录属主和权限。
5. 删除 Volume、执行 prune 前确认数据归属和备份状态。

## 8. 阶段验收

本阶段已完成：
- Volume 创建、检查、挂载、只读访问及容器删除后的数据保留验证。
- Bind Mount 宿主机到容器读取、容器到宿主机写入的双向共享实验。
- tmpfs 临时文件系统实验。
- Volume 备份到宿主机目录、检查归档内容、恢复到新 Volume 的实验。
- UID/GID 与宿主机目录权限冲突实验。
- Mounts 信息阅读、数据不可见、权限不足、容器删除和 Volume 清理等故障分析。
- 阶段面试题复盘。

**阶段 4 完成。下一阶段：阶段 5 Docker 网络管理。**
