# Phase 5b: 音频元数据解析 + 上传时自动填充 + 封面提取 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现 `pkg/metadata` 音频元数据解析库（含封面图片提取），集成到 REST 上传流程中。用户通过 Flutter → gRPC `PublishSong` 手动提交字段后，再 REST 上传文件，服务端从 tag 中**补充用户未填的字段**并自动保存嵌入封面。

**Architecture:** 分两步——(1) Flutter 调用 `PublishSong` gRPC，写入用户手动输入的字段；(2) Flutter 上传音频文件到 REST `/api/v1/files/upload?type=audio&song_id=xxx`，服务端解析 tag 元数据，仅**补充** Song 记录中为空的字段（title/artist/album 等），并提取嵌入封面保存到 Storage。用户手动输入 > tag 自动提取。

**Tech Stack:** Go 1.26+, `github.com/dhowden/tag` (MP3/FLAC/OGG/MP4 标签 + 封面图片), Gin REST, Ent

---

### Task 1: pkg/metadata — 核心元数据解析器（含封面提取）

**Files:**
- Create: `echovault-server/pkg/metadata/metadata.go` — 公开类型 + `ParseFile` / `ParseReader`
- Create: `echovault-server/pkg/metadata/metadata_test.go` — 测试

**核心逻辑：** 用 `github.com/dhowden/tag` 读取音频文件的 ID3/FLAC/Vorbis 标签和嵌入封面图片，补充 SHA256 哈希和 MIME 类型推导。

- [ ] **Step 1: 安装依赖**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
go get github.com/dhowden/tag@latest
go mod tidy
```

- [ ] **Step 2 (RED): 编写测试 — ParseFile 解析 MP3**

```go
// pkg/metadata/metadata_test.go
package metadata_test

import (
	"os"
	"path/filepath"
	"testing"

	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/metadata"
)

// buildMinimalID3v2 构造一个可被 dhowden/tag 解析的最小 ID3v2.4 标签（含 APIC 封面）。
func buildMinimalID3v2(t *testing.T, title, artist, album string, coverData []byte) []byte {
	t.Helper()
	id3 := []byte("ID3\x04\x00\x00")
	frames := []byte{}

	// TIT2 — Title (encoding byte 0x03 = UTF-8)
	titleBytes := []byte(title)
	frames = append(frames, "TIT2"...)
	totalSize := 1 + len(titleBytes) // encoding byte + string
	frames = append(frames,
		byte(totalSize>>24), byte(totalSize>>16),
		byte(totalSize>>8), byte(totalSize))
	frames = append(frames, 0, 0)       // flags
	frames = append(frames, 0x03)       // encoding
	frames = append(frames, titleBytes...)

	// TPE1 — Artist
	artistBytes := []byte(artist)
	frames = append(frames, "TPE1"...)
	totalSize = 1 + len(artistBytes)
	frames = append(frames,
		byte(totalSize>>24), byte(totalSize>>16),
		byte(totalSize>>8), byte(totalSize))
	frames = append(frames, 0, 0)
	frames = append(frames, 0x03)
	frames = append(frames, artistBytes...)

	// TALB — Album (if provided)
	if album != "" {
		albumBytes := []byte(album)
		frames = append(frames, "TALB"...)
		totalSize = 1 + len(albumBytes)
		frames = append(frames,
			byte(totalSize>>24), byte(totalSize>>16),
			byte(totalSize>>8), byte(totalSize))
		frames = append(frames, 0, 0)
		frames = append(frames, 0x03)
		frames = append(frames, albumBytes...)
	}

	// APIC — 封面图片 (if provided)
	if coverData != nil {
		frames = append(frames, "APIC"...)
		mimeType := "image/jpeg"
		apicBody := []byte{0x03}                                         // encoding
		apicBody = append(apicBody, []byte(mimeType)...)                 // MIME type
		apicBody = append(apicBody, 0)                                   // null terminator
		apicBody = append(apicBody, 0x03)                                // picture type: 3=cover front
		apicBody = append(apicBody, []byte{0x00}...)                     // description (empty)
		apicBody = append(apicBody, coverData...)                        // image data
		totalSize = len(apicBody)
		frames = append(frames,
			byte(totalSize>>24), byte(totalSize>>16),
			byte(totalSize>>8), byte(totalSize))
		frames = append(frames, 0, 0) // flags
		frames = append(frames, apicBody...)
	}

	// ID3 tag size (syncsafe integer)
	tagSize := len(frames)
	id3 = append(id3,
		byte((tagSize>>21)&0x7F),
		byte((tagSize>>14)&0x7F),
		byte((tagSize>>7)&0x7F),
		byte(tagSize&0x7F))
	id3 = append(id3, frames...)

	return id3
}

