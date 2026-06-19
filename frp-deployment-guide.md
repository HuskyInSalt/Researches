# FRP 内网穿透完整部署方案

> 适用场景：家里没有公网 IP，通过阿里云 ECS（2核4G/5Mbps/199元）做中转，远程 SSH、传文件、跑命令
> 文档版本：v1.0 / 2025-06
> FRP 版本：v0.62.0（最新稳定版，使用 TOML 格式）

---

## 一、操作系统选择

### 推荐：Ubuntu 22.04 LTS

| 操作系统 | 优点 | 缺点 | 推荐度 |
|---------|------|------|--------|
| **Ubuntu 22.04 LTS** | 文档多、apt 源新、问题好搜 | 资源占用稍大 | ⭐⭐⭐⭐⭐ |
| Debian 12 | 更精简、稳定性强 | 部分新软件包较旧 | ⭐⭐⭐⭐ |
| Rocky Linux 9 | 企业级、CentOS 替代 | 国内源偶尔抽风 | ⭐⭐⭐ |
| Alibaba Cloud Linux 3 | 阿里定制、性能优化 | 出问题排查资料少 | ⭐⭐ |

**选择建议：**
- **第一次用**：Ubuntu 22.04 LTS
- **追求极简**：Debian 12
- **家里服务器**：如果跑 docker/服务多，推荐 Ubuntu 22.04 或 Debian 12

**安装时注意：**
- 不要勾选"安装云监控/云安全中心"等额外组件（这些是收费的）
- 设置一个强 root 密码
- 记住服务器的公网 IP

---

## 二、整体架构

```
┌─────────────────┐                ┌──────────────────┐
│  阿里云 ECS      │                │  家里 Linux       │
│  (frps 服务端)   │ ◄────────────►│  (frpc 客户端)    │
│  公网 IP         │   7000 加密隧道  │  192.168.x.x     │
│  Ubuntu 22.04   │                │  Ubuntu/Debian   │
└─────────────────┘                └──────────────────┘
       ▲
       │ 你在外面（咖啡厅/出差）
       ▼
  ssh -p 6000 root@阿里云IP
       ↓
  frps 收到 → 转发到家里 frpc
       ↓
  家里 frpc 转发到本地 SSH (127.0.0.1:22)
       ↓
  登录成功！
```

---

## 三、端口规划

部署前先把要用的端口规划好，避免后面乱：

| 端口 | 用途 | 服务端 | 客户端 | 阿里云安全组 |
|------|------|--------|--------|--------------|
| 7000 | frp 主控连接 | ✅ 监听 | ❌ | ✅ 放行 TCP |
| 7500 | frp Dashboard | ✅ 监听 | ❌ | ✅ 放行 TCP |
| 6000 | SSH 远程登录 | ❌ | ✅ 远程端口 | ✅ 放行 TCP |
| 6001 | 文件传输/备用 SSH | ❌ | ✅ 远程端口 | ✅ 放行 TCP |
| 6002-6010 | 备用（Web/远程桌面等） | ❌ | ✅ 远程端口 | ✅ 视情况 |

**重要：所有端口都需要在阿里云安全组放行！**

---

## 四、服务端部署（阿里云 ECS）

### Step 1: 初始化系统

```bash
# SSH 登录阿里云服务器
ssh root@你的阿里云公网IP

# 更新系统
apt update && apt upgrade -y

# 安装必要工具
apt install -y curl wget vim ufw

# 设置时区
timedatectl set-timezone Asia/Shanghai
```

### Step 2: 配置防火墙

```bash
# 启用 UFW
ufw default deny incoming
ufw default allow outgoing

# 放行 frp 相关端口
ufw allow 22/tcp      # SSH（管理用）
ufw allow 7000/tcp    # frp 主控
ufw allow 7500/tcp    # frp Dashboard
ufw allow 6000:6010/tcp  # 远程映射端口

# 启用防火墙
ufw enable
ufw status verbose
```

**⚠️ 重要：阿里云安全组也要放行这些端口**
- 登录阿里云控制台 → ECS → 安全组 → 配置规则
- 手动放行 22、7000、7500、6000-6010（**入方向**）

### Step 3: 安装 frps（服务端）

