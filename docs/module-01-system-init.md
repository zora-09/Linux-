# Linux系统初始化
## 1. 实施目标
完成Ubuntu云服务器的基础环境检查，了解系统版本、内核版本、硬件资源、网络状态及系统服务运行情况。

## 2. 实验环境

| 项目 | 信息 |
|---|---|
| 操作系统 | Ubuntu 26.04 |
| 内核版本 | 7.0.0-43-generic |
| CPU | 2 核 |
| 内存 | 约 2GB |
| 磁盘 | 40GB |
| 部署环境 | 云服务器 |

## 3.系统信息检查

```bash
查看系统版本：
cat /etc/os-release

PRETTY_NAME="Ubuntu 26.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="26.04"
VERSION="26.04.1 LTS (Resolute Raccoon)"
VERSION_CODENAME=resolute
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=resolute
LOGO=ubuntu-logo

查看内核版本：
uname -r
7.0.0-43-generic

检查CPU：
lscpu
Architecture:                x86_64
  CPU op-mode(s):            32-bit, 64-bit
  Address sizes:             46 bits physical, 48 bits virtual
  Byte Order:                Little Endian
CPU(s):                      2
  On-line CPU(s) list:       0,1
Vendor ID:                   GenuineIntel
  Model name:                Intel(R) Xeon(R) Gold 6148 CPU @ 2.40GHz
    CPU family:              6
    Model:                   94
    Thread(s) per core:      1
    Core(s) per socket:      2
    Socket(s):               1
    Stepping:                3
    BogoMIPS:                4799.99
..........

检查内存：
free -h
               total        used        free      shared  buff/cache   available(可用内存)
Mem:           1.9Gi       469Mi       777Mi       2.0Mi       869Mi       1.5Gi
Swap:          2.0Gi          0B       2.0Gi

检查磁盘：
df -h

检查inode：
df -i  #存储文件的元数据，即"关于文件的数据"，但不包含文件名和文件内容本身。inode消耗完即使磁盘还有空间也无法创建新的文件


```
## 4.系统服务检查 

```bash
查看失败的 systemd 服务：
systemctl --failed

检查 SSH 服务：
systemctl status ssh

检查网络监听端口：
sudo ss -tulnp

```
## 5.验收结果
·已确认 Ubuntu 26.04 系统环境
·内核版本7.0.0-43-generic
·服务器配置为 2 核 CPU、约 2GB 内存、40GB 磁盘
·SSH 服务正常运行
·当前 systemd 失败服务数量为 0

## 6.模块总结

完成云服务器基础环境检查，掌握系统资源、服务状态和网络监听情况，为后续安全加固提供基础
