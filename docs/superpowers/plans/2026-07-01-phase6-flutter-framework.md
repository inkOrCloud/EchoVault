# Phase 6: Flutter 应用框架 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use subagent-driven-development. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 初始化 Flutter 项目，搭建 gRPC-Web 客户端、drift 本地数据库、认证模块

**Architecture:** Flutter 全平台（Mobile + Desktop + Web），Riverpod 状态管理，grpc-dart 连接 Envoy（:8080），drift 离线 SQLite，dio 传输大文件。

**Tech Stack:** Flutter 3.x, Dart 3.x, Riverpod, grpc-dart, drift, dio, go_router

---

### Task 1: Flutter 项目初始化

**Files:** `echo_vault_app/` 全部文件由 `flutter create` 生成

- [ ] **Step 1: 创建 Flutter 项目**

```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter create --org com.inkorcloud --project-name echo_vault_app --platforms android,ios,windows,macos,linux,web .
```

- [ ] **Step 2: 配置 pubspec.yaml**

```yaml
name: echo_vault_app
description: EchoVault - 音匣音乐管理客户端

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.6.1
  grpc: ^4.0.1
  protobuf: ^3.1.0
  dio: ^5.7.0
  go_router: ^14.6.2
  drift: ^2.22.1
  sqlite3_flutter_libs: ^0.5.28
  path_provider: ^2.1.5
  path: ^1.9.1
  just_audio: ^0.9.42
  shared_preferences: ^2.3.4
  google_sign_in: ^6.2.2  # 可选

dev_dependencies:
  flutter_test:
    sdk: flutter
  drift_dev: ^2.22.1
  build_runner: ^2.4.14
  mocktail: ^1.0.4
```

- [ ] **Step 3: 运行 flutter pub get**

```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter pub get
```

- [ ] **Step 4: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/
git commit -m "feat(flutter): initialize Flutter project with dependencies"
```

---

### Task 2: 服务器地址配置

**核心逻辑：** 首次启动显示配置页，用户输入后端地址。地址存入 SharedPreferences，后续启动自动读取。Web 端默认使用同源地址。

**Files:**
- Create: `echo_vault_app/lib/core/config/server_config.dart`
- Create: `echo_vault_app/lib/features/setup/server_setup_page.dart`

- [ ] **Step 1: 实现 ServerConfig 存储**

```dart
// lib/core/config/server_config.dart
import 'package:flutter/foundation.dart';
import 'package:shared_preferences/shared_preferences.dart';

class ServerConfig {
  static const _keyGrpcUrl = 'grpc_url';
  static const _keyRestUrl = 'rest_url';

  /// 默认 gRPC 端口：Native 直连 9090，Web 走 Envoy 8080
  static int get defaultGrpcPort => kIsWeb ? 8080 : 9090;
  /// 默认 REST 端口：有 Envoy 时 8080，直连时 9091
  static int get defaultRestPort => kIsWeb ? 8080 : 9091;

  /// 返回解析后的 gRPC 和 REST 地址
  static Future<({String host, int grpcPort, int restPort})> getConfig() async {
    final prefs = await SharedPreferences.getInstance();
    final grpcUrl = prefs.getString(_keyGrpcUrl) ?? '';
    final restUrl = prefs.getString(_keyRestUrl) ?? '';

    if (grpcUrl.isEmpty && restUrl.isEmpty) {
      return (host: '', grpcPort: defaultGrpcPort, restPort: defaultRestPort);
    }

    // 解析 gRPC 地址
    final grpcUri = Uri.parse(grpcUrl);
    // 解析 REST 地址（如果未单独设置 REST，则复用 gRPC 的 host/port）
    final restUri = restUrl.isNotEmpty ? Uri.parse(restUrl) : grpcUri;

    // 如果只有 host 没有端口，用默认端口
    return (
      host: grpcUri.host,
      grpcPort: grpcUri.port > 0 ? grpcUri.port : defaultGrpcPort,
      restPort: restUri.port > 0 ? restUri.port : defaultRestPort,
    );
  }