```bash
# 1. 下载最新版本（v0.62.0）
cd /opt
wget https://github.com/fatedier/frp/releases/download/v0.62.0/frp_0.62.0_linux_amd64.tar.gz

# 2. 解压
tar -xzf frp_0.62.0_linux_amd64.tar.gz
mv frp_0.62.0_linux_amd64 frp
cd frp

# 3. 删除客户端文件（服务端只需要 frps）
rm -f frpc frpc.toml

# 4. 创建配置目录
mkdir -p /etc/frp
cp frps.toml /etc/frp/

# 5. 复制可执行文件
cp frps /usr/local/bin/
chmod +x /usr/local/bin/frps
```

### Step 4: 配置 frps.toml

**先生成强 token**（关键！客户端必须用同一个）：

```bash
openssl rand -base64 32
# 输出类似：a8Kj3mN9pQ2rT5vW8xZ1bC4dF6gH7iJ0kL9mN=
# 把这个值复制下来
```

**编辑 `/etc/frp/frps.toml`**：

```toml
# ============================
# frps 服务端配置
# ============================

# 监听地址（一般不改）
bindAddr = "0.0.0.0"

# 客户端连接端口
bindPort = 7000

# ====== 身份认证（关键！客户端必须一致）======
auth.method = "token"
auth.token = "把openssl生成的强密码粘贴到这里"

# ====== Dashboard（可选，用于看连接状态）======
webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "改个强密码"

# ====== 允许客户端使用的端口范围（防止滥用）======
allowPorts = [
  { start = 6000, end = 6010 }
]

# ====== 日志配置 ======
log.to = "/var/log/frps.log"
log.level = "info"
log.maxDays = 7

# ====== 性能调优（5Mbps 带宽下推荐）======
transport.maxPoolCount = 5
transport.tcpKeepalive = 75
```

### Step 5: 配置 systemd 开机自启

创建 `/etc/systemd/system/frps.service`：

```ini
[Unit]
Description=FRP Server
Documentation=https://github.com/fatedier/frp
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
StandardOutput=append:/var/log/frps-stdout.log
StandardError=append:/var/log/frps-stderr.log
Restart=on-failure
RestartSec=5s
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

**启动并设置开机自启**：

```bash
systemctl daemon-reload
systemctl enable frps
systemctl start frps
systemctl status frps
```

### Step 6: 验证服务端

```bash
# 1. 检查服务状态
systemctl status frps

# 2. 检查端口监听
ss -tlnp | grep frps
# 应该看到 7000 和 7500 在监听

# 3. 检查日志
tail -f /var/log/frps.log

# 4. 浏览器访问 Dashboard
# http://你的阿里云IP:7500
# 用配置的 user/password 登录
```

---

## 五、客户端部署（家里 Linux 服务器）

### Step 1: 准备家里服务器

确保家里服务器：
- 能访问外网（家庭宽带能上网）
- SSH 服务在运行（22 端口）
- 有 root 权限

```bash
# 检查 SSH 服务状态
systemctl status sshd

# 如果没装
apt install -y openssh-server
systemctl enable --now sshd
```

### Step 2: 安装 frpc（客户端）

**家里如果是 Linux x86：**

```bash
cd /opt
wget https://github.com/fatedier/frp/releases/download/v0.62.0/frp_0.62.0_linux_amd64.tar.gz
tar -xzf frp_0.62.0_linux_amd64.tar.gz
mv frp_0.62.0_linux_amd64 frp
cd frp
rm -f frps frps.toml

mkdir -p /etc/frp
cp frpc.toml /etc/frp/
cp frpc /usr/local/bin/
chmod +x /usr/local/bin/frpc
```

**ARM 设备（树莓派等）** 把 `amd64` 换成 `arm64`：
```bash
wget https://github.com/fatedier/frp/releases/download/v0.62.0/frp_0.62.0_linux_arm64.tar.gz
```

**家里如果是 Windows/Mac：**

| 平台 | 下载文件 |
|------|---------|
| Windows | `frp_0.62.0_windows_amd64.zip` |
| macOS Intel | `frp_0.62.0_darwin_amd64.tar.gz` |
| macOS Apple Silicon | `frp_0.62.0_darwin_arm64.tar.gz` |

下载解压即可，下面以 Linux 继续。

### Step 3: 配置 frpc.toml

编辑 `/etc/frp/frpc.toml`：

```toml
# ============================
# frpc 客户端配置
# ============================

# 服务端地址（填你的阿里云公网 IP）
serverAddr = "123.45.67.89"   # ← 改成你的阿里云IP
serverPort = 7000

