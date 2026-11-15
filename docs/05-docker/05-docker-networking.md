# Docker 网络管理

## 1. 阶段目标

理解 Docker 常见网络模式及容器通信方式，能够创建和管理自定义 bridge 网络，配置端口映射，并按 DNS、IP、TCP 端口、监听状态和防火墙等层次排查网络故障。

## 2. Docker 网络基础

Docker 网络负责容器之间、容器与宿主机之间，以及容器与外部网络之间的通信。Linux 下常见实现涉及：

- **Network Namespace**：隔离网络设备、路由表、网络规则等网络资源。
- **veth pair**：一对虚拟以太网设备，可理解为连接宿主机与容器网络空间的虚拟网线。
- **Linux Bridge**：二层虚拟交换设备，连接同一 bridge 网络中的容器接口。
- **iptables/nftables 与 NAT**：参与转发、地址转换和端口发布等网络处理。

常见网络驱动/模式：

| 模式 | 特点 | 常见用途 |
|---|---|---|
| bridge | 容器拥有独立网络空间和容器 IP；同一网络中的容器可通信 | 单机容器网络，最常用 |
| host | 容器共享宿主机网络命名空间，没有独立容器 IP | 对网络性能或网络栈有特殊需求的场景 |
| none | 容器保留独立网络命名空间，但不配置常规外部网络接口 | 需要自行配置网络的隔离场景 |
| container | 与指定容器共享网络命名空间 | 多个容器共享网络栈 |
| overlay | 为不同 Docker 主机上的容器提供逻辑网络 | 常与 Swarm 等集群场景配合 |

注意：bridge 既可以指网络驱动类型，也可以指 Docker 默认创建的名为 `bridge` 的网络，两者不是同一个概念。

## 3. 默认 bridge 与自定义 bridge

默认 bridge 网络通常使用 `docker0` 网桥。容器连接默认 bridge 后，可以通过 IP 地址通信，但默认情况下不提供与自定义 bridge 相同的容器名称自动 DNS 解析能力。

自定义 bridge 网络由用户创建，Docker 内置 DNS 可以让同一自定义网络中的容器通过容器名或网络别名互相发现。生产环境通常更推荐按应用或业务边界创建自定义网络。

### 3.1 查看默认网络

```bash
sudo docker network ls
sudo docker network inspect bridge
ip addr show docker0
ip link show
```

本次实验观察到默认 bridge 的网段为 `172.17.0.0/16`，网关为 `172.17.0.1`，宿主机网桥接口为 `docker0`。宿主机还出现了加入 `docker0` 的 veth 接口，说明容器网络接口通过 veth pair 接入宿主机网桥。

### 3.2 默认 bridge 通信实验

创建两个测试容器：

```bash
sudo docker run -dit --name net-test1 lab-alpine:v1
sudo docker run -dit --name net-test2 lab-alpine:v1
```

查看容器 IP：

```bash
sudo docker inspect net-test1 --format '{{.NetworkSettings.Networks.bridge.IPAddress}}'
sudo docker inspect net-test2 --format '{{.NetworkSettings.Networks.bridge.IPAddress}}'
```

实验中，`net-test1` 的默认 bridge IP 为 `172.17.0.2`，`net-test2` 为 `172.17.0.4`。从 net-test1 ping net-test2 的 IP 成功；通过容器名 ping 则提示 `bad address`，验证了默认 bridge 与自定义 bridge 在名称解析方面的差异。

## 4. 创建和管理自定义 bridge

创建自定义网络：

```bash
sudo docker network create --driver bridge --subnet 172.20.0.0/16 my-bridge
sudo docker network inspect my-bridge
```

将已有容器接入网络：

```bash
sudo docker network connect my-bridge net-test1
sudo docker network connect my-bridge net-test2
```

查看容器连接的网络：

```bash
sudo docker inspect net-test1 --format '{{json .NetworkSettings.Networks}}'
```

实验中，自定义网络网关为 `172.20.0.1`，net-test1 地址为 `172.20.0.2`，net-test2 地址为 `172.20.0.3`。两个容器在该网络中既可以通过 IP 通信，也可以通过容器名通信。