func TestParseFile_MP3_BasicTags(t *testing.T) {
	dir := t.TempDir()
	path := filepath.Join(dir, "test.mp3")
	data := buildMinimalID3v2(t, "Test Title", "Test Artist", "Test Album", nil)
	if err := os.WriteFile(path, data, 0644); err != nil {
		t.Fatalf("write test mp3: %v", err)
	}

	meta, err := metadata.ParseFile(path)
	if err != nil {
		t.Fatalf("ParseFile() error = %v", err)
	}
	if meta.Title != "Test Title" {
		t.Errorf("Title = %q, want %q", meta.Title, "Test Title")
	}
	if meta.Artist != "Test Artist" {
		t.Errorf("Artist = %q, want %q", meta.Artist, "Test Artist")
	}
	if meta.Album != "Test Album" {
		t.Errorf("Album = %q, want %q", meta.Album, "Test Album")
	}
	if meta.MIMEType != "audio/mpeg" {
		t.Errorf("MIMEType = %q, want %q", meta.MIMEType, "audio/mpeg")
	}
	if meta.FileHash == "" {
		t.Error("FileHash should not be empty")
	}
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd /home/ink/Desktop/develop/EchoVault && go test ./echovault-server/pkg/metadata/ -v -count=1 -run TestParseFile_MP3_BasicTags`
Expected: compile error — package not found

- [ ] **Step 4 (GREEN): 实现 ParseFile + ParseReader（含封面提取）**

```go
// pkg/metadata/metadata.go
package metadata

import (
	"crypto/sha256"
	"fmt"
	"io"
	"os"
	"path/filepath"
	"strings"

	"github.com/dhowden/tag"
)

// Picture 是嵌入音频文件中的封面图片。
type Picture struct {
	Data     []byte // 原始图片二进制数据
	MIMEType string // 例如 "image/jpeg", "image/png"
	Ext      string // 文件扩展名，例如 "jpg", "png"
}

// AudioMetadata 是音频文件解析后的结构化元数据。
type AudioMetadata struct {
	Title       string   `json:"title"`
	Artist      string   `json:"artist"`
	Album       string   `json:"album"`
	Genre       string   `json:"genre"`
	TrackNumber int      `json:"track_number"`
	DiscNumber  int      `json:"disc_number"`
	Year        int      `json:"year"`
	FileName    string   `json:"file_name"`
	FileHash    string   `json:"file_hash"` // SHA256 hex
	MIMEType    string   `json:"mime_type"`
	Picture     *Picture `json:"picture,omitempty"` // 嵌入封面，可能为 nil
}

// ParseFile 打开文件并解析音频元数据。
func ParseFile(path string) (*AudioMetadata, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, fmt.Errorf("open file: %w", err)
	}
	defer f.Close()

	return ParseReader(f, path)
}

// ParseReader 从 io.ReadSeeker 解析音频元数据。
// filePath 用于推导文件名和 MIME 类型。
func ParseReader(r io.ReadSeeker, filePath string) (*AudioMetadata, error) {
	// SHA256 哈希
	hash := sha256.New()
	if _, err := io.Copy(hash, r); err != nil {
		return nil, fmt.Errorf("hash file: %w", err)
	}
	fileHash := fmt.Sprintf("%x", hash.Sum(nil))

	// 回到文件头供 dhowden/tag 读取
	if _, err := r.Seek(0, io.SeekStart); err != nil {
		return nil, fmt.Errorf("seek to start: %w", err)
	}

	m, err := tag.ReadFrom(r)
	if err != nil {
		// 标签解析失败时，仍然返回基础信息（文件名、哈希、MIME）
		return &AudioMetadata{
			FileName: filepath.Base(filePath),
			FileHash: fileHash,
			MIMEType: mimeTypeByExt(filePath),
		}, nil
	}

	meta := &AudioMetadata{
		Title:    m.Title(),
		Artist:   m.Artist(),
		Album:    m.Album(),
		Genre:    m.Genre(),
		FileName: filepath.Base(filePath),
		FileHash: fileHash,
		MIMEType: mimeTypeByExt(filePath),
	}

	if track, _ := m.Track(); track > 0 {
		meta.TrackNumber = track
	}
	if disc, _ := m.Disc(); disc > 0 {
		meta.DiscNumber = disc
	}
	if year := m.Year(); year > 0 {
		meta.Year = year
	}

	// 提取嵌入封面
	if pic := m.Picture(); pic != nil {
		meta.Picture = &Picture{
			Data:     pic.Data,
			MIMEType: pic.MIMEType,
			Ext:      pic.Ext,
		}
	}

	return meta, nil
}

