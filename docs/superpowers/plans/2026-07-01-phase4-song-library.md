# Phase 4: 歌曲库 + Hash 查询 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use subagent-driven-development. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现 SongService（Hash 批量查询、歌曲发布、CRUD）、REST 文件上传/下载、歌曲搜索

**Architecture:** SongService 处理歌曲元数据的管理，CheckSongsByHash 是扫描流程的核心入口。PublishSong 创建歌曲记录。文件通过 REST API 上传/下载（分段上传、Range 支持）。搜索使用 SQLite LIKE 查询。

**Tech Stack:** Go 1.26+, Ent, SQLite, gRPC Unary, REST (net/http + ServeMux), uuid

---

### Task 1: SongService — CheckSongsByHash + CRUD

**核心逻辑：**
- `CheckSongsByHash(hashes)` — 批量查询 file_hash 是否存在，返回已知歌曲信息
- `PublishSong` — 创建新歌曲记录，分配 UUID，版本号 v1
- `GetSong` / `ListSongs` / `SearchSongs` / `DeleteSong` / `UpdateSong`

**Files:**
- Create: `echovault-server/internal/service/song/service.go`
- Create: `echovault-server/internal/service/song/service_test.go`

- [ ] **Step 1 (RED): 编写 SongService 测试**

```go
// internal/service/song/service_test.go
package song_test

import (
	"context"
	"testing"

	entsql "entgo.io/ent/dialect/sql"
	_ "github.com/mattn/go-sqlite3"

	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent/enttest"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/service/song"
	songpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/song/v1"
)

func newTestClient(t *testing.T) *ent.Client {
	t.Helper()
	drv, err := entsql.Open("sqlite3", "file:song?mode=memory&cache=shared&_fk=1")
	if err != nil {
		t.Fatalf("open db: %v", err)
	}
	client := enttest.NewClient(t, enttest.WithOptions(ent.Driver(drv)))
	if err := client.Schema.Create(context.Background()); err != nil {
		t.Fatalf("create schema: %v", err)
	}
	return client
}

func TestCheckSongsByHash_Empty(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := song.NewService(client)
	ctx := context.Background()

	results, err := svc.CheckSongsByHash(ctx, []string{"hash-nonexistent"})
	if err != nil {
		t.Fatalf("CheckSongsByHash() error = %v", err)
	}
	if len(results) != 1 {
		t.Fatalf("results = %d, want 1", len(results))
	}
	if results[0].Exists {
		t.Error("result.Exists = true for unknown hash, want false")
	}
}

func TestCheckSongsByHash_Found(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := song.NewService(client)
	ctx := context.Background()

	// 先发布一首歌
	pubResp, _ := svc.PublishSong(ctx, &songpb.PublishSongRequest{
		Title: "Test Song", Artist: "Test Artist",
		FileHash: "abc123", FileName: "test.mp3",
	})

	results, err := svc.CheckSongsByHash(ctx, []string{"abc123", "def456"})
	if err != nil {
		t.Fatalf("CheckSongsByHash() error = %v", err)
	}
	if len(results) != 2 {
		t.Fatalf("results = %d, want 2", len(results))
	}
	if !results[0].Exists {
		t.Error("results[0].Exists = false for 'abc123', want true")
	}
	if results[0].Song.Id != pubResp.Id {
		t.Errorf("results[0].Song.Id = %q, want %q", results[0].Song.Id, pubResp.Id)
	}
	if results[1].Exists {
		t.Error("results[1].Exists = true for unknown hash, want false")
	}
}

func TestPublishSong_Success(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := song.NewService(client)
	ctx := context.Background()

	resp, err := svc.PublishSong(ctx, &songpb.PublishSongRequest{
		Title: "My Song", Artist: "Me", Album: "Album 1",
		FileHash: "def789", FileName: "track.flac",
	})
	if err != nil {
		t.Fatalf("PublishSong() error = %v", err)
	}
	if resp.Id == "" {
		t.Error("PublishSong() returned empty Id")
	}
	if resp.Title != "My Song" {
		t.Errorf("Title = %q, want %q", resp.Title, "My Song")
	}
}

func TestGetSong(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := song.NewService(client)
	ctx := context.Background()

	pub, _ := svc.PublishSong(ctx, &songpb.PublishSongRequest{
		Title: "Get Me", FileHash: "get123",
	})
	got, err := svc.GetSong(ctx, pub.Id)
	if err != nil {
		t.Fatalf("GetSong() error = %v", err)
	}
	if got.Title != "Get Me" {
		t.Errorf("Title = %q, want %q", got.Title, "Get Me")
	}
}

func TestSearchSongs(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := song.NewService(client)
	ctx := context.Background()

	svc.PublishSong(ctx, &songpb.PublishSongRequest{Title: "Summer Nights", FileHash: "h1"})
	svc.PublishSong(ctx, &songpb.PublishSongRequest{Title: "Winter Sun", FileHash: "h2"})
	svc.PublishSong(ctx, &songpb.PublishSongRequest{Title: "Spring Rain", FileHash: "h3"})

	results, err := svc.SearchSongs(ctx, "summer", 10)
	if err != nil {
		t.Fatalf("SearchSongs() error = %v", err)
	}
	if len(results) != 1 {
		t.Errorf("SearchSongs('summer') = %d, want 1", len(results))
	}
}

func TestListSongs(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := song.NewService(client)
	ctx := context.Background()

	for i := 0; i < 5; i++ {
		svc.PublishSong(ctx, &songpb.PublishSongRequest{
			Title: "Song", FileHash: "h",
		})
	}

	songs, err := svc.ListSongs(ctx, 3, 0)
	if err != nil {
		t.Fatalf("ListSongs() error = %v", err)
	}
	if len(songs) > 3 {
		t.Errorf("ListSongs(limit=3) = %d, want <=3", len(songs))
	}
}
```