同一个容器可以同时连接多个网络，每个网络有独立的 IP、网关和路由信息。断开一个网络不会自动断开其他网络：

```bash
sudo docker network disconnect bridge net-test1
sudo docker inspect net-test1 --format '{{json .NetworkSettings.Networks}}'
```

断开默认 bridge 后，net-test1 仍可通过 my-bridge 的地址访问 net-test2。之后当 net-test1 又从 my-bridge 断开、且没有连接其他网络时，容器内访问外部网络会出现 `Network unreachable`。重新连接 my-bridge 后通信恢复。

查看容器路由：

```bash
sudo docker exec net-test1 ip route
```

实验中路由表包含默认路由 `default via 172.20.0.1 dev eth0`，以及直连路由 `172.20.0.0/16`。

## 5. 端口映射

端口映射将宿主机端口发布到容器端口，常见格式：

```bash
-p HOST_PORT:CONTAINER_PORT
```

例如：

```bash
sudo docker run -d --name web-demo -p 127.0.0.1:8080:80 nginx
sudo docker port web-demo
```

`127.0.0.1:8080:80` 表示只在宿主机本地地址发布 8080 端口；若使用 `-p 8080:80`，通常会发布到宿主机所有 IPv4 接口。端口映射通常需要在创建容器时配置，不能直接给一个已经创建的容器追加端口发布配置。

**端口映射不等于应用服务本身。** 容器内必须有程序真正监听目标端口，并且应用监听地址要允许容器网络接口访问（例如监听 `0.0.0.0:80`，而不是仅监听容器内的 `127.0.0.1:80`）。

本次使用 BusyBox `nc` 临时监听容器 80 端口，并将宿主机 `127.0.0.1:8080` 映射到容器 80。宿主机 curl 请求得到 `Empty reply from server`，但从容器取出的接收文件中包含完整 HTTP GET 请求头，证明 TCP 请求已到达容器。由于 `nc` 只是 TCP 监听工具，不是 HTTP 服务端，因此没有返回有效 HTTP 响应。

检查端口映射和监听：

```bash
sudo docker port <container>
sudo docker inspect <container> --format '{{json .HostConfig.PortBindings}}'
sudo ss -lntp
sudo docker exec <container> ss -lnt
```

## 6. host 与 none 网络模式

### 6.1 host

```bash
sudo docker run -d --name host-test --network host lab-alpine:v1 sh -c 'nc -l -p 18080'
```

实验中，宿主机和容器的 `/proc/1/ns/net` 都指向 `net:[4026531833]`，且宿主机执行 `ss` 可以看到 18080 端口监听。这验证了容器与宿主机共享网络命名空间。

host 模式没有独立容器 IP，端口与宿主机共用，因此通常不需要也不使用 `-p` 端口映射；端口冲突也会直接影响宿主机网络服务。

### 6.2 none

```bash
sudo docker run -dit --name none-test --network none lab-alpine:v1
sudo docker exec none-test ip addr
```

实验中容器内只看到 lo 接口和 `127.0.0.1`，没有常规 eth0 网络接口；容器的 net namespace inode 与宿主机不同，说明它有独立网络命名空间，但没有连接常规网络。

## 7. TCP 连通性与故障排查

网络故障要分层定位，不能只看容器是否 Up 或 ping 是否成功。

### 7.1 DNS 与 IP 连通性

```bash
sudo docker exec net-test1 getent hosts net-test2
sudo docker exec net-test1 ping -c 2 net-test2
sudo docker inspect net-test1 --format '{{json .NetworkSettings.Networks}}'
sudo docker network inspect my-bridge
```

检查容器是否连接同一个自定义网络、名称解析结果、IP 地址和网络是否存在。

### 7.2 TCP 端口连通性

在 net-test2 中启动临时 TCP 监听：

```bash
sudo docker exec -d net-test2 sh -c 'nc -l -p 8080'
```

查看监听：

```bash
sudo docker exec net-test2 netstat -lnt
```

从 net-test1 测试：

```bash
sudo docker exec net-test1 nc -vz -w 3 net-test2 8080
```

