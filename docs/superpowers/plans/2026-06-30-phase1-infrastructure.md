# Phase 1: 基础设施搭建 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 搭建 EchoVault 后端 Go 项目骨架、Ent ORM 初始化、gRPC 服务框架、Envoy 代理、Docker Compose 开发环境

**Architecture:** 离线优先边缘架构的同步中枢。Go 单体应用按领域分包（user/song/playlist/lyric/sync），通过 Ent 操作 SQLite，对外暴露 gRPC 接口，Envoy 代理提供 gRPC-Web 转换。

**Tech Stack:** Go 1.23+, Ent, SQLite, gRPC, Envoy, Docker Compose, Buf

---

### Task 1: Go 项目脚手架

**Files:**
- Create: `echovault-server/go.mod`
- Create: `echovault-server/go.sum`
- Create: `echovault-server/cmd/server/main.go`
- Create: `echovault-server/Dockerfile`
- Modify: `.gitignore`
- Modify: `docker-compose.yml`

- [ ] **Step 1: 初始化 Go module**

Run: `cd /home/ink/Desktop/develop/EchoVault/echovault-server && go mod init github.com/inkOrCloud/EchoVault/echovault-server`

- [ ] **Step 2: 创建入口文件**

Create `echovault-server/cmd/server/main.go`:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"log/slog"
	"net"
	"os"
	"os/signal"
	"syscall"

	"github.com/spf13/viper"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

func main() {
	// 加载配置
	viper.SetDefault("grpc_port", 9090)
	viper.SetDefault("db_driver", "sqlite3")
	viper.SetDefault("db_path", "data/echovault.db")
	viper.SetDefault("storage_type", "local")
	viper.SetDefault("storage_path", "data/files")
	viper.AutomaticEnv()

	// 启动 gRPC 服务
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", viper.GetInt("grpc_port")))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	s := grpc.NewServer()
	reflection.Register(s)

	// 优雅关闭
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigCh
		slog.Info("shutting down server...")
		cancel()
		s.GracefulStop()
	}()

	slog.Info("starting EchoVault server", "port", viper.GetInt("grpc_port"))
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

- [ ] **Step 3: 创建 Dockerfile**

Create `echovault-server/Dockerfile`:

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

- [ ] **Step 4: 创建 docker-compose.yml（项目根目录）**

Create `/home/ink/Desktop/develop/EchoVault/docker-compose.yml`:

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

- [ ] **Step 5: 更新 .gitignore**

Append to `/home/ink/Desktop/develop/EchoVault/.gitignore`:

```
# EchoVault
data/
.env

# IDE
.idea/
.vscode/

# OS
.DS_Store
Thumbs.db
```

- [ ] **Step 6: 初始化 Go 依赖并提交**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
go get google.golang.org/grpc@latest
go get google.golang.org/protobuf@latest
go get github.com/spf13/viper@latest
go mod tidy
```

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echovault-server/ docker-compose.yml .gitignore
git commit -m "feat: scaffold Go project with gRPC server skeleton"
```

---

### Task 2: Envoy gRPC-Web 代理配置

**Files:**
- Create: `envoy.yaml`（项目根目录）

- [ ] **Step 1: 创建 Envoy 配置文件**

Create `/home/ink/Desktop/develop/EchoVault/envoy.yaml`:

```yaml
static_resources:
  listeners:
  - name: main_listener
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              # gRPC-Web 路由
              - match: { prefix: "/echo_vault.", grpc: {} }
                route:
                  cluster: echovault_grpc
                  timeout: 0s
              # REST 文件传输路由
              - match: { prefix: "/api/v1/" }
                route:
                  cluster: echovault_rest
                  timeout: 0s
          http_filters:
          - name: envoy.filters.http.grpc_web
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
          - name: envoy.filters.http.cors
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors
              allow_origin_string_match:
              - prefix: "*"
              allow_methods: GET, POST, PUT, DELETE, OPTIONS
              allow_headers: content-type, x-grpc-web, grpc-timeout, authorization
              max_age: "86400"
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
  - name: echovault_grpc
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    load_assignment:
      cluster_name: echovault_grpc
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: echovault-server, port_value: 9090 }

  - name: echovault_rest
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: echovault_rest
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: echovault-server, port_value: 9090 }
```

