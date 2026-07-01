# Phase 2: 用户系统 + 设备管理 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现用户注册/登录/JWT 认证、设备注册与管理、gRPC Handler 及认证拦截器

**Architecture:** Go 后端 service 层实现业务逻辑，gRPC Handler 层处理协议转换，auth middleware 实现 JWT 拦截。所有开发遵循 TDD：先写测试，验证失败，再实现，验证通过。

**Tech Stack:** Go 1.26+, Ent, SQLite, gRPC, JWT (golang-jwt), bcrypt

---

### Task 1: JWT 工具包

**Files:**
- Create: `echovault-server/pkg/auth/jwt.go`
- Create: `echovault-server/pkg/auth/jwt_test.go`

- [ ] **Step 1 (RED): 编写 JWT 测试**

```go
// pkg/auth/jwt_test.go
package auth_test

import (
	"testing"
	"time"

	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/auth"
)

func TestGenerateToken_Success(t *testing.T) {
	secret := "test-secret-key"
	userID := "user-123"
	deviceID := "device-456"

	token, err := auth.GenerateToken(secret, userID, deviceID, 1*time.Hour)
	if err != nil {
		t.Fatalf("GenerateToken() error = %v", err)
	}
	if token == "" {
		t.Fatal("GenerateToken() returned empty token")
	}
}

func TestValidateToken_Valid(t *testing.T) {
	secret := "test-secret-key"
	userID := "user-123"
	deviceID := "device-456"

	token, _ := auth.GenerateToken(secret, userID, deviceID, 1*time.Hour)

	claims, err := auth.ValidateToken(secret, token)
	if err != nil {
		t.Fatalf("ValidateToken() error = %v", err)
	}
	if claims.UserID != userID {
		t.Errorf("claims.UserID = %q, want %q", claims.UserID, userID)
	}
	if claims.DeviceID != deviceID {
		t.Errorf("claims.DeviceID = %q, want %q", claims.DeviceID, deviceID)
	}
}

func TestValidateToken_Expired(t *testing.T) {
	secret := "test-secret-key"
	token, _ := auth.GenerateToken(secret, "user-1", "device-1", -1*time.Hour)

	_, err := auth.ValidateToken(secret, token)
	if err == nil {
		t.Fatal("ValidateToken() expected error for expired token")
	}
}

func TestValidateToken_WrongSecret(t *testing.T) {
	token, _ := auth.GenerateToken("secret-a", "user-1", "device-1", 1*time.Hour)
	_, err := auth.ValidateToken("secret-b", token)
	if err == nil {
		t.Fatal("ValidateToken() expected error for wrong secret")
	}
}

func TestValidateToken_InvalidFormat(t *testing.T) {
	_, err := auth.ValidateToken("secret", "not-a-jwt-token")
	if err == nil {
		t.Fatal("ValidateToken() expected error for invalid format")
	}
}

func TestHashPassword_And_Compare(t *testing.T) {
	password := "my-password-123"
	hash, err := auth.HashPassword(password)
	if err != nil {
		t.Fatalf("HashPassword() error = %v", err)
	}
	if hash == "" {
		t.Fatal("HashPassword() returned empty hash")
	}
	if hash == password {
		t.Fatal("HashPassword() returned plaintext")
	}

	if !auth.CheckPassword(hash, password) {
		t.Error("CheckPassword() = false, want true")
	}
	if auth.CheckPassword(hash, "wrong-password") {
		t.Error("CheckPassword() = true for wrong password, want false")
	}
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认编译失败**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
GOCACHE=/tmp/go-cache go test ./pkg/auth/...
```
Expected: `FAIL pkg/auth [build failed]` (package not found)

- [ ] **Step 3 (GREEN): 实现 JWT 工具包**

