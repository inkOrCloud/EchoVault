# Phase 5a: 歌词 + 歌单（后端剩余服务） Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use subagent-driven-development. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成后端所有剩余 Go 服务：LyricService、LRC 解析器、PlaylistService、对应 gRPC Handlers

**Architecture:** 标准三层：Service 业务逻辑 → gRPC Handler 协议层 → RegisterAll 接入。Lyric 存原始 LRC 文本，Playlist 使用浮动位置排序。

**Tech Stack:** Go 1.26+, Ent, SQLite, gRPC Unary, protobuf

---

### Task 1: LyricService（歌词管理）

**Files:**
- Create: `echovault-server/internal/service/lyric/service.go`
- Create: `echovault-server/internal/service/lyric/service_test.go`

- [ ] **Step 1 (RED): 编写测试**

```go
// internal/service/lyric/service_test.go
package lyric_test

import (
	"context"
	"testing"

	entsql "entgo.io/ent/dialect/sql"
	_ "github.com/mattn/go-sqlite3"

	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent/enttest"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/service/lyric"
	lyricpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/lyric/v1"
)

func newTestClient(t *testing.T) *ent.Client {
	t.Helper()
	drv, _ := entsql.Open("sqlite3", "file:lyric?mode=memory&cache=shared&_fk=1")
	client := enttest.NewClient(t, enttest.WithOptions(ent.Driver(drv)))
	if err := client.Schema.Create(context.Background()); err != nil {
		t.Fatalf("create schema: %v", err)
	}
	return client
}

func TestSaveAndGetLyric(t *testing.T) {
	client := newTestClient(t); defer client.Close()
	svc := lyric.NewService(client)
	ctx := context.Background()

	lrc := "[00:01.00]Line 1\n[00:02.00]Line 2"
	saved, err := svc.SaveLyric(ctx, "song-1", lrc, lyricpb.Lyric_TYPE_ORIGINAL, "zh")
	if err != nil { t.Fatalf("SaveLyric() error = %v", err) }
	if saved.SongId != "song-1" { t.Errorf("SongId = %q", saved.SongId) }

	got, err := svc.GetLyric(ctx, "song-1", "", lyricpb.Lyric_TYPE_UNSPECIFIED)
	if err != nil { t.Fatalf("GetLyric() error = %v", err) }
	if got.Content != lrc { t.Errorf("content mismatch") }
}

func TestGetLyric_NotFound(t *testing.T) {
	client := newTestClient(t); defer client.Close()
	svc := lyric.NewService(client)
	_, err := svc.GetLyric(context.Background(), "no-such-song", "", lyricpb.Lyric_TYPE_UNSPECIFIED)
	if err == nil { t.Fatal("expected error for nonexistent song") }
}

func TestSaveLyric_UpdateExisting(t *testing.T) {
	client := newTestClient(t); defer client.Close()
	svc := lyric.NewService(client)
	ctx := context.Background()

	svc.SaveLyric(ctx, "song-1", "[00:01.00]v1", lyricpb.Lyric_TYPE_ORIGINAL, "zh")
	svc.SaveLyric(ctx, "song-1", "[00:01.00]v2", lyricpb.Lyric_TYPE_ORIGINAL, "zh")

	got, _ := svc.GetLyric(ctx, "song-1", "zh", lyricpb.Lyric_TYPE_ORIGINAL)
	if got.Content != "[00:01.00]v2" { t.Errorf("content = %q, want v2", got.Content) }
}

func TestSaveLyric_MultipleLanguages(t *testing.T) {
	client := newTestClient(t); defer client.Close()
	svc := lyric.NewService(client)
	ctx := context.Background()

	svc.SaveLyric(ctx, "song-1", "zh lrc", lyricpb.Lyric_TYPE_ORIGINAL, "zh")
	svc.SaveLyric(ctx, "song-1", "en lrc", lyricpb.Lyric_TYPE_TRANSLATION, "en")

	lyrics, err := svc.GetAllLyrics(ctx, "song-1")
	if err != nil { t.Fatalf("GetAllLyrics() error = %v", err) }
	if len(lyrics) != 2 { t.Errorf("lyrics count = %d, want 2", len(lyrics)) }
}

func TestDeleteLyric(t *testing.T) {
	client := newTestClient(t); defer client.Close()
	svc := lyric.NewService(client)
	ctx := context.Background()

	svc.SaveLyric(ctx, "song-1", "lrc", lyricpb.Lyric_TYPE_ORIGINAL, "zh")
	if err := svc.DeleteLyric(ctx, "song-1", lyricpb.Lyric_TYPE_ORIGINAL, "zh"); err != nil {
		t.Fatalf("DeleteLyric() error = %v", err)
	}
	_, err := svc.GetLyric(ctx, "song-1", "zh", lyricpb.Lyric_TYPE_ORIGINAL)
	if err == nil { t.Fatal("expected error after delete") }
}
```