# 身份认证（必须和服务端一致）
auth.method = "token"
auth.token = "和服务端auth.token完全一致"

# 全局传输配置
transport.tcpMux = true
transport.poolCount = 5

# 客户端管理界面（可选）
webServer.addr = "127.0.0.1"
webServer.port = 7400
webServer.user = "admin"
webServer.password = "强密码"

# 日志
log.to = "/var/log/frpc.log"
log.level = "info"
log.maxDays = 7

# ============================
# 代理配置（你要暴露的服务）
# ============================

# 代理 1：SSH 远程登录
[[proxies]]
name = "home-ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000
# 5Mbps 必开压缩
transport.useCompression = true

# 代理 2：备用 SSH（防止端口冲突）
[[proxies]]
name = "home-ssh-backup"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6001
transport.useCompression = true

# 代理 3：家里 NAS/文件服务（如果有 SMB）
# [[proxies]]
# name = "home-smb"
# type = "tcp"
# localIP = "127.0.0.1"
# localPort = 445
# remotePort = 6445

# 代理 4：家里 Web 服务（如果有）
# [[proxies]]
# name = "home-web"
# type = "http"
# localPort = 80
# customDomains = ["home.example.com"]
```

### Step 4: 配置 systemd 开机自启

创建 `/etc/systemd/system/frpc.service`：

```ini
[Unit]
Description=FRP Client
Documentation=https://github.com/fatedier/frp
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/frpc -c /etc/frp/frpc.toml
StandardOutput=append:/var/log/frpc-stdout.log
StandardError=append:/var/log/frpc-stderr.log
Restart=on-failure
RestartSec=5s
LimitNOFILE=1048576
StartLimitIntervalSec=0

[Install]
WantedBy=multi-user.target
```

**启动**：

```bash
systemctl daemon-reload
systemctl enable frpc
systemctl start frpc
systemctl status frpc
```

### Step 5: 验证客户端

```bash
# 1. 检查服务状态
systemctl status frpc

# 2. 查看连接日志（关键！）
tail -50 /var/log/frpc.log
# 应该看到 "start proxy success" 字样

# 3. 检查 Dashboard
# 浏览器访问：http://阿里云IP:7500
# 登录后能看到家里客户端在线，代理列表里有 home-ssh
```

---

## 六、连通性测试

### 测试 1：SSH 远程登录

**在 Mac/Linux 工作机器上**：

```bash
# 命令格式：ssh -p 远程端口 用户@阿里云IP
ssh -p 6000 root@阿里云公网IP
# 第一次会提示输入 yes
# 然后输入家里服务器的密码
```

**在 Windows 上**：
- PowerShell：`ssh -p 6000 root@阿里云公网IP`
- 或用 PuTTY：主机填阿里云IP，端口填 6000

### 测试 2：传文件

```bash
# 传单个文件
scp -P 6000 -C 本地文件.txt root@阿里云IP:/目标路径/

# 传整个目录
scp -P 6000 -C -r 本地目录/ root@阿里云IP:/目标路径/

# 拉取家里文件到本地
scp -P 6000 -C root@阿里云IP:/家里文件.txt ./
```

### 测试 3：性能验证

```bash
# 在家里服务器上生成 100MB 测试文件
dd if=/dev/zero of=/tmp/testfile bs=1M count=100

# 从工作机器拉取，观察速度
scp -P 6000 -C root@阿里云IP:/tmp/testfile /tmp/
# 5Mbps 理论上限 625KB/s，100MB 大约 2-3 分钟
```

---

## 七、安全加固

### 1. SSH 密钥登录（强烈推荐）

**在你工作机器上生成密钥**：

```bash
# Mac/Linux
ssh-keygen -t ed25519 -C "your_email@example.com"

# 公钥传到家里服务器（首次还是用密码登录）
ssh-copy-id -p 22 root@家里内网IP
# 或者手动：把 ~/.ssh/id_ed25519.pub 内容追加到家里的 ~/.ssh/authorized_keys

# 测试免密登录
ssh root@家里内网IP
```

**关闭家里服务器的密码登录**：

```bash
vim /etc/ssh/sshd_config

# 修改这两行
PasswordAuthentication no
PermitRootLogin prohibit-password  # 或 yes 但配合密钥

