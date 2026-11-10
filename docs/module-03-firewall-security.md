## 防火墙安全策略及Fail2ban日志监控行为

### 1.实施目标

使用ufw管理服务器网络访问规则，通过fail2ban监控ssh登录失败行为，降低未经授权访问的风险。

### 2.ufw防火墙配置

检查防火墙配置：

```bash
sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
443/tcp (v6)               ALLOW IN    Anywhere (v6)      ##IPv4 和 IPv6 均配置了放行规则。
```

默认策略：

- 入站流量：deny
- 出站流量：allow
- 路由转发：disabled

### 3.Fail2ban 部署与配置

`fail2ban` 是一个日志监控工具。它会实时扫描 SSH 等服务的安全日志，当发现某个 IP 在短时间内连续多次登录失败时，就自动调用防火墙（如 iptables）把这个 IP 临时封禁一段时间

1.安装启动并检查服务状态：

```bash
sudo apt update
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
sudo systemctl status fail2ban
```

2.核心配置：

**不要直接修改** `/etc/fail2ban/jail.conf`，升级时会被覆盖。应该新建或编辑 `/etc/fail2ban/jail.d/sshd.local` 文件：

```bash
[sshd]
enabled = true
backend = systemd
maxretry = 5
findtime = 10m
bantime = 1h    #10分钟内输错密码5次，就封禁该IP1小时

生效并验证：
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     4
|  `- Journal matches:  _SYSTEMD_UNIT=ssh.service + _COMM=sshd
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:
```

### 4.验收结果

- UFW 已启用。
- 默认拒绝入站流量。
- SSH、HTTP、HTTPS 端口已放行。
- Fail2ban 服务运行正常。
- sshd Jail 已启用。
- 当前检查未发现已封禁 IP，不能据此认定实际封禁测试成功

### 5.模块总结

通过UFW限制服务器入站访问范围，通过fail2ban对ssh登录失败行为进行监控和防护