  static Future<void> save({
    required String host,
    int grpcPort = 9090,
    int restPort = 9091,
  }) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(_keyGrpcUrl, 'http://$host:$grpcPort');
    await prefs.setString(_keyRestUrl, 'http://$host:$restPort');
  }

  static Future<bool> isConfigured() async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.containsKey(_keyGrpcUrl);
  }
}
```

- [ ] **Step 2: 实现服务器配置页面**

```dart
// lib/features/setup/server_setup_page.dart
class ServerSetupPage extends ConsumerStatefulWidget {
  @override
  ConsumerState<ServerSetupPage> createState() => _ServerSetupPageState();
}

class _ServerSetupPageState extends ConsumerState<ServerSetupPage> {
  final _urlController = TextEditingController();
  final _grpcPortController = TextEditingController(text: '9090');
  final _restPortController = TextEditingController(text: '9091');
  bool _useSamePort = true;  // 有反代时勾选，统一端口
  String? _error;

  Future<void> _connect() async {
    final host = _urlController.text.trim();
    if (host.isEmpty) {
      setState(() => _error = '请输入服务器地址');
      return;
    }
    final grpcPort = int.tryParse(_grpcPortController.text.trim()) ?? 9090;
    final restPort = _useSamePort ? grpcPort : (int.tryParse(_restPortController.text.trim()) ?? 9091);
    try {
      final manager = ref.read(grpcClientProvider);
      await manager.connect(host: host, grpcPort: grpcPort, webPort: grpcPort);
      await ServerConfig.save(host: host, grpcPort: grpcPort, restPort: restPort);
      ref.read(serverConfigProvider.notifier).state = (
        host: host, grpcPort: grpcPort, restPort: restPort,
      );
      if (mounted) context.go('/login');
    } catch (e) {
      setState(() => _error = '连接失败: $e');
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('连接服务器')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Text('输入 EchoVault 服务器地址'),
            const SizedBox(height: 24),
            TextField(
              controller: _urlController,
              decoration: const InputDecoration(
                labelText: '服务器地址',
                hintText: '192.168.1.100 或 music.example.com',
                prefixIcon: Icon(Icons.dns),
              ),
            ),
            const SizedBox(height: 16),
            TextField(
              controller: _portController,
              decoration: const InputDecoration(
                labelText: '端口',
                hintText: '8080',
                prefixIcon: Icon(Icons.tag),
              ),
              keyboardType: TextInputType.number,
            ),
            if (_error != null) ...[
              const SizedBox(height: 16),
              Text(_error!, style: const TextStyle(color: Colors.red)),
            ],
            const SizedBox(height: 24),
            ElevatedButton(
              onPressed: _connect,
              child: const Text('连接'),
            ),
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 3: 添加 Riverpod Provider**

```dart
// lib/providers/server_provider.dart
final serverConfigProvider = StateProvider<({String host, int grpcPort, int restPort})?>((ref) => null);
```

- [ ] **Step 4: 更新路由 — 未配置时跳转设置页**

在 router.dart 的 redirect 中添加：
```dart
final configured = ref.read(serverConfigProvider) != null;
if (!configured && state.matchedLocation != '/setup') return '/setup';
```

- [ ] **Step 5: 提交**

---

### Task 3: gRPC-Web 客户端封装

**核心逻辑：** 封装 grpc-dart `ClientChannel`，通过 Envoy（localhost:8080）连接后端，自动附加 JWT Token。

**Files:**
- Create: `echo_vault_app/lib/core/grpc/client.dart`
- Create: `echo_vault_app/lib/core/grpc/auth_interceptor.dart`
- Create: `echo_vault_app/lib/core/grpc/grpc_provider.dart` (Riverpod provider)

- [ ] **Step 1: 实现 gRPC 客户端管理器**

```dart
// lib/core/grpc/client.dart
import 'package:flutter/foundation.dart' show kIsWeb;
import 'package:grpc/grpc.dart' as grpc;
import 'package:grpc/grpc_web.dart' as grpc_web;

class GrpcClientManager {
  late final grpc.ClientChannelBase _channel;
  String? _accessToken;

  Future<void> connect({
    required String host,
    int grpcPort = 9090,   // Native 直连 gRPC
    int webPort = 8080,    // Web 走 Envoy gRPC-Web
  }) async {
    if (kIsWeb) {
      // Web: gRPC-Web, 走 Envoy
      _channel = grpc_web.GrpcWebClientChannel(
        Uri.parse('http://$host:$webPort'),
      );
    } else {
      // Native: HTTP/2 直连 Go gRPC server
      _channel = grpc.ClientChannel(
        host,
        port: grpcPort,
        options: const grpc.ChannelOptions(
          connectionTimeout: Duration(seconds: 10),
        ),
      );
    }
  }

  Map<String, String> get metadata {
    if (_accessToken != null && _accessToken!.isNotEmpty) {
      return {'authorization': 'Bearer $_accessToken'};
    }
    return {};
  }

  void setToken(String token) => _accessToken = token;
  void clearToken() => _accessToken = null;
  Future<void> disconnect() async => _channel.shutdown();
  grpc.ClientChannelBase get channel => _channel;
}
```

- [ ] **Step 2: 创建全部 5 个 gRPC Service 的 Provider**

```dart
// lib/core/grpc/grpc_provider.dart
import 'package:grpc/grpc.dart';
import 'package:riverpod/riverpod.dart';

import 'client.dart';
import '../../../proto/echo_vault/user/v1/user_service.pbgrpc.dart';
import '../../../proto/echo_vault/song/v1/song_service.pbgrpc.dart';
// ... etc

final grpcClientProvider = Provider<GrpcClientManager>((ref) {
  return GrpcClientManager();
});

final userServiceProvider = Provider<UserServiceClient>((ref) {
  final manager = ref.watch(grpcClientProvider);
  return UserServiceClient(manager.channel);
});

final songServiceProvider = Provider<SongServiceClient>((ref) {
  final manager = ref.watch(grpcClientProvider);
  return SongServiceClient(manager.channel);
});
// ... sync, lyric, playlist
```

- [ ] **Step 3: 提交**

---

### Task 4: drift 本地数据库

**核心逻辑：** 离线优先架构的本地数据层。表结构与服务端一致，增加 sync_status 等本地字段。

**Files:**
- Create: `echo_vault_app/lib/core/db/database.dart`
- Create: `echo_vault_app/lib/core/db/tables.dart`
- Create: `echo_vault_app/lib/core/db/dao/`

- [ ] **Step 1: 定义 drift 表结构**

```dart
// lib/core/db/tables.dart
import 'package:drift/drift.dart';

class Songs extends Table {
  TextColumn get id => text()();
  TextColumn get title => text()();
  TextColumn get artist => text().nullable()();
  TextColumn get album => text().nullable()();
  TextColumn get fileHash => text().nullable()();
  IntColumn get durationMs => integer().nullable()();
  IntColumn get fileSize => integer().nullable()();
  TextColumn get syncStatus => text().withDefault(const Constant('pending'))();
  TextColumn get localPath => text().nullable()();
  BoolColumn get isDownloaded => boolean().withDefault(const Constant(false))();
  IntColumn get playCount => integer().withDefault(const Constant(0))();
  
  @override
  Set<Column> get primaryKey => {id};
}
```

- [ ] **Step 2: 初始化 Database**

```dart
// lib/core/db/database.dart
import 'package:drift/native.dart';
import 'package:drift/drift.dart';
import 'package:path_provider/path_provider.dart';
import 'package:path/path.dart' as p;
import 'tables.dart';

part 'database.g.dart';

@DriftDatabase(tables: [Songs, Playlists, PlaylistSongs, Lyrics, SyncLogs])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());

  @override
  int get schemaVersion => 1;
}

LazyDatabase _openConnection() {
  return LazyDatabase(() async {
    final dbFolder = await getApplicationDocumentsDirectory();
    final file = File(p.join(dbFolder.path, 'echovault.db'));
    return NativeDatabase.createInBackground(file);
  });
}
```

- [ ] **Step 3: 运行 build_runner 生成代码**

```bash
cd echo_vault_app
flutter pub run build_runner build
```

- [ ] **Step 4: 提交**

---

### Task 5: 认证模块

**核心逻辑：** 登录/注册页面，Token 本地持久化（SharedPreferences），自动登录。

**Files:**
- Create: `echo_vault_app/lib/services/auth_service.dart`
- Create: `echo_vault_app/lib/providers/auth_provider.dart`
- Create: `echo_vault_app/lib/features/auth/login_page.dart`
- Create: `echo_vault_app/lib/features/auth/register_page.dart`
- Create: `echo_vault_app/lib/app.dart`

- [ ] **Step 1: RED — 编写 AuthService 测试**

```dart
// test/services/auth_service_test.dart
void main() {
  test('login with valid credentials', () async {
    final service = AuthService();
    // 需要 mock gRPC client
  });
}
```

- [ ] **Step 2: GREEN — 实现 AuthService**

```dart
// lib/services/auth_service.dart
class AuthService {
  final UserServiceClient _userClient;

  AuthService(this._userClient);

  Future<AuthResult> login(String username, String password, String deviceId) async {
    final resp = await _userClient.login(LoginRequest(
      username: username, password: password, deviceId: deviceId,
    ));
    return AuthResult(resp.user, resp.accessToken);
  }

  Future<AuthResult> register(String username, String password, String displayName) async {
    final resp = await _userClient.register(RegisterRequest(
      username: username, password: password, displayName: displayName,
    ));
    return AuthResult(resp.user, resp.accessToken);
  }
}
```

- [ ] **Step 3: RED — 编写登录页面 widget test**
- [ ] **Step 4: GREEN — 实现登录/注册页面**
- [ ] **Step 5: 提交**

---

### Task 6: 应用入口 + 路由

**Files:**
- Modify: `echo_vault_app/lib/main.dart`
- Create: `echo_vault_app/lib/app.dart`
- Create: `echo_vault_app/lib/router.dart`

- [ ] **Step 1: 配置 go_router**

```dart
// lib/router.dart
final router = GoRouter(
  initialLocation: '/login',
  redirect: (context, state) {
    final isLoggedIn = ref.read(authProvider).isLoggedIn;
    if (!isLoggedIn && state.matchedLocation != '/login' && state.matchedLocation != '/register') {
      return '/login';
    }
    return null;
  },
  routes: [
    GoRoute(path: '/login', builder: (_, __) => const LoginPage()),
    GoRoute(path: '/register', builder: (_, __) => const RegisterPage()),
    ShellRoute(
      builder: (_, __, child) => MainShell(child: child),
      routes: [
        GoRoute(path: '/library', builder: (_, __) => const LibraryPage()),
        GoRoute(path: '/playlists', builder: (_, __) => const PlaylistsPage()),
        GoRoute(path: '/settings', builder: (_, __) => const SettingsPage()),
      ],
    ),
  ],
);
```

- [ ] **Step 2: 实现 main.dart + ProviderScope**

```dart
// lib/main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final grpcManager = GrpcClientManager();
  await grpcManager.connect(host: 'localhost', port: 8080);

  runApp(
    ProviderScope(
      overrides: [grpcClientProvider.overrideWithValue(grpcManager)],
      child: const EchoVaultApp(),
    ),
  );
}
```

- [ ] **Step 3: 提交**

---

### Phase 6 自审清单

- [ ] Flutter 项目编译通过 (`flutter build apk --debug`)
- [ ] gRPC-Web 客户端可连接 Envoy
- [ ] drift 数据库表创建成功
- [ ] 登录/注册页面可访问
- [ ] Token 持久化 + 自动登录
- [ ] 路由守卫正常工作（未登录 → /login）
- [ ] 所有提交推送到 GitHub

**入口：** `Phase 7: 歌曲库 + 扫描器 UI`
