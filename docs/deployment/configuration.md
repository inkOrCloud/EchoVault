# EchoVault 配置参考

> 版本：v1.0  
> 更新日期：2026-07-09

---

## 目录

1. [配置方式](#1-配置方式)
2. [环境变量参考](#2-环境变量参考)
3. [Go 后端配置](#3-go-后端配置)
4. [Flutter 客户端配置](#4-flutter-客户端配置)
5. [Envoy 配置](#5-envoy-配置)
6. [Docker Compose 配置](#6-docker-compose-配置)

---

## 1. 配置方式

EchoVault 支持以下配置方式（优先级从高到低）：

1. **环境变量** — 最高优先级，适合 Docker 部署
2. **配置文件** — `config.yaml` 或 `config.yml`（位于工作目录或 `/app/`）
3. **默认值** — 代码内置的合理默认值

### 1.1 环境变量命名规则

所有环境变量大写、下划线分隔，对应配置项的蛇形命名大写化：

```
db_path      → DB_PATH
jwt_secret   → JWT_SECRET
grpc_port    → GRPC_PORT
```

### 1.2 配置文件示例

```yaml
# config.yaml
grpc_port: 9090
rest_port: 9091
db_driver: sqlite3
db_path: data/echovault.db
storage_type: local
storage_path: data/files
jwt_secret: change-me-in-production
```

---

## 2. 环境变量参考

### 2.1 核心配置

| 变量名 | 默认值 | 说明 |
|:---|:---:|:---|
| `GRPC_PORT` | `9090` | gRPC 服务监听端口 |
| `REST_PORT` | `9091` | REST 文件服务监听端口 |
| `DB_DRIVER` | `sqlite3` | 数据库驱动：`sqlite3` 或 `postgres` |
| `DB_PATH` | `data/echovault.db` | 数据库路径（PostgreSQL 时为连接字符串） |
| `STORAGE_TYPE` | `local` | 文件存储类型：`local` 或 `s3` |
| `STORAGE_PATH` | `data/files` | 文件存储路径（local 模式） |
| `JWT_SECRET` | `change-me-in-production` | JWT 签名密钥（**生产环境必须修改**） |

### 2.2 S3 存储配置（可选）

当 `STORAGE_TYPE=s3` 时需要以下变量：

| 变量名 | 默认值 | 说明 |
|:---|:---:|:---|
| `AWS_ACCESS_KEY_ID` | - | S3 访问密钥 |
| `AWS_SECRET_ACCESS_KEY` | - | S3 密钥 |
| `AWS_REGION` | `us-east-1` | S3 区域 |
| `S3_BUCKET` | `echovault` | S3 存储桶名 |
| `S3_ENDPOINT` | - | S3 兼容服务端点（如 MinIO） |

### 2.3 PostgreSQL 配置（可选）

当 `DB_DRIVER=postgres` 时，`DB_PATH` 格式为连接字符串：

```
host=postgres user=echovault password=YOUR_PASS dbname=echovault sslmode=disable
```

---

## 3. Go 后端配置

### 3.1 代码结构

配置加载逻辑位于 `echovault-server/pkg/config/config.go`：

```go
// 加载顺序：
// 1. 读取 config.yaml（可选）
// 2. 环境变量覆盖
// 3. 默认值填充

viper.SetDefault("grpc_port", 9090)
viper.SetDefault("db_driver", "sqlite3")
viper.SetDefault("db_path", "data/echovault.db")
viper.AutomaticEnv()
```

### 3.2 默认值说明

| 配置项 | 默认值 | 安全风险 |
|:---|:---|:---:|
| `JWT_SECRET` | `change-me-in-production` | 🔴 高 — 密钥泄露可伪造令牌 |
| `GRPC_PORT` | `9090` | 🟢 无 |
| `DB_PATH` | `data/echovault.db` | 🟢 无 |

---

## 4. Flutter 客户端配置

### 4.1 服务器地址配置

在客户端首次启动时，通过服务器配置页面设置：

| 配置项 | 说明 | 示例 |
|:---|:---|:---|
| 服务器地址 | Envoy 代理的 URL（含端口） | `http://192.168.1.100:8080` |
| 用户名 | 注册时使用的用户名 | `admin` |
| 密码 | 注册时使用的密码 | `********` |

### 4.2 配置持久化

客户端配置存储在本地的 `SharedPreferences` 中：

```dart
// 代码位置：lib/core/config/server_config.dart
class ServerConfig {
  final String serverUrl;
  final String? authToken;
  // ...
}
```

---

## 5. Envoy 配置

Envoy 配置文件位于项目根目录 `envoy.yaml`。

### 5.1 端口说明

| 监听端口 | 用途 |
|:---:|:---|
| `8080` | 统一入口（Flutter 客户端连接此端口） |

### 5.2 路由规则

```yaml
routes:
  # gRPC 请求
  - match: { prefix: "/echo_vault.", grpc: {} }
    route:
      cluster: echovault_grpc
      timeout: 0s

  # REST 请求
  - match: { prefix: "/api/v1/" }
    route:
      cluster: echovault_rest
      timeout: 0s
```

### 5.3 CORS 配置

```yaml
http_filters:
- name: envoy.filters.http.cors
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors
    allow_origin_string_match:
    - prefix: "*"              # 允许所有来源
    allow_methods: GET, POST, PUT, DELETE, OPTIONS
    allow_headers: content-type, x-grpc-web, grpc-timeout, authorization
    max_age: "86400"           # 预检缓存 24 小时
```

---

## 6. Docker Compose 配置

### 6.1 完整配置示例

```yaml
version: "3.8"

services:
  envoy:
    image: envoyproxy/envoy:v1.32-latest
    ports:
      - "8080:8080"
    volumes:
      - ./envoy.yaml:/etc/envoy/envoy.yaml
    depends_on:
      - echovault-server

  echovault-server:
    build:
      context: ./echovault-server
      dockerfile: Dockerfile
    ports:
      - "9090:9090"
    volumes:
      - ./data:/app/data
    environment:
      DB_DRIVER: sqlite3
      DB_PATH: /app/data/echovault.db
      STORAGE_TYPE: local
      STORAGE_PATH: /app/data/files
      JWT_SECRET: ${JWT_SECRET:-change-me-in-production}
```

### 6.2 `.env` 文件模板

```bash
# .env — EchoVault 环境变量
# 复制此文件为 .env 并修改配置

# 必填：JWT 签名密钥（安全关键！）
JWT_SECRET=your-secure-random-secret

# 可选：存储配置
# STORAGE_TYPE=local
# STORAGE_PATH=./data/files

# 可选：数据库配置
# DB_DRIVER=sqlite3
# DB_PATH=./data/echovault.db

# 可选：PostgreSQL 配置
# DB_DRIVER=postgres
# DB_PATH=host=postgres user=echovault password=pass dbname=echovault sslmode=disable

# 可选：S3 存储配置
# STORAGE_TYPE=s3
# AWS_ACCESS_KEY_ID=your-key
# AWS_SECRET_ACCESS_KEY=your-secret
# AWS_REGION=us-east-1
# S3_BUCKET=echovault

# 可选：端口配置
# GRPC_PORT=9090
# REST_PORT=9091
```
