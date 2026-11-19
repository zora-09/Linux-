# 模块 06：网站 HTTPS 配置与证书管理

## 1. 模块目标与完成范围

本模块学习 HTTP、HTTPS、TLS 与 X.509 证书基础，并使用 OpenSSL 完成证书材料生成和验证。模块状态标记为**已完成**，表示本轮学习与整理已收尾；实际操作记录与原理知识分别记录，未执行的公网 CA 申请和 Nginx 部署不作为已完成实操。

### 已完成的实操

- 检查 Ubuntu 环境中的 OpenSSL 版本。
- 生成并检查 RSA 2048 私钥。
- 生成并查看 CSR。
- 生成带 DNS SAN 的自签名证书。
- 查看证书 Subject、Issuer、有效期和 SAN。
- 验证证书公钥与私钥对应。
- 使用 OpenSSL 验证自签名证书。

### 本轮未实施的部署操作

公网 CA 证书申请、Certbot/ACME 自动签发与续期、Nginx HTTPS 配置、HTTP 到 HTTPS 跳转，以及真实站点 TLS 故障演练没有在本轮执行。相关原理、配置示例和排查方法作为知识储备保留。

## 2. HTTPS、HTTP 与 TLS

HTTP 通常使用 TCP 80 端口；HTTPS 是运行在 TLS 之上的 HTTP，常见服务端口为 TCP 443。

TLS 主要提供三类保护：

1. **机密性**：通过握手协商得到的对称会话密钥加密通信数据。
2. **身份认证**：客户端验证服务器证书、访问域名和证书信任链。
3. **完整性**：TLS 记录层提供完整性保护，帮助检测通信数据是否被篡改。

证书本身并不独立完成上述全部工作。服务器证书主要将服务器身份信息与公钥绑定，并由签发者签名；TLS 协议负责握手、密钥协商和后续通信保护。

浏览器验证证书时会检查访问域名是否匹配证书 SAN、证书有效期、签名链及证书用途等。自签名证书通常不在浏览器默认信任库中，因此实验环境访问时可能出现信任警告。将证书显式加入某台设备的信任库，只改变该设备的信任配置，不代表公共浏览器默认信任它。

## 3. 证书类型与证书链

### 3.1 DV、OV、EV

| 类型 | 主要验证内容 |
|---|---|
| DV | 申请者对域名具有控制权 |
| OV | 域名控制权及申请组织身份 |
| EV | 更严格的组织身份核验 |

DV、OV、EV 是身份验证要求的差异，不是 TLS 加密强度等级。

### 3.2 证书链

常见结构：

```text
受信任的根 CA
      |
      v
中间 CA（可能有多个）
      |
      v
服务器证书
```

- 根 CA：客户端信任库中的信任起点。
- 中间 CA：由上级 CA 签发，用于分层签发证书并减少根 CA 私钥的日常使用。
- 服务器证书：绑定具体服务器域名或 IP。

Nginx 部署时通常需要服务器证书及必要的中间证书链。根 CA 通常由客户端信任库提供。

### 3.3 企业内网证书

企业可以部署内部 CA，为内网服务签发证书。员工设备需要通过受控方式安装并信任企业根 CA 证书，才能验证内部证书链。企业根 CA 私钥必须严格保护，不能分发到员工设备。若通过 IP 访问服务，SAN 中应包含对应的 IP 类型地址。

## 4. 常见证书文件

| 文件 | 作用 | 安全要求 |
|---|---|---|
| `.key` | 私钥 | 必须保密，不能提交代码仓库 |
| `.csr` | 证书签名请求，包含公钥和申请信息 | 提交 CA 前检查内容 |
| `.crt` / `.cer` | 常见证书文件 | 通常可以公开 |
| `.pem` | 常见文本封装格式，可保存证书、私钥或证书链 | 必须根据实际内容判断，不能仅凭扩展名判断 |

CSR 由申请者使用私钥生成，通常提交给 CA。CA 完成必要验证后签发证书。私钥应留在申请者控制的服务器或安全环境中，不应提交给 CA。

Nginx 常见配置指令：

```nginx
ssl_certificate     /path/to/fullchain.pem;
ssl_certificate_key /path/to/privkey.pem;
```

证书文件通常指向服务器证书及所需中间证书组成的完整链文件；私钥文件必须妥善保护。

## 5. OpenSSL 实验记录

### 5.1 环境检查

实验环境：Ubuntu 虚拟机。

执行：

```bash
openssl version
which openssl
```

实际版本：

```text
OpenSSL 3.5.5 27 Jan 2026 (Library: OpenSSL 3.5.5 27 Jan 2026)
/usr/bin/openssl
```

创建实验目录：

```bash
mkdir -p ~/https-lab
cd ~/https-lab
```

### 5.2 生成 RSA 私钥

```bash
openssl genrsa -out server.key 2048
```

检查私钥：

```bash
openssl rsa -in server.key -check -noout
```

