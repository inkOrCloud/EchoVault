# Docker Compose 部署指南

> 版本：v1.0  
> 更新日期：2026-07-09  
> 适用版本：EchoVault v1.0

---

## 目录

1. [环境要求](#1-环境要求)
2. [快速启动](#2-快速启动)
3. [服务架构](#3-服务架构)
4. [容器详解](#4-容器详解)
5. [数据持久化](#5-数据持久化)
6. [网络配置](#6-网络配置)
7. [生产部署](#7-生产部署)
8. [运维命令](#8-运维命令)
9. [故障排除](#9-故障排除)

---

## 1. 环境要求

| 组件 | 最低版本 | 说明 |
|:---|:---:|:---|
| Docker Engine | 24.0+ | 容器运行时 |
| Docker Compose | 2.20+ | 容器编排（或 docker compose plugin） |
| 磁盘空间 | 1 GB + 曲库容量 | 系统镜像约 200MB，数据库及文件按需增长 |
| 内存 | 512 MB | 单机部署建议 1 GB |

**验证环境：**

```bash
docker --version
# Docker version 24.0.7, build afdd53b

docker compose version
# Docker Compose version v2.20.3
```

## 2. 快速启动

### 2.1 获取项目

```bash
git clone --recurse-submodules https://github.com/inkOrCloud/EchoVault.git
cd EchoVault
```

> `--recurse-submodules` 参数会自动拉取 `echovault-server` 和 `echo_vault_app` 子模块。

### 2.2 配置环境变量

创建 `.env` 文件（可选，不创建则使用默认值）：

```bash
# 安全密钥（生产环境必须修改）
JWT_SECRET=your-strong-secret-key-here

# 可选：自定义数据目录
# STORAGE_PATH=./data/files
# DB_PATH=./data/echovault.db
```

### 2.3 启动服务

```bash
docker compose up -d
```

首次启动会自动构建镜像，耗时约 2-5 分钟。成功后：

```bash
# 验证所有容器运行中
docker compose ps
# NAME                    IMAGE                       STATUS   PORTS
# echovault-envoy-1       envoyproxy/envoy:v1.32     Up       0.0.0.0:8080->8080/tcp
# echovault-server-1      echovault-echovault-server  Up       0.0.0.0:9090->9090/tcp

# 查看日志
docker compose logs -f
```

### 2.4 验证部署

```bash
# 检查 gRPC 端点（通过 Envoy）
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/echo_vault.user.v1.UserService/Register
# 预期：404（gRPC 需要 POST + 正确 payload）

# 检查 REST 端点
curl -s http://localhost:9091/api/v1/health
# 预期：{"status":"ok"}
```

### 2.5 启动 Flutter 客户端

```bash
cd echo_vault_app
flutter run
```

在客户端配置页输入服务器地址：`http://<服务器IP>:8080`

## 3. 服务架构

Docker Compose 部署包含三个容器：

```
┌──────────────┐      gRPC-Web / REST       ┌──────────────────┐
│   Flutter     │ ──────────────────────►     │  Envoy Proxy     │
│   客户端      │     HTTP/1.1 :8080          │  (gRPC-Web 转换) │
└──────────────┘                              └────────┬─────────┘
                                                       │
                                          ┌────────────┴────────────┐
                                          │                         │
                                   gRPC :9090                 REST :9091
                                          │                         │
                                          ▼                         ▼
                                   ┌──────────────────┐  ┌──────────────────┐
                                   │  Go 同步服务器    │  │  REST 文件服务    │
                                   │  echovault-server │  │  (内嵌在 Gin 中) │
                                   └──────────────────┘  └──────────────────┘
```

**请求流程说明：**

1. Flutter 客户端通过 HTTP/1.1 发送 gRPC-Web 请求到 Envoy（端口 8080）
2. Envoy 将 gRPC-Web 转换为 HTTP/2 gRPC，转发给 Go 后端（端口 9090）
3. REST 文件上传/下载请求由 Envoy 按路径前缀 `/api/v1/` 路由到 Go 后端的 REST 端口（9091）

## 4. 容器详解

### 4.1 echovault-server（Go 后端）

| 属性 | 值 |
|:---|:---|
| 镜像 | 基于 `golang:1.23-alpine` 多阶段构建 |
| 基础运行时 | `alpine:3.20`（~15MB） |
| 暴露端口 | `9090`（gRPC）, `9091`（REST） |
| 数据库 | SQLite（默认），支持 PostgreSQL |
| 文件存储 | 本地文件系统（默认），支持 S3 |

**构建说明：**

```dockerfile
FROM golang:1.23-alpine AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /build/server ./cmd/server

FROM alpine:3.20
RUN apk add --no-cache ca-certificates tzdata
WORKDIR /app
COPY --from=builder /build/server .
EXPOSE 9090
ENTRYPOINT ["./server"]
```

> 注意：CGO_ENABLED=0 编译，不使用 C 扩展，减少镜像体积。

### 4.2 Envoy Proxy

| 属性 | 值 |
|:---|:---|
| 镜像 | `envoyproxy/envoy:v1.32-latest` |
| 暴露端口 | `8080`（统一入口） |
| 功能 | gRPC-Web 协议转换、CORS 支持、请求路由 |

**路由规则：**

| 路径前缀 | 目标 | 协议 |
|:---|:---|:---:|
| `/echo_vault.`（gRPC 服务名） | `echovault-server:9090` | HTTP/2 gRPC |
| `/api/v1/` | `echovault-server:9091` | HTTP/1.1 REST |

**CORS 配置：**
- 允许所有来源（`*`）
- 允许方法：GET, POST, PUT, DELETE, OPTIONS
- 允许 Headers：`content-type`, `x-grpc-web`, `grpc-timeout`, `authorization`
- 预检缓存时间：86400 秒（24 小时）

## 5. 数据持久化

### 5.1 数据卷映射

```yaml
volumes:
  - ./data:/app/data
```

`/app/data` 目录结构：

```
data/
├── echovault.db              # SQLite 数据库（自动创建）
└── files/
    ├── songs/                 # 用户上传的音频文件
    │   └── {song_id}.mp3
    ├── covers/                # 专辑封面图片
    │   └── {song_id}.jpg
    └── lyrics/                # 歌词文件
        └── {song_id}.lrc
```

### 5.2 备份建议

```bash
# 一键备份
tar czf echovault-backup-$(date +%Y%m%d).tar.gz ./data

# 仅备份数据库
sqlite3 ./data/echovault.db ".backup 'backup-$(date +%Y%m%d).db'"

# 恢复
tar xzf echovault-backup-20260709.tar.gz
```

## 6. 网络配置

### 6.1 端口映射

| 宿主机端口 | 容器端口 | 用途 |
|:---:|:---:|:---|
| 8080 | 8080 | Envoy 统一入口（gRPC-Web + REST） |
| 9090 | 9090 | gRPC 直连（调试用，可选暴露） |

### 6.2 修改端口

编辑 `docker-compose.yml`：修改 `ports` 段即可。

## 7. 生产部署

### 7.1 安全建议

1. **修改 JWT 密钥**
   ```bash
   echo "JWT_SECRET=$(openssl rand -base64 32)" >> .env
   ```

2. **限制 Envoy 监听地址**
   ```yaml
   ports:
     - "127.0.0.1:8080:8080"  # 仅本地访问
   ```

3. **启用 HTTPS 反向代理**
   推荐前置 Nginx/Caddy 提供 TLS 终止：

   ```nginx
   # nginx 配置示例
   server {
       listen 443 ssl;
       server_name music.example.com;

       ssl_certificate /path/to/cert.pem;
       ssl_certificate_key /path/to/key.pem;

       location / {
           proxy_pass http://127.0.0.1:8080;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
       }
   }
   ```

4. **使用 PostgreSQL 替代 SQLite**（多用户场景）
   ```yaml
   services:
     postgres:
       image: postgres:16-alpine
       environment:
         POSTGRES_DB: echovault
         POSTGRES_USER: echovault
         POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
     echovault-server:
       environment:
         DB_DRIVER: postgres
         DB_PATH: "host=postgres user=echovault password=${POSTGRES_PASSWORD} dbname=echovault sslmode=disable"
   ```

### 7.2 资源限制

```yaml
services:
  echovault-server:
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.5'
```

## 8. 运维命令

### 8.1 日常运维

```bash
# 查看状态
docker compose ps

# 查看日志
docker compose logs -f --tail=100

# 重启服务
docker compose restart

# 重新构建并启动
docker compose up -d --build

# 停止服务
docker compose down

# 停止并清理数据卷（谨慎！）
docker compose down -v
```

### 8.2 更新服务

```bash
# 拉取最新代码
git pull --recurse-submodules

# 重新构建并启动
docker compose up -d --build

# 验证
docker compose logs --tail=20
```

### 8.3 健康检查

```bash
# gRPC 服务健康检查（通过 Envoy）
grpcurl -plaintext localhost:8080 echo_vault.user.v1.UserService/Register

# 检查数据库完整性
docker compose exec echovault-server sqlite3 /app/data/echovault.db "PRAGMA integrity_check;"
```

## 9. 故障排除

### 9.1 容器无法启动

**现象：** `docker compose up -d` 后容器退出

**排查步骤：**

```bash
# 查看详细日志
docker compose logs echovault-server

# 常见原因：
# 1. 端口被占用 → 修改 ports 映射
# 2. 数据目录权限 → sudo chown -R 1000:1000 ./data
# 3. SQLite 损坏 → 删除 data/echovault.db 重新创建
```

### 9.2 Envoy 连接 refused

**现象：** Flutter 客户端无法连接 Envoy

**排查：**

```bash
# 检查 Envoy 是否在监听
docker compose exec envoy curl -s http://localhost:8080

# 检查 Envoy 配置是否加载
docker compose exec envoy curl -s http://localhost:9901/config_dump
```

### 9.3 gRPC 调用报错

**现象：** 客户端调用 gRPC 返回 "Unavailable" 或 "Internal"

**排查：**

```bash
# 确认后端服务运行中
docker compose ps

# 查看 gRPC 服务端日志
docker compose logs echovault-server

# 测试 gRPC 连通性
grpcurl -plaintext localhost:9090 list
```
