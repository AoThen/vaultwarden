# Vaultwarden 部署指南

本文档提供 Vaultwarden 的完整部署说明，帮助你快速搭建自己的密码管理服务。

---

## 目录

- [快速开始](#快速开始)
- [配置说明](#配置说明)
- [数据库配置](#数据库配置)
- [反向代理配置](#反向代理配置)
- [HTTPS 配置](#https-配置)
- [邮件服务配置](#邮件服务配置)
- [备份与恢复](#备份与恢复)
- [安全建议](#安全建议)
- [常见问题](#常见问题)

---

## 快速开始

### 前置要求

- Docker 20.10+
- Docker Compose v2+
- 至少 512MB 可用内存

### 最小化部署

```bash
# 1. 克隆或下载配置文件
mkdir vaultwarden && cd vaultwarden

# 2. 下载 docker-compose.yml
curl -O https://raw.githubusercontent.com/dani-garcia/vaultwarden/main/examples/docker-compose.yml

# 3. 创建数据目录
mkdir data

# 4. 启动服务
docker compose up -d

# 5. 查看日志
docker compose logs -f

# 6. 访问服务
# 打开浏览器访问 http://localhost:8080
```

### 创建管理员账户

首次部署后，你需要：

1. 访问 `http://localhost:8080`
2. 点击「创建账户」注册第一个用户
3. 此账户将成为管理员

---

## 配置说明

### 核心配置项

| 环境变量 | 说明 | 默认值 | 必填 |
|---------|------|--------|------|
| `DOMAIN` | 服务访问域名 | `http://localhost` | **是** |
| `ADMIN_TOKEN` | 管理员面板访问令牌 | - | 推荐 |
| `SIGNUPS_ALLOWED` | 是否允许新用户注册 | `true` | 否 |
| `INVITATIONS_ALLOWED` | 是否允许邀请用户 | `true` | 否 |
| `ENABLE_WEBSOCKET` | 启用 WebSocket 实时同步 | `true` | 否 |
| `SHOW_PASSWORD_HINT` | 显示密码提示 | `false` | 否 |

### 域名配置 (DOMAIN)

**重要**: `DOMAIN` 变量必须与你的实际访问地址一致，否则以下功能将无法正常工作：

- 附件下载
- 邮件链接
- U2F/FIDO2 认证
- WebSocket 连接

```yaml
# 示例配置
- DOMAIN=https://vault.example.com          # 标准配置
- DOMAIN=https://vault.example.com:8443     # 自定义端口
- DOMAIN=https://example.com/vault          # 子路径部署
```

### 管理员令牌 (ADMIN_TOKEN)

管理员面板位于 `/admin`，需要设置 `ADMIN_TOKEN` 才能访问。

**推荐使用 Argon2 哈希**：

```bash
# 生成 Argon2 哈希令牌
docker exec -it vaultwarden /vaultwarden hash

# 输出示例:
# Argon2id hash: $argon2id$v=19$m=65540,t=3,p=4$...
```

```yaml
# 在 docker-compose.yml 中配置
environment:
  - ADMIN_TOKEN=$argon2id$v=19$m=65540,t=3,p=4$...
```

**注意**: 在 `docker-compose.yml` 中使用 `$` 需要转义为 `$$`：

```yaml
- ADMIN_TOKEN=$$argon2id$$v=19$$m=65540,t=3,p=4$$...
```

---

## 数据库配置

### SQLite (默认)

默认使用 SQLite，无需额外配置，数据存储在 `/data/db.sqlite3`。

**适用场景**：
- 个人使用
- 小团队 (< 50 用户)
- 单机部署

### PostgreSQL (推荐生产环境)

```yaml
services:
  vaultwarden:
    environment:
      - DATABASE_URL=postgresql://vaultwarden:密码@postgres:5432/vaultwarden
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_USER=vaultwarden
      - POSTGRES_PASSWORD=你的密码
      - POSTGRES_DB=vaultwarden
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U vaultwarden"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### MySQL / MariaDB

```yaml
services:
  vaultwarden:
    environment:
      - DATABASE_URL=mysql://vaultwarden:密码@mysql:3306/vaultwarden

  mysql:
    image: mysql:8
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=你的root密码
      - MYSQL_USER=vaultwarden
      - MYSQL_PASSWORD=你的密码
      - MYSQL_DATABASE=vaultwarden
    volumes:
      - ./mysql-data:/var/lib/mysql
```

---

## 反向代理配置

### Nginx 配置示例

```nginx
server {
    listen 80;
    listen 443 ssl http2;
    server_name vault.example.com;

    # SSL 证书
    ssl_certificate /etc/nginx/ssl/vault.example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/vault.example.com.key;

    # 客户端上传大小限制
    client_max_body_size 100M;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket 支持
    location /notifications/hub {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Caddy 配置示例

```
vault.example.com {
    encode gzip
    
    # 客户端上传大小限制
    request_body {
        max_size 100MB
    }
    
    reverse_proxy localhost:8080 {
        header_up X-Real-IP {remote_host}
    }
    
    # WebSocket 自动支持
}
```

### Traefik 配置示例

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.vaultwarden.rule=Host(`vault.example.com`)"
      - "traefik.http.routers.vaultwarden.entrypoints=websecure"
      - "traefik.http.routers.vaultwarden.tls.certresolver=myresolver"
      - "traefik.http.services.vaultwarden.loadbalancer.server.port=80"
```

---

## HTTPS 配置

### 方案一：反向代理处理 SSL (推荐)

Vaultwarden 运行 HTTP，由反向代理处理 HTTPS。

### 方案二：Vaultwarden 直接启用 TLS

```yaml
environment:
  - ROCKET_TLS={certs="/ssl/cert.pem",key="/ssl/key.pem"}
volumes:
  - ./ssl:/ssl:ro
```

---

## 邮件服务配置

### SMTP 配置

```yaml
environment:
  - DOMAIN=https://vault.example.com      # 必填，邮件链接需要
  - SMTP_HOST=smtp.example.com
  - SMTP_FROM=vaultwarden@example.com
  - SMTP_FROM_NAME=Vaultwarden
  - SMTP_SECURITY=starttls                # starttls | force_tls | off
  - SMTP_PORT=587
  - SMTP_USERNAME=your_username
  - SMTP_PASSWORD=your_password
```

### 常用邮件服务商配置

| 服务商 | SMTP_HOST | SMTP_PORT | SMTP_SECURITY |
|--------|-----------|-----------|---------------|
| Gmail | smtp.gmail.com | 587 | starttls |
| Outlook | smtp-mail.outlook.com | 587 | starttls |
| QQ邮箱 | smtp.qq.com | 587 | starttls |
| 163邮箱 | smtp.163.com | 587 | starttls |
| 阿里云企业邮箱 | smtp.qiye.aliyun.com | 587 | starttls |

---

## 备份与恢复

### SQLite 备份

```bash
# 简单备份
cp ./data/db.sqlite3 ./backup/db.sqlite3.$(date +%Y%m%d)

# 使用 sqlite3 在线备份
sqlite3 ./data/db.sqlite3 ".backup './backup/db.sqlite3'"

# 使用 docker exec
docker exec vaultwarden sqlite3 /data/db.sqlite3 ".backup '/data/backup.sqlite3'"
```

### PostgreSQL 备份

```bash
# 备份
docker exec vaultwarden-postgres pg_dump -U vaultwarden vaultwarden > backup.sql

# 恢复
cat backup.sql | docker exec -i vaultwarden-postgres psql -U vaultwarden vaultwarden
```

### 完整备份脚本

```bash
#!/bin/bash
BACKUP_DIR="/backup/vaultwarden"
DATE=$(date +%Y%m%d_%H%M%S)

# 创建备份目录
mkdir -p "$BACKUP_DIR"

# 备份数据目录
tar -czvf "$BACKUP_DIR/data_$DATE.tar.gz" ./data

# 删除 30 天前的备份
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +30 -delete

echo "Backup completed: $BACKUP_DIR/data_$DATE.tar.gz"
```

---

## 安全建议

### 1. 禁用注册 (生产环境)

```yaml
- SIGNUPS_ALLOWED=false
```

首次创建管理员账户后，建议禁用注册。

### 2. 启用管理员面板保护

```yaml
- ADMIN_TOKEN=your_secure_token
- ADMIN_RATELIMIT_SECONDS=300
- ADMIN_RATELIMIT_MAX_BURST=3
```

### 3. 配置登录速率限制

```yaml
- LOGIN_RATELIMIT_SECONDS=60
- LOGIN_RATELIMIT_MAX_BURST=10
```

### 4. 使用 HTTPS

生产环境**必须**使用 HTTPS，否则：
- 密码明文传输
- U2F/FIDO2 无法工作
- 浏览器安全警告

### 5. 限制附件大小

```yaml
- ORG_ATTACHMENT_LIMIT=1048576     # 组织附件限制 1GB
- USER_ATTACHMENT_LIMIT=1048576    # 用户附件限制 1GB
```

### 6. 定期备份

设置定时任务进行自动备份。

---

## 常见问题

### Q: 忘记管理员令牌怎么办？

在 `data/config.json` 中可以查看或修改 `admin_token`。

### Q: 如何重置用户密码？

管理员无法重置用户密码，因为数据是端到端加密的。用户需要使用主密码恢复密钥。

### Q: WebSocket 不工作？

确保：
1. `ENABLE_WEBSOCKET=true`
2. 反向代理正确配置 WebSocket 升级
3. `DOMAIN` 配置正确

### Q: 附件上传失败？

检查：
1. 反向代理的 `client_max_body_size` 设置
2. 磁盘空间是否充足
3. 用户附件限制配置

### Q: 邮件发送失败？

检查：
1. SMTP 配置是否正确
2. `DOMAIN` 是否正确设置
3. 防火墙是否允许 SMTP 出站

---

## 更多资源

- [官方文档](https://github.com/dani-garcia/vaultwarden/wiki)
- [反向代理示例](https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples)
- [环境变量完整列表](https://github.com/dani-garcia/vaultwarden/blob/main/.env.template)

---

## 许可证

Vaultwarden 使用 AGPL-3.0 许可证。
