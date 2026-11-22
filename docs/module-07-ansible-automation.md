# 模块07：Ansible 自动化运维

## 1. 实验环境

### 1.1 实验架构

采用 Ubuntu 作为 Ansible 控制端，Rocky Linux 作为被控端，通过 SSH 密钥认证实现远程自动化管理。

| 主机 | 操作系统 | IP 地址 | 角色 |
|---|---|---|---|
| Ubuntu | Ubuntu 26.04.1 LTS | 10.2.0.15 | Ansible 控制端 |
| Rocky Linux | Rocky Linux 10.2 | 10.2.4.7 | Ansible 被控端 |

环境信息：

- Ansible Core：2.20.1
- 控制端 Python：3.14.4
- 被控端 Python：3.12
- SSH 认证方式：Ed25519 公钥认证
- 两台主机通过云服务器内网通信

### 1.2 安装 Ansible

在 Ubuntu 控制端安装：

```bash
sudo apt update
sudo apt install ansible-core -y
ansible --version
```

### 1.3 配置 SSH 密钥认证

生成 Ed25519 密钥：

```bash
ssh-keygen -t ed25519
```

将公钥复制到 Rocky 被控端：

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@10.2.4.7
```

验证免密登录：

```bash
ssh root@10.2.4.7
```

能够直接登录 Rocky Linux，说明 SSH 密钥认证配置成功。

---

## 2. Ansible 项目初始化与主机管理

### 2.1 创建项目目录

```bash
mkdir -p ~/ansible-lab
cd ~/ansible-lab

mkdir -p inventory playbooks roles templates files
```

项目目录结构：

```text
ansible-lab/
├── ansible.cfg
├── inventory/
│   └── hosts.ini
├── playbooks/
├── roles/
├── templates/
└── files/
```

### 2.2 配置 Inventory

编辑 `inventory/hosts.ini`：

```ini
[rocky_servers]
rocky ansible_host=10.2.4.7 ansible_user=root
```

查看主机清单：

```bash
ansible-inventory --graph
```

### 2.3 配置 ansible.cfg

项目根目录下的 `ansible.cfg`：

```ini
[defaults]
inventory=./inventory/hosts.ini
remote_user=root
host_key_checking=True
interpreter_python=auto_silent
deprecation_warnings=False
roles_path=./roles
log_path=./ansible.log
```

主要参数：

| 参数 | 作用 |
|---|---|
| inventory | 指定默认主机清单 |
| remote_user | 默认 SSH 登录用户 |
| host_key_checking | SSH 主机密钥检查 |
| interpreter_python | Python 解释器自动发现策略 |
| deprecation_warnings | 控制弃用警告 |
| roles_path | 指定 Role 搜索目录 |
| log_path | 指定 Ansible 日志文件 |

查看生效配置：

```bash
ansible-config dump --only-changed
```

### 2.4 测试主机连通性

```bash
ansible rocky_servers -m ping
```

成功时返回 `pong`。

查看远程主机信息：

```bash
ansible rocky_servers -m command -a "uptime"
ansible rocky_servers -m command -a "free -h"
ansible rocky_servers -m setup -a "filter=ansible_distribution*"
```

Facts 可以获取远程主机的操作系统、内存、CPU、网络接口等信息。

---

## 3. 批量主机配置管理

### 3.1 软件包管理

使用 package 模块安装软件包：

```bash
ansible rocky_servers -m package -a "name=nginx state=present"
```

检查软件包：

```bash
ansible rocky_servers -m command -a "rpm -q nginx"
```

`state=present` 表示确保软件包已经安装。

### 3.2 服务管理

启动 Nginx 并设置开机自启：

```bash
ansible rocky_servers -m service -a "name=nginx state=started enabled=yes"
```

检查服务：

```bash
ansible rocky_servers -m command -a "systemctl is-active nginx"
ansible rocky_servers -m command -a "systemctl is-enabled nginx"
```

### 3.3 防火墙管理

安装 Ansible POSIX Collection：

```bash
ansible-galaxy collection install ansible.posix
```

使用 firewalld 模块放行 HTTP 服务：

```bash
ansible rocky_servers -m ansible.posix.firewalld -a "service=http zone=public permanent=yes state=enabled immediate=yes"
```

检查防火墙规则：

```bash
ansible rocky_servers -m command -a "firewall-cmd --zone=public --list-services"
```

### 3.4 用户、目录和文件管理

创建运维用户：

```bash
ansible rocky_servers -m user -a "name=opuser comment='Operations User' shell=/bin/bash create_home=yes state=present"
```

检查用户：

```bash
ansible rocky_servers -m command -a "id opuser"
```

创建目录：

```bash
ansible rocky_servers -m file -a "path=/opt/ops/data state=directory owner=opuser group=opuser mode=0750"
```

创建文件并设置内容、权限：

```bash
ansible rocky_servers -m copy -a "content='Managed by Ansible\\nOperations data directory\\n' dest=/opt/ops/data/README.txt owner=opuser group=opuser mode=0640"
```

检查文件：

```bash
ansible rocky_servers -m command -a "cat /opt/ops/data/README.txt"
```

### 3.5 幂等性

幂等性是 Ansible 自动化运维的重要特性。

当目标主机已经满足任务要求时，Ansible 不需要再次修改系统状态。例如：

```yaml
- name: Ensure nginx is installed
  ansible.builtin.package:
    name: nginx
    state: present
