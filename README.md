# OpenClaw 部署指南（Ubuntu / systemd / loopback）

本仓库只保留两样东西：
1) `openclaw-gateway.ubuntu.service`（OpenClaw 以 ubuntu 用户运行的 systemd service 模板）
2) 两条必用命令：`daemon-reload` + `restart`

目标：让 gateway 常驻运行，并且只监听本机回环 `127.0.0.1:18789`。

---

## 1) 安装并初始化（一次性）

```bash
sudo npm install -g openclaw@latest
openclaw onboard
```

---

## 2) 写入 systemd service（ubuntu 运行）

把仓库里的 `openclaw-gateway.ubuntu.service` 内容写到：
`/etc/systemd/system/openclaw-gateway.service`

（或直接复制粘贴）。

---

## 3) 使其生效（只用这两条）

```bash
sudo systemctl daemon-reload
sudo systemctl restart openclaw-gateway
```