// mimeTypeByExt 根据文件扩展名推测 MIME 类型。
func mimeTypeByExt(path string) string {
	ext := strings.ToLower(filepath.Ext(path))
	switch ext {
	case ".mp3":
		return "audio/mpeg"
	case ".flac":
		return "audio/flac"
	case ".ogg":
		return "audio/ogg"
	case ".m4a", ".mp4":
		return "audio/mp4"
	case ".wav":
		return "audio/wav"
	case ".aac":
		return "audio/aac"
	case ".wma":
		return "audio/x-ms-wma"
	case ".opus":
		return "audio/opus"
	default:
		return "application/octet-stream"
	}
}

// MIMETypeByExt 根据文件扩展名返回 MIME 类型（导出供外部使用）。
func MIMETypeByExt(path string) string {
	return mimeTypeByExt(path)
}

// AudioFileExts 返回支持的音频文件扩展名集合。
func AudioFileExts() map[string]bool {
	return map[string]bool{
		".mp3": true, ".flac": true, ".ogg": true,
		".m4a": true, ".mp4": true, ".wav": true,
		".aac": true, ".wma": true, ".opus": true,
	}
}

// IsAudioFile 检查文件扩展名是否为支持的音频格式。
func IsAudioFile(path string) bool {
	ext := strings.ToLower(filepath.Ext(path))
	return AudioFileExts()[ext]
}
```

- [ ] **Step 5: Run test**

Run: `cd /home/ink/Desktop/develop/EchoVault && go test ./echovault-server/pkg/metadata/ -v -count=1 -run TestParseFile_MP3_BasicTags`
Expected: PASS

- [ ] **Step 6 (RED): 编写测试 — FLAC + 封面提取 + 非音频回退**

```go
// pkg/metadata/metadata_test.go — 追加

func writeTestFLAC(t *testing.T, dir, title, artist string) string {
	t.Helper()
	path := filepath.Join(dir, "test.flac")
	data := []byte("fLaC")
	// STREAMINFO: type=0(block), last=false(0), size=34
	data = append(data, 0x00, 0x00, 0x00, 0x22)
	data = append(data, make([]byte, 34)...)

	// VORBIS_COMMENT block: type=4, last=true(0x80)
	commentData := fmt.Sprintf("TITLE=%s\x00ARTIST=%s", title, artist)
	commentLen := len(commentData)
	vorbis := make([]byte, 0)
	vorbis = append(vorbis, 0x84)
	vorbis = append(vorbis,
		byte(commentLen>>16),
		byte((commentLen>>8)&0xFF),
		byte(commentLen&0xFF))
	vendor := "libFLAC 1.4.3"
	vLen := len(vendor)
	vorbis = append(vorbis, byte(vLen), byte(vLen>>8), byte(vLen>>16), byte(vLen>>24))
	vorbis = append(vorbis, []byte(vendor)...)
	vorbis = append(vorbis, 2, 0, 0, 0)
	vorbis = append(vorbis, byte(len([]byte(title))))
	vorbis = append(vorbis, []byte(title)...)
	vorbis = append(vorbis, byte(len([]byte(artist))))
	vorbis = append(vorbis, []byte(artist)...)

	data = append(data, vorbis...)
	if err := os.WriteFile(path, data, 0644); err != nil {
		t.Fatalf("write flac: %v", err)
	}
	return path
}

func TestParseFile_FLAC(t *testing.T) {
	dir := t.TempDir()
	path := writeTestFLAC(t, dir, "FLAC Title", "FLAC Artist")

	meta, err := metadata.ParseFile(path)
	if err != nil {
		t.Fatalf("ParseFile() error = %v", err)
	}
	if meta.Title != "FLAC Title" {
		t.Errorf("Title = %q, want %q", meta.Title, "FLAC Title")
	}
	if meta.MIMEType != "audio/flac" {
		t.Errorf("MIMEType = %q, want %q", meta.MIMEType, "audio/flac")
	}
}

func TestParseFile_WithCoverArt(t *testing.T) {
	dir := t.TempDir()
	coverData := []byte("fake jpeg data here") // 模拟封面
	path := filepath.Join(dir, "cover.mp3")
	data := buildMinimalID3v2(t, "Cover Song", "Cover Artist", "Cover Album", coverData)
	if err := os.WriteFile(path, data, 0644); err != nil {
		t.Fatalf("write mp3 with cover: %v", err)
	}

	meta, err := metadata.ParseFile(path)
	if err != nil {
		t.Fatalf("ParseFile() error = %v", err)
	}
	if meta.Picture == nil {
		t.Fatal("Picture should not be nil for file with APIC frame")
	}
	if len(meta.Picture.Data) != len(coverData) {
		t.Errorf("Picture data length = %d, want %d", len(meta.Picture.Data), len(coverData))
	}
}

