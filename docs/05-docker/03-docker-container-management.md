# 模块 05：Docker 容器管理

> 阶段：第三阶段（Docker 容器管理）  
> 状态：已完成  
> 实施日期：2026-11-11 至 2026-11-13  
> 实施环境：Ubuntu 26.04.1 LTS（amd64），Docker Engine 29.9.3

## 1. 阶段目标

掌握容器生命周期、创建与启动、停止与删除、exec 与 attach、日志查看、重启策略、资源限制和容器启动故障排查。通过轻量级 Alpine 容器完成操作验证，并结合实际输出理解容器状态和退出码。

本阶段不包含数据卷、网络、Dockerfile 和 Compose 的专项实操，这些内容分别安排在后续阶段。

## 2. 实验环境与初始检查

服务器资源有限：2 vCPU、约 1.9 GiB 内存。本阶段使用已有的 Alpine 镜像，避免额外拉取大型镜像。

阶段开始时执行：

```bash
sudo docker ps -a
sudo docker images
sudo docker stats --no-stream
```

当时观察到 4 个历史容器均为 Exited (0)，没有运行中的容器，因此 `docker stats --no-stream` 没有输出。镜像列表中 `alpine:latest` 与 `lab-alpine:v1` 指向相同镜像 ID（`f59abe656b69`），`hello-world:latest` 也仍然存在。

实验中没有删除这些历史容器。后续新建独立测试容器，避免影响此前阶段的实验记录。

## 3. 容器生命周期与常用命令

### 3.1 常见状态

| 状态 | 含义 |
|---|---|
| Created | 容器对象已创建，主进程尚未成功运行 |
| Running | 容器主进程正在运行 |
| Paused | 容器进程已暂停 |
| Restarting | 容器正在根据重启策略重新启动 |
| Exited | 容器主进程已经退出，可能正常结束，也可能异常退出 |
| Dead | 容器处于无法正常恢复的异常状态 |

判断容器是否异常不能只看 `Exited`。需要结合退出码、日志和 State.Error 分析。退出码 0 通常表示正常结束，非 0 通常表示执行失败；具体含义仍取决于应用程序。

### 3.2 常用命令

| 命令 | 作用 |
|---|---|
| `docker ps` | 查看运行中的容器 |
| `docker ps -a` | 查看所有容器，包括已停止容器 |
| `docker create` | 创建容器但不启动 |
| `docker run` | 创建并启动容器，通常可理解为 create + start |
| `docker start` | 启动已有容器 |
| `docker stop` | 停止运行中的容器 |
| `docker restart` | 重启容器 |
| `docker rm` | 删除容器对象 |
| `docker inspect` | 查看容器详细配置和状态 |
| `docker top` | 查看容器内进程 |
| `docker stats` | 查看容器资源使用情况 |
| `docker logs` | 查看容器标准输出和标准错误日志 |

### 3.3 实际生命周期实验

使用 `lab-alpine:v1` 创建 `test-demo`：

```bash
sudo docker create --name test-demo lab-alpine:v1 sleep 600
sudo docker ps -a --filter name=test-demo
sudo docker start test-demo
sudo docker ps -a --filter name=test-demo
sudo docker top test-demo
sudo docker stop test-demo
sudo docker restart test-demo
sudo docker ps -a --filter name=test-demo
sudo docker stop test-demo
sudo docker rm test-demo
sudo docker ps -a --filter name=test-demo
```

实际观察：

1. create 后状态为 `Created`。
2. start 后状态变为 `Up`，`docker top` 能看到 `sleep 600` 进程。
3. stop 后状态为 `Exited (0)`。
4. restart 后重新变为 `Up`。
5. stop 后执行 rm，容器记录消失；镜像并未因此删除。

结论：停止容器不等于删除容器。停止后的容器仍可再次启动，并且仍然引用原镜像。

## 4. 前台/后台运行、PID 1 与 exec / attach

### 4.1 容器主进程

容器生命周期与容器主进程密切相关。容器主进程（容器 PID 1）退出后，容器通常也会进入 Exited 状态。使用后台模式 `-d` 并不会让一个已经结束的进程持续运行。

### 4.2 docker exec