- [ ] **Step 2: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add envoy.yaml
git commit -m "feat: add Envoy gRPC-Web + REST proxy configuration"
```

---

### Task 3: Ent Schema 定义

**Files:**
- Create: `echovault-server/internal/ent/schema/user.go`
- Create: `echovault-server/internal/ent/schema/song.go`
- Create: `echovault-server/internal/ent/schema/lyric.go`
- Create: `echovault-server/internal/ent/schema/playlist.go`
- Create: `echovault-server/internal/ent/schema/playlist_song.go`
- Create: `echovault-server/internal/ent/schema/device.go`
- Create: `echovault-server/internal/ent/schema/sync_log.go`

- [ ] **Step 1: 安装 Ent CLI 并初始化**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
go get entgo.io/ent@latest
go run entgo.io/ent/cmd/ent init --target internal/ent/schema User Song Lyric Playlist PlaylistSong Device SyncLog
```

- [ ] **Step 2: 定义 User Schema**

Write `echovault-server/internal/ent/schema/user.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type User struct {
	ent.Schema
}

func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").Unique().Immutable(),
		field.String("username").Unique().MaxLen(64),
		field.String("display_name").MaxLen(128).Default(""),
		field.String("password_hash"),
		field.String("role").Default("user"), // "admin" | "user"
		field.Bool("is_deleted").Default(false),
		field.Int64("version").Default(0),
		field.Time("created_at"),
		field.Time("updated_at"),
	}
}

func (User) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("username"),
	}
}
```

- [ ] **Step 3: 定义 Song Schema**

Write `echovault-server/internal/ent/schema/song.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type Song struct {
	ent.Schema
}

func (Song) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").Unique().Immutable(),
		field.String("title").MaxLen(512),
		field.String("artist").MaxLen(256).Default(""),
		field.String("album").MaxLen(256).Default(""),
		field.String("genre").MaxLen(128).Default(""),
		field.Int32("track_number").Default(0),
		field.Int32("disc_number").Default(1),
		field.Int32("duration_ms").Default(0),
		field.Int32("year").Default(0),
		field.String("file_name").MaxLen(512).Default(""),
		field.Int64("file_size").Default(0),
		field.String("file_hash").MaxLen(64).Default(""),   // SHA256
		field.String("mime_type").MaxLen(64).Default(""),
		field.Int32("bitrate").Default(0),
		field.Int32("sample_rate").Default(0),
		field.String("source").Default("local"),             // local | uploaded | synced
		field.String("file_status").Default("local_only"),   // local_only | uploaded | downloaded | cloud_only
		field.String("owner_id").Default(""),
		field.Bool("is_deleted").Default(false),
		field.Int64("version").Default(0),
		field.Time("created_at"),
		field.Time("updated_at"),
	}
}

func (Song) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("file_hash"),
		index.Fields("owner_id"),
		index.Fields("title", "artist"),
	}
}
```

- [ ] **Step 4: 定义 Lyric Schema**

Write `echovault-server/internal/ent/schema/lyric.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type Lyric struct {
	ent.Schema
}

func (Lyric) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").Unique().Immutable(),
		field.String("song_id"),
		field.Text("content"),        // LRC 格式
		field.String("type").Default("original"),  // original | translation | phonetic
		field.String("language").Default(""),
		field.Int32("offset_ms").Default(0),
		field.String("source").Default("manual"),  // embedded | manual | fetched
		field.Bool("is_deleted").Default(false),
		field.Int64("version").Default(0),
		field.Time("created_at"),
		field.Time("updated_at"),
	}
}

func (Lyric) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("song_id"),
	}
}
```

- [ ] **Step 5: 定义 Playlist Schema**

Write `echovault-server/internal/ent/schema/playlist.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

type Playlist struct {
	ent.Schema
}

func (Playlist) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").Unique().Immutable(),
		field.String("name").MaxLen(256),
		field.String("description").MaxLen(1024).Default(""),
		field.String("cover_url").Default(""),
		field.String("type").Default("user"),  // user | favorite | auto
		field.String("owner_id"),
		field.Bool("is_public").Default(false),
		field.Int32("song_count").Default(0),
		field.Bool("is_deleted").Default(false),
		field.Int64("version").Default(0),
		field.Time("created_at"),
		field.Time("updated_at"),
	}
}
```

