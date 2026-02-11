# OpenClaw 部署指南（Ubuntu / systemd / loopback）

这份文档整理了从 0 到 1 在 Ubuntu 服务器上安装并以 systemd 方式运行 OpenClaw Gateway 的一套命令（**仅本机监听**），适合 2G 小机器。

---

## 0) 前置

- Ubuntu 20.04/22.04/24.04（任意一版都可）
- 能 sudo
- 对外只需要暴露你自己的 Web/反代（如 nginx），OpenClaw Gateway 建议只在本机回环监听

---

## 1) 安装基础依赖

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg jq
```

---

## 2) 安装 Node.js 22（NodeSource）

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg

echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_22.x nodistro main" \
  | sudo tee /etc/apt/sources.list.d/nodesource.list >/dev/null

sudo apt update
sudo apt install -y nodejs

node -v
npm -v
```

---

## 3) 安装 OpenClaw

```bash
sudo npm install -g openclaw@latest
openclaw --version
```

> 建议：生产环境可以把 `@latest` 换成固定版本，避免自动升级带来不确定性。

---

## 4) 初始化配置（认证/生成配置文件）

```bash
openclaw onboard
```

会在 `~/.openclaw/` 下生成配置与工作区（具体路径以向导输出为准）。

---

## 5) 创建 systemd Gateway 服务（仅本机监听）

下面给两份可直接复制粘贴的模板（任选其一）：

### 方案 A：root 运行

```bash
sudo tee /etc/systemd/system/openclaw-gateway.service >/dev/null <<'EOF'
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/root
Environment=HOME=/root
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ExecStart=/usr/bin/openclaw gateway --port 18789 --bind loopback
Restart=always
RestartSec=3
StandardOutput=journal
StandardError=journal
KillMode=mixed
TimeoutStopSec=60
KillSignal=SIGTERM

[Install]
WantedBy=multi-user.target
EOF
```

### 方案 B：ubuntu 运行（不新增用户）

```bash
sudo tee /etc/systemd/system/openclaw-gateway.service >/dev/null <<'EOF'
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu
Environment=HOME=/home/ubuntu
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ExecStart=/usr/bin/openclaw gateway --port 18789 --bind loopback
Restart=always
RestartSec=3
StandardOutput=journal
StandardError=journal
KillMode=mixed
TimeoutStopSec=60
KillSignal=SIGTERM

[Install]
WantedBy=multi-user.target
EOF
```

注意：如果你切换到 `User=ubuntu`，需要确保 ubuntu 的配置存在于：`/home/ubuntu/.openclaw/openclaw.json`，并且 workspace 路径正确。

---

## 6) 启动并设置开机自启

首次部署/首次启用：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway
```

当你“改过 service 内容”后（例如从 root 切到 ubuntu，或修改 ExecStart/环境变量），就用这两条让改动生效：

```bash
sudo systemctl daemon-reload
sudo systemctl restart openclaw-gateway
```

常用管理命令：

```bash
# 停止/启动/重启 gateway
sudo systemctl stop openclaw-gateway
sudo systemctl start openclaw-gateway
sudo systemctl restart openclaw-gateway
```

进入 TUI（主会话）：

```bash
openclaw tui --session main
```

---

## 7) 查看运行状态/日志

```bash
sudo systemctl status openclaw-gateway --no-pager -l

# 实时日志
sudo journalctl -u openclaw-gateway -f

# 确认只在本机监听
ss -ltnp | grep 18789
```

---

## 8) 远程控制「电脑 Chrome」的 Browser Relay（Windows Node + Chrome 扩展）

> 目标：Gateway 跑在服务器上，让 OpenClaw 能控制你**本机 Chrome 已登录态**（GSC/GA/Cloudflare/X 等）。

### 8.1 Node.js 下载（Windows）

- Node.js 官方下载页（推荐 Node 22 LTS）：https://nodejs.org/en/download
- Windows 预构建安装器直达：https://nodejs.org/en/download/prebuilt-installer

### 8.2 Windows：安装 OpenClaw CLI

```powershell
npm i -g openclaw
openclaw --version
```

### 8.3 Windows：安装/运行 Node Host（让服务器能控制你电脑）

> 需要管理员 PowerShell（创建 Scheduled Task）。

```powershell
openclaw node install --force --host 43.156.245.19 --port 18789
openclaw node restart
openclaw node status
```

前台诊断（最直观，任意 PowerShell）：

```powershell
openclaw node run --host 43.156.245.19 --port 18789
```

如果提示 `pairing required`：去服务器执行 `openclaw devices approve <requestId>`（见 8.6）。

### 8.4 Windows：安装 Chrome 扩展（unpacked）

```powershell
openclaw browser extension install
openclaw browser extension path
```

Chrome 操作：
- 打开 `chrome://extensions`
- 打开 **Developer mode**
- **Load unpacked** → 选择 `openclaw browser extension path` 打印出来的目录
- Pin 扩展到工具栏

### 8.5 Chrome：attach tab（这一步不是命令行）

- 打开你要控制的网站页面
- 点击工具栏的 **OpenClaw Browser Relay** 扩展图标
- 角标显示 **ON** 才表示该 tab 已 attach（可控）

扩展常见状态：
- `ON`：已 attach
- `!`：relay 不可达（通常是 node host 未连接/未启动，或本机 18792 未通）

本机 relay 端口检查：

```powershell
Test-NetConnection 127.0.0.1 -Port 18792
```

### 8.6 服务器：首次配对/批准（解决 pairing required）

node host 的配对走的是 **devices pairing**（不是 nodes pending）：

```bash
openclaw devices list --json
openclaw devices approve <requestId>
```

### 8.7 今天踩坑复盘（高频问题）

1) **pairing required 不会出现在 nodes pending**
- node host 需要用 `openclaw devices list/approve` 批准。

2) **走 Nginx:80 反代 WebSocket 容易 token_missing / ECONNRESET**
- 更稳：node host 直接连 Gateway 的 `:18789`（不走反代）。

3) **Windows Scheduled Task 后台常驻经常拿不到 token**
- 现象：前台 `openclaw node run ...` 能连；一 `openclaw node restart` 就掉线。
- 推荐解法：把 token 写到脚本里（`C:\Users\<user>\.openclaw\node.cmd`）：

```bat
set OPENCLAW_GATEWAY_TOKEN=你的token
openclaw node run --host 43.156.245.19 --port 18789
```

（或者用 Machine 环境变量 `OPENCLAW_GATEWAY_TOKEN`。）

4) **不要在 node.cmd 里再 start /b 二次拉起（双实例抢文件）**
- 会出现“另一个程序正在使用此文件”。
- Scheduled Task 只跑一个实例即可。

5) **Options 页不是 attach**
- Options 页只显示 relay 状态/端口；必须在目标 tab 点扩展让角标 ON。

### 8.8 常用运维补充命令

（已合并到 **6) 启动并设置开机自启**，避免重复。）