实际输出：

```text
RSA key ok
```

### 5.3 生成 CSR

```bash
openssl req -new -key server.key -out server.csr
```

交互填写的主体信息：

| 字段 | 实验值 |
|---|---|
| C | CN |
| ST | Tianjin |
| L | Tianjin |
| O | Lab |
| OU | IT |
| CN | example.com |

查看 CSR：

```bash
openssl req -in server.csr -noout -text
```

检查结果显示 Subject 包含上述主体信息，公钥算法为 RSA，公钥长度为 2048 bit，CSR 使用 SHA-256 with RSA 签名。

本次 CSR 还包含一个可选的 challengePassword 属性。它不是 HTTPS 网站访问密码，也不是私钥。实际申请通常不需要设置该属性；不要将含有不必要敏感属性的 CSR 随意公开。

本次 CSR 未配置 SAN；后续生成自签名证书时通过 `-addext` 添加 SAN。

### 5.4 生成自签名证书

实际执行：

```bash
openssl req -x509 -new \
  -key server.key \
  -out server.crt \
  -days 365 \
  -sha256 \
  -subj "/C=CN/ST=Tianjin/L=Tianjin/O=Lab/OU=IT/CN=example.com" \
  -addext "subjectAltName=DNS:example.com,DNS:www.example.com"
```

参数说明：

- `-x509`：输出 X.509 证书。
- `-key server.key`：使用指定私钥。
- `-out server.crt`：指定证书输出文件。
- `-days 365`：证书有效期 365 天。
- `-sha256`：使用 SHA-256 摘要算法签名。
- `-subj`：直接指定证书主体。
- `-addext`：添加证书扩展，本次设置 DNS SAN。

**说明：** 这条命令直接使用私钥生成自签名证书，没有读取前面生成的 `server.csr`。CSR 实验用于理解证书申请流程；真实 CA 签发时会提交 CSR，由 CA 验证并签发证书。

### 5.5 检查文件与权限

```bash
ls -l server.crt server.csr server.key
```

实际输出：

```text
-rw-rw-r-- 1 ubuntu ubuntu 1379 Nov 19 10:18 server.crt
-rw-rw-r-- 1 ubuntu ubuntu 1062 Nov 19 10:14 server.csr
-rw------- 1 ubuntu ubuntu 1704 Nov 19 10:09 server.key
```

私钥权限为 `600`，仅文件所有者可读写。私钥不能上传 GitHub、不能提交代码仓库，也不能发送给无关人员。

### 5.6 查看 Subject、Issuer 与有效期

```bash
openssl x509 -in server.crt -noout -subject -issuer -dates
```

实际输出：

```text
subject=C=CN, ST=Tianjin, L=Tianjin, O=Lab, OU=IT, CN=example.com
issuer=C=CN, ST=Tianjin, L=Tianjin, O=Lab, OU=IT, CN=example.com
notBefore=Nov 19 02:18:38 2026 GMT
notAfter=Nov 19 02:18:38 2027 GMT
```

Subject 与 Issuer 相同，符合本次自签名证书的特征。有效期为 2026-11-19 至 2027-11-19（GMT）。

### 5.7 检查 SAN

```bash
openssl x509 -in server.crt -noout -ext subjectAltName
```

实际输出：

```text
X509v3 Subject Alternative Name:
    DNS:example.com, DNS:www.example.com
```

证书 SAN 包含 `example.com` 和 `www.example.com`。现代客户端主要依据 SAN 验证访问域名是否匹配证书。

### 5.8 验证证书与私钥匹配

从证书提取公钥并计算摘要：

```bash
openssl x509 -in server.crt -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | sha256sum
```

从私钥导出对应公钥并计算摘要：

```bash
openssl pkey -in server.key -pubout -outform DER \
  | sha256sum
```

两条命令实际输出的 SHA-256 摘要相同：

```text
77cee65dbd540a552cc68d805adc5e624921650dec1a62b772241903ddb6577a  -
```

这说明证书中的公钥与私钥对应的公钥一致，证书和私钥匹配。

### 5.9 验证自签名证书

```bash
openssl verify -CAfile server.crt server.crt
```

实际输出：

```text
server.crt: OK
```

这里将自签名证书显式指定为信任证书，因此 OpenSSL 验证通过。这不表示公共浏览器默认信任该证书。

### 5.10 实验文件结构

```text
https-lab/
├── server.key    # RSA 私钥，必须保密
├── server.csr    # 证书签名请求
└── server.crt    # 自签名证书
```

## 6. 后续知识：公共 CA、ACME 与 Nginx

以下内容作为原理和配置参考保留，本轮没有执行对应部署。

### 6.1 公共 CA 与 Let's Encrypt

公共 CA 的根证书已被主流操作系统或浏览器信任库收录。服务器通过域名控制权等验证后，可以获得由受信任 CA 签发的证书。