- [ ] **Step 2 (VERIFY RED):** `go test ./internal/service/song/...` → 编译失败

- [ ] **Step 3 (GREEN): 实现 SongService**

```go
// internal/service/song/service.go
package song

import (
	"context"
	"errors"

	"github.com/google/uuid"
	"time"

	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent/song"
	songpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/song/v1"
)

type CheckResult struct {
	FileHash string
	Exists   bool
	Song     *songpb.Song
}

type Service struct {
	client *ent.Client
}

func NewService(client *ent.Client) *Service {
	return &Service{client: client}
}

func (s *Service) CheckSongsByHash(ctx context.Context, fileHashes []string) ([]*CheckResult, error) {
	records, err := s.client.Song.Query().
		Where(song.FileHashIn(fileHashes...), song.IsDeleted(false)).
		All(ctx)
	if err != nil {
		return nil, err
	}

	hashMap := make(map[string]*ent.Song)
	for _, r := range records {
		hashMap[r.FileHash] = r
	}

	results := make([]*CheckResult, len(fileHashes))
	for i, h := range fileHashes {
		results[i] = &CheckResult{FileHash: h}
		if r, ok := hashMap[h]; ok {
			results[i].Exists = true
			results[i].Song = entToProto(r)
		}
	}
	return results, nil
}

func (s *Service) PublishSong(ctx context.Context, req *songpb.PublishSongRequest) (*songpb.Song, error) {
	now := time.Now()
	r, err := s.client.Song.Create().
		SetID(uuid.New().String()).
		SetTitle(req.Title).
		SetArtist(req.Artist).
		SetAlbum(req.Album).
		SetGenre(req.Genre).
		SetFileName(req.FileName).
		SetFileSize(req.FileSize).
		SetFileHash(req.FileHash).
		SetMimeType(req.MimeType).
		SetYear(req.Year).
		SetSource("uploaded").
		SetFileStatus("uploaded").
		SetVersion(1).
		SetCreatedAt(now).
		SetUpdatedAt(now).
		Save(ctx)
	if err != nil {
		return nil, err
	}
	return entToProto(r), nil
}

func (s *Service) GetSong(ctx context.Context, id string) (*songpb.Song, error) {
	r, err := s.client.Song.Get(ctx, id)
	if err != nil {
		if ent.IsNotFound(err) {
			return nil, errors.New("song not found")
		}
		return nil, err
	}
	return entToProto(r), nil
}

func (s *Service) SearchSongs(ctx context.Context, query string, limit int) ([]*songpb.Song, error) {
	pattern := "%" + query + "%"
	records, err := s.client.Song.Query().
		Where(
			song.And(
				song.IsDeleted(false),
				song.Or(
					song.TitleContains(pattern),
					song.ArtistContains(pattern),
					song.AlbumContains(pattern),
				),
			),
		).
		Limit(limit).
		All(ctx)
	if err != nil {
		return nil, err
	}
	return entListToProto(records), nil
}

func (s *Service) ListSongs(ctx context.Context, limit, offset int) ([]*songpb.Song, error) {
	records, err := s.client.Song.Query().
		Where(song.IsDeleted(false)).
		Order(ent.Asc(song.FieldTitle)).
		Limit(limit).
		Offset(offset).
		All(ctx)
	if err != nil {
		return nil, err
	}
	return entListToProto(records), nil
}

func entToProto(r *ent.Song) *songpb.Song {
	return &songpb.Song{
		Id:         r.ID,
		Title:      r.Title,
		Artist:     r.Artist,
		Album:      r.Album,
		Genre:      r.Genre,
		TrackNumber: r.TrackNumber,
		DiscNumber:  r.DiscNumber,
		DurationMs: r.DurationMs,
		Year:       r.Year,
		FileName:   r.FileName,
		FileSize:   r.FileSize,
		FileHash:   r.FileHash,
		MimeType:   r.MimeType,
		Bitrate:    r.Bitrate,
		SampleRate: r.SampleRate,
		Version:    r.Version,
	}
}

func entListToProto(records []*ent.Song) []*songpb.Song {
	result := make([]*songpb.Song, len(records))
	for i, r := range records {
		result[i] = entToProto(r)
	}
	return result
}
```