- [ ] **Step 6: 定义 PlaylistSong Schema**

Write `echovault-server/internal/ent/schema/playlist_song.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type PlaylistSong struct {
	ent.Schema
}

func (PlaylistSong) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").Unique().Immutable(),
		field.String("playlist_id"),
		field.String("song_id"),
		field.Int32("position").Default(0),
		field.String("added_by").Default(""),
		field.Int64("version").Default(0),
		field.Time("created_at"),
	}
}

func (PlaylistSong) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("playlist_id", "song_id").Unique(),
		index.Fields("playlist_id", "position"),
	}
}
```

- [ ] **Step 7: 定义 Device Schema**

Write `echovault-server/internal/ent/schema/device.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type Device struct {
	ent.Schema
}

func (Device) Fields() []ent.Field {
	return []ent.Field{
		field.String("device_id").Unique().Immutable(),
		field.String("device_name").MaxLen(128).Default(""),
		field.String("platform").Default(""),
		field.String("os_version").Default(""),
		field.String("client_version").Default(""),
		field.String("user_id"),
		field.Time("last_sync_at").Optional().Nillable(),
		field.String("sync_state").Default("pending"),
		field.Time("created_at"),
		field.Time("updated_at"),
	}
}

func (Device) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("user_id"),
	}
}
```

- [ ] **Step 8: 定义 SyncLog Schema**

Write `echovault-server/internal/ent/schema/sync_log.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type SyncLog struct {
	ent.Schema
}

func (SyncLog) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").Unique().Immutable(),
		field.String("device_id"),
		field.String("entity_type"),   // song | playlist | playlist_song | lyric
		field.String("entity_id"),
		field.String("action"),        // create | update | delete
		field.Int64("version"),
		field.Bytes("data").Optional(), // 完整实体 protobuf 序列化
		field.Time("timestamp"),
		field.Bool("acked").Default(false),
	}
}

func (SyncLog) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("device_id", "version"),
		index.Fields("acked"),
	}
}
```

- [ ] **Step 9: 生成 Ent 代码**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
go generate ./internal/ent/...
```

Expected output: Generated files in `internal/ent/` (client.go, migrate/, enttest/, etc.)

- [ ] **Step 10: 创建数据库初始化逻辑**

Modify `echovault-server/cmd/server/main.go` to add DB initialization:

```go
package main

import (
	// ... existing imports ...
	_ "github.com/mattn/go-sqlite3"
	"entgo.io/ent/dialect"
	entsql "entgo.io/ent/dialect/sql"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
)

func main() {
	// ... viper config ...
	
	// 初始化数据库
	dbPath := viper.GetString("db_path")
	drv, err := entsql.Open(dialect.SQLite, dbPath+"?_journal_mode=WAL&_fk=1")
	if err != nil {
		log.Fatalf("failed to open database: %v", err)
	}
	defer drv.Close()
	
	client := ent.NewClient(ent.Driver(drv))
	defer client.Close()
	
	// 自动迁移
	ctx := context.Background()
	if err := client.Schema.Create(ctx); err != nil {
		log.Fatalf("failed to create schema: %v", err)
	}

	// ... gRPC server ...
}
```

- [ ] **Step 11: 安装 SQLite 驱动并提交**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
go get github.com/mattn/go-sqlite3@latest
go mod tidy
```

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echovault-server/internal/
git commit -m "feat: add Ent schema definitions for all 7 entities"
```

---

### Task 4: Buf 代码生成 + gRPC 服务注册

**Files:**
- Modify: `proto/`（子模块）
- Create: `echovault-server/api/grpc/generated/`（Buf 生成代码）
- Create: `echovault-server/internal/grpc/`——服务注册
- Create: `echo_vault_app/lib/models/generated/`（Buf 生成代码）

- [ ] **Step 1: 初始化 proto 子模块**

```bash
cd /home/ink/Desktop/develop/EchoVault
git submodule update --init --recursive
```

- [ ] **Step 2: 用 Buf 生成 Go 代码**

```bash
cd /home/ink/Desktop/develop/EchoVault/proto
buf generate
```

Expected: Go proto + gRPC code in `../echovault-server/api/grpc/generated/echo_vault/`

- [ ] **Step 3: 创建 gRPC 服务注册器**

Create `echovault-server/internal/grpc/register.go`:

```go
package grpc

