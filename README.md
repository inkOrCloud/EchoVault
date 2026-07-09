# 🎵 EchoVault（音匣）

> **混合模式音乐管理平台** — 本地优先，云端同步，全平台覆盖

EchoVault 是一个开源的家庭音乐管理系统，让你可以：

- 🎶 **管理本地音乐** — 扫描本地音频文件，自动提取元数据
- ☁️ **多设备同步** — 曲库、歌单、歌词自动跨设备同步
- 📤 **文件共享** — 上传音乐到私有服务器，全家人共享
- 📴 **离线可用** — 离线优先架构，无网络也能正常使用

---

## ✨ 功能特性

| 功能 | 描述 |
|:---|:---|
| **曲库管理** | 扫描本地音频（MP3/FLAC/M4A/OGG），自动提取标题、艺术家、专辑等元数据 |
| **音频播放** | 全功能播放器，支持顺序/随机/循环播放，迷你播放器常驻 |
| **歌单系统** | 创建/管理歌单，跨设备同步 |
| **同步歌词** | 支持 LRC 格式歌词，原词+翻译对照显示 |
| **文件上传** | 发布本地音乐到服务器，SHA256 去重 |
| **多设备同步** | 基于版本的增量同步，离线优先设计 |
| **设备管理** | 查看/管理已连接的设备 |
| **REST API** | 支持分段上传、断点续传、Range 请求 |

---

## 🏗️ 系统架构

```
Flutter 客户端 (全平台)
    │
    │ gRPC-Web (CRUD) / REST (文件)
    ▼
Envoy Proxy (gRPC-Web 转换 + CORS)
    │
    ├── gRPC :9090 ──► Go 同步服务器 (Ent + SQLite/PostgreSQL)
    └── REST  :9091 ──► Go 文件服务 (本地/S3 存储)
```

### 技术栈

| 层 | 技术 |
|:---|:---|
| **客户端** | Flutter 3.x + Riverpod + Drift(SQLite) + just_audio |
| **后端** | Go 1.23 + gRPC + Ent ORM + Gin |
| **代理** | Envoy Proxy (gRPC-Web) |
| **数据库** | SQLite（默认）/ PostgreSQL（可选）|
| **存储** | 本地文件系统 / S3 兼容 |
| **部署** | Docker Compose |

---

## 🚀 快速开始

### 前置条件

- Docker Engine 24.0+ & Docker Compose 2.20+
- Flutter 3.x（客户端开发，可选）

### 一键部署

```bash
# 克隆项目
git clone --recurse-submodules https://github.com/inkOrCloud/EchoVault.git
cd EchoVault

# 生成安全密钥
echo "JWT_SECRET=$(openssl rand -base64 32)" > .env

# 启动服务
docker compose up -d

# 验证运行
docker compose ps
```

Flutter 客户端连接地址：`http://<服务器IP>:8080`

> 详细部署说明见 [部署文档](docs/deployment/first-time-setup.md)

---

## 📦 项目结构

```
EchoVault/
├── docker-compose.yml           # Docker Compose 编排
├── envoy.yaml                   # Envoy 代理配置
├── echovault-server/            # Go 后端
│   ├── cmd/server/main.go       # 入口
│   ├── internal/
│   │   ├── ent/                 # Ent ORM（数据模型 + 迁移）
│   │   │   └── schema/          # Schema 定义
│   │   ├── grpc/                # gRPC 处理器 + 鉴权
│   │   ├── rest/                # REST 文件服务
│   │   ├── service/             # 业务逻辑层
│   │   │   ├── user/            # 用户服务
│   │   │   ├── song/            # 歌曲服务
│   │   │   ├── playlist/        # 歌单服务
│   │   │   ├── lyric/           # 歌词服务
│   │   │   └── sync/            # 同步引擎
│   │   └── ... 
│   ├── pkg/
│   │   ├── auth/                # JWT 认证
│   │   ├── config/              # 配置加载
│   │   ├── metadata/            # 音频元数据解析
│   │   └── storage/             # 文件存储 (local/S3)
│   └── tests/e2e/               # 端到端测试
├── echo_vault_app/              # Flutter 客户端
│   └── lib/
│       ├── core/                # 基础层(db/gRPC/REST/配置)
│       ├── features/            # 功能模块
│       │   ├── auth/            # 登录/注册
│       │   ├── library/         # 曲库
│       │   ├── player/          # 播放器
│       │   ├── playlist/        # 歌单
│       │   ├── publish/         # 发布
│       │   ├── device/          # 设备管理
│       │   ├── sync/            # 同步状态
│       │   ├── setup/           # 服务器配置
│       │   └── navigation/      # 导航
│       └── providers/           # 全局状态
├── docs/
│   ├── deployment/              # 部署文档
│   ├── user/                    # 用户文档
│   └── superpowers/             # 设计文档和计划
└── proto/                       # gRPC Protobuf 定义
```

---

## 📚 文档

| 文档 | 说明 |
|:---|:---|
| [架构设计](docs/superpowers/specs/2026-06-30-echovault-design.md) | 完整架构设计文档 |
| [Docker 部署](docs/deployment/docker-compose.md) | Docker Compose 详细指南 |
| [配置参考](docs/deployment/configuration.md) | 环境变量和配置说明 |
| [首次配置](docs/deployment/first-time-setup.md) | 从零开始的部署指南 |
| [用户手册](docs/user/user-manual.md) | 功能使用说明 |
| [FAQ](docs/user/faq.md) | 常见问题解答 |
| [故障排除](docs/user/troubleshooting.md) | 常见错误解决方案 |

---

## 🧪 测试

```bash
# 后端单元测试
cd echovault-server
go test ./internal/... -v

# 后端端到端测试
go test ./tests/e2e/... -v

# Flutter 测试
cd echo_vault_app
flutter test
```

---

## 🔧 开发指南

### 生成 Protobuf 代码

```bash
cd proto
buf generate
```

### 生成 Ent 代码

```bash
cd echovault-server
go generate ./internal/ent/...
```

### 代码规范

```bash
# 后端 lint
cd echovault-server
golangci-lint run ./...

# Flutter lint
cd echo_vault_app
dart analyze
```

---

## 📋 项目状态

| 阶段 | 状态 |
|:---|:---:|
| Phase 1-5: Go 后端（数据层、业务逻辑、同步引擎） | ✅ 完成 |
| Phase 6-9: Flutter 前端（UI、播放器、设备管理、同步） | ✅ 完成 |
| Phase 10a: 部署文档 | ✅ 完成 |
| Phase 10b: 用户文档 | ✅ 完成 |
| Phase 10c: 端到端测试 | ✅ 完成 |
| Phase 10d: 性能优化 | ✅ 完成 |

---

## 📄 许可证

本项目基于 Apache License 2.0 开源。详见 [LICENSE](LICENSE) 文件。
