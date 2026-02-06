# OpenClaw 部署指南（Ubuntu / systemd / loopback）

这份文档整理了从 0 到 1 在 Ubuntu 服务器上安装并以 systemd 方式运行 OpenClaw Gateway 的一套命令（**仅本机监听**），适合 2G 小机器。

> 说明：下面的命令以 **root** 执行为例（和我当前机器一致）。更稳的生产做法是改成专用用户（见「可选：非 root 运行」）。

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

### ubuntu 是否需要重新 `openclaw onboard`？

- **新机器首次用 ubuntu 跑**：用 ubuntu 身份执行一次：
  ```bash
  sudo -u ubuntu -H openclaw onboard
  ```

- **从 root 迁移到 ubuntu**：如果已经把 `/root/.openclaw/` 完整复制到 `/home/ubuntu/.openclaw/`，通常不需要再 onboard。
---

## 6) 启动并设置开机自启（推荐）

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway
```

> 如果你后续修改了 service 文件（比如切换 User=root/ubuntu），再执行：
> `sudo systemctl daemon-reload && sudo systemctl restart openclaw-gateway`

---

## 7) 查看运行状态/日志

```bash
sudo systemctl status openclaw-gateway --no-pager

# 实时日志
sudo journalctl -u openclaw-gateway -f

# 确认只在本机监听
ss -ltnp | grep 18789
```

---

## 关于 Chrome headless（是否包含？）

- **上面的安装/启动命令本身不安装/不强制启动 Chrome**。
- 但 OpenClaw 的配置里通常会存在 `browser` 配置项（例如 `browser.enabled=true`、`browser.headless=true`）。
- 这表示：
  - 功能层面允许调用浏览器自动化
  - 运行时只有在需要时才会启动浏览器进程（不一定常驻）

如果你是 2G 小机器，且暂时用不到浏览器自动化，可以考虑在配置中禁用 browser 以节省内存（此项按需，不在本指南默认步骤中修改）。

---

## 可选：非 root 运行（更推荐）

大致思路：

1) 创建专用用户：
```bash
sudo useradd -m -s /bin/bash openclaw
```

2) 用该用户运行 `openclaw onboard`（或把配置迁到 `/home/openclaw`）

3) systemd 里改：
- `User=openclaw`
- `WorkingDirectory=/home/openclaw`
- `Environment=HOME=/home/openclaw`

---

## 可选：systemd 硬化（建议）

在 `[Service]` 里额外加：

```ini
NoNewPrivileges=true
PrivateTmp=true
```

更激进的保护（需你规划可写路径）：

```ini
ProtectSystem=strict
ProtectHome=true
```

---

## License

MIT