import (
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"

	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
)

// RegisterAll 注册所有 gRPC 服务到给定服务器
func RegisterAll(s *grpc.Server, client *ent.Client) {
	// Phase 2 及以后在此注册具体服务实现
	// userpb.RegisterUserServiceServer(s, userService)
	// songpb.RegisterSongServiceServer(s, songService)
	// ...

	reflection.Register(s)
}
```

- [ ] **Step 4: 更新 main.go 使用注册器**

Modify `echovault-server/cmd/server/main.go` 调用 `grpc.RegisterAll(s, client)` 替代 `reflection.Register(s)`

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echovault-server/api/ echovault-server/internal/grpc/
git commit -m "feat: add Buf-generated gRPC code and service registrar"
```

---

### Task 5: 存储抽象层

**Files:**
- Create: `echovault-server/pkg/storage/storage.go`
- Create: `echovault-server/pkg/storage/local.go`
- Create: `echovault-server/pkg/storage/s3.go`

- [ ] **Step 1: 定义 Storage 接口**

Create `echovault-server/pkg/storage/storage.go`:

```go
package storage

import (
	"context"
	"io"
)

// Storage 文件存储抽象
type Storage interface {
	// SaveAudio 保存音频文件
	SaveAudio(ctx context.Context, songID, filename string, reader io.Reader) error
	// GetAudio 获取音频文件
	GetAudio(ctx context.Context, songID string) (io.ReadCloser, int64, error)
	// SaveCover 保存封面
	SaveCover(ctx context.Context, songID string, reader io.Reader) error
	// GetCover 获取封面
	GetCover(ctx context.Context, songID string) (io.ReadCloser, int64, error)
	// DeleteSongFiles 删除歌曲相关文件
	DeleteSongFiles(ctx context.Context, songID string) error
}
```

- [ ] **Step 2: 实现本地文件系统存储**

Create `echovault-server/pkg/storage/local.go`:

```go
package storage

import (
	"context"
	"fmt"
	"io"
	"os"
	"path/filepath"
)

type LocalStorage struct {
	basePath string
}

func NewLocalStorage(basePath string) (*LocalStorage, error) {
	if err := os.MkdirAll(filepath.Join(basePath, "songs"), 0755); err != nil {
		return nil, fmt.Errorf("create songs dir: %w", err)
	}
	if err := os.MkdirAll(filepath.Join(basePath, "covers"), 0755); err != nil {
		return nil, fmt.Errorf("create covers dir: %w", err)
	}
	return &LocalStorage{basePath: basePath}, nil
}

func (s *LocalStorage) SaveAudio(_ context.Context, songID, filename string, reader io.Reader) error {
	dir := filepath.Join(s.basePath, "songs", songID)
	if err := os.MkdirAll(dir, 0755); err != nil {
		return err
	}
	dst, err := os.Create(filepath.Join(dir, filename))
	if err != nil {
		return err
	}
	defer dst.Close()
	_, err = io.Copy(dst, reader)
	return err
}

func (s *LocalStorage) GetAudio(_ context.Context, songID string) (io.ReadCloser, int64, error) {
	dir := filepath.Join(s.basePath, "songs", songID)
	entries, err := os.ReadDir(dir)
	if err != nil {
		return nil, 0, err
	}
	if len(entries) == 0 {
		return nil, 0, os.ErrNotExist
	}
	fpath := filepath.Join(dir, entries[0].Name())
	fi, err := os.Stat(fpath)
	if err != nil {
		return nil, 0, err
	}
	f, err := os.Open(fpath)
	if err != nil {
		return nil, 0, err
	}
	return f, fi.Size(), nil
}

func (s *LocalStorage) SaveCover(_ context.Context, songID string, reader io.Reader) error {
	dst, err := os.Create(filepath.Join(s.basePath, "covers", songID+".jpg"))
	if err != nil {
		return err
	}
	defer dst.Close()
	_, err = io.Copy(dst, reader)
	return err
}

func (s *LocalStorage) GetCover(_ context.Context, songID string) (io.ReadCloser, int64, error) {
	fpath := filepath.Join(s.basePath, "covers", songID+".jpg")
	fi, err := os.Stat(fpath)
	if err != nil {
		return nil, 0, err
	}
	f, err := os.Open(fpath)
	if err != nil {
		return nil, 0, err
	}
	return f, fi.Size(), nil
}

func (s *LocalStorage) DeleteSongFiles(_ context.Context, songID string) error {
	os.RemoveAll(filepath.Join(s.basePath, "songs", songID))
	os.Remove(filepath.Join(s.basePath, "covers", songID+".jpg"))
	return nil
}
```