Let's Encrypt 是提供免费自动化证书签发服务的公共 CA，使用 ACME 协议自动化证书申请、域名验证、签发和续期。申请真实域名证书需要拥有域名控制权；本实验中的 `example.com` 是示例域名，不能据此申请属于自己的真实网站证书。

### 6.2 ACME 常见验证方式

- **HTTP-01**：CA 要求申请者在域名对应 Web 服务的指定 HTTP 路径提供验证内容，通常需要公网可访问的 TCP 80 端口。
- **DNS-01**：申请者在域名 DNS 中添加指定 TXT 记录，可用于无法开放 HTTP 验证端口或需要申请通配符证书等场景。

### 6.3 Certbot 与续期

Certbot 是常见的 ACME 客户端，可协助申请、安装和续期证书。真实部署时需要结合操作系统、Web 服务、域名解析和网络条件选择安装及验证方式。

自动续期还应检查定时任务、续期结果、证书有效期，并安排续期后的服务加载或重载。

### 6.4 Nginx HTTPS 配置示意

以下为配置结构示意，未在本轮实验环境中执行：

```nginx
server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    location / {
        proxy_pass http://app;
    }
}
```

HTTP 到 HTTPS 跳转通常通过单独监听 80 端口的 Server 块返回 301 或 308 重定向。实际部署前应确认域名解析、证书 SAN、证书链、私钥权限和上游服务地址。

### 6.5 HTTPS 常见排查方向

| 现象 | 优先检查 |
|---|---|
| 浏览器提示证书不受信任 | 签发者、证书链、客户端信任库 |
| 域名不匹配 | SAN 是否包含当前访问域名 |
| 证书过期 | notAfter、续期任务和实际部署的证书文件 |
| Nginx 无法启动 | 配置语法、证书路径、私钥路径和权限 |
| TLS 握手失败 | TLS 版本、密码套件、证书链及客户端兼容性 |
| HTTPS 返回 502 | Nginx 上游应用状态、网络和代理配置 |

常用检查命令：

```bash
# 检查 Nginx 配置语法
sudo nginx -t

# 查看证书信息
openssl x509 -in server.crt -noout -text

# 检查 TLS 连接与证书链（替换为真实域名）
openssl s_client -connect example.com:443 -servername example.com

# 查看证书有效期
openssl x509 -in server.crt -noout -dates
```

## 7. 面试知识复盘

**HTTPS 与 HTTP 有什么区别？**

HTTP 本身不提供 TLS 传输保护；HTTPS 是 HTTP 运行在 TLS 之上，通过 TLS 提供通信加密、身份认证和完整性保护。

**证书的作用是什么？**

服务器证书将服务器身份信息与公钥绑定，并由签发者签名。客户端通过验证证书和信任链确认服务器身份；TLS 再通过握手协商会话密钥，保护后续通信。

**为什么自签名证书会出现浏览器警告？**

自签名证书通常不在浏览器默认信任链中。它可用于测试环境建立加密连接，但客户端无法仅凭公共信任链确认其身份。受控内网可通过部署内部 CA 并向客户端分发根证书建立信任。

**CSR 是什么？为什么要提交给 CA？**

CSR 是证书签名请求，包含申请者信息和公钥，并由对应私钥签名。申请者将 CSR 提交给 CA，CA 完成必要验证后签发证书。私钥留在申请者控制的服务器或安全环境中。

**证书和私钥必须匹配吗？**

必须。证书中的公钥必须与服务器持有的私钥相对应。可以分别提取公钥并比较摘要验证。

**SAN 是什么？**

SAN（Subject Alternative Name）用于列出证书适用的 DNS 名称或 IP 地址。现代客户端主要依据 SAN 判断证书是否匹配访问目标。

## 8. 阶段验收与总结

- [x] 理解 HTTP、HTTPS 与 TLS 的关系。
- [x] 理解 TLS 的机密性、身份认证和完整性保护。
- [x] 理解证书、私钥、CSR、SAN 和证书链。
- [x] 了解 DV、OV、EV 的区别。
- [x] 检查 OpenSSL 环境。
- [x] 生成 RSA 2048 私钥并检查私钥。
- [x] 生成 CSR 并查看请求内容。
- [x] 生成包含 DNS SAN 的自签名证书。
- [x] 查看证书 Subject、Issuer 和有效期。
- [x] 验证证书 SAN。
- [x] 验证证书公钥与私钥匹配。
- [x] 使用 OpenSSL 验证自签名证书。
- [ ] 公共 CA 证书申请与 Certbot 自动续期（本轮未实施）。
- [ ] Nginx HTTPS 部署与 HTTP 跳转（本轮未实施）。
- [ ] 真实站点 TLS 验证与故障演练（本轮未实施）。

本模块已完成当前学习范围内的原理学习与 OpenSSL 实操。公共 CA 签发、Nginx HTTPS 上线配置和自动续期保留为后续可补充的实践内容。记录以实际执行结果为准，不将配置示例写成已部署结果。