```

如果 Nginx 已经安装，任务通常返回 `ok`，不会产生实际修改。如果软件包被卸载，再次执行 Playbook 时，Ansible 会重新安装。

幂等性不是简单地表示任务只执行一次，而是保证多次执行后目标状态保持一致。

---

## 4. Playbook 工程化

### 4.1 Playbook 基本结构

Playbook 使用 YAML 格式定义自动化任务：

```yaml
---
- name: Example Playbook
  hosts: rocky_servers
  become: true
  tasks:
    - name: Show system information
      ansible.builtin.command:
        cmd: uptime
      changed_when: false
```

语法检查和执行：

```bash
ansible-playbook playbook.yml --syntax-check
ansible-playbook playbook.yml
```

### 4.2 变量管理

变量可以减少重复配置，提高 Playbook 的可维护性。

示例：

```yaml
---
- name: Deploy nginx
  hosts: rocky_servers
  become: true

  vars:
    nginx_package: nginx
    nginx_dir: /opt/nginx-backup
    nginx_dir_mode: '0755'

  tasks:
    - name: Install nginx
      package:
        name: "{{ nginx_package }}"
        state: present

    - name: Create nginx directory
      file:
        path: "{{ nginx_dir }}"
        state: directory
        owner: root
        group: root
        mode: "{{ nginx_dir_mode }}"
```

通过命令行传入变量：

```bash
ansible-playbook playbooks/nginx-deploy.yml -e "nginx_dir=/opt/nginx-test"
```

命令行传入的 Extra Vars 可以覆盖 Playbook 中定义的普通变量。

### 4.3 Facts 与条件判断

Facts 用于获取被控端信息。示例：

```yaml
---
- name: Check operating system
  hosts: all
  gather_facts: true

  tasks:
    - name: Show distribution
      debug:
        msg: "{{ ansible_distribution }}"

    - name: Execute only on Rocky Linux
      debug:
        msg: "This is Rocky Linux"
      when: ansible_distribution == "Rocky"
```

`when` 用于根据条件决定是否执行任务。

### 4.4 循环 loop

使用 loop 批量创建目录：

```yaml
---
- name: Create directories
  hosts: rocky_servers
  become: true

  tasks:
    - name: Create operations directories
      file:
        path: "{{ item }}"
        state: directory
        owner: opuser
        group: opuser
        mode: "0750"
      loop:
        - /opt/ops/logs
        - /opt/ops/scripts
        - /opt/ops/backup
```

`item` 表示当前循环元素。

### 4.5 Handlers

Handlers 用于在指定任务发生变化时执行后续操作。例如，修改 Nginx 配置后重新加载服务：

```yaml
tasks:
  - name: Deploy nginx configuration
    copy:
      content: |
        server {
            listen 8080;
            server_name localhost;
        }
      dest: /etc/nginx/conf.d/ansible-handler.conf
    notify: Reload nginx

handlers:
  - name: Reload nginx
    service:
      name: nginx
      state: reloaded
