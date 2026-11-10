# SSH安全加固

## 1. 实施目标
通过ssh配置调整，实现基于公钥的远程登录，禁止Root直接登录和密码认证，降低暴力破解风险

## 2. 关键配置文件

- /etc/ssh/sshd_config
- /etc/ssh/sshd_config.d/50-cloud-init.conf    #50-cloud-init.conf 是 cloud-init 工具在系统初始化时自动生成的配置，专门用来覆盖主配置文件中的设置。子文件的配置优先级更高（可删掉）

## 3. 核心配置

主配置文件中设置：

```text
PermitRootLogin no

cloud-init 配置文件中设置：
PasswordAuthentication no    

```
## 4.配置检查

```bash
检查 SSH 配置语法：
sudo sshd -t

查看最终生效配置：
sudo sshd -T | grep -Ei \
'^(port|permitrootlogin|passwordauthentication|pubkeyauthentication)'

实际检查结果：
port 22
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no

```
## 5.公钥登录验证
1.生成 SSH 密钥对（公钥 + 私钥）的标准命令是 ssh-keygen（客户端）

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
-t ed25519：指定密钥类型，推荐 ed25519（现代、短、安全）。老系统不支持时用 -t rsa -b 4096。
-C：注释，通常写邮箱或用途标识，会附加在公钥末尾，方便识别。
```

执行后会交互式询问：

- 保存路径：默认 ~/.ssh/id_ed25519，直接回车即可

- passphrase：给私钥加一层密码保护，可留空（回车两次）。留空方便自动化，但私钥文件泄露风险更高

- 生成后得到两个文件：

  ~/.ssh/id_ed25519 —— 私钥，绝不能外传

  ~/.ssh/id_ed25519.pub —— 公钥，可以随便分发

2.客户端使用 Ed25519 密钥进行 SSH 登录

```bash
ssh -i ~/.ssh/id_ed25519 user@server_ip
```
3.服务端公钥文件：

```text
/home/ubuntu/.ssh/authorized_keys
```
4.客户端密钥：

```text
客户端 Ed25519 私钥（保存在本地，不上传仓库）
```
5.把公钥传到服务器

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server  #它会自动把公钥追加到服务器的 ~/.ssh/authorized_keys 并设好权限。如果没有 ssh-copy-id，就手动复制 id_ed25519.pub 的内容，粘贴到服务器上的 ~/.ssh/authorized_keys 里
```
验证结果：
- 使用 SSH 公钥登录成功。
- 禁用公钥、仅使用密码进行测试时，

  ```bash
  ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no user@server_ip
  返回：
  Permission denied (publickey).
  ```

说明密码认证已被禁止，公钥认证正常工作。