func TestParseFile_UnknownFormat_ReturnsBasicInfo(t *testing.T) {
	dir := t.TempDir()
	path := filepath.Join(dir, "readme.txt")
	if err := os.WriteFile(path, []byte("hello world"), 0644); err != nil {
		t.Fatalf("write txt: %v", err)
	}

	meta, err := metadata.ParseFile(path)
	if err != nil {
		t.Fatalf("ParseFile() error = %v", err)
	}
	if meta.FileName != "readme.txt" {
		t.Errorf("FileName = %q, want %q", meta.FileName, "readme.txt")
	}
	if meta.FileHash == "" {
		t.Error("FileHash should not be empty")
	}
	if meta.Picture != nil {
		t.Error("Picture should be nil for non-audio file")
	}
}

func TestParseFile_NotFound(t *testing.T) {
	_, err := metadata.ParseFile("/nonexistent/file.mp3")
	if err == nil {
		t.Fatal("expected error for nonexistent file")
	}
}

func TestMIMETypeByExt_AllFormats(t *testing.T) {
	tests := []struct {
		path     string
		expected string
	}{
		{"song.mp3", "audio/mpeg"},
		{"song.flac", "audio/flac"},
		{"song.ogg", "audio/ogg"},
		{"song.m4a", "audio/mp4"},
		{"song.mp4", "audio/mp4"},
		{"song.wav", "audio/wav"},
		{"song.aac", "audio/aac"},
		{"song.wma", "audio/x-ms-wma"},
		{"song.opus", "audio/opus"},
		{"song.unknown", "application/octet-stream"},
	}
	for _, tt := range tests {
		t.Run(tt.path, func(t *testing.T) {
			got := metadata.MIMETypeByExt(tt.path)
			if got != tt.expected {
				t.Errorf("MIMETypeByExt(%q) = %q, want %q", tt.path, got, tt.expected)
			}
		})
	}
}

func TestIsAudioFile(t *testing.T) {
	if !metadata.IsAudioFile("song.mp3") {
		t.Error("IsAudioFile('song.mp3') = false, want true")
	}
	if !metadata.IsAudioFile("song.flac") {
		t.Error("IsAudioFile('song.flac') = false, want true")
	}
	if metadata.IsAudioFile("readme.txt") {
		t.Error("IsAudioFile('readme.txt') = true, want false")
	}
}
```

- [ ] **Step 7: Run all metadata tests**

Run: `cd /home/ink/Desktop/develop/EchoVault && go test ./echovault-server/pkg/metadata/ -v -count=1`
Expected: PASS (7 tests)

- [ ] **Step 8: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echovault-server/pkg/metadata/ echovault-server/go.mod echovault-server/go.sum
git commit -m "feat: add pkg/metadata — audio metadata parser with cover art extraction (MP3/FLAC)"
```

---

### Task 2: REST 上传 — 补充空字段 + 自动保存封面

**Files:**
- Create: `echovault-server/internal/rest/publisher.go` — `SongUpdater` 接口定义
- Modify: `echovault-server/internal/rest/handler.go` — 上传时补充元数据 + 保存封面
- Modify: `echovault-server/internal/rest/handler_test.go` — 测试
- Modify: `echovault-server/cmd/server/main.go` — 注入 SongUpdater adapter

**核心逻辑：** REST 上传处理流程：
1. 保存上传文件到临时文件
2. 调用 `metadata.ParseFile` 提取标签 + 封面
3. 如果有封面 → `Storage.SaveCover`
4. 调用 `SongUpdater.UpdateFromScan` → 后台查 Song 记录，**只补充用户未填的字段**

- [ ] **Step 1: 创建 SongUpdater 接口**

```go
// internal/rest/publisher.go
package rest

// SongUpdater 定义上传音频后补充元数据的接口。
// 由 adapter 在 main.go 中对接 SongService。
// 实现应查询 Song 记录，仅当字段值为空时才用 metadata 填充。
type SongUpdater interface {
	// UpdateFromScan 用音频文件提取的元数据补充 Song 记录中用户未填的字段。
	// 包括文本标签和嵌入封面。
	// - songID: 上传时指定的歌曲 ID
	// - fileHash: 音频文件 SHA256
	// - fileSize: 文件字节数
	UpdateFromScan(songID, fileHash string, fileSize int64) error
}
```

- [ ] **Step 2: 修改 REST Handler — upload 处理流程**