```

特点：

- 普通任务通过 `notify` 通知 Handler。
- 只有触发通知时才会执行。
- 同一个 Handler 在一次 Play 中通常只执行一次。
- Handler 默认在当前 Play 的任务结束后执行。

### 4.6 Templates

Jinja2 模板可以根据变量动态生成配置文件。

模板文件 `templates/nginx/site.conf.j2`：

```nginx
server{
   listen {{listen_port}};
   server_name {{server_domain}};

   location / {
   return 200 "hello from ansible template\\n";
   }
}
```

Playbook：

```yaml
---
- name: Deploy nginx template
  hosts: rocky_servers
  become: true

  vars:
    listen_port: 8082
    server_domain: localhost

  tasks:
    - name: Deploy nginx configuration
      template:
        src: ../templates/nginx/site.conf.j2
        dest: /etc/nginx/conf.d/ansible-template.conf
        owner: root
        group: root
        mode: "0644"
      notify: Reload nginx

  handlers:
    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
```

模板中的变量会在执行时替换成实际值。

### 4.7 Roles

Roles 用于将任务、变量、模板和 Handler 组织成独立组件。

创建 Role：

```bash
ansible-galaxy role init roles/nginx
ansible-galaxy role init roles/common
```

项目中的 Role 结构：

```text
roles/
├── common/
│   ├── defaults/main.yml
│   └── tasks/main.yml
└── nginx/
    ├── defaults/main.yml
    ├── handlers/main.yml
    ├── tasks/main.yml
    └── templates/site.conf.j2
```

Common Role 默认变量：

```yaml
---
common_user: opuser
common_directories:
  - /opt/ops/logs
  - /opt/ops/scripts
  - /opt/ops/backup
```

Common Role 任务：

```yaml
---
- name: create files
  file:
    path: "{{item}}"
    state: directory
    owner: "{{common_user}}"
    group: "{{common_user}}"
    mode: "0750"
  loop: "{{common_directories}}"
```

Nginx Role 默认变量：

```yaml
---
nginx_package: nginx
listen_port: 8082
server_domain: localhost
```

Nginx Role 任务：

```yaml
---
- name: install nginx
  package:
    name: "{{nginx_package}}"
    state: present

- name: start and enable nginx
  service:
    name: "{{nginx_package}}"
    state: started
    enabled: true

- name: deploy nginx
  template:
    src: site.conf.j2
    dest: /etc/nginx/conf.d/ansible-role.conf
    owner: root
    group: root
    mode: "0644"
  notify: reload nginx
```

Nginx Role Handler：

```yaml
---
- name: reload nginx
  service:
    name: "{{nginx_package}}"
    state: reloaded
```

Nginx Role 模板的实际部署内容使用监听端口变量和域名变量，返回文本为：

```nginx
server{
   listen {{listen_port}};
   server_name {{server_domain}};

   location / {
   return 200 "hello from ansible role v2 \\n";
   }
}
```

调用 Role：

```yaml
---
- name: deploy nginx using role
  hosts: rocky_servers
  become: true
  roles:
    - nginx
```

---

## 5. 安全与复杂任务

### 5.1 Ansible Vault

Vault 用于加密 Ansible 中的敏感变量文件。

创建加密文件：

```bash
ansible-vault create vault/secrets.yaml
```

查看加密文件：

```bash
ansible-vault view vault/secrets.yaml
```

编辑：

```bash
ansible-vault edit vault/secrets.yaml
```

其他常用命令：

```bash
ansible-vault encrypt secrets.yaml
ansible-vault decrypt secrets.yaml
ansible-vault rekey secrets.yaml
```

执行引用 Vault 文件的 Playbook：

```bash
ansible-playbook playbooks/vault-demo.yml --ask-vault-pass
```

也可以与 Check Mode 配合：

```bash
ansible-playbook playbooks/vault-demo.yml --check --ask-vault-pass
```

Vault 文件可以保存加密后的内容，但 Vault 密码必须单独安全保管，不能提交到 GitHub。不要将真实密码写入普通示例、日志或代码仓库。

### 5.2 权限提升 become

常见配置：

```yaml
---
- name: Configure server
  hosts: rocky_servers
  remote_user: opuser
  become: true
  become_method: sudo
  become_user: root

  tasks:
    - name: Check system
      command: uptime
      changed_when: false