- [ ] **Step 3: 创建 S3 桩代码**

Create `echovault-server/pkg/storage/s3.go`:

```go
package storage

import (
	"context"
	"fmt"
	"io"
)

// S3Storage S3 兼容存储（预留，后续实现）
type S3Storage struct{}

func NewS3Storage(bucket, region string) (*S3Storage, error) {
	return nil, fmt.Errorf("S3 storage not yet implemented")
}

func (s *S3Storage) SaveAudio(_ context.Context, _, _ string, _ io.Reader) error {
	return fmt.Errorf("S3 storage not yet implemented")
}

func (s *S3Storage) GetAudio(_ context.Context, _ string) (io.ReadCloser, int64, error) {
	return nil, 0, fmt.Errorf("S3 storage not yet implemented")
}

func (s *S3Storage) SaveCover(_ context.Context, _ string, _ io.Reader) error {
	return fmt.Errorf("S3 storage not yet implemented")
}

func (s *S3Storage) GetCover(_ context.Context, _ string) (io.ReadCloser, int64, error) {
	return nil, 0, fmt.Errorf("S3 storage not yet implemented")
}

func (s *S3Storage) DeleteSongFiles(_ context.Context, _ string) error {
	return fmt.Errorf("S3 storage not yet implemented")
}
```

- [ ] **Step 4: 添加 Storage 工厂函数**

Append to `echovault-server/pkg/storage/storage.go`:

```go
// NewStorage 根据配置创建存储后端
func NewStorage(storageType, storagePath string) (Storage, error) {
	switch storageType {
	case "local":
		return NewLocalStorage(storagePath)
	case "s3":
		return NewS3Storage(storagePath, "")
	default:
		return NewLocalStorage(storagePath)
	}
}
```

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echovault-server/pkg/storage/
git commit -m "feat: add file storage abstraction layer with local FS implementation"
```

---

### Task 6: 配置与管理工具

**Files:**
- Create: `echovault-server/pkg/config/config.go`
- Create: `echovault-server/Makefile`
- Modify: `echovault-server/cmd/server/main.go`

- [ ] **Step 1: 创建配置管理**

Create `echovault-server/pkg/config/config.go`:

```go
package config

import "github.com/spf13/viper"

type Config struct {
	GRPCPort   int
	DBDriver   string
	DBPath     string
	StorageType string
	StoragePath string
	JWTSecret  string
}

func Load() (*Config, error) {
	viper.SetConfigName("config")
	viper.SetConfigType("yaml")
	viper.AddConfigPath(".")
	viper.AddConfigPath("/app")

	viper.SetDefault("grpc_port", 9090)
	viper.SetDefault("db_driver", "sqlite3")
	viper.SetDefault("db_path", "data/echovault.db")
	viper.SetDefault("storage_type", "local")
	viper.SetDefault("storage_path", "data/files")
	viper.SetDefault("jwt_secret", "change-me-in-production")

	viper.AutomaticEnv()

	if err := viper.ReadInConfig(); err != nil {
		if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
			return nil, err
		}
	}

	return &Config{
		GRPCPort:    viper.GetInt("grpc_port"),
		DBDriver:    viper.GetString("db_driver"),
		DBPath:      viper.GetString("db_path"),
		StorageType: viper.GetString("storage_type"),
		StoragePath: viper.GetString("storage_path"),
		JWTSecret:   viper.GetString("jwt_secret"),
	}, nil
}
```

- [ ] **Step 2: 创建 Makefile**

Create `echovault-server/Makefile`:

```makefile
.PHONY: generate build run test clean