```go
// internal/rest/handler.go — 完整版
package rest

import (
	"io"
	"net/http"
	"os"
	"path/filepath"
	"strconv"

	"github.com/gin-gonic/gin"

	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/metadata"
	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/storage"
)

// Handler 是 REST 文件服务处理器。
type Handler struct {
	Storage       storage.Storage
	SongUpdater   SongUpdater
	router        *gin.Engine
}

// NewHandler 创建 REST Handler。
// songUpdater 可为 nil（此时不更新 Song 记录）。
func NewHandler(s storage.Storage, songUpdater SongUpdater) *Handler {
	gin.SetMode(gin.ReleaseMode)
	h := &Handler{Storage: s, SongUpdater: songUpdater}
	router := gin.New()
	router.Use(gin.Recovery())

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
	fileType := c.Query("type")
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
		// 保存到临时文件以便解析元数据
		tmpFile, err := os.CreateTemp("", "echovault-upload-*"+filepath.Ext(header.Filename))
		if err != nil {
			c.JSON(500, gin.H{"error": "create temp file: " + err.Error()})
			return
		}
		tmpPath := tmpFile.Name()
		defer os.Remove(tmpPath)

		if _, err := io.Copy(tmpFile, file); err != nil {
			c.JSON(500, gin.H{"error": "save temp file: " + err.Error()})
			return
		}
		tmpFile.Close()

		// 解析元数据
		meta, _ := metadata.ParseFile(tmpPath)

		// 保存音频文件到持久存储
		tmpForRead, _ := os.Open(tmpPath)
		defer tmpForRead.Close()
		if err := h.Storage.SaveAudio(c.Request.Context(), songID, header.Filename, tmpForRead); err != nil {
			c.JSON(500, gin.H{"error": "save audio: " + err.Error()})
			return
		}

		// 如果有嵌入封面，保存到 Storage
		if meta != nil && meta.Picture != nil {
			// 将封面数据包装为 io.Reader
			coverReader := &pictureReader{data: meta.Picture.Data}
			if err := h.Storage.SaveCover(c.Request.Context(), songID, coverReader); err != nil {
				// 封面保存失败不阻止主流程
				// 只记录日志（生产环境用 slog）
			}
		}

		// 补充 Song 记录中用户未填的字段
		if h.SongUpdater != nil && meta != nil {
			if err := h.SongUpdater.UpdateFromScan(songID, meta.FileHash, header.Size); err != nil {
				// 元数据补充失败不阻止上传成功
			}
		}

	case "cover":
		if err := h.Storage.SaveCover(c.Request.Context(), songID, file); err != nil {
			c.JSON(500, gin.H{"error": "save cover: " + err.Error()})
			return
		}
	default:
		c.JSON(400, gin.H{"error": "invalid type: " + fileType})
		return
	}

	c.JSON(200, gin.H{"status": "ok"})
}

// pictureReader 将 []byte 包装为 io.Reader，用于 Storage.SaveCover。
type pictureReader struct {
	data   []byte
	offset int
}

func (r *pictureReader) Read(p []byte) (int, error) {
	if r.offset >= len(r.data) {
		return 0, io.EOF
	}
	n := copy(p, r.data[r.offset:])
	r.offset += n
	return n, nil
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

- [ ] **Step 3 (RED): 编写测试 — 上传 + 元数据补充 + 封面提取**

```go
// internal/rest/handler_test.go
package rest_test

import (
	"bytes"
	"io"
	"mime/multipart"
	"net/http"
	"net/http/httptest"
	"os"
	"path/filepath"
	"testing"

	"github.com/inkOrCloud/EchoVault/echovault-server/internal/rest"
	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/storage"
)

// mockSongUpdater 记录 UpdateFromScan 调用。
type mockSongUpdater struct {
	LastSongID string
	LastHash   string
	LastSize   int64
	CallCount  int
}

func (m *mockSongUpdater) UpdateFromScan(songID, fileHash string, fileSize int64) error {
	m.CallCount++
	m.LastSongID = songID
	m.LastHash = fileHash
	m.LastSize = fileSize
	return nil
}

// buildTestMP3WithCover 构造带 ID3v2 标签 + APIC 封面的 MP3 文件。
func buildTestMP3WithCover(t *testing.T, dir, title, artist string, coverData []byte) string {
	t.Helper()
	id3 := []byte("ID3\x04\x00\x00")
	frames := []byte{}

	// TIT2
	tBytes := append([]byte{0x03}, []byte(title)...)
	frames = append(frames, "TIT2"...)
	frames = append(frames,
		byte(len(tBytes)>>24), byte(len(tBytes)>>16),
		byte(len(tBytes)>>8), byte(len(tBytes)))
	frames = append(frames, 0, 0)
	frames = append(frames, tBytes...)

	// TPE1
	aBytes := append([]byte{0x03}, []byte(artist)...)
	frames = append(frames, "TPE1"...)
	frames = append(frames,
		byte(len(aBytes)>>24), byte(len(aBytes)>>16),
		byte(len(aBytes)>>8), byte(len(aBytes)))
	frames = append(frames, 0, 0)
	frames = append(frames, aBytes...)

	// APIC 封面
	if coverData != nil {
		apicBody := []byte{0x03}
		apicBody = append(apicBody, "image/jpeg"...)
		apicBody = append(apicBody, 0)
		apicBody = append(apicBody, 0x03) // cover front
		apicBody = append(apicBody, 0)
		apicBody = append(apicBody, coverData...)
		frames = append(frames, "APIC"...)
		frames = append(frames,
			byte(len(apicBody)>>24), byte(len(apicBody)>>16),
			byte(len(apicBody)>>8), byte(len(apicBody)))
		frames = append(frames, 0, 0)
		frames = append(frames, apicBody...)
	}

	tagSize := len(frames)
	id3 = append(id3,
		byte((tagSize>>21)&0x7F),
		byte((tagSize>>14)&0x7F),
		byte((tagSize>>7)&0x7F),
		byte(tagSize&0x7F))
	id3 = append(id3, frames...)

	path := filepath.Join(dir, "test.mp3")
	if err := os.WriteFile(path, id3, 0644); err != nil {
		t.Fatalf("write mp3: %v", err)
	}
	return path
}

