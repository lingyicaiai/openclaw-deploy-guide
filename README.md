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

仓库里提供了两份模板（推荐直接复制使用）：

- `systemd/openclaw-gateway.root.service`（root 运行）
- `systemd/openclaw-gateway.ubuntu.service`（ubuntu 运行）

### 方案 A：root 运行（当前默认示例）

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

后续你修改了 service 文件（比如从 root 切到 ubuntu，或改了 ExecStart/环境变量），用这两条让它生效：

```bash
sudo systemctl daemon-reload
sudo systemctl restart openclaw-gateway
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

## License

MIT