# 生成 Ent 代码
generate:
	go generate ./internal/ent/...

# 生成 proto 代码（需要在 proto 目录执行 buf generate）
proto-gen:
	cd ../proto && buf generate

# 构建
build:
	go build -o bin/server ./cmd/server

# 运行
run:
	go run ./cmd/server

# 测试
test:
	go test ./... -v -race -count=1

# 清理
clean:
	rm -rf bin/ data/
```

- [ ] **Step 3: 重构 main.go 使用 Config**

Update `echovault-server/cmd/server/main.go` to use `config.Load()`

- [ ] **Step 4: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echovault-server/pkg/config/ echovault-server/Makefile
git commit -m "chore: add config management and Makefile"
```

---

### Self-Review Checklist

Phase 1 完成后：

1. ✅ Go 项目可以 `go run ./cmd/server` 启动，监听 9090
2. ✅ Ent 自动建表（SQLite）
3. ✅ `docker-compose up` 启动三个容器（server + envoy），端口 8080 可用
4. ✅ Buf 可以正常生成 Go proto 代码
5. ✅ `go test ./...` 全部通过
6. ✅ `make generate` + `make build` 正常工作

**入口：** `Phase 2: 用户系统 + 设备管理`

---

### Task 7: Service Converter 层（ent ↔ proto 双向映射）

**Files:**
- Create: `echovault-server/internal/service/song/convert.go`
- Create: `echovault-server/internal/service/user/convert.go`
- Create: `echovault-server/internal/service/playlist/convert.go`
- Create: `echovault-server/internal/service/lyric/convert.go`
- Create: `echovault-server/internal/service/sync/convert.go`
- Create: `echovault-server/pkg/convert/helpers.go`

- [ ] **Step 1: 创建公共转换辅助函数**

Create `echovault-server/pkg/convert/helpers.go`:

```go
// Package convert 提供 ent ↔ proto 转换的共享辅助函数
package convert

import (
	"time"

	"google.golang.org/protobuf/types/known/timestamppb"
)

// PTime 将 time.Time 转为 protobuf Timestamp
func PTime(t time.Time) *timestamppb.Timestamp {
	if t.IsZero() {
		return nil
	}
	return timestamppb.New(t)
}

// FileSourceToEnt 将 proto 枚举转为 ent 存储字符串
func FileSourceToProto(s string) string {
	// ent 存储 "local" | "uploaded" | "synced"
	// proto 枚举 FILE_SOURCE_LOCAL | FILE_SOURCE_UPLOADED | FILE_SOURCE_SYNCED
	switch s {
	case "local":
		return "FILE_SOURCE_LOCAL"
	case "uploaded":
		return "FILE_SOURCE_UPLOADED"
	case "synced":
		return "FILE_SOURCE_SYNCED"
	default:
		return "FILE_SOURCE_UNSPECIFIED"
	}
}

func FileSourceToEnt(s string) string {
	switch s {
	case "FILE_SOURCE_LOCAL":
		return "local"
	case "FILE_SOURCE_UPLOADED":
		return "uploaded"
	case "FILE_SOURCE_SYNCED":
		return "synced"
	default:
		return "local"
	}
}

// FileStatusToProto / ToEnt 同理
func FileStatusToProto(s string) string {
	switch s {
	case "local_only":
		return "FILE_STATUS_LOCAL_ONLY"
	case "uploaded":
		return "FILE_STATUS_UPLOADED"
	case "downloaded":
		return "FILE_STATUS_DOWNLOADED"
	case "cloud_only":
		return "FILE_STATUS_CLOUD_ONLY"
	default:
		return "FILE_STATUS_UNSPECIFIED"
	}
}

func FileStatusToEnt(s string) string {
	switch s {
	case "FILE_STATUS_LOCAL_ONLY":
		return "local_only"
	case "FILE_STATUS_UPLOADED":
		return "uploaded"
	case "FILE_STATUS_DOWNLOADED":
		return "downloaded"
	case "FILE_STATUS_CLOUD_ONLY":
		return "cloud_only"
	default:
		return "local_only"
	}
}
```