systemctl restart sshd
```

### 2. 阿里云安全组最小化

只放行需要的端口，其他全部拒绝：
- 22（管理 SSH）
- 7000（frp 主控）
- 6000-6010（远程映射端口）
- 7500（Dashboard，建议改成高位端口 + 强密码）

**Dashboard 限制访问 IP（推荐）：**
阿里云安全组里，把 7500 端口的访问来源改成你的常用 IP（比如家里宽带 IP）。**只允许指定 IP 访问** Dashboard。

### 3. frp token 强密码

确保 `auth.token` 至少 32 位，包含大小写字母、数字：
```bash
openssl rand -base64 32
```

### 4. 启用传输加密和压缩

在 `frpc.toml` 代理配置里：
```toml
[[proxies]]
name = "home-ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000

transport.useEncryption = true   # 加密
transport.useCompression = true  # 压缩（小带宽必开）
```

### 5. 定期更新 frp

```bash
# 服务端升级（重复安装步骤，最后重启）
cd /opt/frp
# 下载新版本
wget https://github.com/fatedier/frp/releases/download/v0.6X.0/frp_0.6X.0_linux_amd64.tar.gz
# ... 解压替换
systemctl restart frps
```

---

## 八、5Mbps 带宽下的体验优化

### SSH 提速（家里服务器 `~/.ssh/config`）

```
Host home
  HostName 阿里云IP
  Port 6000
  User root
  Compression yes
  ControlMaster auto
  ControlPath ~/.ssh/ssh-%r@%h:%p
  ControlPersist 10m
  ServerAliveInterval 30
  ServerAliveCountMax 3
```

之后直接 `ssh home` 就连上。第一次会要求输入密码，之后免密。

### rsync 传文件优化

```bash
# 启用压缩 + 部分传输
rsync -avz --progress --partial 本地文件/ root@阿里云IP:/目标/

# 大量小文件用 tar 先打包
tar czf - 源目录/ | ssh -p 6000 root@阿里云IP "tar xzf - -C /目标/"
```

### scp 限速

```bash
# -l 4000 限制 4Mbps（kbit/s），避免占满带宽影响其他操作
scp -P 6000 -C -l 4000 大文件.zip root@阿里云IP:/目标/
```

### Git 加速

```bash
# 全局开启压缩
git config --global core.compression 9
git config --global pack.windowMemory 256m
```

---

## 九、进阶使用场景

### 场景 1：远程桌面（Windows）

**Windows RDP：**
```toml
# frpc.toml 追加
[[proxies]]
name = "home-rdp"
type = "tcp"
localIP = "127.0.0.1"
localPort = 3389
remotePort = 6389
```

使用：Windows 远程桌面客户端 → `阿里云IP:6389`

### 场景 2：家里 NAS 文件共享

**SMB 共享（Windows 网络邻居）：**
```toml
[[proxies]]
name = "home-smb"
type = "tcp"
localIP = "127.0.0.1"
localPort = 445
remotePort = 6445
```

**WebDAV（更安全）：**
```toml
[[proxies]]
name = "home-webdav"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8080
remotePort = 68080
```

### 场景 3：家里 Web 服务

```toml
# frpc.toml
[[proxies]]
name = "home-blog"
type = "http"
localPort = 80
customDomains = ["home.example.com"]
```

需要：
1. 域名解析到阿里云 IP
2. 阿里云安全组放行 80 端口
3. frps 配置：`vhostHTTPPort = 80`

### 场景 4：远程开发

```toml
# Jupyter Notebook
[[proxies]]
name = "jupyter"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8888
remotePort = 6888

# VSCode Server
[[proxies]]
name = "code-server"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8080
remotePort = 6808
```

---

## 十、监控与维护

### 1. 监控脚本（家里服务器）

创建 `/usr/local/bin/check-frp.sh`：

```bash
#!/bin/bash
# 检查 frp 服务状态
ALERT_LOG="/var/log/frp-alert.log"

if ! pgrep frpc > /dev/null; then
  echo "[$(date)] 警告：frpc 进程不存在，正在重启..." >> $ALERT_LOG
  systemctl restart frpc
fi

# 检查外网连通性
if ! curl -s --max-time 5 http://阿里云IP:7500 > /dev/null; then
  echo "[$(date)] 警告：frps 服务端不可达！" >> $ALERT_LOG
fi
```

**加 crontab 定时执行**：

```bash
chmod +x /usr/local/bin/check-frp.sh
crontab -e
# 添加：
*/5 * * * * /usr/local/bin/check-frp.sh
```

### 2. 常用排查命令

```bash
# 查看 frpc 状态
systemctl status frpc
journalctl -u frpc -f  # 实时日志

