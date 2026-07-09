# EchoVault 故障排除指南

> 更新日期：2026-07-09

---

## 目录

1. [服务器无法启动](#1-服务器无法启动)
2. [数据库问题](#2-数据库问题)
3. [网络连接问题](#3-网络连接问题)
4. [客户端问题](#4-客户端问题)
5. [文件上传/下载问题](#5-文件上传下载问题)
6. [同步问题](#6-同步问题)
7. [性能问题](#7-性能问题)

---

## 1. 服务器无法启动

### 1.1 Docker 容器退出

**现象：** `docker compose up -d` 后容器状态为 `Exited`

**排查步骤：**

```bash
# 1. 查看完整日志
docker compose logs echovault-server

# 2. 常见错误及解决
```

**错误 1：端口被占用**

```
listen tcp :9090: bind: address already in use
```

**解决：**

```bash
# 查找占用端口的进程
sudo lsof -i :9090

# 或修改 docker-compose.yml 中的端口映射
# 将 "9090:9090" 改为 "9092:9090"
```

**错误 2：数据库无法打开**

```
failed to open database: unable to open database file
```

**解决：**

```bash
# 检查数据目录权限
ls -la ./data
sudo chown -R 1000:1000 ./data

# 或删除损坏的数据库重建
rm -f ./data/echovault.db
docker compose restart
```

**错误 3：权限不足**

```
permission denied
```

**解决：**

```bash
# 确保 Docker 用户有权访问数据目录
sudo chown -R $(id -u):$(id -g) ./data
```

### 1.2 构建失败

**现象：** `docker compose up -d --build` 报错

**解决：**

```bash
# 清除 Docker 构建缓存
docker compose build --no-cache

# 或手动构建测试
cd echovault-server
CGO_ENABLED=0 go build ./cmd/server
```

---

## 2. 数据库问题

### 2.1 数据库损坏

**现象：** 查询报错 `database disk image is malformed`

**解决：**

```bash
# 1. 停止服务
docker compose down

# 2. 备份损坏的数据库
cp ./data/echovault.db ./data/echovault.db.corrupt

# 3. 尝试恢复
sqlite3 ./data/echovault.db ".recover" | sqlite3 ./data/echovault-recovered.db

# 4. 替换并重启
mv ./data/echovault-recovered.db ./data/echovault.db
docker compose up -d
```

### 2.2 数据库迁移失败

**现象：** 升级版本后启动报错

**解决：**

```bash
# 查看迁移错误详情
docker compose logs echovault-server

# 如果 schema 变更不兼容，可能需要重建数据库
# 注意：这将丢失所有数据！
docker compose down
rm -f ./data/echovault.db
docker compose up -d
```

---

## 3. 网络连接问题

### 3.1 客户端无法连接服务器

**排查流程：**

```bash
# 1. 确认服务运行
docker compose ps

# 2. 检查服务器防火墙
sudo ufw status
# 如有必要：sudo ufw allow 8080/tcp

# 3. 测试网络连通性（在客户端设备上）
curl -v http://<服务器IP>:8080
# 预期输出包含 envoy 响应

# 4. 检查 Envoy 是否正常
docker compose logs envoy
```

### 3.2 gRPC 调用失败

**现象：** 客户端操作返回 "gRPC error"

**常见原因：**

| 错误码 | 原因 | 解决 |
|:---:|:---|:---|
| 14 (Unavailable) | 服务器未就绪或网络不可达 | 检查服务状态和网络 |
| 16 (Unauthenticated) | JWT 令牌无效或过期 | 重新登录 |
| 3 (InvalidArgument) | 请求参数错误 | 检查输入数据 |
| 13 (Internal) | 服务器内部错误 | 检查服务器日志 |

---

## 4. 客户端问题

### 4.1 应用崩溃

**Android：**

```bash
# 查看崩溃日志
adb logcat | grep -i "echovault"

# 清除应用数据
adb shell pm clear com.echovault.app
```

**iOS：**

查看 Xcode 设备日志（Organizer → Devices → View Device Logs）

**通用解决：**

1. 重启应用
2. 检查是否有新版本
3. 清除应用缓存（设置 → 清除缓存）

### 4.2 连接配置丢失

**现象：** 每次打开应用都需要重新配置服务器地址

**解决：** 可能是 SharedPreferences 数据损坏。清除应用数据后重新配置。

### 4.3 歌曲列表不刷新

**现象：** 上传新歌后列表未更新

**解决：** 下拉刷新曲库页面，或重启应用。

---

## 5. 文件上传/下载问题

### 5.1 上传进度卡住

**排查步骤：**

1. 检查网络连接是否稳定
2. 检查服务器磁盘空间：`docker compose exec echovault-server df -h /app/data`
3. 查看服务器日志：`docker compose logs --tail=50 echovault-server`
4. 尝试上传较小的文件

### 5.2 文件哈希冲突

**现象：** 上传文件提示"文件已存在"

**说明：** 这是正常行为。EchoVault 通过 SHA256 文件哈希去重，如果同一个文件已经上传过，不会重复存储。

### 5.3 下载失败

**现象：** 从云端下载歌曲失败

**解决：**

1. 检查文件在服务器上是否存在：`ls -la ./data/files/songs/`
2. 检查磁盘空间
3. 查看服务器 REST 日志

---

## 6. 同步问题

### 6.1 同步一直停留在"同步中"

**排查：**

1. 检查网络连接
2. 查看同步状态详情页的错误信息
3. 检查服务器日志中是否有同步错误

### 6.2 设备间数据不一致

**原因：** 同步有一定延迟（1-2 分钟），也可能是某台设备未联网

**解决：**

1. 确保所有设备联网
2. 手动触发同步（同步页面 → 立即同步）
3. 等待片刻后刷新

### 6.3 同步冲突

**现象：** 两台设备修改了同一数据，后修改的覆盖了先修改的

**说明：** 这是当前冲突解决策略（last-write-wins）的设计行为。如果这对你的使用造成困扰，请在 GitHub 提交 Issue 讨论。

---

## 7. 性能问题

### 7.1 服务器响应慢

**优化建议：**

1. 检查服务器资源使用：`docker stats`
2. 考虑增加内存限制
3. 使用 PostgreSQL 替代 SQLite（多用户场景）
4. 确保数据库文件在 SSD 上

### 7.2 客户端启动慢

**优化建议：**

1. 减少自动扫描的目录数量
2. 清除本地缓存：设置 → 清除缓存
3. 升级设备硬件

### 7.3 大量歌曲时列表卡顿

**优化建议：**

1. 使用搜索功能而非滚动浏览
2. 关闭不必要的动画效果（如果系统提供此选项）
3. 分批加载，耐心等待