- [ ] **Step 2: 创建 Song Converter**

Create `echovault-server/internal/service/song/convert.go`:

```go
package song

import (
	commonpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/common/v1"
	songpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/song/v1"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/convert"
)

// EntToProto 将 ent.Song 转为 proto Song
func EntToProto(s *ent.Song) *songpb.Song {
	if s == nil {
		return nil
	}
	return &songpb.Song{
		Id:         s.ID,
		Title:      s.Title,
		Artist:     s.Artist,
		Album:      s.Album,
		Genre:      s.Genre,
		TrackNumber: s.TrackNumber,
		DiscNumber: s.DiscNumber,
		DurationMs: s.DurationMs,
		Year:       s.Year,
		FileName:   s.FileName,
		FileSize:   s.FileSize,
		FileHash:   s.FileHash,
		MimeType:   s.MimeType,
		Bitrate:    s.Bitrate,
		SampleRate: s.SampleRate,
		Source:     commonpb.FileSource(commonpb.FileSource_value[convert.FileSourceToProto(s.Source)]),
		FileStatus: commonpb.FileStatus(commonpb.FileStatus_value[convert.FileStatusToProto(s.FileStatus)]),
		OwnerId:    s.OwnerID,
		Version:    s.Version,
		IsDeleted:  s.IsDeleted,
		CreatedAt:  convert.PTime(s.CreatedAt),
		UpdatedAt:  convert.PTime(s.UpdatedAt),
	}
}

// ProtoToEntCreate 将 proto PublishSongRequest 转为 ent.Song 创建参数
func ProtoToEntCreate(req *songpb.PublishSongRequest) *ent.Song {
	return &ent.Song{
		Title:     req.Title,
		Artist:    req.Artist,
		Album:     req.Album,
		Genre:     req.Genre,
		TrackNumber: req.TrackNumber,
		DiscNumber: req.DiscNumber,
		Year:      req.Year,
		FileName:  req.FileName,
		FileSize:  req.FileSize,
		FileHash:  req.FileHash,
		MimeType:  req.MimeType,
		FileStatus: "uploaded",
	}
}

// EntsToProtoList 批量转换
func EntsToProtoList(songs []*ent.Song) []*songpb.Song {
	result := make([]*songpb.Song, len(songs))
	for i, s := range songs {
		result[i] = EntToProto(s)
	}
	return result
}
```

- [ ] **Step 3: 创建 User Converter**

Create `echovault-server/internal/service/user/convert.go`:

```go
package user

import (
	userpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/user/v1"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/convert"
)

func EntToProto(u *ent.User) *userpb.User {
	if u == nil {
		return nil
	}
	return &userpb.User{
		Id:          u.ID,
		Username:    u.Username,
		DisplayName: u.DisplayName,
		Role:        u.Role,
		CreatedAt:   convert.PTime(u.CreatedAt),
		UpdatedAt:   convert.PTime(u.UpdatedAt),
	}
}

func EntDeviceToProto(d *ent.Device) *userpb.Device {
	if d == nil {
		return nil
	}
	return &userpb.Device{
		DeviceId:    d.DeviceID,
		DeviceName:  d.DeviceName,
		Platform:    d.Platform,
		OsVersion:   d.OsVersion,
		ClientVersion: d.ClientVersion,
		LastSyncAt:  convert.PTime(d.LastSyncAt),
		RegisteredAt: convert.PTime(d.CreatedAt),
	}
}
```

- [ ] **Step 4: 创建 Playlist Converter**

Create `echovault-server/internal/service/playlist/convert.go`:

```go
package playlist

import (
	playlistpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/playlist/v1"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/convert"
)

func EntToProto(p *ent.Playlist) *playlistpb.Playlist {
	if p == nil {
		return nil
	}
	return &playlistpb.Playlist{
		Id:          p.ID,
		Name:        p.Name,
		Description: p.Description,
		CoverUrl:    p.CoverURL,
		Type:        playlistTypeToProto(p.Type),
		OwnerId:     p.OwnerID,
		IsPublic:    p.IsPublic,
		SongCount:   p.SongCount,
		Version:     p.Version,
		CreatedAt:   convert.PTime(p.CreatedAt),
		UpdatedAt:   convert.PTime(p.UpdatedAt),
	}
}

func playlistTypeToProto(s string) playlistpb.Playlist_Type {
	switch s {
	case "user":
		return playlistpb.Playlist_TYPE_USER
	case "favorite":
		return playlistpb.Playlist_TYPE_FAVORITE
	case "auto":
		return playlistpb.Playlist_TYPE_AUTO
	default:
		return playlistpb.Playlist_TYPE_UNSPECIFIED
	}
}
```

- [ ] **Step 5: 创建 Lyric Converter**

Create `echovault-server/internal/service/lyric/convert.go`:

```go
package lyric

import (
	lyricpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/lyric/v1"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/convert"
)

func EntToProto(l *ent.Lyric) *lyricpb.Lyric {
	if l == nil {
		return nil
	}
	return &lyricpb.Lyric{
		SongId:   l.SongID,
		Content:  l.Content,
		Type:     lyricTypeToProto(l.Type),
		Language: l.Language,
		OffsetMs: l.OffsetMs,
		Source:   lyricSourceToProto(l.Source),
		Version:  l.Version,
		UpdatedAt: convert.PTime(l.UpdatedAt),
	}
}

func lyricTypeToProto(s string) lyricpb.Lyric_Type {
	switch s {
	case "original":
		return lyricpb.Lyric_TYPE_ORIGINAL
	case "translation":
		return lyricpb.Lyric_TYPE_TRANSLATION
	case "phonetic":
		return lyricpb.Lyric_TYPE_PHONETIC
	default:
		return lyricpb.Lyric_TYPE_UNSPECIFIED
	}
}

func lyricSourceToProto(s string) lyricpb.Lyric_Source {
	switch s {
	case "embedded":
		return lyricpb.Lyric_SOURCE_EMBEDDED
	case "manual":
		return lyricpb.Lyric_SOURCE_MANUAL
	case "fetched":
		return lyricpb.Lyric_SOURCE_FETCHED
	default:
		return lyricpb.Lyric_SOURCE_UNSPECIFIED
	}
}
```

- [ ] **Step 6: 创建 Sync Converter**

Create `echovault-server/internal/service/sync/convert.go`:

```go
package sync

import (
	syncpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/sync/v1"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/convert"
)

func EntToProto(s *ent.SyncLog) *syncpb.SyncChange {
	if s == nil {
		return nil
	}
	return &syncpb.SyncChange{
		EntityType: s.EntityType,
		EntityId:   s.EntityID,
		Action:     actionToProto(s.Action),
		Version:    s.Version,
		Data:       s.Data,
		Timestamp:  convert.PTime(s.Timestamp),
		DeviceId:   s.DeviceID,
	}
}

func actionToProto(s string) syncpb.SyncChange_Action {
	switch s {
	case "create":
		return syncpb.SyncChange_ACTION_CREATE
	case "update":
		return syncpb.SyncChange_ACTION_UPDATE
	case "delete":
		return syncpb.SyncChange_ACTION_DELETE
	default:
		return syncpb.SyncChange_ACTION_UNSPECIFIED
	}
}

func ProtoToEnt(pb *syncpb.SyncChange) *ent.SyncLog {
	return &ent.SyncLog{
		EntityType: pb.EntityType,
		EntityID:   pb.EntityID,
		Action:     pb.Action.String(),
		Version:    pb.Version,
		Data:       pb.Data,
		Timestamp:  pb.Timestamp.AsTime(),
		DeviceID:   pb.DeviceId,
	}
}
```

- [ ] **Step 7: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echovault-server/internal/service/ echovault-server/pkg/convert/
git commit -m "feat: add ent ↔ proto converter layer for all 5 services"
```