func TestUploadAudio_CallsUpdater(t *testing.T) {
	tmpDir := t.TempDir()
	s, err := storage.NewLocalStorage(filepath.Join(tmpDir, "files"))
	if err != nil {
		t.Fatalf("NewLocalStorage() error = %v", err)
	}

	updater := &mockSongUpdater{}
	h := rest.NewHandler(s, updater)

	mp3Dir := t.TempDir()
	mp3Path := buildTestMP3WithCover(t, mp3Dir, "Auto Title", "Auto Artist", nil)

	body := &bytes.Buffer{}
	writer := multipart.NewWriter(body)
	fw, _ := writer.CreateFormFile("file", "song.mp3")
	mp3Data, _ := os.ReadFile(mp3Path)
	fw.Write(mp3Data)
	writer.Close()

	req := httptest.NewRequest("POST", "/api/v1/files/upload?type=audio&song_id=test-song-001", body)
	req.Header.Set("Content-Type", writer.FormDataContentType())
	w := httptest.NewRecorder()
	h.ServeHTTP(w, req)

	if w.Code != 200 {
		t.Fatalf("status = %d, want 200; body=%s", w.Code, w.Body.String())
	}
	if updater.CallCount != 1 {
		t.Errorf("CallCount = %d, want 1", updater.CallCount)
	}
	if updater.LastSongID != "test-song-001" {
		t.Errorf("SongID = %q, want %q", updater.LastSongID, "test-song-001")
	}
	if updater.LastHash == "" {
		t.Error("FileHash should not be empty")
	}
}

func TestUploadAudio_WithCover_SavesCover(t *testing.T) {
	tmpDir := t.TempDir()
	storageDir := filepath.Join(tmpDir, "files")
	s, err := storage.NewLocalStorage(storageDir)
	if err != nil {
		t.Fatalf("NewLocalStorage() error = %v", err)
	}

	h := rest.NewHandler(s, nil) // 无 updater，只测试封面保存

	coverData := []byte("fake-cover-image-bytes")
	mp3Dir := t.TempDir()
	mp3Path := buildTestMP3WithCover(t, mp3Dir, "Cover Song", "Cover Artist", coverData)

	body := &bytes.Buffer{}
	writer := multipart.NewWriter(body)
	fw, _ := writer.CreateFormFile("file", "cover-song.mp3")
	mp3Data, _ := os.ReadFile(mp3Path)
	fw.Write(mp3Data)
	writer.Close()

	req := httptest.NewRequest("POST", "/api/v1/files/upload?type=audio&song_id=test-cover-song", body)
	req.Header.Set("Content-Type", writer.FormDataContentType())
	w := httptest.NewRecorder()
	h.ServeHTTP(w, req)

	if w.Code != 200 {
		t.Fatalf("status = %d, want 200; body=%s", w.Code, w.Body.String())
	}

	// 验证封面已保存
	coverReader, coverSize, err := s.GetCover(req.Context(), "test-cover-song")
	if err != nil {
		t.Fatalf("GetCover() error = %v (cover was not saved)", err)
	}
	defer coverReader.Close()

	if coverSize != int64(len(coverData)) {
		t.Errorf("cover size = %d, want %d", coverSize, len(coverData))
	}
	readData, _ := io.ReadAll(coverReader)
	if string(readData) != string(coverData) {
		t.Errorf("cover data mismatch")
	}
}

func TestUploadAudio_NoMetadata_StillSucceeds(t *testing.T) {
	tmpDir := t.TempDir()
	s, err := storage.NewLocalStorage(filepath.Join(tmpDir, "files"))
	if err != nil {
		t.Fatalf("NewLocalStorage() error = %v", err)
	}

	h := rest.NewHandler(s, nil)

	// 上传一个没有标签的二进制文件
	body := &bytes.Buffer{}
	writer := multipart.NewWriter(body)
	fw, _ := writer.CreateFormFile("file", "noise.bin")
	fw.Write([]byte("not an audio file at all but that's ok"))
	writer.Close()

	req := httptest.NewRequest("POST", "/api/v1/files/upload?type=audio&song_id=test-no-meta", body)
	req.Header.Set("Content-Type", writer.FormDataContentType())
	w := httptest.NewRecorder()
	h.ServeHTTP(w, req)

	if w.Code != 200 {
		t.Fatalf("status = %d, want 200; body=%s", w.Code, w.Body.String())
	}
}