```

| 参数 | 作用 |
|---|---|
| remote_user | SSH 登录用户 |
| become | 是否启用权限提升 |
| become_method | 权限提升方式 |
| become_user | 提权后的目标用户 |

当前实验使用 root SSH 登录，因此部分任务中的 `become: true` 并非必须。生产环境通常建议使用普通运维账户配合 sudo。

### 5.3 Tags

Tags 用于选择性执行任务。

示例：

```yaml
---
- name: Nginx management
  hosts: rocky_servers
  become: true

  tasks:
    - name: Install nginx
      package:
        name: nginx
        state: present
      tags:
        - install

    - name: Deploy nginx configuration
      copy:
        content: "server { listen 80; }"
        dest: /etc/nginx/conf.d/tag-demo.conf
        mode: "0644"
      tags:
        - config

    - name: Check nginx status
      command: systemctl is-active nginx
      changed_when: false
      tags:
        - check
```

查看 Tags：

```bash
ansible-playbook playbooks/nginx-tags.yml --list-tags
```

只执行配置任务：

```bash
ansible-playbook playbooks/nginx-tags.yml --tags config
```

排除指定标签：

```bash
ansible-playbook playbooks/nginx-tags.yml --skip-tags install
```

### 5.4 错误处理

#### ignore_errors

忽略单个任务的失败，让后续任务继续执行：

```yaml
- name: Simulate failure
  command: /bin/false
  ignore_errors: true
```

不应滥用，否则可能掩盖真正的故障。

#### failed_when

根据任务结果自定义失败条件：

```yaml
- name: Check application output
  command: echo "ERROR database connection failed"
  register: log_result
  failed_when: "'ERROR' in log_result.stdout"
```

即使命令返回码为 0，只要输出满足失败条件，任务也会被判定为失败。

#### block、rescue、always

```yaml
tasks:
  - block:
      - name: Execute first task
        debug:
          msg: "Start"

      - name: Simulate failure
        command: /bin/false

      - name: Execute after failure
        debug:
          msg: "This task will not run"

    rescue:
      - name: Handle failure
        debug:
          msg: "Failure detected"

    always:
      - name: Always execute
        debug:
          msg: "Cleanup or final check"
```

- `block`：正常任务区域。
- `rescue`：block 中任务失败后执行补救任务。
- `always`：无论 block 成功还是失败，都会执行。

### 5.5 执行控制

#### 限制目标主机

```bash
ansible-playbook playbooks/nginx-deploy.yml --limit rocky
ansible-playbook playbooks/nginx-deploy.yml --list-hosts
```

#### serial

控制分批执行的主机数量：

```yaml
- name: Rolling update
  hosts: web_servers
  serial: 2
```

例如有10台服务器，`serial: 3` 会分成4批执行。

#### forks

控制 Ansible 全局并发任务数。

查看默认值：

```bash
ansible-config dump | grep '^DEFAULT_FORKS'
```

本实验环境默认值为 5。

#### run_once

```yaml
- name: Execute once
  command: hostname
  run_once: true
```

在当前 Play 的执行批次中只执行一次。如果与 `serial` 配合使用，则每个批次都会执行一次。

#### throttle

限制单个任务的最大并发数：

```yaml
- name: Limit task concurrency
  command: hostname
  throttle: 1
```

| 参数 | 控制范围 |
|---|---|
| forks | 全局并发任务数 |
| serial | 每批处理的主机数量 |
| throttle | 单个任务的并发数 |

---

## 6. 自动化运维综合项目

### 6.1 项目目标

使用 Ansible 自动化完成 Rocky Linux 服务器的基础初始化和 Nginx 部署。

综合使用：

- Inventory 主机清单
- Ansible 配置文件
- Common Role
- Nginx Role
- Jinja2 模板
- Handler
- 变量管理
- 幂等性检查

### 6.2 统一入口 site.yml

```yaml
---
- name: configure rocky linux servers
  hosts: rocky_servers
  become: true
  roles:
    - common
    - nginx
