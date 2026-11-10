## Linux安全基线

### 1.实施目标

检查服务器用户权限、账号密码状态、家目录权限、系统更新状态和网络监听端口，建立基础 Linux 安全基线。

### 2.用户账号检查

例：lighthouse

```bash
ubuntu@VM-0-15-ubuntu:~$ id lighthouse
uid=1001(lighthouse) gid=1001(lighthouse) groups=1001(lighthouse)
```

检查密码状态：

```bash
ubuntu@VM-0-15-ubuntu:~$ sudo passwd -S lighthouse
lighthouse L 2026-11-09 0 99999 7 -1   #L表示密码锁定
```

家目录权限检查：

```bash
ubuntu@VM-0-15-ubuntu:~$ ls -ld /home/lighthouse/
drwxr-x--- 2 lighthouse lighthouse 4096 Nov  9 09:17 /home/lighthouse/
```

权限说明：

| 权限 |            含义            |
| :--: | :------------------------: |
|  7   | 用户拥有读、写、执行、权限 |
|  5   |  同组用户拥有读和执行权限  |
|  0   |       其他用户无权限       |

### 3.系统更新检查

检查待更新软件包：

```bash
sudo apt update
apt list --upgradable
```

### 4.网络监听端口检查

```bash
sudo ss -anptul
```

### 5.systemd服务检查

```bash
ubuntu@VM-0-15-ubuntu:~$ systemctl --failed
UNIT LOAD ACTIVE SUB DESCRIPTION

0 loaded units listed.     #当前没有处于failed状态的systemd服务
```

### 6.模块总结

完成用户账号、密码状态、家目录权限、
系统更新和网络监听端口检查。

通过最小权限原则和服务暴露面检查，
降低不必要的账号权限和网络访问风险。