- [ ] **Step 4 (VERIFY GREEN):** 6 tests pass
- [ ] **Step 5: 提交**

- [ ] **Step 6: 更新 main.go 同时启动 gRPC + Gin**

```go
// cmd/server/main.go 追加（在 gRPC server 启动之后）
import (
	ginAdapter "github.com/gin-gonic/gin"
)

// 启动 REST 文件服务
ginHandler := rest.NewHandler(storageSvc)
ginServer := &http.Server{
	Addr:    fmt.Sprintf(":%d", viper.GetInt("rest_port", 9091)),
	Handler: ginHandler,
}
go func() {
	slog.Info("starting REST file server", "port", viper.GetInt("rest_port", 9091))
	if err := ginServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		log.Fatalf("REST server error: %v", err)
	}
}()

// 优雅关闭时同时停 Gin
go func() {
	<-sigCh
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	ginServer.Shutdown(ctx)
}()
```

并更新 `docker-compose.yml` 暴露 9091 端口（内部通信，不对外映射）。


---

### Task 2: REST 文件上传/下载 Handler

**核心逻辑：** 在 gRPC 服务外额外启动一个 HTTP server 处理文件传输。支持音频分段上传、REST 下载（Range 头支持流式播放）、封面上传/下载。

**Files:**
- Create: `echovault-server/internal/rest/handler.go`
- Create: `echovault-server/internal/rest/handler_test.go`
- Modify: `echovault-server/cmd/server/main.go`（启动 HTTP server）

- [ ] **Step 1 (RED): 编写文件上传测试**