`docker exec` 在一个正在运行的容器中额外启动一个进程，常用于进入容器排查问题或执行临时命令。

实际实验：

```bash
sudo docker run -d --name exec-demo lab-alpine:v1 sleep 600
sudo docker exec -it exec-demo sh
```

在容器内执行 `ps`、`echo $$` 后使用 `exit` 退出 Shell，再在宿主机执行：

```bash
sudo docker ps --filter name=exec-demo
```

实际结果：退出 exec 创建的 Shell 后，`exec-demo` 仍为 Up。因为 `sleep 600` 是容器主进程，exec 启动的 Shell 是额外进程。

### 4.3 docker attach

`docker attach` 连接容器已有主进程的标准输入/输出，不会额外创建一个 Shell。

实际实验：

```bash
sudo docker run -dit --name attach-demo lab-alpine:v1 sh
sudo docker attach attach-demo
```

容器内执行 `ps`，观察到 PID 1 为 `sh`；执行 `echo $$` 输出 1，确认当前 Shell 就是容器主进程。随后使用 Docker 默认 detach 快捷键 Ctrl+P，再按 Ctrl+Q 脱离终端连接，宿主机检查容器仍为 Up。

注意：不要把 attach 当作 exec 使用。对交互式主进程输入 `exit` 会结束该主进程；Ctrl+C 可能向前台进程发送 SIGINT，具体行为取决于进程和终端信号处理方式。日常进入容器排障通常优先使用 exec。

### 4.4 exec 与 attach 对比

| 对比项 | exec | attach |
|---|---|---|
| 作用 | 启动额外进程 | 连接已有主进程 |
| 是否新建 Shell | 使用 `exec -it ... sh` 时会 | 不会 |
| 退出影响 | 退出额外 Shell 通常不影响主进程 | 退出主进程会导致容器停止 |
| 常见用途 | 进入容器排障、执行命令 | 连接交互式主进程 |

## 5. 容器日志

容器化应用通常将日志写到标准输出 stdout 和标准错误 stderr，Docker 可通过 `docker logs` 查看。

常用命令：

```bash
sudo docker logs 容器名
sudo docker logs --tail 100 容器名
sudo docker logs -f 容器名
sudo docker logs -t 容器名
```

- 不加参数：查看已有日志。
- `--tail 100`：只查看最后 100 行。
- `-f`：持续跟踪新日志。
- `-t`：显示时间戳。

### 5.1 实际日志实验

第一次执行命令时，命令中的分隔符被输入成中文分号，导致 sleep 被当作 echo 的普通文本，没有真正等待。纠正为英文半角分号后重新实验：

```bash
sudo docker run -d --name logs-demo2 lab-alpine:v1 \
  sh -c 'echo "应用启动"; sleep 3; echo "处理请求中"; sleep 3; echo "应用结束"'

sudo docker logs -t logs-demo2
```

实际日志时间：

```text
2026-11-12T03:27:09.674177824Z 应用启动
2026-11-12T03:27:12.676962773Z 处理请求中
2026-11-12T03:27:15.677882809Z 应用结束
```

三条日志之间分别相隔约 3 秒，说明命令按预期执行。该容器执行完命令后退出属于正常现象。

## 6. 容器自动重启策略

创建容器时可以设置重启策略：

| 策略 | 含义 |
|---|---|
| `no` | 默认值，不自动重启 |
| `always` | 容器退出后始终尝试重启 |
| `unless-stopped` | 除非容器被用户明确停止，否则 Docker 会尝试重新启动 |
| `on-failure[:max-retries]` | 仅在非零退出码时重启，可设置最大重试次数 |

实际使用 `on-failure:3` 模拟程序异常：

```bash
sudo docker run -d \
  --name restart-demo \
  --restart=on-failure:3 \
  lab-alpine:v1 \
  sh -c 'echo "模拟程序异常退出"; exit 1'

sudo docker ps -a --filter name=restart-demo
sudo docker inspect --format \
'状态={{.State.Status}} 退出码={{.State.ExitCode}} 重启次数={{.RestartCount}} 策略={{.HostConfig.RestartPolicy.Name}}' \
restart-demo
```

实际结果：