- [ ] **Step 2 (VERIFY RED)**
- [ ] **Step 3 (GREEN): 实现**

```go
// internal/service/lyric/service.go
package lyric

import (
	"context"
	"errors"
	"time"

	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent/lyric"
	lyricpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/lyric/v1"
)

type Service struct{ client *ent.Client }
func NewService(client *ent.Client) *Service { return &Service{client: client} }

func (s *Service) SaveLyric(ctx context.Context, songID, content string, typ lyricpb.Lyric_Type, language string) (*lyricpb.Lyric, error) {
	typStr := typ.String()
	now := time.Now()
	// 检查是否已存在
	existing, _ := s.client.Lyric.Query().
		Where(lyric.SongID(songID), lyric.Type(typStr), lyric.Language(language)).
		First(ctx)
	if existing != nil {
		r, err := s.client.Lyric.UpdateOne(existing).
			SetContent(content).SetUpdatedAt(now).Save(ctx)
		if err != nil { return nil, err }
		return entToProto(r), nil
	}
	r, err := s.client.Lyric.Create().
		SetSongID(songID).SetContent(content).
		SetType(typStr).SetLanguage(language).
		Save(ctx)
	if err != nil { return nil, err }
	return entToProto(r), nil
}

func (s *Service) GetLyric(ctx context.Context, songID, language string, typ lyricpb.Lyric_Type) (*lyricpb.Lyric, error) {
	query := s.client.Lyric.Query().Where(lyric.SongID(songID))
	if language != "" { query = query.Where(lyric.Language(language)) }
	if typ != lyricpb.Lyric_TYPE_UNSPECIFIED { query = query.Where(lyric.Type(typ.String())) }
	r, err := query.First(ctx)
	if err != nil {
		if ent.IsNotFound(err) { return nil, errors.New("lyric not found") }
		return nil, err
	}
	return entToProto(r), nil
}

func (s *Service) GetAllLyrics(ctx context.Context, songID string) ([]*lyricpb.Lyric, error) {
	records, err := s.client.Lyric.Query().Where(lyric.SongID(songID)).All(ctx)
	if err != nil { return nil, err }
	result := make([]*lyricpb.Lyric, len(records))
	for i, r := range records { result[i] = entToProto(r) }
	return result, nil
}

func (s *Service) DeleteLyric(ctx context.Context, songID string, typ lyricpb.Lyric_Type, language string) error {
	n, err := s.client.Lyric.Delete().Where(
		lyric.SongID(songID), lyric.Type(typ.String()), lyric.Language(language),
	).Exec(ctx)
	if err != nil { return err }
	if n == 0 { return errors.New("lyric not found") }
	return nil
}

func entToProto(r *ent.Lyric) *lyricpb.Lyric {
	t := lyricpb.Lyric_TYPE_UNSPECIFIED
	switch r.Type { case "TYPE_ORIGINAL": t = lyricpb.Lyric_TYPE_ORIGINAL; case "TYPE_TRANSLATION": t = lyricpb.Lyric_TYPE_TRANSLATION; case "TYPE_PHONETIC": t = lyricpb.Lyric_TYPE_PHONETIC }
	s := lyricpb.Lyric_SOURCE_UNSPECIFIED
	switch r.Source { case "SOURCE_EMBEDDED": s = lyricpb.Lyric_SOURCE_EMBEDDED; case "SOURCE_MANUAL": s = lyricpb.Lyric_SOURCE_MANUAL; case "SOURCE_FETCHED": s = lyricpb.Lyric_SOURCE_FETCHED }
	return &lyricpb.Lyric{
		SongId: r.SongID, Content: r.Content, Type: t,
		Language: r.Language, OffsetMs: r.OffsetMs, Source: s,
		Version: r.Version, UpdatedAt: timestamppb.New(r.UpdatedAt),
	}
}
```

- [ ] **Step 4 (VERIFY GREEN)**
- [ ] **Step 5: 提交**

---

### Task 2: LRC 解析器

**Files:**
- Create: `echovault-server/pkg/lrc/parser.go`
- Create: `echovault-server/pkg/lrc/parser_test.go`

- [ ] **Step 1 (RED): 编写测试**