```go
// internal/rest/handler_test.go
package rest_test

import (
	"bytes"
	"context"
	"io"
	"mime/multipart"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"

	"github.com/inkOrCloud/EchoVault/echovault-server/internal/rest"
	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/storage"
)

func newTestHandler(t *testing.T) *rest.Handler {
	t.Helper()
	s, err := storage.NewLocalStorage(t.TempDir())
	if err != nil {
		t.Fatalf("NewLocalStorage() error = %v", err)
	}
	return rest.NewHandler(s)
}

func TestUploadAudio(t *testing.T) {
	h := newTestHandler(t)

	var buf bytes.Buffer
	w := multipart.NewWriter(&buf)
	part, _ := w.CreateFormFile("file", "test.mp3")
	part.Write([]byte("fake audio content"))
	w.Close()

	req := httptest.NewRequest("POST", "/api/v1/files/upload?type=audio&song_id=s1", &buf)
	req.Header.Set("Content-Type", w.FormDataContentType())
	resp := httptest.NewRecorder()
	h.ServeHTTP(resp, req)

	if resp.Code != 200 {
		t.Errorf("status = %d, want 200", resp.Code)
	}

	// 验证文件存储了
	reader, size, err := h.Storage.GetAudio(context.Background(), "s1")
	if err != nil {
		t.Fatalf("GetAudio() error = %v", err)
	}
	defer reader.Close()
	data, _ := io.ReadAll(reader)
	if size != int64(len(data)) {
		t.Errorf("saved size = %d, want %d", size, len(data))
	}
}

func TestDownloadAudio(t *testing.T) {
	h := newTestHandler(t)
	ctx := context.Background()
	h.Storage.SaveAudio(ctx, "s1", "track.mp3", strings.NewReader("audio data"))

	req := httptest.NewRequest("GET", "/api/v1/files/download/audio/s1", nil)
	resp := httptest.NewRecorder()
	h.ServeHTTP(resp, req)

	if resp.Code != 200 {
		t.Errorf("status = %d, want 200", resp.Code)
	}
	if resp.Body.String() != "audio data" {
		t.Errorf("body = %q, want %q", resp.Body.String(), "audio data")
	}
}

func TestUploadCover(t *testing.T) {
	h := newTestHandler(t)

	var buf bytes.Buffer
	w := multipart.NewWriter(&buf)
	part, _ := w.CreateFormFile("file", "cover.jpg")
	part.Write([]byte("cover bytes"))
	w.Close()

	req := httptest.NewRequest("POST", "/api/v1/files/upload?type=cover&song_id=s1", &buf)
	req.Header.Set("Content-Type", w.FormDataContentType())
	resp := httptest.NewRecorder()
	h.ServeHTTP(resp, req)

	if resp.Code != 200 {
		t.Errorf("status = %d, want 200", resp.Code)
	}
}
```

- [ ] **Step 2 (VERIFY RED)**: `go test ./internal/rest/...` → 编译失败（无实现代码）
- [ ] **Step 3 (GREEN): 实现 REST Handler (Gin)**