func TestUploadCover_DoesNotCallUpdater(t *testing.T) {
	tmpDir := t.TempDir()
	s, err := storage.NewLocalStorage(filepath.Join(tmpDir, "files"))
	if err != nil {
		t.Fatalf("NewLocalStorage() error = %v", err)
	}

	updater := &mockSongUpdater{}
	h := rest.NewHandler(s, updater)

	body := &bytes.Buffer{}
	writer := multipart.NewWriter(body)
	fw, _ := writer.CreateFormFile("file", "cover.jpg")
	fw.Write([]byte("fake jpeg bytes"))
	writer.Close()

	req := httptest.NewRequest("POST", "/api/v1/files/upload?type=cover&song_id=test-cover", body)
	req.Header.Set("Content-Type", writer.FormDataContentType())
	w := httptest.NewRecorder()
	h.ServeHTTP(w, req)

	if w.Code != 200 {
		t.Fatalf("status = %d, want 200; body=%s", w.Code, w.Body.String())
	}
	if updater.CallCount != 0 {
		t.Errorf("CallCount = %d, want 0 for cover upload", updater.CallCount)
	}
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `cd /home/ink/Desktop/develop/EchoVault && go test ./echovault-server/internal/rest/ -v -count=1`
Expected: compile error — `NewHandler` 签名已改变

- [ ] **Step 5: 创建 SongUpdater adapter — main.go 中对接 SongService**

需要先在 SongService 中增加一个 `UpdateFromScan` 方法：

```go
// internal/service/song/service.go — 追加

// UpdateFromScan 用扫描提取的元数据补充 Song 记录中用户未填的字段。
// 只补充空字段（title/artist/album/genre/track_number/disc_number/year），
// 并更新 file_hash/file_size/mime_type。
func (s *Service) UpdateFromScan(ctx context.Context, songID, title, artist, album, genre string,
	trackNumber, discNumber, year int32,
	fileHash, fileName, mimeType string, fileSize int64) error {

	r, err := s.client.Song.Get(ctx, songID)
	if err != nil {
		return err
	}

	updater := s.client.Song.UpdateOneID(songID)

	// 仅当用户未填时用 tag 值补充
	if r.Title == "" && title != "" {
		updater.SetTitle(title)
	}
	if r.Artist == "" && artist != "" {
		updater.SetArtist(artist)
	}
	if r.Album == "" && album != "" {
		updater.SetAlbum(album)
	}
	if r.Genre == "" && genre != "" {
		updater.SetGenre(genre)
	}
	if r.TrackNumber == 0 && trackNumber > 0 {
		updater.SetTrackNumber(trackNumber)
	}
	if r.DiscNumber == 0 && discNumber > 0 {
		updater.SetDiscNumber(discNumber)
	}
	if r.Year == 0 && year > 0 {
		updater.SetYear(year)
	}

	// 文件信息总是更新（上传时才有确切值）
	updater.SetFileHash(fileHash)
	updater.SetFileSize(fileSize)
	updater.SetMimeType(mimeType)
	updater.SetFileName(fileName)
	updater.SetFileStatus("uploaded")
	updater.SetSource("uploaded")

	return updater.Exec(ctx)
}
```

- [ ] **Step 5b (RED): 编写 SongService.UpdateFromScan 测试**

```go
// internal/service/song/service_test.go — 追加

func TestUpdateFromScan_FillsEmptyFields(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := song.NewService(client)
	ctx := context.Background()

	// 用户手动发布，Title 已填，Artist 留空
	pubResp, _ := svc.PublishSong(ctx, &songpb.PublishSongRequest{
		Title: "User Title", Artist: "",
		FileHash: "old-hash", FileName: "user.mp3",
	})

	// 上传后从 tag 提取的元数据
	err := svc.UpdateFromScan(ctx, pubResp.Id,
		"Tag Title",    // title (不应覆盖用户已填的)
		"Tag Artist",   // artist (应填入，因为用户留空)
		"Tag Album",
		"Tag Genre",
		3, 1, 2024,
		"new-hash-abc", "song.mp3", "audio/mpeg", 12345)
	if err != nil {
		t.Fatalf("UpdateFromScan() error = %v", err)
	}

	// 验证
	updated, _ := svc.GetSong(ctx, pubResp.Id)
	if updated.Title != "User Title" {
		t.Errorf("Title = %q, want %q (should NOT be overwritten)", updated.Title, "User Title")
	}
	if updated.Artist != "Tag Artist" {
		t.Errorf("Artist = %q, want %q (should be filled)", updated.Artist, "Tag Artist")
	}
	if updated.FileHash != "new-hash-abc" {
		t.Errorf("FileHash = %q, want %q", updated.FileHash, "new-hash-abc")
	}
}
```

- [ ] **Step 6: 更新 main.go — 注入 adapter**

```go
// cmd/server/main.go — 在 rest.NewHandler 调用处

// SongUpdater adapter: 将 REST handler 的 UpdateFromScan 桥接到 SongService
type songUpdaterAdapter struct {
	svc *song.Service
}

func (a *songUpdaterAdapter) UpdateFromScan(songID, fileHash string, fileSize int64) error {
	// 注意：实际添加时需要传递元数据字段
	// 简化方案：在 handler 中直接调用 SongService.UpdateFromScan
	// 实际上 SongUpdater 接口需要包含 metadata 参数
	// 这里使用改进的接口设计（见下方改进方案）
}
```

等等 —— 这里有个接口设计问题。`SongUpdater.UpdateFromScan` 只传了 `songID, fileHash, fileSize`，但 `SongService.UpdateFromScan` 需要 title/artist 等元数据字段。接口应该传递完整的元数据。

让我修正接口定义：

```go
// internal/rest/publisher.go — 修正版
package rest

import "github.com/inkOrCloud/EchoVault/echovault-server/pkg/metadata"

// SongUpdater 定义上传音频后补充元数据的接口。
type SongUpdater interface {
	UpdateFromScan(songID string, meta *metadata.AudioMetadata, fileSize int64) error
}
```

然后 adapter 实现：

```go
// cmd/server/main.go — adapter
type songUpdaterAdapter struct {
	svc *song.Service
}

func (a *songUpdaterAdapter) UpdateFromScan(songID string, meta *metadata.AudioMetadata, fileSize int64) error {
	ctx := context.Background()
	_, err := a.svc.UpdateFromScan(ctx, songID,
		meta.Title, meta.Artist, meta.Album, meta.Genre,
		int32(meta.TrackNumber), int32(meta.DiscNumber), int32(meta.Year),
		meta.FileHash, meta.FileName, meta.MIMEType, fileSize)
	return err
}
```

- [ ] **Step 7: Run all tests**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault && go test ./echovault-server/pkg/metadata/ -v -count=1
cd /home/ink/Desktop/develop/EchoVault && go test ./echovault-server/internal/service/song/ -v -count=1 -run TestUpdateFromScan
cd /home/ink/Desktop/develop/EchoVault && go test ./echovault-server/internal/rest/ -v -count=1
```
Expected: all 3 suites PASS

- [ ] **Step 8: 跑全量测试**

Run: `cd /home/ink/Desktop/develop/EchoVault && go test ./echovault-server/... -count=1 2>&1 | tail -30`
Expected: all tests PASS (regression free)

- [ ] **Step 9: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echovault-server/internal/rest/ echovault-server/internal/service/song/service.go echovault-server/internal/service/song/service_test.go echovault-server/cmd/server/main.go
git commit -m "feat: integrate pkg/metadata into upload flow — auto-fill empty Song fields + save embedded cover"
```

---

### Phase 5b 自审清单

- [ ] `pkg/metadata.ParseFile` 解析 MP3 ID3v2 标签（标题/艺术家/专辑/曲目号）
- [ ] `pkg/metadata.ParseFile` 解析 FLAC Vorbis Comment 标签
- [ ] `pkg/metadata.ParseFile` 提取嵌入封面图片（APIC frame），返回 `*Picture`
- [ ] `pkg/metadata.ParseFile` 对非音频文件返回基础信息（不崩溃）
- [ ] `pkg/metadata.MIMETypeByExt` 覆盖 9 种音频格式
- [ ] `pkg/metadata.IsAudioFile` 正确识别音频扩展名
- [ ] `SongService.UpdateFromScan` **仅补充用户未填的字段**（title/artist 等不覆盖）
- [ ] `SongService.UpdateFromScan` 始终更新 file_hash/file_size/mime_type
- [ ] REST 上传音频 → 自动提取元数据 → 补充 Song 空字段
- [ ] REST 上传音频 → 自动保存嵌入封面到 Storage
- [ ] 元数据解析失败不阻止上传成功（容错）
- [ ] 上传封面（type=cover）不触发 Song 更新
- [ ] 全量测试通过（~90 tests）

**全局测试数：** 现有 78 + 新增 ~10 = **~88 测试全部通过**

**入口：** `Phase 7: 歌曲库 + 扫描器 UI (Flutter)`