```text
状态=exited 退出码=1 重启次数=3 策略=on-failure
```

说明容器第一次启动后异常退出，Docker 自动重启 3 次，达到最大重试次数后停止继续重试。RestartCount 表示自动重启次数，不包括第一次启动。

查看策略和日志：

```bash
sudo docker inspect --format '{{json .HostConfig.RestartPolicy}}' restart-demo
sudo docker logs restart-demo
```

运维注意：重启策略不能代替应用健康检查和故障根因分析。对于持续失败的应用，无限制重启可能形成重启循环；应结合日志、退出码、重试上限和告警判断。

## 7. 容器资源限制

Docker 通过 Linux Cgroups 对容器资源进行限制和统计。Namespace 负责隔离资源视图，Cgroups 负责控制资源使用量。

| 参数 | 作用 |
|---|---|
| `--memory` | 设置容器内存使用上限 |
| `--cpus` | 设置 CPU 使用配额 |
| `--pids-limit` | 限制容器中的进程/线程任务数量 |

实际创建资源受限容器（实际名称为 `resource`）：

```bash
sudo docker run -d \
  --name resource \
  --memory=128m \
  --cpus=0.5 \
  --pids-limit=100 \
  lab-alpine:v1 sleep 600
```

查看资源使用：

```bash
sudo docker stats --no-stream resource
```

实际观察到内存使用约 364 KiB / 128 MiB、CPU 约 0.00%、PIDS 为 1。PIDS 是当前任务数，不是上限。

查看配置：

```bash
sudo docker inspect --format \
'Memory={{.HostConfig.Memory}} NanoCpus={{.HostConfig.NanoCpus}} PidsLimit={{.HostConfig.PidsLimit}}' \
resource
```

实际输出：

```text
Memory=134217728 NanoCpus=500000000 PidsLimit=100
```

其中 134217728 字节等于 128 MiB；500000000 NanoCPUs 对应 0.5 CPU；PidsLimit 为 100。

注意：资源限制是使用上限，不是资源预留。实验环境只有约 1.9 GiB 内存，因此只验证限制配置，没有执行可能造成宿主机 OOM 的高负载实验。

## 8. 容器启动故障排查

排查容器启动失败或启动后退出时，建议按顺序检查：

1. `docker ps -a`：确认容器状态。
2. `docker logs`：查看应用 stdout/stderr 日志；进程启动前失败时，日志可能为空。
3. `docker inspect`：查看 State、ExitCode、Error、Entrypoint、Cmd、重启策略等。
4. 检查镜像、启动命令、应用配置和资源限制。
5. 根据故障发生阶段区分容器进程创建失败与应用运行后退出。

### 8.1 故障一：启动命令不存在

执行：

```bash
sudo docker run --name failed lab-alpine:v1 nonexistent-command
```

实际错误核心内容：

```text
exec: "nonexistent-command": executable file not found in $PATH
```

随后使用 inspect 检查：

```bash
sudo docker inspect --format \
'状态码={{.State.Status}} 退出码={{.State.ExitCode}} 错误={{.State.Error}}' \
failed
```

实际结果：

```text
状态码=created 退出码=127 错误=failed to create task ... exec: "nonexistent-command": executable file not found in $PATH
```

本次错误发生在 OCI Runtime 创建容器进程时，主进程没有成功运行，容器对象仍处于 Created。退出码 127 通常表示命令无法找到。该类故障需要检查镜像中是否存在指定程序，以及 Entrypoint/Cmd 是否正确。因为应用进程尚未正常启动，`docker logs` 可能没有应用日志。

### 8.2 故障二：应用启动后异常退出

执行：

```bash
sudo docker run --name failed-exit lab-alpine:v1 \
  sh -c 'echo "应用启动"; echo "模拟程序异常" >&2; exit 42'
```

检查：

```bash
sudo docker ps -a --filter name=failed-exit
sudo docker logs --tail 50 failed-exit
sudo docker inspect --format \
'状态={{.State.Status}} 退出码={{.State.ExitCode}} 错误={{.State.Error}}' \
failed-exit
```

实际结果：

```text
状态=exited 退出码=42 错误=
```

日志内容：

```text
应用启动
模拟程序异常
```

