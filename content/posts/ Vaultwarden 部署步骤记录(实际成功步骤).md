---
author: ["qyzzyqlqj"]
title: "Vaultwarden 部署步骤记录（实际成功步骤）"
date: "2026-08-03"
description: ""
summary: ""
tags: ["经验", "教程", "web", "服务器", "部署", "安全", "网络", "实用项目", "开源"]
categories: ["教程"]
series: ["教程"]
ShowToc: true
TocOpen: false
draft: false
---


> 环境: Debian 13 (bookworm) | 1C/256MB RAM | 3GB 硬盘 | Podman NAT 小鸡
> 日期: 2026-08-02
> GitHub 仓库: [qyzzyqlqj/Vaultwarden_Backup](https://github.com/qyzzyqlqj/Vaultwarden_Backup)

---

## 一、提取 Vaultwarden 二进制文件

由于小鸡是 Podman/LXC 嵌套容器，Docker 和 Podman 均无法运行（`cannot clone: Permission denied`），因此使用 `docker-image-extract` 脚本从 Alpine 镜像中直接提取二进制文件。

### 1.1 创建提取目录并下载脚本

```bash
mkdir -p ~/vw-extract && cd ~/vw-extract
wget https://raw.githubusercontent.com/jjlin/docker-image-extract/main/docker-image-extract
chmod +x docker-image-extract
```

### 1.2 提取镜像

> 注意: 必须显式指定 `-p linux/amd64`，否则会报 `No image digest found`。

```bash
./docker-image-extract -p linux/amd64 vaultwarden/server:latest-alpine
```

预期输出: `Image contents extracted into ./output.`

### 1.3 部署到运行目录

```bash
mkdir -p ~/vaultwarden
cp output/vaultwarden ~/vaultwarden/
cp -r output/web-vault ~/vaultwarden/
cd ~/vaultwarden
```

### 1.4 清理临时文件

```bash
rm -rf ~/vw-extract
```

---

## 二、启动 Vaultwarden 服务

### 2.1 首次启动（绑定 127.0.0.1）

> 说明: 2.1-2.3 节使用 `nohup` 快速验证服务能否正常启动。
> 验证通过后，建议直接跳到 2.4 节配置 systemd 服务（更稳定，支持开机自启和崩溃重启）。

```bash
cd ~/vaultwarden
nohup env ROCKET_PORT=8080 DATA_FOLDER=./data WEB_VAULT_FOLDER=./web-vault WEB_VAULT_ENABLED=true ./vaultwarden > vaultwarden.log 2>&1 &
```

> 数据目录: 上述命令使用相对路径 `./data`，实际指向 `/root/vaultwarden/data`。
> 如需在 systemd 开机自启或其他非 `~/vaultwarden` 目录下启动，应使用绝对路径:
> `DATA_FOLDER=/root/vaultwarden/data WEB_VAULT_FOLDER=/root/vaultwarden/web-vault`

检查日志:

```bash
cat vaultwarden.log
```

预期输出: `Rocket has launched from http://127.0.0.1:8080`

本地验证:

```bash
curl -I http://127.0.0.1:8080
```

预期: `HTTP/1.1 200 OK`

### 2.2 修复绑定地址（改为 0.0.0.0）

由于 `127.0.0.1` 仅限本地访问，外部 NAT 端口转发无法连接，需要改为监听所有网络接口。

```bash
ps aux | grep vaultwarden
kill -9 <PID>
cd ~/vaultwarden && nohup env ROCKET_ADDRESS=0.0.0.0 ROCKET_PORT=8080 DATA_FOLDER=./data WEB_VAULT_FOLDER=./web-vault WEB_VAULT_ENABLED=true ./vaultwarden > vaultwarden.log 2>&1 &
```

> 绝对路径等价写法（推荐用于 systemd 等场景）:
> `nohup env ROCKET_ADDRESS=0.0.0.0 ROCKET_PORT=8080 DATA_FOLDER=/root/vaultwarden/data WEB_VAULT_FOLDER=/root/vaultwarden/web-vault WEB_VAULT_ENABLED=true /root/vaultwarden/vaultwarden > /root/vaultwarden/vaultwarden.log 2>&1 &`

验证日志:

```bash
cat vaultwarden.log
```

预期: `Rocket has launched from http://0.0.0.0:8080`

### 2.3 验证内网可达

```bash
curl -I http://10.91.0.51:8080
```

预期: `HTTP/1.1 200 OK`

> 注意: IPv6 方案（`ROCKET_ADDRESS="[::]"`）会导致 Rocket 框架解析报错退出（Exit 101），因此最终稳定使用 `0.0.0.0`。

### 2.4 配置 systemd 开机自启与保活（推荐）

小鸡支持 systemd（PID 1 运行中），因此可以用 systemd 服务替代 `nohup`，实现崩溃秒级重启和开机自启。

```bash
cat << 'EOF' > /etc/systemd/system/vaultwarden.service
[Unit]
Description=Vaultwarden Password Manager
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/vaultwarden
Environment="ROCKET_ADDRESS=0.0.0.0"
Environment="ROCKET_PORT=8080"
Environment="DATA_FOLDER=./data"
Environment="WEB_VAULT_FOLDER=./web-vault"
Environment="WEB_VAULT_ENABLED=true"
ExecStart=/root/vaultwarden/vaultwarden
Restart=always
RestartSec=5s
MemoryMax=180M

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now vaultwarden.service
```

检查服务状态:

```bash
systemctl status vaultwarden
```

预期: `Active: active (running)`

> 优势: 崩溃后 5 秒内自动拉起 | 小鸡重启后自动启动 | 内存限制 180M，给系统留 76M 缓冲
> 注意: 启用 systemd 服务后，之前的 `nohup` 后台进程可以停掉（`kill <PID>`），避免两个实例冲突。

---

## 三、Cloudflare Tunnel 配置 HTTPS 外网访问

由于商家 NAT 端口转发（8080 -> 15510）返回 502 Bad Gateway，且 Bitwarden 客户端强制要求 HTTPS，因此使用 Cloudflare Tunnel 实现外网访问。

### 3.1 安装 cloudflared

```bash
cd ~/vaultwarden
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
dpkg -i cloudflared-linux-amd64.deb
rm cloudflared-linux-amd64.deb
```

### 3.2 安装为系统服务（网页控制台模式）

在 Cloudflare Dashboard -> Zero Trust -> Networks -> Tunnels 中创建 Tunnel，复制生成的安装命令:

```bash
cloudflared service install <YOUR_TOKEN>
```

预期输出: `Linux service for cloudflared installed successfully`

### 3.3 配置路由

在 Cloudflare 网页控制台配置:

| 配置项 | 值 |
|--------|-----|
| Subdomain | 自定义（如 `vault`） |
| Domain | 你的域名 |
| Service Type | HTTP |
| URL | `127.0.0.1:8080` |

> 说明: 内部使用 HTTP 是安全的。流量在 cloudflared 与 Vaultwarden 之间走本机环回，公网段全程由 Cloudflare 提供 HTTPS 加密。

---

## 四、GitHub 自动备份（每 5 分钟）

### 4.1 生成 SSH Deploy Key

```bash
ssh-keygen -t ed25519 -C "vaultwarden-backup" -f ~/.ssh/vw_backup -N ""
cat ~/.ssh/vw_backup.pub
```

输出示例: `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIE3WAjjxE4GCtyhoZUQxFDuvcOlU5b+SB6+Kd08FMtch vaultwarden-backup`

将公钥添加到 GitHub 仓库 Settings -> Deploy keys -> Add deploy key（勾选 Allow write access）。

### 4.2 配置 SSH

```bash
cat << 'EOF' >> ~/.ssh/config
Host github.com
    HostName github.com
    IdentityFile ~/.ssh/vw_backup
    User git
EOF
chmod 600 ~/.ssh/config
```

### 4.3 初始化本地备份仓库

```bash
mkdir -p ~/vw-repo && cd ~/vw-repo
git init
git config user.name "Vaultwarden Backup"
git config user.email "backup@local"
git branch -M main
git remote add origin git@github.com:qyzzyqlqj/Vaultwarden_Backup.git
```

### 4.4 测试 SSH 连接

```bash
ssh -T git@github.com
```

预期输出: `Hi qyzzyqlqj/Vaultwarden_Backup! You've successfully authenticated...`

### 4.5 编写备份脚本

```bash
#!/bin/bash
DATA_DIR="/root/vaultwarden/data"
REPO_DIR="/root/vw-repo"
mkdir -p "$REPO_DIR"
CURRENT_HASH=$(sqlite3 "$DATA_DIR/db.sqlite3" ".dump" | sha256sum | awk '{print $1}')
OLD_HASH_FILE="$REPO_DIR/.db_hash"
[ -f "$OLD_HASH_FILE" ] && OLD_HASH=$(cat "$OLD_HASH_FILE") || OLD_HASH=""
if [ "$CURRENT_HASH" != "$OLD_HASH" ]; then
    sqlite3 "$DATA_DIR/db.sqlite3" ".backup '$REPO_DIR/db.sqlite3'"
    echo "$CURRENT_HASH" > "$OLD_HASH_FILE"
fi
for f in rsa_key.pem rsa_key.pub.pem; do
    if [ -f "$DATA_DIR/$f" ]; then
        if [ -f "$REPO_DIR/$f" ]; then
            if ! cmp -s "$DATA_DIR/$f" "$REPO_DIR/$f"; then
                cp "$DATA_DIR/$f" "$REPO_DIR/$f"
            fi
        else
            cp "$DATA_DIR/$f" "$REPO_DIR/$f"
        fi
    fi
done
if [ -d "$DATA_DIR/attachments" ]; then
    if command -v rsync &>/dev/null; then
        rsync -a --delete "$DATA_DIR/attachments/" "$REPO_DIR/attachments/"
    else
        mkdir -p "$REPO_DIR/attachments"
        for f in "$DATA_DIR/attachments/"*; do
            [ -f "$f" ] || continue
            fname=$(basename "$f")
            if [ -f "$REPO_DIR/attachments/$fname" ]; then
                if ! cmp -s "$f" "$REPO_DIR/attachments/$fname"; then
                    cp "$f" "$REPO_DIR/attachments/$fname"
                fi
            else
                cp "$f" "$REPO_DIR/attachments/$fname"
            fi
        done
    fi
fi
cd "$REPO_DIR" || exit
git add .
if ! git diff --cached --quiet; then
    TIME=$(date "+%Y-%m-%d %H:%M:%S")
    git commit -m "Auto backup: $TIME"
    git push origin main
fi
```

赋予执行权限并测试:

```bash
chmod +x ~/backup.sh
~/backup.sh
```

首次运行成功输出:

```
[main (root-commit) 8182cf1] Auto backup: 2026-08-02 17:13:17
 2 files changed, 27 insertions(+)
 create mode 100644 db.sqlite3
 create mode 100644 rsa_key.pem
To github.com:qyzzyqlqj/Vaultwarden_Backup.git
 * [new branch]      main -> main
```

### 4.6 设置 crontab 定时任务

```bash
apt install -y cron
crontab -e
```

在 crontab 中添加以下行:

```
*/1 * * * * /bin/bash /root/backup.sh >/dev/null 2>&1
```

启动 cron 服务并验证:

```bash
service cron start
systemctl status cron
```

预期: `Active: active (running)`

---

## 五、最终架构

```
用户浏览器
    |
    | HTTPS (Cloudflare 自动提供 SSL 证书)
    |
Cloudflare 边缘节点 (CDN + SSL 终止)
    |
    | Cloudflare Tunnel (加密隧道)
    |
cloudflared (小鸡系统服务)
    |
    | HTTP (本机环回 127.0.0.1:8080)
    |
Vaultwarden (端口 8080)
    - 数据目录: ~/vaultwarden/data
    - 前端页面: ~/vaultwarden/web-vault
    |
    | cron 每 1 分钟触发
    |
backup.sh
    - SQLite 热备份
    - Git 增量提交
    - Push 到 GitHub 私有仓库
```

---

## 六、灾备恢复步骤

如果小鸡故障，在新服务器上恢复:

1. 在新服务器上用同样方法部署 Vaultwarden
2. 克隆备份仓库:
   ```bash
   git clone git@github.com:qyzzyqlqj/Vaultwarden_Backup.git ~/vw-repo
   ```
3. 停止 Vaultwarden 服务
4. 恢复数据:
   ```bash
   cp ~/vw-repo/db.sqlite3 ~/vaultwarden/data/
   [ -f ~/vw-repo/rsa_key.pem ] && cp ~/vw-repo/rsa_key.pem ~/vaultwarden/data/
   [ -f ~/vw-repo/rsa_key.pub.pem ] && cp ~/vw-repo/rsa_key.pub.pem ~/vaultwarden/data/
   [ -d ~/vw-repo/attachments ] && cp -r ~/vw-repo/attachments ~/vaultwarden/data/
   ```
5. 重新启动 Vaultwarden，用原主密码登录即可

---

## 附: 失败尝试记录

| 尝试方案 | 失败原因 |
|---------|---------|
| 下载 GitHub Releases 二进制 | 官方已取消独立 Release 包，下载到的是 9 字节的 HTML |
| 使用 `docker run` | Docker daemon 未运行（嵌套容器限制） |
| 使用 `podman run` | `cannot clone: Permission denied`（嵌套容器限制） |
| 从 Docker 镜像 `docker cp` 提取 | Docker daemon 未运行 |
| 绑定 `ROCKET_ADDRESS="[::]"` | Rocket 框架解析 IPv6 语法失败（Exit 101） |
| 使用 `iptables -F` 放行端口 | `Permission denied`（容器内无 `cap_net_admin`） |
| `ufw allow 8080` | 系统未安装 ufw |
| `cloudflared tunnel login` + 本地配置文件 | 改用网页控制台模式更简单，最终还是走了网页模式 |

---

## 问题修复记录

### 备份脚本过于频繁（一晚上 Push 上百次）

**现象:** 每 1 分钟 cron 触发一次备份，每次都会产生 commit 并 push 到 GitHub，即使数据没有实际变化。

**根因:** 原始 backup.sh 中 `sqlite3 ".backup"` 命令每次生成的 SQLite 文件二进制内容不同（内部页面排序、元数据版本号存在差异），导致 `git add` 后 `git diff --cached` 总检测到"变更"，从而每次都会提交。

**修复内容**（已在 backup.sh 中更新）:

1. 使用 `sqlite3 .dump | sha256sum` 计算数据库实际内容的哈希值（仅计算用户数据，忽略 SQLite 内部元数据），存入 `.db_hash` 文件作为比对基准
2. 每次备份时比对哈希，只有数据真实变化时才执行 `.backup` 热备份并更新哈希文件
3. 密钥文件 `rsa_key.pem` / `rsa_key.pub.pem` 使用 `cmp -s` 对比后再覆盖
4. 附件目录优先使用 `rsync -a` 同步（仅传输有变化的文件），无 rsync 时逐文件 `cmp -s` 对比

**方案验证:**

```bash
vim ~/backup.sh
chmod +x ~/backup.sh
~/backup.sh
```

观察输出: 如果数据无变化，应无输出且不产生新 commit；如果数据有变化，只产生一次 commit。