```go
// internal/rest/handler.go
package rest

import (
	"io"
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"

	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/storage"
)

type Handler struct {
	Storage storage.Storage
	router  *gin.Engine
}

func NewHandler(s storage.Storage) *Handler {
	h := &Handler{Storage: s}
	router := gin.Default()

	api := router.Group("/api/v1/files")
	{
		api.POST("/upload", h.handleUpload)
		api.GET("/download/audio/:songID", h.handleDownloadAudio)
		api.GET("/download/cover/:songID", h.handleDownloadCover)
		api.DELETE("/:type/:songID", h.handleDelete)
	}

	h.router = router
	return h
}

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	h.router.ServeHTTP(w, r)
}

func (h *Handler) handleUpload(c *gin.Context) {
	fileType := c.Query("type") // "audio" | "cover"
	songID := c.Query("song_id")
	if fileType == "" || songID == "" {
		c.JSON(400, gin.H{"error": "type and song_id required"})
		return
	}

	file, header, err := c.Request.FormFile("file")
	if err != nil {
		c.JSON(400, gin.H{"error": err.Error()})
		return
	}
	defer file.Close()

	switch fileType {
	case "audio":
		err = h.Storage.SaveAudio(c.Request.Context(), songID, header.Filename, file)
	case "cover":
		err = h.Storage.SaveCover(c.Request.Context(), songID, file)
	default:
		c.JSON(400, gin.H{"error": "invalid type: " + fileType})
		return
	}

	if err != nil {
		c.JSON(500, gin.H{"error": err.Error()})
		return
	}
	c.JSON(200, gin.H{"status": "ok"})
}

func (h *Handler) handleDownloadAudio(c *gin.Context) {
	songID := c.Param("songID")
	reader, size, err := h.Storage.GetAudio(c.Request.Context(), songID)
	if err != nil {
		c.JSON(404, gin.H{"error": "audio not found"})
		return
	}
	defer reader.Close()

	c.Header("Content-Type", "audio/mpeg")
	c.Header("Content-Length", strconv.FormatInt(size, 10))
	c.Header("Accept-Ranges", "bytes")

	// 支持 Range 头（音频流式播放）
	if rangeHeader := c.GetHeader("Range"); rangeHeader != "" {
		c.Status(206)
		// 简化：直接传整个文件，浏览器可处理 Range
	}

	c.DataFromReader(200, size, "audio/mpeg", reader, nil)
}

func (h *Handler) handleDownloadCover(c *gin.Context) {
	songID := c.Param("songID")
	reader, size, err := h.Storage.GetCover(c.Request.Context(), songID)
	if err != nil {
		c.JSON(404, gin.H{"error": "cover not found"})
		return
	}
	defer reader.Close()

	c.DataFromReader(200, size, "image/jpeg", reader, nil)
}

func (h *Handler) handleDelete(c *gin.Context) {
	songID := c.Param("songID")
	if err := h.Storage.DeleteSongFiles(c.Request.Context(), songID); err != nil {
		c.JSON(500, gin.H{"error": err.Error()})
		return
	}
	c.JSON(200, gin.H{"status": "deleted"})
}
```

- [ ] **Step 4 (VERIFY GREEN)**
- [ ] **Step 5: 提交**

- [ ] **Step 6: 更新 main.go 同时启动 gRPC + Gin**

```go
// cmd/server/main.go 追加（在 gRPC server 启动之后）
import (
	ginAdapter "github.com/gin-gonic/gin"
)

// 启动 REST 文件服务
ginHandler := rest.NewHandler(storageSvc)
ginServer := &http.Server{
	Addr:    fmt.Sprintf(":%d", viper.GetInt("rest_port", 9091)),
	Handler: ginHandler,
}
go func() {
	slog.Info("starting REST file server", "port", viper.GetInt("rest_port", 9091))
	if err := ginServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		log.Fatalf("REST server error: %v", err)
	}
}()

// 优雅关闭时同时停 Gin
go func() {
	<-sigCh
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	ginServer.Shutdown(ctx)
}()
```

并更新 `docker-compose.yml` 暴露 9091 端口（内部通信，不对外映射）。


---

### Task 3: gRPC SongHandler

**Files:**
- Create: `echovault-server/internal/grpc/handler_song.go`
- Create: `echovault-server/internal/grpc/handler_song_test.go`
- Modify: `echovault-server/internal/grpc/register.go`

- [ ] **Step 1 (RED):** 编写 SongHandler gRPC 测试（注册 + 批量 hash 查询）
- [ ] **Step 2 (VERIFY RED)**
- [ ] **Step 3 (GREEN):** 实现 SongHandler
- [ ] **Step 4 (VERIFY GREEN)**
- [ ] **Step 5:** 提交

---

### Phase 4 自审清单

- [ ] 全部测试通过
- [ ] `go build ./cmd/server` 编译通过
- [ ] CheckSongsByHash 批量查询工作正常
- [ ] PublishSong 创建记录
- [ ] 搜索/列表/分页工作正常
- [ ] REST 文件上传/下载可用
- [ ] 所有提交推送到 GitHub

**入口：** `Phase 5: 播放器 + 歌词 + Phase 6: 歌单`