说明容器进程成功启动并输出日志，随后应用主动返回退出码 42。Docker 没有记录到进程创建阶段错误，因此 State.Error 为空。

### 8.3 两类故障对比

| 对比项 | failed | failed-exit |
|---|---|---|
| 故障阶段 | 容器进程创建阶段 | 应用运行阶段 |
| 状态 | Created | Exited |
| 退出码 | 127 | 42 |
| State.Error | 有 OCI Runtime 错误 | 空 |
| 排查重点 | 镜像中命令是否存在、Entrypoint/Cmd | 应用日志、退出码、程序配置 |

重要：`Exited` 不一定代表异常，需结合退出码；`Created` 也可能是容器进程启动失败后留下的状态。不要只凭状态名称下结论。

## 9. 常用排障命令速查

```bash
# 所有容器
sudo docker ps -a

# 查看容器日志
sudo docker logs --tail 100 容器名

# 持续跟踪日志
sudo docker logs -f 容器名

# 查看容器状态、退出码、OOM 情况和错误
sudo docker inspect --format \
'Status={{.State.Status}} ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}} Error={{.State.Error}}' \
容器名

# 查看启动命令配置
sudo docker inspect --format \
'Entrypoint={{json .Config.Entrypoint}} Cmd={{json .Config.Cmd}}' \
容器名

# 查看容器内进程
sudo docker top 容器名

# 查看资源使用情况
sudo docker stats --no-stream
```

## 10. 阶段面试题与参考回答

### 1. Docker 容器有哪些常见生命周期状态？

Created 表示容器对象已创建但主进程尚未成功运行；Running 表示主进程运行中；Exited 表示主进程已退出，可能正常结束也可能异常退出，需要结合退出码和日志判断。还可能看到 Paused、Restarting、Dead 等状态。

### 2. docker run、create 和 start 有什么区别？

run 会创建并启动容器，通常可理解为 create 加 start；create 只创建不启动；start 启动已创建或已停止的容器。容器主进程执行结束后，即使使用 run，容器也会进入 Exited。

### 3. docker exec 和 attach 有什么区别？

exec 在运行中的容器内额外启动进程，常用于排障；attach 连接容器已有主进程的标准输入输出。日常排障通常优先使用 exec，避免误操作容器主进程。

### 4. 容器启动后马上退出，如何排查？

先用 docker ps -a 看状态，再用 docker logs 查看应用日志，然后用 docker inspect 查看退出码、State.Error、Entrypoint 和 Cmd。若是命令不存在，检查镜像和启动命令；若应用运行后退出，则结合日志和退出码分析。

### 5. Docker 如何限制容器 CPU、内存和进程数量？

创建容器时可以使用 --cpus、--memory、--pids-limit 等参数。底层主要依赖 Linux Cgroups 进行资源限制和统计；Namespace 负责资源视图隔离。可以使用 docker stats 观察实际资源使用情况。

## 11. 阶段验收清单

- [x] 理解 Created、Running、Exited 等常见容器状态
- [x] 完成 create、start、stop、restart、rm 生命周期实验
- [x] 理解容器主进程与容器生命周期的关系
- [x] 使用 exec 进入容器并验证退出额外 Shell 不影响主进程
- [x] 使用 attach 连接 PID 1，并安全脱离终端
- [x] 使用 logs、tail、时间戳查看容器日志
- [x] 验证 on-failure:3 重启策略和重启次数
- [x] 使用 memory、cpus、pids-limit 配置资源上限
- [x] 使用 docker stats 和 inspect 验证资源限制
- [x] 模拟命令不存在和应用运行后异常退出两类故障
- [x] 完成五道阶段面试题

## 12. 阶段总结与后续计划

第三阶段完成了容器生命周期管理、exec/attach、日志查看、重启策略、资源限制和两类启动故障的排查。实验验证了容器停止与删除的区别、主进程对容器生命周期的影响，以及 Cgroups 资源限制配置。

下一阶段进入 **阶段 4：Docker 数据卷与数据持久化**，学习 Volume、Bind Mount、tmpfs、数据持久化、权限和备份恢复。本阶段不提前开展数据卷或网络专项实验。