# 查看 frps 状态
systemctl status frps
journalctl -u frps -f

# 检查端口监听
ss -tlnp | grep -E '7000|6000|7500'

# 测试连通性
nc -zv 阿里云IP 7000

# 查看 Dashboard API
curl -u admin:密码 http://阿里云IP:7500/api/status
```

### 3. 配置文件热重载

修改 `frpc.toml` 后，无需重启：

```bash
# 在家里服务器上
frpc reload -c /etc/frp/frpc.toml
```

服务端配置热重载（v0.52+）：

```bash
# 在阿里云上
frps reload -c /etc/frp/frps.toml
```

---

## 十一、常见问题排查

### Q1：客户端连不上服务端

```bash
# 1. 检查 frpc 日志
tail -50 /var/log/frpc.log
# 常见错误：connection refused / dial timeout / auth failed

# 2. 检查服务端是否在运行
ssh root@阿里云IP "systemctl status frps"

# 3. 检查阿里云安全组：7000 端口是否放行
# 阿里云控制台 → ECS → 安全组 → 手动检查

# 4. 检查服务端防火墙
ssh root@阿里云IP "ufw status"

# 5. 测试连通性
ssh root@阿里云IP "nc -zv 阿里云IP 7000"
```

### Q2：能连上但 SSH 失败

```bash
# 检查家里 SSH 服务
systemctl status sshd
ss -tlnp | grep :22

# 检查 frpc 代理配置
grep -A 5 "home-ssh" /etc/frp/frpc.toml

# 手动测试代理
ssh -p 6000 -v root@阿里云IP
# -v 看详细日志
```

### Q3：连接经常断

```bash
# 编辑 frpc.service，增加重试
vim /etc/systemd/system/frpc.service
# 确保有以下配置：
Restart=always
RestartSec=5
StartLimitIntervalSec=0

systemctl daemon-reload
systemctl restart frpc
```

### Q4：传大文件很慢

- 确认启用了 `transport.useCompression = true`（5Mbps 必开）
- 家里宽带上行带宽是多少？联系运营商确认
- 阿里云 5Mbps 是固定带宽，速度应该稳定
- 检查家里路由器的 QoS 设置

### Q5：忘记 Dashboard 密码

```bash
# 编辑配置文件
vim /etc/frp/frps.toml
# 改 webServer.password 的值
systemctl restart frps
```

### Q6：Token 不匹配

- 服务端和客户端的 `auth.token` 必须**完全一致**（包括大小写、空格、换行）
- 改完后两边都要 `systemctl restart`

---

## 十二、备份与恢复

### 备份

```bash
# 备份关键文件（服务端和客户端都做）
tar czf frp-backup-$(date +%Y%m%d).tar.gz \
  /etc/frp/ \
  /etc/systemd/system/frps.service \
  /etc/systemd/system/frpc.service

# 上传到对象存储或下载到本地
```

### 恢复

```bash
# 上传备份文件
tar xzf frp-backup-20250605.tar.gz -C /
systemctl daemon-reload
systemctl restart frps   # 服务端
systemctl restart frpc   # 客户端
```

---

## 十三、一键安装脚本

如果你想用脚本一键部署，参考下面这两个：

### 服务端一键安装（阿里云）

```bash
#!/bin/bash
# install-frps.sh - 在阿里云 ECS 上执行
set -e

echo "🚀 开始安装 frps 服务端..."

# 安装依赖
apt update && apt install -y wget ufw

# 配置防火墙
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 7000/tcp
ufw allow 7500/tcp
ufw allow 6000:6010/tcp
ufw --force enable

# 下载 frp
cd /opt
wget -q https://github.com/fatedier/frp/releases/download/v0.62.0/frp_0.62.0_linux_amd64.tar.gz
tar -xzf frp_0.62.0_linux_amd64.tar.gz
mv frp_0.62.0_linux_amd64 frp
cd frp
rm -f frpc frpc.toml

mkdir -p /etc/frp
cp frps.toml /etc/frp/
cp frps /usr/local/bin/
chmod +x /usr/local/bin/frps

# 生成 token
TOKEN=$(openssl rand -base64 32)
DASHBOARD_PASS=$(openssl rand -base64 16)

