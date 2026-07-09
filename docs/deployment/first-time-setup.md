# EchoVault 首次配置指南

> 版本：v1.0  
> 更新日期：2026-07-09

---

本文档指导你从零开始部署并运行 EchoVault 音乐管理系统。

---

## 目录

1. [准备工作](#1-准备工作)
2. [服务器部署](#2-服务器部署)
3. [初始化配置](#3-初始化配置)
4. [客户端连接](#4-客户端连接)
5. [上传音乐](#5-上传音乐)
6. [多设备同步](#6-多设备同步)
7. [常见问题](#7-常见问题)

---

## 1. 准备工作

### 1.1 硬件需求

- Linux 服务器（建议 Ubuntu 22.04+ 或 Debian 12+）
- 至少 512 MB 内存，1 GB 可用磁盘空间
- 如果需要外网访问，请确保服务器有公网 IP 或配置了内网穿透

### 1.2 软件安装

**安装 Docker：**

```bash
# Ubuntu / Debian
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
# 重新登录终端使组生效

# 验证安装
docker --version
docker compose version
```

### 1.3 准备域名（可选）

如需外网访问，建议配置域名解析到服务器 IP，并在后续步骤中配置 HTTPS。

---

## 2. 服务器部署

### 2.1 克隆项目

```bash
git clone --recurse-submodules https://github.com/inkOrCloud/EchoVault.git
cd EchoVault
```

### 2.2 配置安全密钥

```bash
# 生成安全的 JWT 密钥
echo "JWT_SECRET=$(openssl rand -base64 32)" > .env

# 确认写入成功
cat .env
# 输出：JWT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 2.3 启动服务

```bash
# 启动所有服务（后台运行）
docker compose up -d

# 等待启动完成（约 10-30 秒）
sleep 10

# 检查服务状态
docker compose ps
```

**预期输出：**

```
NAME                    IMAGE                       STATUS   PORTS
echovault-envoy-1       envoyproxy/envoy:v1.32     Up       0.0.0.0:8080->8080/tcp
echovault-server-1      echovault-echovault-server  Up       0.0.0.0:9090->9090/tcp
```

### 2.4 验证服务

```bash
# 查看启动日志
docker compose logs --tail=20 echovault-server
```

**正常日志应包含：**

```
database migrated successfully
starting EchoVault server port=9090
starting REST file server port=9091
```

---

## 3. 初始化配置

### 3.1 注册管理员账号

通过 Flutter 客户端注册，或在服务器端直接注册：

```bash
# 通过 gRPC 注册（需要 grpcurl）
SERVICE="echo_vault.user.v1.UserService/Register"
PAYLOAD='{"username":"admin","password":"your-password","display_name":"Admin"}'
grpcurl -plaintext -d "$PAYLOAD" localhost:8080 $SERVICE
```

**成功响应：**

```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "username": "admin",
    "displayName": "Admin"
  }
}
```

### 3.2 配置存储路径

确保数据目录有足够空间：

```bash
# 查看数据目录
ls -la ./data/

# 查看磁盘空间
df -h ./data
```

---

## 4. 客户端连接

### 4.1 安装 Flutter 客户端

**从源代码构建：**

```bash
cd echo_vault_app
flutter pub get
flutter run
```

**Android APK 构建：**

```bash
cd echo_vault_app
flutter build apk --release
# 安装包位于：build/app/outputs/flutter-apk/app-release.apk
```

### 4.2 首次连接

1. 启动 Flutter 客户端
2. 进入**服务器配置**页面
3. 输入服务器地址：`http://<服务器IP>:8080`
4. 点击"连接测试"
5. 如果显示连接成功，进入登录页面
6. 使用注册的账号密码登录

**局域网连接示例：**

```
服务器 IP：192.168.1.100
客户端填写：http://192.168.1.100:8080
```

### 4.3 故障排查

| 现象 | 原因 | 解决 |
|:---|:---|:---|
| 连接超时 | 防火墙未开放 8080 端口 | `sudo ufw allow 8080/tcp` |
| CORS 错误 | Envoy 配置问题 | 检查 `envoy.yaml` CORS 配置 |
| 证书错误 | 未配置 HTTPS | 本地开发用 `http://` |

---

## 5. 上传音乐

### 5.1 通过客户端上传

1. 登录后进入"我的曲库"页面
2. 点击"扫描"按钮，选择音频文件或目录
3. 系统自动解析音频元数据（标题、艺术家、专辑等）
4. 确认元数据无误后，点击"发布到服务器"
5. 文件上传完成即可在所有设备上访问

### 5.2 支持的音频格式

| 格式 | 说明 | 元数据支持 |
|:---|:---|:---:|
| MP3 | 最广泛兼容 | ID3v1/ID3v2 |
| FLAC | 无损格式 | Vorbis Comment |
| M4A / AAC | Apple 生态 | MP4 metadata |
| OGG | 开放格式 | Vorbis Comment |
| WAV | 未压缩 | RIFF metadata |
| OPUS | 现代编码 | Vorbis Comment |

---

## 6. 多设备同步

### 6.1 同步机制

EchoVault 采用**离线优先**同步策略：

```
设备 A ──► 上传歌曲 ──► 服务器 ──► 下发变更 ──► 设备 B
设备 B ──► 创建歌单 ──► 服务器 ──► 下发变更 ──► 设备 A
```

- 离线时：所有操作存储在本地 SQLite
- 联网后：自动增量同步，无需手动操作
- 冲突解决：基于版本号（last-write-wins）

### 6.2 设备管理

1. 在设备列表页查看已注册的设备
2. 每台设备注册时自动命名（如"XiaoMi-14-Pro"）
3. 可重命名或移除不再使用的设备

### 6.3 查看同步状态

- 主页顶部的同步指示器显示当前同步状态
- 同步详情页展示同步历史记录
- 支持手动触发同步（在同步页面点击"立即同步"）

---

## 7. 常见问题

### 7.1 如何备份数据？

```bash
# 一键备份所有数据
cd EchoVault
tar czf echovault-backup-$(date +%Y%m%d).tar.gz ./data .env
```

### 7.2 如何升级？

```bash
cd EchoVault
git pull --recurse-submodules
docker compose up -d --build
```

### 7.3 如何迁移到新服务器？

```bash
# 旧服务器：备份
cd EchoVault
docker compose down
tar czf echovault-migrate.tar.gz ./data .env

# 新服务器：安装 Docker，克隆项目
git clone --recurse-submodules https://github.com/inkOrCloud/EchoVault.git
cd EchoVault

# 恢复数据
tar xzf ../echovault-migrate.tar.gz

# 启动
docker compose up -d
```

### 7.4 如何更改 JWT 密钥？

修改 `.env` 中的 `JWT_SECRET`，然后重启服务：

```bash
docker compose restart
```

注意：更改密钥后所有现有登录令牌将失效，用户需要重新登录。

### 7.5 如何迁移到 PostgreSQL？

参见 [configuration.md](configuration.md#23-postgresql-配置) 中的 PostgreSQL 配置说明。