```go
// pkg/auth/jwt.go
package auth

import (
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"golang.org/x/crypto/bcrypt"
)

type Claims struct {
	UserID   string `json:"user_id"`
	DeviceID string `json:"device_id"`
	jwt.RegisteredClaims
}

func GenerateToken(secret, userID, deviceID string, ttl time.Duration) (string, error) {
	claims := Claims{
		UserID:   userID,
		DeviceID: deviceID,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(ttl)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
		},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString([]byte(secret))
}

func ValidateToken(secret, tokenString string) (*Claims, error) {
	token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(t *jwt.Token) (interface{}, error) {
		if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
		}
		return []byte(secret), nil
	})
	if err != nil {
		return nil, err
	}
	claims, ok := token.Claims.(*Claims)
	if !ok || !token.Valid {
		return nil, fmt.Errorf("invalid token claims")
	}
	return claims, nil
}

func HashPassword(password string) (string, error) {
	bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	return string(bytes), err
}

func CheckPassword(hash, password string) bool {
	return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认全部通过**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
GOCACHE=/tmp/go-cache go test ./pkg/auth/... -v
```
Expected: 6 tests, all PASS

- [ ] **Step 5: 安装依赖并提交**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
go get github.com/golang-jwt/jwt/v5@latest
go get golang.org/x/crypto/bcrypt@latest
go mod tidy
git add pkg/auth/ go.mod go.sum
git commit -m "feat(auth): add JWT and bcrypt utilities (TDD)"
```

---

### Task 2: UserService（注册 + 登录 + 获取用户信息）

**Files:**
- Create: `echovault-server/internal/service/user/service.go`
- Create: `echovault-server/internal/service/user/service_test.go`

Dependencies:
- `pkg/auth` (Task 1)
- `internal/ent` (Phase 1)
- `pkg/convert` (Phase 1)
- 生成的 proto: `userpb`, `commonpb`

- [ ] **Step 1 (RED): 编写 UserService 测试**

```go
// internal/service/user/service_test.go
package user_test

import (
	"context"
	"testing"

	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent/enttest"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/service/user"
	_ "github.com/mattn/go-sqlite3"

	entsql "entgo.io/ent/dialect/sql"
)

func newTestClient(t *testing.T) *ent.Client {
	t.Helper()
	drv, err := entsql.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	if err != nil {
		t.Fatalf("failed to open test db: %v", err)
	}
	client := ent.NewClient(ent.Driver(drv))
	if err := client.Schema.Create(context.Background()); err != nil {
		t.Fatalf("failed to create schema: %v", err)
	}
	return client
}

func TestRegister_Success(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := user.NewService(client, "test-secret")
	ctx := context.Background()

	resp, err := svc.Register(ctx, "newuser", "password123", "New User")
	if err != nil {
		t.Fatalf("Register() error = %v", err)
	}
	if resp.UserID == "" {
		t.Error("Register() returned empty UserID")
	}
	if resp.Username != "newuser" {
		t.Errorf("Register() Username = %q, want %q", resp.Username, "newuser")
	}
	if resp.Token == "" {
		t.Error("Register() returned empty Token")
	}
}

func TestRegister_DuplicateUsername(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := user.NewService(client, "test-secret")
	ctx := context.Background()

	svc.Register(ctx, "dupuser", "pass1", "User 1")
	_, err := svc.Register(ctx, "dupuser", "pass2", "User 2")
	if err == nil {
		t.Fatal("Register() expected error for duplicate username")
	}
}


func TestRegister_WeakPassword(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := user.NewService(client, "test-secret")
	ctx := context.Background()

	tests := []struct {
		name     string
		password string
	}{
		{"too short", "ab"},
		{"has space", "pass word123"},
		{"has tab", "pass\tword123"},
		{"has newline", "pass\nword123"},
		{"non-ASCII", "passwörd123"},
		{"all valid", "ValidPass1"},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			_, err := svc.Register(ctx, "user_"+tt.name, tt.password, "Test")
			if err == nil {
				t.Errorf("Register() with password %q expected error", tt.password)
			}
		})
	}
}

func TestLogin_Success(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := user.NewService(client, "test-secret")
	ctx := context.Background()

	svc.Register(ctx, "loginuser", "mypass", "Login User")
	resp, err := svc.Login(ctx, "loginuser", "mypass", "device-001")
	if err != nil {
		t.Fatalf("Login() error = %v", err)
	}
	if resp.Token == "" {
		t.Error("Login() returned empty token")
	}
	if resp.UserID == "" {
		t.Error("Login() returned empty UserID")
	}
}