# 写配置
cat > /etc/frp/frps.toml << EOF
bindAddr = "0.0.0.0"
bindPort = 7000

auth.method = "token"
auth.token = "$TOKEN"

webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "$DASHBOARD_PASS"

allowPorts = [
  { start = 6000, end = 6010 }
]

log.to = "/var/log/frps.log"
log.level = "info"
log.maxDays = 7

transport.maxPoolCount = 5
transport.tcpKeepalive = 75
EOF

# 写 systemd
cat > /etc/systemd/system/frps.service << 'EOF'
[Unit]
Description=FRP Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
StandardOutput=append:/var/log/frps-stdout.log
StandardError=append:/var/log/frps-stderr.log
Restart=on-failure
RestartSec=5s
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF

# 启动
systemctl daemon-reload
systemctl enable frps
systemctl start frps

echo ""
echo "✅ frps 安装完成！"
echo ""
echo "================================="
echo "Token: $TOKEN"
echo "Dashboard 密码: $DASHBOARD_PASS"
echo "================================="
echo "⚠️ 请把上面这两个值记下来，配置客户端要用！"
echo "Dashboard 地址: http://$(curl -s ifconfig.me):7500"
```

### 客户端一键安装（家里服务器）

```bash
#!/bin/bash
# install-frpc.sh - 在家里服务器上执行
# 用法：bash install-frpc.sh <阿里云IP> <TOKEN>

set -e

if [ $# -ne 2 ]; then
  echo "用法: bash install-frpc.sh <阿里云IP> <TOKEN>"
  echo "示例: bash install-frpc.sh 123.45.67.89 abc123..."
  exit 1
fi

SERVER_IP=$1
TOKEN=$2

echo "🚀 开始安装 frpc 客户端..."

# 安装依赖
apt update && apt install -y wget

# 下载 frp
cd /opt
wget -q https://github.com/fatedier/frp/releases/download/v0.62.0/frp_0.62.0_linux_amd64.tar.gz
tar -xzf frp_0.62.0_linux_amd64.tar.gz
mv frp_0.62.0_linux_amd64 frp
cd frp
rm -f frps frps.toml

mkdir -p /etc/frp
cp frpc.toml /etc/frp/
cp frpc /usr/local/bin/
chmod +x /usr/local/bin/frpc

# 写配置
cat > /etc/frp/frpc.toml << EOF
serverAddr = "$SERVER_IP"
serverPort = 7000

auth.method = "token"
auth.token = "$TOKEN"

transport.tcpMux = true
transport.poolCount = 5

webServer.addr = "127.0.0.1"
webServer.port = 7400
webServer.user = "admin"
webServer.password = "$(openssl rand -base64 16)"

log.to = "/var/log/frpc.log"
log.level = "info"
log.maxDays = 7

[[proxies]]
name = "home-ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000
transport.useCompression = true

[[proxies]]
name = "home-ssh-backup"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6001
transport.useCompression = true
EOF

# 写 systemd
cat > /etc/systemd/system/frpc.service << 'EOF'
[Unit]
Description=FRP Client
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/frpc -c /etc/frp/frpc.toml
StandardOutput=append:/var/log/frpc-stdout.log
StandardError=append:/var/log/frpc-stderr.log
Restart=on-failure
RestartSec=5s
LimitNOFILE=1048576
StartLimitIntervalSec=0

[Install]
WantedBy=multi-user.target
EOF

# 启动
systemctl daemon-reload
systemctl enable frpc
systemctl start frpc

echo ""
echo "✅ frpc 安装完成！"
echo ""
echo "测试连接：在你的工作机器上执行："
echo "  ssh -p 6000 root@$SERVER_IP"
echo ""
echo "查看日志："
echo "  tail -f /var/log/frpc.log"
```

**使用方式**：

```bash
# 1. 先在阿里云上跑服务端脚本
bash install-frps.sh
# 记下输出的 Token

# 2. 在家里服务器上跑客户端脚本
bash install-frpc.sh <阿里云IP> <刚才的Token>
```

---

## 十四、参考链接

- FRP 官方仓库：https://github.com/fatedier/frp
- FRP 官方文档：https://gofrp.org/zh-cn/docs/
- 阿里云 ECS 控制台：https://ecs.console.aliyun.com/
- 阿里云安全组配置：https://help.aliyun.com/document_detail/25471.html

---

**🎉 部署完成后，你就可以像在同一个局域网一样操作家里的服务器了！**