实验中端口监听时显示 `open`；监听进程退出后，测试命令返回非零退出码；重新启动监听后再次显示 `open`。这说明容器 Up、DNS 解析成功、IP 可达，都不能单独证明应用 TCP 端口正常。

### 7.3 推荐排查顺序

1. 使用 `docker ps -a` 确认相关容器状态。
2. 使用 `docker network ls`、`docker network inspect` 确认网络存在且容器已接入。
3. 使用 `getent hosts` 检查容器名称解析。
4. 使用 `ping`（在环境允许 ICMP 的情况下）检查基础 IP 连通性。
5. 使用 `ss -lnt` 或 `netstat -lnt` 检查目标容器内服务是否监听正确端口和地址。
6. 使用 `nc -vz` 测试 TCP 端口连通性。
7. 检查应用配置、防火墙规则、宿主机端口映射和云安全组。
8. 使用 `docker logs`、应用日志及 `docker inspect` 继续定位。

常见判断：

| 现象 | 优先检查 |
|---|---|
| IP 可达，容器名无法解析 | 是否接入同一自定义 bridge、DNS 名称/别名 |
| 容器 Up，但 TCP 端口不通 | 应用是否启动、监听端口和监听地址 |
| 容器间端口可通，外部无法访问 | 端口映射、宿主机防火墙、云安全组 |
| 容器无法访问其他网络 | 网络连接、IP、路由表、网关和转发规则 |
| ping 成功但业务连接失败 | TCP 端口、应用状态、访问控制和协议层问题 |

注意：TCP 连接失败不一定都是 `Connection refused`。目标端口没有监听时可能被拒绝；防火墙丢弃数据包时也可能表现为超时。

## 8. 网络清理

查看网络及其连接情况：

```bash
sudo docker network ls
sudo docker network inspect my-bridge
```

删除未被容器使用的自定义网络：

```bash
sudo docker network prune
```

实验中创建了没有连接容器的 `prune-demo` 网络，执行 prune 后该网络被删除；仍被 net-test1、net-test2 使用的 `my-bridge` 保留。

清理前应检查网络是否仍被容器或业务依赖，避免影响运行中的服务。默认的 bridge、host、none 网络属于 Docker 内置网络，不应作为普通自定义网络随意删除。

## 9. Overlay 网络认知

本机执行：

```bash
sudo docker info --format '{{.Swarm.LocalNodeState}}'
```

结果为 `inactive`，表示当前 Docker 节点未加入 Swarm。本阶段只学习 Overlay 的用途和原理，没有启动 Swarm 或进行跨主机实操。Overlay 为不同 Docker 主机上的容器提供逻辑网络，常与 Swarm 配合，底层可使用 VXLAN 等封装机制。

## 10. 阶段验收与总结

本阶段已完成：

- 查看 Docker 默认网络、docker0 网桥和 veth 接口。
- 验证默认 bridge 的 IP 通信及容器名解析差异。
- 创建自定义 bridge，验证容器名 DNS 解析和 IP 通信。
- 验证容器多网络连接、断开网络、重新连接和路由表。
- 配置并检查端口映射，理解端口发布与应用监听的区别。
- 验证 host、none 网络模式及网络命名空间差异。
- 使用 `nc`、`ss`/ `netstat` 进行 TCP 端口监听和连通性排查。
- 模拟网络断开和 TCP 监听退出故障，并完成恢复。
- 使用 `docker network prune` 清理未使用的自定义网络。
- 理解 Overlay 网络用途，确认当前 Swarm 状态为 inactive。

**阶段结论：** Docker 网络排查应从容器状态和网络连接开始，逐层检查 DNS、IP、路由、TCP 端口、应用监听及防火墙/端口映射。容器状态 Up 不代表业务可用，ping 成功也不代表 TCP 服务正常。

## 11. 实验环境清理提醒

本阶段的 `net-test1`、`net-test2`、`attach-demo` 和 `my-bridge` 是测试环境对象。清理前应再次检查它们是否仍需用于后续实验，不要直接对所有 Docker 资源执行无差别 prune。