```

### 6.3 执行综合项目

语法检查：

```bash
ansible-playbook playbooks/site.yml --syntax-check
```

执行：

```bash
ansible-playbook playbooks/site.yml
```

### 6.4 验证 Common Role

检查目录：

```bash
ansible rocky_servers -m command -a "ls -l /opt/ops"
```

确认目录存在：

- `/opt/ops/logs`
- `/opt/ops/scripts`
- `/opt/ops/backup`

### 6.5 验证 Nginx Role

检查服务：

```bash
ansible rocky_servers -m command -a "systemctl is-active nginx"
```

检查配置语法：

```bash
ansible rocky_servers -m command -a "nginx -t"
```

本机回环地址验证：

```bash
ansible rocky_servers -m command -a "curl -s http://127.0.0.1:8082"
```

实际返回：

```text
hello from ansible role v2
```

综合项目使用本机回环地址验证服务，没有为了实验额外开放云服务器外部访问端口。

### 6.6 Nginx 配置冲突排查

执行 `nginx -t` 时曾出现：

```text
conflicting server name "localhost" on 0.0.0.0:8082, ignored
```

原因是多个配置文件同时使用：

```nginx
listen 8082;
server_name localhost;
```

排查配置：

```bash
grep -R -nE 'listen|server_name' /etc/nginx/conf.d/
```

发现 `ansible-role.conf` 与 `ansible-template.conf` 都使用 8082，配置内容重复。使用 Ansible file 模块删除了多余的 `ansible-template.conf`。

再次检查：

```bash
nginx -t
```

冲突警告消失，Nginx 配置检查成功。

---

## 7. 自动化质量与故障排查

### 7.1 Check Mode

检查 Playbook 可能产生的变化，但不实际应用修改：

```bash
ansible-playbook playbooks/site.yml --check
```

本次实验结果为 `changed=0`、`failed=0`，说明当前主机已经满足 Playbook 定义的目标状态。

注意：Check Mode 是模拟执行，并不是所有模块都能准确预测实际执行结果。

### 7.2 Diff Mode

查看配置文件变更前后的差异：

```bash
ansible-playbook playbooks/site.yml --check --diff
```

实验中修改 Nginx 模板返回内容后，Diff Mode 显示了配置文件的变化。确认变更后，正式执行：

```bash
ansible-playbook playbooks/site.yml
```

### 7.3 幂等性验证

再次执行：

```bash
ansible-playbook playbooks/site.yml
```

最终结果：

```text
ok=5
changed=0
failed=0
```

说明综合项目再次执行时没有产生不必要的修改，幂等性验证成功。

### 7.4 Ansible 日志

在 `ansible.cfg` 中配置：

```ini
log_path=./ansible.log
```

执行 Playbook 后自动生成日志文件。

查看最近日志：

```bash
tail -n 20 ansible.log
```

日志可以用于：

- 记录 Playbook 执行时间
- 查看任务执行结果
- 定位失败任务
- 分析主机变更情况

日志可能包含主机信息和敏感输出，提交 GitHub 前需要检查并决定是否忽略。

### 7.5 常见故障排查

#### 故障一：Vault 解密失败

错误信息：

```text
Attempting to decrypt but no vault secrets found
```

原因：执行时没有提供 Vault 密码。

解决：

```bash
ansible-playbook playbooks/vault-demo.yml --ask-vault-pass
```

#### 故障二：Role 找不到

原因：Ansible 没有在项目目录中找到 Role。

在 `ansible.cfg` 中配置：

```ini
roles_path=./roles
```

#### 故障三：Nginx 配置冲突

原因：多个 server block 使用相同监听端口和 server_name。

解决步骤：

1. 使用 grep 检查配置文件。
2. 找到重复的 server block。
3. 确认是否存在重复配置。
4. 删除不需要的配置。
5. 使用 `nginx -t` 再次检查。

#### 故障四：软件包不存在

实验中使用了不存在的软件包名称：

```yaml
- name: Install wrong package
  package:
    name: nginx-package-not-exist
    state: present
