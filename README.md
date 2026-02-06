# OpenClaw 部署指南（Ubuntu / systemd / loopback）

这份仓库是一份“最小可用”的 OpenClaw 部署笔记：在 Ubuntu 服务器上安装 OpenClaw，并用 systemd 让 gateway 常驻运行（**仅本机回环监听 127.0.0.1:18789**）。

适用场景：2G 小机器、希望稳定跑起来、通过 nginx/反代对外暴露 UI（如需）。

---

## 0) 前置

- Ubuntu 20.04/22.04/24.04
- 能 sudo

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

---

## 4) 初始化配置（认证/生成配置文件）

```bash
openclaw onboard
```

会在 `~/.openclaw/` 下生成配置与工作区（具体路径以向导输出为准）。

---

## 5) 创建 systemd Gateway 服务（仅本机监听）

下面以 **root 运行**为例（更省事、最不容易遇到权限问题）。

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

---

## 6) 启动并设置开机自启（推荐）

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway
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