```go
// pkg/lrc/parser_test.go
package lrc_test

import (
	"testing"
	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/lrc"
)

func TestParse_Basic(t *testing.T) {
	lines := lrc.Parse("[00:01.00]Line 1\n[00:02.50]Line 2")
	if len(lines) != 2 { t.Fatalf("count = %d", len(lines)) }
	if lines[0].Text != "Line 1" { t.Errorf("text = %q", lines[0].Text) }
	if lines[0].Timestamp != 1000 { t.Errorf("ts = %d", lines[0].Timestamp) }
	if lines[1].Timestamp != 2500 { t.Errorf("ts = %d", lines[1].Timestamp) }
}

func TestParse_WithOffset(t *testing.T) {
	lines := lrc.Parse("[offset:+500]\n[00:01.00]Line")
	if len(lines) != 1 { t.Fatalf("count = %d", len(lines)) }
	if lines[0].Timestamp != 1500 { t.Errorf("ts = %d, want 1500", lines[0].Timestamp) }
}

func TestParse_Empty(t *testing.T) {
	if lines := lrc.Parse(""); len(lines) != 0 { t.Errorf("empty should return 0") }
}

func TestParse_InvalidLine(t *testing.T) {
	lines := lrc.Parse("not a lrc line\n[00:01.00]valid")
	if len(lines) != 1 { t.Errorf("count = %d, want 1", len(lines)) }
}

func TestFormat(t *testing.T) {
	lines := []lrc.Line{{Timestamp: 1000, Text: "Hello"}, {Timestamp: 2500, Text: "World"}}
	out := lrc.Format(lines)
	expected := "[00:01.00]Hello\n[00:02.50]World\n"
	if out != expected { t.Errorf("got %q, want %q", out, expected) }
}
```

- [ ] **Step 2 (VERIFY RED)**
- [ ] **Step 3 (GREEN): 实现**

```go
// pkg/lrc/parser.go
package lrc

import (
	"fmt"
	"regexp"
	"strconv"
	"strings"
)

type Line struct {
	Timestamp int64  // 毫秒
	Text      string
}

var lineRe = regexp.MustCompile(`\[(\d+):(\d+(?:\.\d+)?)\](.*)`)

func Parse(lrc string) []Line {
	var lines []Line
	offset := int64(0)
	for _, raw := range strings.Split(lrc, "\n") {
		raw = strings.TrimSpace(raw)
		if raw == "" { continue }
		if strings.HasPrefix(raw, "[offset:") {
			o := strings.TrimSuffix(strings.TrimPrefix(raw, "[offset:"), "]")
			if v, err := strconv.ParseInt(o, 10, 64); err == nil { offset = v }
			continue
		}
		m := lineRe.FindStringSubmatch(raw)
		if m == nil { continue }
		min, _ := strconv.ParseInt(m[1], 10, 64)
		sec, _ := strconv.ParseFloat(m[2], 64)
		ts := min*60000 + int64(sec*1000) + offset
		lines = append(lines, Line{Timestamp: ts, Text: m[3]})
	}
	return lines
}

func Format(lines []Line) string {
	var b strings.Builder
	for _, l := range lines {
		min := l.Timestamp / 60000
		sec := float64(l.Timestamp%60000) / 1000
		b.WriteString(fmt.Sprintf("[%02d:%05.2f]%s\n", min, sec, l.Text))
	}
	return b.String()
}
```

- [ ] **Step 4 (VERIFY GREEN)**
- [ ] **Step 5: 提交**

---

### Task 3: PlaylistService（歌单管理）

**Files:**
- Create: `echovault-server/internal/service/playlist/service.go`
- Create: `echovault-server/internal/service/playlist/service_test.go`

- [ ] **Step 1 (RED): 编写测试**

6 个测试：创建歌单、获取歌单、添加歌曲、移除歌曲、重新排序、删除歌单

- [ ] **Step 2 (VERIFY RED)**
- [ ] **Step 3 (GREEN): 实现 PlaylistService**

核心逻辑：
- 创建/获取/更新/删除歌单
- AddSong 使用 position = max + 1000 的浮动排序
- ReorderSongs 重新分配 1000 间隔
- RemoveSong 从歌单移除

- [ ] **Step 4 (VERIFY GREEN)**
- [ ] **Step 5: 提交**

---

### Task 4: gRPC LyricHandler + PlaylistHandler

**Files:**
- Create: `echovault-server/internal/grpc/handler_lyric.go`
- Create: `echovault-server/internal/grpc/handler_lyric_test.go`
- Create: `echovault-server/internal/grpc/handler_playlist.go`
- Create: `echovault-server/internal/grpc/handler_playlist_test.go`
- Modify: `echovault-server/internal/grpc/register.go`

- [ ] **Step 1 (RED):** 各 2 个集成测试
- [ ] **Step 2 (VERIFY RED)**
- [ ] **Step 3 (GREEN):** 实现 Handler + 注册
- [ ] **Step 4 (VERIFY GREEN)**
- [ ] **Step 5: 提交全量验证**

---

### 自审清单

- [ ] LyricService 6 测试通过
- [ ] LRC 解析器 6 测试通过
- [ ] PlaylistService 6 测试通过
- [ ] gRPC Handlers 4 测试通过
- [ ] `go test ./... -count=1` 全部通过
- [ ] `go build ./cmd/server` 编译通过
- [ ] 所有提交推送到 GitHub

**入口：** `Phase 6: Flutter 前端开发`