```

执行时报错：

```text
No package nginx-package-not-exist available.
```

原因是软件仓库中没有对应的软件包。修改为正确的软件包名称后重新执行，任务成功。

#### 故障五：通配符没有展开

使用 command 模块执行带通配符的命令时，通配符不会自动展开，因为 `command` 模块不会通过 Shell 解释命令。

需要 Shell 特性时可以使用：

```yaml
- name: Find nginx configuration files
  shell: ls /etc/nginx/conf.d/*.conf
```

生产环境应优先考虑使用适合的 Ansible 模块，而不是依赖 Shell 命令。

---

## 8. 常用命令速查

| 命令 | 用途 |
|---|---|
| `ansible --version` | 查看 Ansible 版本 |
| `ansible-inventory --graph` | 查看主机清单结构 |
| `ansible-inventory --list` | 查看完整 Inventory |
| `ansible all -m ping` | 测试主机连通性 |
| `ansible all -m setup` | 收集远程主机 Facts |
| `ansible-playbook xxx.yml` | 执行 Playbook |
| `ansible-playbook xxx.yml --syntax-check` | 语法检查 |
| `ansible-playbook xxx.yml --check` | 模拟执行 |
| `ansible-playbook xxx.yml --diff` | 显示变更差异 |
| `ansible-playbook xxx.yml --list-hosts` | 查看目标主机 |
| `ansible-playbook xxx.yml --list-tasks` | 查看任务列表 |
| `ansible-playbook xxx.yml --list-tags` | 查看标签 |
| `ansible-playbook xxx.yml --tags config` | 按标签执行 |
| `ansible-playbook xxx.yml --limit rocky` | 限制执行主机 |
| `ansible-config dump --only-changed` | 查看修改过的配置 |
| `ansible-galaxy role init roles/name` | 初始化 Role |
| `ansible-vault create file.yml` | 创建加密文件 |
| `ansible-vault view file.yml` | 查看加密文件 |

---

## 9. 面试知识点

### 9.1 Ansible 是什么？

Ansible 是一种开源自动化运维工具，可以通过 SSH 管理远程主机，实现软件部署、配置管理、服务管理和批量任务执行。

主要特点：

- 通常不需要在被控端安装 Agent
- 使用 SSH 进行远程管理
- 使用 YAML 编写 Playbook
- 支持 Inventory 主机清单
- 通过模块执行具体任务
- 支持幂等性

### 9.2 Ad-Hoc 与 Playbook 的区别

| 对比 | Ad-Hoc | Playbook |
|---|---|---|
| 使用方式 | 命令行 | YAML 文件 |
| 适用场景 | 临时操作 | 自动化流程 |
| 任务组织 | 单条命令 | 多任务编排 |
| 复用性 | 较低 | 较高 |

### 9.3 什么是 Ansible Role？

Role 是 Ansible 的任务组织方式，可以将任务、变量、模板、文件和 Handler 等拆分到不同目录中，适合大型项目和重复使用的自动化配置。

### 9.4 什么是 Handler？

Handler 是由任务通知触发的特殊任务，常用于配置文件发生变化后重启或重新加载服务。

### 9.5 什么是 Ansible Vault？

Vault 用于加密敏感变量和文件，例如数据库密码、API Token 等，防止敏感信息以明文形式保存在代码仓库中。

### 9.6 什么是幂等性？

幂等性表示多次执行同一个自动化任务后，系统最终保持在相同的目标状态。例如，确保 Nginx 已安装的任务不会在每次执行时都重新安装软件包。

### 9.7 如何排查 Playbook 执行失败？

常用排查步骤：

1. 使用 `--syntax-check` 检查语法。
2. 使用 `--list-hosts` 确认目标主机。
3. 使用 `--check` 预检查任务。
4. 查看失败任务的错误信息。
5. 检查远程主机服务、软件包、权限和配置。
6. 使用 `-vvv` 获取详细执行信息。
7. 修复问题后重新执行并验证结果。

---

## 10. 本模块总结

本模块围绕 Ubuntu 控制端和 Rocky Linux 被控端，完成了 Ansible 自动化运维实践。

主要完成内容：

- Ansible 控制端安装与配置
- SSH Ed25519 密钥认证
- Inventory 主机清单管理
- Ad-Hoc 批量执行
- 软件包、服务、防火墙、用户和文件管理
- Playbook 变量、Facts、条件、循环
- Handlers、Jinja2 Templates 和 Roles
- Ansible Vault 敏感信息加密
- become 权限提升
- Tags、错误处理和执行控制
- Common 与 Nginx Role 综合部署
- Check Mode、Diff Mode 和幂等性验证
- Ansible 日志与故障排查

最终通过 `playbooks/site.yml` 统一调用 Common 和 Nginx Role，实现 Rocky Linux 服务器的基础目录初始化与 Nginx 自动化部署。

本模块重点从单条 Ansible 命令过渡到可复用、可维护的自动化运维项目。