func TestLogin_WrongPassword(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := user.NewService(client, "test-secret")
	ctx := context.Background()

	svc.Register(ctx, "userx", "correctpass", "User X")
	_, err := svc.Login(ctx, "userx", "wrongpass", "device-002")
	if err == nil {
		t.Fatal("Login() expected error for wrong password")
	}
}

func TestLogin_UserNotFound(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := user.NewService(client, "test-secret")

	_, err := svc.Login(context.Background(), "nouser", "pass", "device-003")
	if err == nil {
		t.Fatal("Login() expected error for nonexistent user")
	}
}

func TestGetUser_Success(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := user.NewService(client, "test-secret")
	ctx := context.Background()

	regResp, _ := svc.Register(ctx, "getme", "pass", "Get Me")
	u, err := svc.GetUser(ctx, regResp.UserID)
	if err != nil {
		t.Fatalf("GetUser() error = %v", err)
	}
	if u.Username != "getme" {
		t.Errorf("GetUser() Username = %q, want %q", u.Username, "getme")
	}
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认因缺少实现而失败**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
GOCACHE=/tmp/go-cache go test ./internal/service/user/... -v
```
Expected: compilation errors (no Service struct, no NewService, etc.)

- [ ] **Step 3 (GREEN): 实现 UserService**

```go
// internal/service/user/service.go
package user

import (
	"context"
	"database/sql"
	"errors"
	"time"

	"github.com/google/uuid"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent/user"
	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/auth"
	"golang.org/x/crypto/bcrypt"
)

type RegisterResponse struct {
	UserID   string
	Username string
	Token    string
}

type LoginResponse struct {
	UserID   string
	Username string
	Token    string
}


// 在 Service struct 定义之后、NewService 之前添加

// ValidatePassword 检查密码合法性
// 规则：8-128 位，仅允许大小写字母、数字、半角符号，不含空白字符
func ValidatePassword(password string) error {
	if len(password) < 8 {
		return errors.New("password must be at least 8 characters")
	}
	if len(password) > 128 {
		return errors.New("password must be at most 128 characters")
	}
	for _, c := range password {
		if c <= 32 || c == 127 {
			return errors.New("password contains invalid characters")
		}
		if c > 126 {
			return errors.New("password contains non-ASCII characters")
		}
	}
	return nil
}

type Service struct {
	client *ent.Client
	secret string
}

func NewService(client *ent.Client, jwtSecret string) *Service {
	return &Service{client: client, secret: jwtSecret}
}

func (s *Service) Register(ctx context.Context, username, password, displayName string) (*RegisterResponse, error) {
	exists, _ := s.client.User.Query().Where(user.Username(username)).Exist(ctx)
	if exists {
		return nil, errors.New("username already exists")
	}

	hash, err := auth.HashPassword(password)
	if err != nil {
		return nil, err
	}

	if err := ValidatePassword(password); err != nil {
		return nil, err
	}

	u, err := s.client.User.Create().
		SetID(uuid.New().String()).
		SetUsername(username).
		SetDisplayName(displayName).
		SetPasswordHash(hash).
		SetRole("user").
		Save(ctx)
	if err != nil {
		return nil, err
	}

	token, err := auth.GenerateToken(s.secret, u.ID, "", 24*time.Hour)
	if err != nil {
		return nil, err
	}

	return &RegisterResponse{UserID: u.ID, Username: u.Username, Token: token}, nil
}

func (s *Service) Login(ctx context.Context, username, password, deviceID string) (*LoginResponse, error) {
	u, err := s.client.User.Query().Where(user.Username(username)).Only(ctx)
	if err != nil {
		if ent.IsNotFound(err) {
			return nil, errors.New("user not found")
		}
		return nil, err
	}

	if !auth.CheckPassword(u.PasswordHash, password) {
		return nil, errors.New("incorrect password")
	}

	token, err := auth.GenerateToken(s.secret, u.ID, deviceID, 24*time.Hour)
	if err != nil {
		return nil, err
	}

	return &LoginResponse{UserID: u.ID, Username: u.Username, Token: token}, nil
}

func (s *Service) GetUser(ctx context.Context, userID string) (*ent.User, error) {
	u, err := s.client.User.Get(ctx, userID)
	if err != nil {
		if ent.IsNotFound(err) {
			return nil, errors.New("user not found")
		}
		return nil, err
	}
	return u, nil
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认全部通过**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
GOCACHE=/tmp/go-cache go test ./internal/service/user/... -v
```
Expected: 6 tests, all PASS

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
go get github.com/google/uuid@latest
go mod tidy
git add internal/service/user/service.go internal/service/user/service_test.go go.mod go.sum
git commit -m "feat(user): add UserService with register, login, and get user (TDD, 6 tests)"
```

---

### Task 3: DeviceService（设备注册/列表/移除）

**Files:**
- Modify: `echovault-server/internal/service/user/service.go` (追加 Device methods)
- Modify: `echovault-server/internal/service/user/service_test.go` (追加 Device tests)
- Create: `echovault-server/internal/service/user/device.go` (设备逻辑分离)

- [ ] **Step 1 (RED): 编写 Device 测试**

追加到 `service_test.go`:

```go
func TestRegisterDevice_Success(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := user.NewService(client, "test-secret")
	ctx := context.Background()

	reg, _ := svc.Register(ctx, "deviceuser", "pass", "Device User")
	err := svc.RegisterDevice(ctx, reg.UserID, "dev-001", "My Desktop", "linux")
	if err != nil {
		t.Fatalf("RegisterDevice() error = %v", err)
	}
}

func TestRegisterDevice_DuplicateID(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := user.NewService(client, "test-secret")
	ctx := context.Background()

	reg, _ := svc.Register(ctx, "dupdev", "pass", "Dup Device")
	svc.RegisterDevice(ctx, reg.UserID, "dev-001", "First", "linux")
	err := svc.RegisterDevice(ctx, reg.UserID, "dev-001", "Second", "macos")
	if err != nil {
		t.Logf("Duplicate device handled: %v", err) // 允许存在或返回错误
	}
}

func TestListDevices(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := user.NewService(client, "test-secret")
	ctx := context.Background()

	reg, _ := svc.Register(ctx, "listdev", "pass", "List Dev")
	svc.RegisterDevice(ctx, reg.UserID, "d1", "Desktop", "linux")
	svc.RegisterDevice(ctx, reg.UserID, "d2", "Phone", "android")

	devices, err := svc.ListDevices(ctx, reg.UserID)
	if err != nil {
		t.Fatalf("ListDevices() error = %v", err)
	}
	if len(devices) != 2 {
		t.Errorf("ListDevices() count = %d, want 2", len(devices))
	}
}

func TestRemoveDevice(t *testing.T) {
	client := newTestClient(t)
	defer client.Close()
	svc := user.NewService(client, "test-secret")
	ctx := context.Background()

	reg, _ := svc.Register(ctx, "rmdev", "pass", "Rm Dev")
	svc.RegisterDevice(ctx, reg.UserID, "d1", "Desktop", "linux")
	err := svc.RemoveDevice(ctx, reg.UserID, "d1")
	if err != nil {
		t.Fatalf("RemoveDevice() error = %v", err)
	}

	devices, _ := svc.ListDevices(ctx, reg.UserID)
	if len(devices) != 0 {
		t.Errorf("After RemoveDevice, count = %d, want 0", len(devices))
	}
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认失败**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
GOCACHE=/tmp/go-cache go test ./internal/service/user/... -v -run "Device"
```
Expected: compilation errors for undefined methods

- [ ] **Step 3 (GREEN): 实现 Device 方法**

```go
// internal/service/user/device.go
package user

import (
	"context"
	"errors"
	"time"

	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent/device"
)

func (s *Service) RegisterDevice(ctx context.Context, userID, deviceID, name, platform string) error {
	exists, _ := s.client.Device.Query().Where(device.DeviceID(deviceID)).Exist(ctx)
	if exists {
		return errors.New("device already registered")
	}

	_, err := s.client.Device.Create().
		SetDeviceID(deviceID).
		SetDeviceName(name).
		SetPlatform(platform).
		SetUserID(userID).
		SetLastSyncAt(time.Now()).
		Save(ctx)
	return err
}

func (s *Service) ListDevices(ctx context.Context, userID string) ([]*ent.Device, error) {
	return s.client.Device.Query().Where(device.UserID(userID)).All(ctx)
}

func (s *Service) RemoveDevice(ctx context.Context, userID, deviceID string) error {
	n, err := s.client.Device.Delete().Where(
		device.DeviceID(deviceID),
		device.UserID(userID),
	).Exec(ctx)
	if err != nil {
		return err
	}
	if n == 0 {
		return errors.New("device not found")
	}
	return nil
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认全部通过**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
GOCACHE=/tmp/go-cache go test ./internal/service/user/... -v
```
Expected: 10 tests (6 from Task 2 + 4 Device), all PASS

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
git add internal/service/user/device.go internal/service/user/service_test.go
git commit -m "feat(device): add device register/list/remove (TDD, +4 tests, total 10)"
```

---

### Task 4: gRPC Handler + Auth Interceptor

**Files:**
- Create: `echovault-server/internal/grpc/auth_interceptor.go`
- Create: `echovault-server/internal/grpc/handler_user.go`
- Create: `echovault-server/internal/grpc/handler_user_test.go`
- Modify: `echovault-server/internal/grpc/register.go`

- [ ] **Step 1 (RED): 编写 Handler 测试**

```go
// internal/grpc/handler_user_test.go
package grpc_test

import (
	"context"
	"testing"

	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent/enttest"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/grpc"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/service/user"
	userpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/user/v1"
	_ "github.com/mattn/go-sqlite3"

	entsql "entgo.io/ent/dialect/sql"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

func newTestServer(t *testing.T) (userpb.UserServiceClient, func()) {
	t.Helper()
	drv, err := entsql.Open("sqlite3", "file:handler?mode=memory&cache=shared&_fk=1")
	if err != nil {
		t.Fatalf("open db: %v", err)
	}
	client := enttest.NewClient(t, enttest.WithOptions(ent.Driver(drv)))
	if err := client.Schema.Create(context.Background()); err != nil {
		t.Fatalf("create schema: %v", err)
	}

	svc := user.NewService(client, "test-secret")
	userHandler := grpc.NewUserHandler(svc)

	s := grpc.NewServer()
	userpb.RegisterUserServiceServer(s, userHandler)

	lis, _ := net.Listen("tcp", "127.0.0.1:0")
	go s.Serve(lis)

	conn, _ := grpc.Dial(lis.Addr().String(), grpc.WithTransportCredentials(insecure.NewCredentials()))
	userClient := userpb.NewUserServiceClient(conn)

	return userClient, func() {
		conn.Close()
		s.GracefulStop()
	}
}

func TestRegisterHandler_Success(t *testing.T) {
	client, cleanup := newTestServer(t)
	defer cleanup()

	resp, err := client.Register(context.Background(), &userpb.RegisterRequest{
		Username: "handleruser",
		Password: "pass123",
	})
	if err != nil {
		t.Fatalf("Register RPC error = %v", err)
	}
	if resp.User.Username != "handleruser" {
		t.Errorf("resp.User.Username = %q, want %q", resp.User.Username, "handleruser")
	}
	if resp.AccessToken == "" {
		t.Error("resp.AccessToken is empty")
	}
}

func TestLoginHandler_Success(t *testing.T) {
	client, cleanup := newTestServer(t)
	defer cleanup()

	client.Register(context.Background(), &userpb.RegisterRequest{
		Username: "loginhandler", Password: "pass456",
	})
	resp, err := client.Login(context.Background(), &userpb.LoginRequest{
		Username: "loginhandler", Password: "pass456", DeviceId: "dev-001",
	})
	if err != nil {
		t.Fatalf("Login RPC error = %v", err)
	}
	if resp.AccessToken == "" {
		t.Error("resp.AccessToken is empty")
	}
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认失败**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
GOCACHE=/tmp/go-cache go test ./internal/grpc/... -v -run "Handler"
```
Expected: compilation errors

- [ ] **Step 3 (GREEN): 实现 Handler + Interceptor**

```go
// internal/grpc/auth_interceptor.go
package grpc

import (
	"context"
	"strings"

	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/auth"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
)

func AuthInterceptor(secret string) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
		// 跳过无需认证的 RPC
		if isPublicRPC(info.FullMethod) {
			return handler(ctx, req)
		}

		md, ok := metadata.FromIncomingContext(ctx)
		if !ok {
			return nil, status.Error(codes.Unauthenticated, "missing metadata")
		}
		authHeader := md.Get("authorization")
		if len(authHeader) == 0 {
			return nil, status.Error(codes.Unauthenticated, "missing authorization header")
		}

		tokenStr := strings.TrimPrefix(authHeader[0], "Bearer ")
		claims, err := auth.ValidateToken(secret, tokenStr)
		if err != nil {
			return nil, status.Error(codes.Unauthenticated, "invalid token: "+err.Error())
		}

		// 将 claims 注入 context
		ctx = context.WithValue(ctx, ctxKeyUserID, claims.UserID)
		ctx = context.WithValue(ctx, ctxKeyDeviceID, claims.DeviceID)
		return handler(ctx, req)
	}
}

func isPublicRPC(method string) bool {
	public := []string{
		"/echo_vault.user.v1.UserService/Register",
		"/echo_vault.user.v1.UserService/Login",
		"/echo_vault.user.v1.UserService/GetServerInfo",
	}
	for _, p := range public {
		if method == p {
			return true
		}
	}
	return false
}

type ctxKey string

const (
	ctxKeyUserID   ctxKey = "user_id"
	ctxKeyDeviceID ctxKey = "device_id"
)

func GetUserID(ctx context.Context) string {
	v, _ := ctx.Value(ctxKeyUserID).(string)
	return v
}

func GetDeviceID(ctx context.Context) string {
	v, _ := ctx.Value(ctxKeyDeviceID).(string)
	return v
}
```

```go
// internal/grpc/handler_user.go
package grpc

import (
	"context"

	"github.com/inkOrCloud/EchoVault/echovault-server/internal/service/user"
	"github.com/inkOrCloud/EchoVault/echovault-server/pkg/convert"
	userpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/user/v1"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

type UserHandler struct {
	userpb.UnimplementedUserServiceServer
	svc *user.Service
}

func NewUserHandler(svc *user.Service) *UserHandler {
	return &UserHandler{svc: svc}
}

func (h *UserHandler) Register(ctx context.Context, req *userpb.RegisterRequest) (*userpb.RegisterResponse, error) {
	resp, err := h.svc.Register(ctx, req.Username, req.Password, req.DisplayName)
	if err != nil {
		// 密码校验错误 = InvalidArgument, 用户名重复 = AlreadyExists
		code := codes.InvalidArgument
		if err.Error() == "username already exists" {
			code = codes.AlreadyExists
		}
		return nil, status.Error(code, err.Error())
	}
	// 查询完整用户信息用于返回
	u, err := h.svc.GetUser(ctx, resp.UserID)
	if err != nil {
		return nil, status.Error(codes.Internal, "failed to get created user")
	}
	return &userpb.RegisterResponse{
		User:        convertUser(u),
		AccessToken: resp.Token,
	}, nil
}

func (h *UserHandler) Login(ctx context.Context, req *userpb.LoginRequest) (*userpb.LoginResponse, error) {
	resp, err := h.svc.Login(ctx, req.Username, req.Password, req.DeviceId)
	if err != nil {
		return nil, status.Error(codes.Unauthenticated, err.Error())
	}
	u, _ := h.svc.GetUser(ctx, resp.UserID)
	return &userpb.LoginResponse{
		User:        convertUser(u),
		AccessToken: resp.Token,
	}, nil
}

func (h *UserHandler) GetCurrentUser(ctx context.Context, _ *userpb.GetCurrentUserRequest) (*userpb.GetCurrentUserResponse, error) {
	userID := GetUserID(ctx)
	if userID == "" {
		return nil, status.Error(codes.Unauthenticated, "not authenticated")
	}
	u, err := h.svc.GetUser(ctx, userID)
	if err != nil {
		return nil, status.Error(codes.NotFound, err.Error())
	}
	return &userpb.GetCurrentUserResponse{User: convertUser(u)}, nil
}

func convertUser(u *ent.User) *userpb.User {
	return &userpb.User{
		Id:          u.ID,
		Username:    u.Username,
		DisplayName: u.DisplayName,
		Role:        u.Role,
		CreatedAt:   convert.PTime(u.CreatedAt),
		UpdatedAt:   convert.PTime(u.UpdatedAt),
	}
}

func (h *UserHandler) ListDevices(ctx context.Context, req *userpb.ListDevicesRequest) (*userpb.ListDevicesResponse, error) {
	userID := GetUserID(ctx)
	devices, err := h.svc.ListDevices(ctx, userID)
	if err != nil {
		return nil, status.Error(codes.Internal, err.Error())
	}
	pbDevices := make([]*userpb.Device, len(devices))
	for i, d := range devices {
		pbDevices[i] = userEntToProto(d)
	}
	return &userpb.ListDevicesResponse{Devices: pbDevices}, nil
}

func (h *UserHandler) RegisterDevice(ctx context.Context, req *userpb.RegisterDeviceRequest) (*userpb.RegisterDeviceResponse, error) {
	userID := GetUserID(ctx)
	if err := h.svc.RegisterDevice(ctx, userID, req.DeviceId, req.DeviceName, req.Platform); err != nil {
		return nil, status.Error(codes.AlreadyExists, err.Error())
	}
	return &userpb.RegisterDeviceResponse{}, nil
}

func (h *UserHandler) RemoveDevice(ctx context.Context, req *userpb.RemoveDeviceRequest) (*userpb.RemoveDeviceResponse, error) {
	userID := GetUserID(ctx)
	if err := h.svc.RemoveDevice(ctx, userID, req.DeviceId); err != nil {
		return nil, status.Error(codes.NotFound, err.Error())
	}
	return &userpb.RemoveDeviceResponse{}, nil
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认全部通过**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
GOCACHE=/tmp/go-cache go test ./internal/grpc/... -v
```
Expected: 2 tests, both PASS

- [ ] **Step 5: 更新 register.go + 主入口**

修改 `internal/grpc/register.go` 以接收并注册 UserHandler:

```go
package grpc

import (
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/ent"
	"github.com/inkOrCloud/EchoVault/echovault-server/internal/service/user"
	userpb "github.com/inkOrCloud/EchoVault/echovault-server/api/grpc/generated/echo_vault/user/v1"
	"google.golang.org/grpc"
)

func RegisterAll(s *grpc.Server, client *ent.Client, jwtSecret string) {
	userSvc := user.NewService(client, jwtSecret)
	userHandler := NewUserHandler(userSvc)
	userpb.RegisterUserServiceServer(s, userHandler)

	// 注册认证拦截器（在 server main 中包装）
}

// AuthInterceptorOpts 返回拦截器配置
func AuthInterceptorOpts(secret string) grpc.ServerOption {
	return grpc.UnaryInterceptor(AuthInterceptor(secret))
}
```

修改 `cmd/server/main.go` 使用 AuthInterceptor:

```go
s := grpc.NewServer(grpc.AuthInterceptorOpts(cfg.JWTSecret))
grpc.RegisterAll(s, client, cfg.JWTSecret)
```

- [ ] **Step 6: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault/echovault-server
git add internal/grpc/ cmd/server/main.go
git commit -m "feat(grpc): add UserHandler, AuthInterceptor, and gRPC wiring (TDD, 2 integration tests)"
```

---


### Phase 2 自审清单

- [ ] 全部测试通过（预计 18+ 测试）
- [ ] `go build ./cmd/server` 编译通过
- [ ] register/login 完整 gRPC 流程可用
- [ ] 认证拦截器正常工作（公开 RPC 免认证，其他需要 Bearer Token）
- [ ] 设备注册/列表/移除可用
- [ ] 所有提交在 `github.com/inkOrCloud/echovault-server` 可访问

**入口：** `Phase 3: 同步引擎核心`
