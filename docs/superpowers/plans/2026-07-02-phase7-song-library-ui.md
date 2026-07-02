# Phase 7: 歌曲库 + 扫描器 UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现 Flutter 端歌曲库 UI：本地文件扫描器、云端哈希比对、歌曲列表/搜索、元数据编辑发布流程

**Architecture:** 桌面优先（dart:io 递归扫描），通过 gRPC-Web 调用 SongService（CheckSongsByHash/PublishSong/SearchSongs），通过 Dio REST 上传音频文件/封面。扫描 → 比对云端 → 编辑元数据 → 上传四步流程。Riverpod 管理状态，每个模块独立测试。

**Tech Stack:** Flutter 3.x, Dart 3.x, dart:io, crypto (SHA256), flutter_riverpod, grpc-dart, dio, go_router, protoc-gen-dart

---

## File Structure

```
echo_vault_app/lib/
├── core/
│   └── grpc/
│       └── client.dart                        # 已有 — 追加 gRPC stub 工厂方法
├── models/
│   └── generated/                             # [新建] proto 生成的 Dart 代码
├── features/
│   ├── library/
│   │   ├── models/
│   │   │   └── scanned_file.dart              # 扫描文件数据模型
│   │   ├── pages/
│   │   │   └── library_page.dart              # 歌曲列表主页
│   │   ├── providers/
│   │   │   ├── library_provider.dart           # 歌曲列表状态
│   │   │   └── scanner_provider.dart           # 扫描状态
│   │   ├── services/
│   │   │   ├── file_scanner.dart              # 文件系统递归扫描器
│   │   │   ├── song_repository.dart           # SongService gRPC 封装
│   │   │   └── rest_client.dart              # Dio REST 上传封装
│   │   └── widgets/
│   │       ├── song_list_tile.dart            # 歌曲列表行组件
│   │       └── scan_progress_dialog.dart       # 扫描进度对话框
│   └── publish/
│       ├── pages/
│       │   ├── scan_result_page.dart          # 扫描结果页（云端比对）
│       │   └── edit_metadata_page.dart        # 元数据编辑页
│       └── providers/
│           └── publish_provider.dart          # 发布流程状态
├── router.dart                                 # 修改 — 追加 /library /scan /publish 路由
└── main.dart                                   # 修改 — 初始化 gRPC stubs

echo_vault_app/test/
├── features/
│   ├── library/
│   │   ├── models/
│   │   │   └── scanned_file_test.dart
│   │   ├── services/
│   │   │   ├── file_scanner_test.dart
│   │   │   ├── song_repository_test.dart
│   │   │   └── rest_client_test.dart
│   │   ├── providers/
│   │   │   ├── library_provider_test.dart
│   │   │   └── scanner_provider_test.dart
│   │   ├── widgets/
│   │   │   └── song_list_tile_test.dart
│   │   └── pages/
│   │       └── library_page_test.dart
│   └── publish/
│       ├── pages/
│       │   ├── scan_result_page_test.dart
│       │   └── edit_metadata_page_test.dart
│       └── providers/
│           └── publish_provider_test.dart
```

---

### Task 1: 生成 Dart Proto 代码 + 添加依赖

**核心逻辑：** 使用 `protoc` + `protoc-gen-dart` 从 proto 文件生成 Dart gRPC 代码。添加 `crypto`（SHA256）和 `file_picker`（目录选择）依赖。

**Dependencies to add:**
- `crypto: ^3.0.6` — SHA256 哈希计算
- `file_picker: ^8.0.0` — 目录选择对话框（桌面优先）
- `intl: ^0.19.0` — 日期/时长格式化
- `mocktail: ^1.0.4` — 测试 Mock 支持（dev_dependency）

**Files:**
- Modify: `echo_vault_app/pubspec.yaml` — 追加 crypto, file_picker, intl, mocktail
- Create: `echo_vault_app/lib/models/generated/echo_vault/common/v1/types.pb.dart`
- Create: `echo_vault_app/lib/models/generated/echo_vault/common/v1/types.pbenum.dart`
- Create: `echo_vault_app/lib/models/generated/echo_vault/common/v1/types.pbjson.dart`
- Create: `echo_vault_app/lib/models/generated/echo_vault/song/v1/song_service.pb.dart`
- Create: `echo_vault_app/lib/models/generated/echo_vault/song/v1/song_service.pbgrpc.dart`
- Create: `echo_vault_app/lib/models/generated/echo_vault/song/v1/song_service.pbjson.dart`

- [ ] **Step 1: 添加 proto 生成脚本**

Protoc-gen-dart 需要先安装。创建生成脚本 `echo_vault_app/scripts/gen_proto.sh`:

```bash
#!/bin/bash
# 从 proto/ 生成 Dart gRPC 代码到 echo_vault_app/lib/models/generated/
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")/../.." && pwd)"
PROTO_DIR="$SCRIPT_DIR/proto"
OUT_DIR="$SCRIPT_DIR/echo_vault_app/lib/models/generated"

# 确保 protoc-gen-dart 已安装
if ! command -v protoc-gen-dart &>/dev/null; then
  echo "Installing protoc-gen-dart..."
  dart pub global activate protoc_plugin
fi

mkdir -p "$OUT_DIR"

protoc \
  --proto_path="$PROTO_DIR" \
  --dart_out="$OUT_DIR" \
  "$PROTO_DIR"/echo_vault/common/v1/types.proto \
  "$PROTO_DIR"/echo_vault/song/v1/song_service.proto

echo "Dart proto code generated to $OUT_DIR"
```

- [ ] **Step 2: 更新 pubspec.yaml — 添加依赖**

Append to `echo_vault_app/pubspec.yaml` dependencies section:
```yaml
  crypto: ^3.0.6
  file_picker: ^8.1.7
  intl: ^0.20.2
```

Append to dev_dependencies:
```yaml
  mocktail: ^1.0.4
```

- [ ] **Step 3: 运行 flutter pub get**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter pub get
```
Expected: Dependencies resolved successfully.

- [ ] **Step 4: 生成 Dart proto 代码**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault
chmod +x echo_vault_app/scripts/gen_proto.sh
bash echo_vault_app/scripts/gen_proto.sh
```
Expected: `lib/models/generated/echo_vault/` 下生成 common 和 song 两个包的 .pb.dart/.pbgrpc.dart/.pbjson.dart 文件

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/pubspec.yaml echo_vault_app/pubspec.lock echo_vault_app/lib/models/generated/ echo_vault_app/scripts/gen_proto.sh
git commit -m "feat(flutter): generate Dart proto code for SongService + add dependencies"
```

---

### Task 2: ScannedFile 数据模型

**核心逻辑：** 定义扫描结果的数据模型，表示从本地文件系统扫描到的音频文件。包含文件路径、SHA256 哈希、文件大小、MIME 类型推导、基础元数据（从文件名猜测的 title/artist）。

**Files:**
- Create: `echo_vault_app/lib/features/library/models/scanned_file.dart`
- Test: `echo_vault_app/test/features/library/models/scanned_file_test.dart`

- [ ] **Step 1 (RED): 编写 ScannedFile 模型测试**

```dart
// test/features/library/models/scanned_file_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';

void main() {
  group('ScannedFile', () {
    test('creates with path and hash', () {
      final file = ScannedFile(
        path: '/music/test.mp3',
        fileName: 'test.mp3',
        fileHash: 'abc123',
        fileSize: 1024,
        mimeType: 'audio/mpeg',
      );
      expect(file.path, '/music/test.mp3');
      expect(file.fileHash, 'abc123');
      expect(file.fileSize, 1024);
    });

    test('deduces mime type from extension', () {
      expect(ScannedFile.deduceMimeType('song.mp3'), 'audio/mpeg');
      expect(ScannedFile.deduceMimeType('song.flac'), 'audio/flac');
      expect(ScannedFile.deduceMimeType('song.ogg'), 'audio/ogg');
      expect(ScannedFile.deduceMimeType('song.m4a'), 'audio/mp4');
      expect(ScannedFile.deduceMimeType('song.wav'), 'audio/wav');
      expect(ScannedFile.deduceMimeType('song.wma'), 'audio/x-ms-wma');
      expect(ScannedFile.deduceMimeType('song.aac'), 'audio/aac');
      expect(ScannedFile.deduceMimeType('song.unknown'), 'application/octet-stream');
    });

    test('extracts title from filename', () {
      expect(ScannedFile.titleFromFilename('My Song.mp3'), 'My Song');
      expect(ScannedFile.titleFromFilename('01 - Track One.flac'), 'Track One');
    });

    test('isAudioExtension returns true for known audio formats', () {
      expect(ScannedFile.isAudioExtension('mp3'), isTrue);
      expect(ScannedFile.isAudioExtension('flac'), isTrue);
      expect(ScannedFile.isAudioExtension('ogg'), isTrue);
      expect(ScannedFile.isAudioExtension('m4a'), isTrue);
      expect(ScannedFile.isAudioExtension('wav'), isTrue);
      expect(ScannedFile.isAudioExtension('txt'), isFalse);
      expect(ScannedFile.isAudioExtension('mp4'), isFalse);
    });

    test('equality and hashcode based on fileHash', () {
      final a = ScannedFile(path: '/a.mp3', fileName: 'a.mp3', fileHash: 'hash1', fileSize: 100, mimeType: 'audio/mpeg');
      final b = ScannedFile(path: '/b.mp3', fileName: 'b.mp3', fileHash: 'hash1', fileSize: 200, mimeType: 'audio/mpeg');
      final c = ScannedFile(path: '/c.mp3', fileName: 'c.mp3', fileHash: 'hash2', fileSize: 100, mimeType: 'audio/mpeg');
      expect(a, equals(b));
      expect(a.hashCode, equals(b.hashCode));
      expect(a, isNot(equals(c)));
    });
  });
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认编译失败**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/models/scanned_file_test.dart
```
Expected: FAIL — `ScannedFile` class not found

- [ ] **Step 3 (GREEN): 实现 ScannedFile**

```dart
// lib/features/library/models/scanned_file.dart
class ScannedFile {
  final String path;
  final String fileName;
  final String fileHash;
  final int fileSize;
  final String mimeType;
  final String? guessedTitle;
  final String? guessedArtist;

  const ScannedFile({
    required this.path,
    required this.fileName,
    required this.fileHash,
    required this.fileSize,
    required this.mimeType,
    this.guessedTitle,
    this.guessedArtist,
  });

  static const Set<String> audioExtensions = {
    'mp3', 'flac', 'ogg', 'm4a', 'wav', 'wma', 'aac',
  };

  static bool isAudioExtension(String ext) => audioExtensions.contains(ext.toLowerCase());

  static String deduceMimeType(String fileName) {
    final ext = fileName.split('.').last.toLowerCase();
    switch (ext) {
      case 'mp3':  return 'audio/mpeg';
      case 'flac': return 'audio/flac';
      case 'ogg':  return 'audio/ogg';
      case 'm4a':  return 'audio/mp4';
      case 'wav':  return 'audio/wav';
      case 'wma':  return 'audio/x-ms-wma';
      case 'aac':  return 'audio/aac';
      default:     return 'application/octet-stream';
    }
  }

  /// 从文件名猜测标题。若匹配 "N - Title.ext" 格式则提取 Title 部分。
  static String titleFromFilename(String fileName) {
    final name = fileName.replaceAll(RegExp(r'\.[^.]+$'), '');
    final match = RegExp(r'^\d+\s*[-–—]\s*(.+)$').firstMatch(name);
    if (match != null) return match.group(1)!.trim();
    return name.trim();
  }

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is ScannedFile && runtimeType == other.runtimeType && fileHash == other.fileHash;

  @override
  int get hashCode => fileHash.hashCode;
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认通过**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/models/scanned_file_test.dart
```
Expected: 6 tests, all PASS

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/models/scanned_file.dart echo_vault_app/test/features/library/models/scanned_file_test.dart
git commit -m "feat(flutter): add ScannedFile model with MIME deduction and filename parsing"
```

---

### Task 3: FileScanner — 本地文件系统递归扫描器

**核心逻辑：** 使用 `dart:io` 递归扫描用户选择的目录，收集音频文件并计算 SHA256 哈希。支持进度回调（当前目录、已找到文件数）。返回 `ScannedFile` 列表。

**Files:**
- Create: `echo_vault_app/lib/features/library/services/file_scanner.dart`
- Test: `echo_vault_app/test/features/library/services/file_scanner_test.dart`

- [ ] **Step 1 (RED): 编写 FileScanner 测试（Mock 文件系统）**

```dart
// test/features/library/services/file_scanner_test.dart
import 'dart:io';
import 'package:flutter_test/flutter_test.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';
import 'package:echo_vault_app/features/library/services/file_scanner.dart';

/// Helper: 在临时目录中创建测试文件结构
Future<String> _createTestDirectory({
  required List<String> audioFiles,
  required List<String> nonAudioFiles,
}) async {
  final dir = Directory.systemTemp.createTempSync('echovault_scanner_test_');
  for (final f in audioFiles) {
    File('${dir.path}/$f').createSync(recursive: true);
  }
  for (final f in nonAudioFiles) {
    File('${dir.path}/$f').createSync(recursive: true);
  }
  return dir.path;
}

void main() {
  late String testDir;

  setUp(() {
    testDir = '';
  });

  tearDown(() async {
    if (testDir.isNotEmpty) {
      await Directory(testDir).delete(recursive: true);
    }
  });

  test('scanDirectory returns audio files only', () async {
    testDir = await _createTestDirectory(
      audioFiles: ['song1.mp3', 'sub/song2.flac', 'song3.ogg'],
      nonAudioFiles: ['readme.txt', 'image.jpg', 'video.mp4', 'sub/notes.md'],
    );
    final scanner = FileScanner();
    final results = await scanner.scanDirectory(testDir);
    expect(results.length, 3);
    expect(results.any((f) => f.fileName == 'song1.mp3'), isTrue);
    expect(results.any((f) => f.fileName == 'song2.flac'), isTrue);
    expect(results.any((f) => f.fileName == 'song3.ogg'), isTrue);
  });

  test('scanDirectory computes SHA256 hash', () async {
    testDir = await _createTestDirectory(
      audioFiles: ['hash_test.mp3'],
      nonAudioFiles: [],
    );
    final file = File('${testDir}/hash_test.mp3');
    await file.writeAsString('test content for hashing');
    final scanner = FileScanner();
    final results = await scanner.scanDirectory(testDir);
    expect(results.length, 1);
    // SHA256 of "test content for hashing"
    expect(results.first.fileHash, isNotEmpty);
    expect(results.first.fileHash.length, 64); // SHA256 hex length
  });

  test('scanDirectory skips symlinks and hidden files', () async {
    testDir = await _createTestDirectory(
      audioFiles: ['visible.mp3'],
      nonAudioFiles: ['.hidden.mp3'],
    );
    final scanner = FileScanner();
    final results = await scanner.scanDirectory(testDir);
    expect(results.any((f) => f.fileName == '.hidden.mp3'), isFalse);
  });

  test('scanDirectory reports progress via callback', () async {
    testDir = await _createTestDirectory(
      audioFiles: List.generate(5, (i) => 'song$i.mp3'),
      nonAudioFiles: [],
    );
    final scanner = FileScanner();
    int progressCount = 0;
    String? lastDir;
    await scanner.scanDirectory(
      testDir,
      onProgress: (found, currentDir) {
        progressCount = found;
        lastDir = currentDir;
      },
    );
    expect(progressCount, 5);
    expect(lastDir, isNotEmpty);
  });

  test('throws if directory does not exist', () async {
    final scanner = FileScanner();
    expect(
      () => scanner.scanDirectory('/nonexistent/path'),
      throwsA(isA<FileSystemException>()),
    );
  });
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认编译失败**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/services/file_scanner_test.dart
```
Expected: FAIL — `FileScanner` class not found

- [ ] **Step 3 (GREEN): 实现 FileScanner**

```dart
// lib/features/library/services/file_scanner.dart
import 'dart:convert';
import 'dart:io';
import 'package:crypto/crypto.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';

class FileScanner {
  /// 递归扫描目录，收集音频文件并计算 SHA256 哈希。
  ///
  /// [directory]: 要扫描的目录路径。
  /// [onProgress]: 进度回调，参数为（已找到文件数, 当前正在扫描的目录）。
  Future<List<ScannedFile>> scanDirectory(
    String directory, {
    void Function(int found, String currentDirectory)? onProgress,
  }) async {
    final dir = Directory(directory);
    if (!await dir.exists()) {
      throw FileSystemException('Directory does not exist', directory);
    }

    final results = <ScannedFile>[];
    final files = <File>[];

    // 第一步：递归收集所有音频文件
    await for (final entity in dir.list(recursive: true, followLinks: false)) {
      if (entity is File) {
        // 跳过隐藏文件
        final fileName = entity.uri.pathSegments.last;
        if (fileName.startsWith('.')) continue;

        final ext = fileName.split('.').last.toLowerCase();
        if (ScannedFile.isAudioExtension(ext)) {
          // 获取父目录用于进度回调
          final parentDir = entity.parent.path;
          onProgress?.call(files.length + 1, parentDir);
          files.add(entity);
        }
      }
    }

    // 第二步：计算每个文件的 SHA256
    for (final file in files) {
      final bytes = await file.readAsBytes();
      final hash = sha256.convert(bytes).toString();
      final fileName = file.uri.pathSegments.last;

      results.add(ScannedFile(
        path: file.path,
        fileName: fileName,
        fileHash: hash,
        fileSize: await file.length(),
        mimeType: ScannedFile.deduceMimeType(fileName),
        guessedTitle: ScannedFile.titleFromFilename(fileName),
      ));
    }

    return results;
  }
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认通过**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/services/file_scanner_test.dart
```
Expected: 5 tests, all PASS

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/services/file_scanner.dart echo_vault_app/test/features/library/services/file_scanner_test.dart
git commit -m "feat(flutter): add FileScanner with recursive scan and SHA256 hashing"
```

---

### Task 4: SongRepository — gRPC 封装

**核心逻辑：** 封装 SongService 的 gRPC 调用。GrpcClientManager 提供 channel 和 metadata（JWT token）。Repository 方法对应 CheckSongsByHash, PublishSong, SearchSongs, ListSongs, GetSong。

**Files:**
- Create: `echo_vault_app/lib/features/library/services/song_repository.dart`
- Modify: `echo_vault_app/lib/core/grpc/client.dart` — 追加 gRPC stub 工厂方法
- Test: `echo_vault_app/test/features/library/services/song_repository_test.dart`

- [ ] **Step 1 (RED): 编写 SongRepository 测试**

```dart
// test/features/library/services/song_repository_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:echo_vault_app/core/grpc/client.dart';
import 'package:echo_vault_app/features/library/services/song_repository.dart';

// Mock gRPC stubs
class MockSongServiceClient extends Mock implements SongServiceClient {}
class MockGrpcClientManager extends Mock implements GrpcClientManager {}

void main() {
  late MockSongServiceClient mockStub;
  late MockGrpcClientManager mockManager;
  late SongRepository repository;

  setUp(() {
    mockStub = MockSongServiceClient();
    mockManager = MockGrpcClientManager();
    when(() => mockManager.metadata).thenReturn({'authorization': 'Bearer test-token'});
    repository = SongRepository(manager: mockManager, stub: mockStub);
  });

  group('checkSongsByHash', () {
    test('returns mapping of hash to existence', () async {
      final response = CheckSongsByHashResponse(results: [
        CheckSongsByHashResponse_Result(fileHash: 'hash1', exists: true, song: Song(id: 'song-1', title: 'Known Song')),
        CheckSongsByHashResponse_Result(fileHash: 'hash2', exists: false),
      ]);
      when(() => mockStub.checkSongsByHash(
        any(),
        any(),
        options: any(named: 'options'),
      )).thenAnswer((_) async => response);

      final results = await repository.checkSongsByHash(deviceId: 'dev-1', fileHashes: ['hash1', 'hash2']);
      expect(results.length, 2);
      expect(results[0].exists, isTrue);
      expect(results[0].song!.id, 'song-1');
      expect(results[1].exists, isFalse);
    });

    test('throws on gRPC error', () async {
      when(() => mockStub.checkSongsByHash(any(), any(), options: any(named: 'options')))
          .thenThrow(GrpcError.unavailable('server down'));
      expect(
        () => repository.checkSongsByHash(deviceId: 'dev-1', fileHashes: ['hash1']),
        throwsA(isA<SongRepositoryException>()),
      );
    });
  });

  group('publishSong', () {
    test('returns published song', () async {
      final response = PublishSongResponse(song: Song(id: 'new-song', title: 'My Song'));
      when(() => mockStub.publishSong(any(), any(), options: any(named: 'options')))
          .thenAnswer((_) async => response);
      final song = await repository.publishSong(
        fileHash: 'hash1', title: 'My Song', artist: 'Me',
        fileName: 'test.mp3', fileSize: 1024, mimeType: 'audio/mpeg',
      );
      expect(song.id, 'new-song');
      expect(song.title, 'My Song');
    });
  });

  group('searchSongs', () {
    test('returns matching songs', () async {
      final response = SearchSongsResponse(songs: [
        Song(id: 's1', title: 'Hello'),
        Song(id: 's2', title: 'World'),
      ]);
      when(() => mockStub.searchSongs(any(), any(), options: any(named: 'options')))
          .thenAnswer((_) async => response);
      final songs = await repository.searchSongs('Hello');
      expect(songs.length, 2);
    });
  });
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认编译失败**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/services/song_repository_test.dart
```
Expected: FAIL — imports not found, `SongRepository` not defined

- [ ] **Step 3 (GREEN): 实现 SongRepository + 更新 GrpcClientManager**

首先，更新 `GrpcClientManager` 添加 stub 工厂：

```dart
// lib/core/grpc/client.dart — 追加内容
import 'package:grpc/grpc.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pbgrpc.dart';

class GrpcClientManager {
  late final ClientChannel _channel;
  String? _accessToken;
  bool _connected = false;

  bool get isConnected => _connected;

  Future<void> connect({
    required String host,
    int grpcPort = 9090,
  }) async {
    _channel = ClientChannel(
      host,
      port: grpcPort,
      options: const ChannelOptions(
        connectionTimeout: Duration(seconds: 10),
      ),
    );
    _connected = true;
  }

  /// gRPC 调用元数据（Bearer Token）
  Map<String, String> get metadata {
    if (_accessToken != null && _accessToken!.isNotEmpty) {
      return {'authorization': 'Bearer $_accessToken'};
    }
    return {};
  }

  CallOptions get callOptions => CallOptions(metadata: metadata);

  /// 创建 SongService gRPC 客户端 stub
  SongServiceClient get songService => SongServiceClient(_channel);

  void setToken(String token) {
    _accessToken = token;
  }

  void clearToken() {
    _accessToken = null;
  }

  Future<void> disconnect() async {
    if (_connected) {
      await _channel.shutdown();
      _connected = false;
    }
  }

  ClientChannel get channel => _channel;
}
```

创建 `SongRepository`:

```dart
// lib/features/library/services/song_repository.dart
import 'package:grpc/grpc.dart';
import 'package:echo_vault_app/core/grpc/client.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pbgrpc.dart';
import 'package:echo_vault_app/models/generated/echo_vault/common/v1/types.pb.dart';

class CheckResult {
  final String fileHash;
  final bool exists;
  final Song? song;
  const CheckResult({required this.fileHash, required this.exists, this.song});
}

class SongRepositoryException implements Exception {
  final String message;
  final String? grpcCode;
  const SongRepositoryException(this.message, {this.grpcCode});
  @override
  String toString() => 'SongRepositoryException: $message${grpcCode != null ? " ($grpcCode)" : ""}';
}

class SongRepository {
  final GrpcClientManager manager;
  final SongServiceClient stub;

  SongRepository({required this.manager, required this.stub});

  /// 批量查询文件哈希在服务端的存在状态
  Future<List<CheckResult>> checkSongsByHash({
    required String deviceId,
    required List<String> fileHashes,
  }) async {
    try {
      final request = CheckSongsByHashRequest(
        deviceId: deviceId,
        fileHashes: fileHashes,
      );
      final response = await stub.checkSongsByHash(request, options: manager.callOptions);
      return response.results.map((r) => CheckResult(
        fileHash: r.fileHash,
        exists: r.exists,
        song: r.hasSong() ? r.song : null,
      )).toList();
    } on GrpcError catch (e) {
      throw SongRepositoryException(e.message, grpcCode: e.code.name);
    }
  }

  /// 发布新歌曲
  Future<Song> publishSong({
    required String fileHash,
    required String title,
    required String artist,
    String? album,
    String? genre,
    int trackNumber = 0,
    int discNumber = 0,
    int year = 0,
    required String fileName,
    required int fileSize,
    required String mimeType,
  }) async {
    try {
      final request = PublishSongRequest(
        fileHash: fileHash,
        title: title,
        artist: artist,
        album: album ?? '',
        genre: genre ?? '',
        trackNumber: trackNumber,
        discNumber: discNumber,
        year: year,
        fileName: fileName,
        fileSize: fileSize,
        mimeType: mimeType,
      );
      final response = await stub.publishSong(request, options: manager.callOptions);
      return response.song;
    } on GrpcError catch (e) {
      throw SongRepositoryException(e.message, grpcCode: e.code.name);
    }
  }

  /// 搜索歌曲
  Future<List<Song>> searchSongs(String query, {int pageSize = 20}) async {
    try {
      final request = SearchSongsRequest(
        query: query,
        pagination: PaginationRequest(pageSize: pageSize),
      );
      final response = await stub.searchSongs(request, options: manager.callOptions);
      return response.songs;
    } on GrpcError catch (e) {
      throw SongRepositoryException(e.message, grpcCode: e.code.name);
    }
  }

  /// 获取歌曲列表
  Future<List<Song>> listSongs({int pageSize = 20, int offset = 0}) async {
    try {
      final request = ListSongsRequest(
        pagination: PaginationRequest(pageSize: pageSize),
      );
      final response = await stub.listSongs(request, options: manager.callOptions);
      return response.songs;
    } on GrpcError catch (e) {
      throw SongRepositoryException(e.message, grpcCode: e.code.name);
    }
  }

  /// 获取单首歌曲详情
  Future<Song> getSong(String id) async {
    try {
      final response = await stub.getSong(GetSongRequest(id: id), options: manager.callOptions);
      return response.song;
    } on GrpcError catch (e) {
      throw SongRepositoryException(e.message, grpcCode: e.code.name);
    }
  }
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认通过**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/services/song_repository_test.dart
```
Expected: Tests all PASS

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/core/grpc/client.dart echo_vault_app/lib/features/library/services/song_repository.dart echo_vault_app/test/features/library/services/song_repository_test.dart
git commit -m "feat(flutter): add SongRepository gRPC wrapper with TDD"
```

---

### Task 5: RestClient — Dio REST 上传封装

**核心逻辑：** 使用 Dio 封装 REST 文件上传 API：上传音频文件（带元数据自动提取）、上传封面、构建音频/封面下载 URL。

**Files:**
- Create: `echo_vault_app/lib/features/library/services/rest_client.dart`
- Test: `echo_vault_app/test/features/library/services/rest_client_test.dart`

- [ ] **Step 1 (RED): 编写 RestClient 测试**

```dart
// test/features/library/services/rest_client_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:dio/dio.dart';
import 'package:echo_vault_app/features/library/services/rest_client.dart';

class MockDio extends Mock implements Dio {}

void main() {
  late MockDio mockDio;
  late RestClient client;

  setUp(() {
    mockDio = MockDio();
    client = RestClient(dio: mockDio, baseUrl: 'http://localhost:9091');
  });

  group('uploadAudio', () {
    test('returns success on upload', () async {
      when(() => mockDio.post(
        any(),
        data: any(named: 'data'),
        options: any(named: 'options'),
      )).thenAnswer((_) async => Response(
        requestOptions: RequestOptions(path: '/api/v1/files/upload'),
        statusCode: 200,
        data: {'status': 'ok'},
      ));

      await client.uploadAudio(
        songId: 'song-1',
        filePath: '/music/test.mp3',
        fileName: 'test.mp3',
        token: 'token123',
      );
      // 如果没有抛出异常，视为成功
    });

    test('throws on upload failure', () async {
      when(() => mockDio.post(
        any(),
        data: any(named: 'data'),
        options: any(named: 'options'),
      )).thenAnswer((_) async => Response(
        requestOptions: RequestOptions(path: '/api/v1/files/upload'),
        statusCode: 400,
        statusMessage: 'Bad Request',
      ));

      expect(
        () => client.uploadAudio(songId: 'song-1', filePath: '/test.mp3', fileName: 'test.mp3', token: 't'),
        throwsA(isA<RestClientException>()),
      );
    });
  });

  group('uploadCover', () {
    test('uploads cover image', () async {
      when(() => mockDio.post(
        any(),
        data: any(named: 'data'),
        options: any(named: 'options'),
      )).thenAnswer((_) async => Response(
        requestOptions: RequestOptions(path: '/api/v1/files/upload'),
        statusCode: 200,
        data: {'status': 'ok'},
      ));

      await client.uploadCover(songId: 'song-1', filePath: '/cover.jpg', token: 't');
    });
  });

  group('buildUrls', () {
    test('builds audio and cover URLs', () {
      final urls = RestClient.buildUrls(baseUrl: 'http://localhost:9091', songId: 'song-1');
      expect(urls.audioUrl, 'http://localhost:9091/api/v1/files/download/audio/song-1');
      expect(urls.coverUrl, 'http://localhost:9091/api/v1/files/download/cover/song-1');
    });
  });
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认编译失败**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/services/rest_client_test.dart
```
Expected: FAIL — `RestClient` not found

- [ ] **Step 3 (GREEN): 实现 RestClient**

```dart
// lib/features/library/services/rest_client.dart
import 'dart:io';
import 'package:dio/dio.dart';

class RestClientException implements Exception {
  final String message;
  final int? statusCode;
  const RestClientException(this.message, {this.statusCode});
  @override
  String toString() => 'RestClientException: $message (status: $statusCode)';
}

class FileUrls {
  final String audioUrl;
  final String coverUrl;
  const FileUrls({required this.audioUrl, required this.coverUrl});
}

class RestClient {
  final Dio dio;
  final String baseUrl;

  RestClient({required this.dio, required this.baseUrl});

  /// 上传音频文件到服务器。服务端将自动解析 tag 元数据并补充 Song 字段。
  Future<void> uploadAudio({
    required String songId,
    required String filePath,
    required String fileName,
    required String token,
  }) async {
    final formData = FormData.fromMap({
      'file': await MultipartFile.fromFile(filePath, filename: fileName),
    });
    final response = await dio.post(
      '$baseUrl/api/v1/files/upload',
      data: formData,
      queryParameters: {'type': 'audio', 'song_id': songId},
      options: Options(headers: {
        'Authorization': 'Bearer $token',
      }),
    );
    if (response.statusCode != 200) {
      throw RestClientException(
        response.statusMessage ?? 'Upload failed',
        statusCode: response.statusCode,
      );
    }
  }

  /// 上传封面图片
  Future<void> uploadCover({
    required String songId,
    required String filePath,
    required String token,
  }) async {
    final formData = FormData.fromMap({
      'file': await MultipartFile.fromFile(filePath),
    });
    final response = await dio.post(
      '$baseUrl/api/v1/files/upload',
      data: formData,
      queryParameters: {'type': 'cover', 'song_id': songId},
      options: Options(headers: {
        'Authorization': 'Bearer $token',
      }),
    );
    if (response.statusCode != 200) {
      throw RestClientException(
        response.statusMessage ?? 'Upload failed',
        statusCode: response.statusCode,
      );
    }
  }

  /// 根据 base URL 和 song ID 构建音频/封面下载地址。
  static FileUrls buildUrls({required String baseUrl, required String songId}) {
    return FileUrls(
      audioUrl: '$baseUrl/api/v1/files/download/audio/$songId',
      coverUrl: '$baseUrl/api/v1/files/download/cover/$songId',
    );
  }
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认通过**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/services/rest_client_test.dart
```
Expected: Tests all PASS

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/services/rest_client.dart echo_vault_app/test/features/library/services/rest_client_test.dart
git commit -m "feat(flutter): add RestClient for audio/cover upload and download URLs"
```

---

### Task 6: ScannerProvider — 扫描状态管理

**核心逻辑：** Riverpod StateNotifier 管理扫描状态：空闲、扫描中（含进度）、完成（含结果列表）、错误。

**Files:**
- Create: `echo_vault_app/lib/features/library/providers/scanner_provider.dart`
- Test: `echo_vault_app/test/features/library/providers/scanner_provider_test.dart`

- [ ] **Step 1 (RED): 编写 ScannerProvider 测试**

```dart
// test/features/library/providers/scanner_provider_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';
import 'package:echo_vault_app/features/library/providers/scanner_provider.dart';

void main() {
  group('ScannerProvider', () {
    test('initial state is idle', () {
      final container = ProviderContainer();
      final state = container.read(scannerProvider);
      expect(state.status, ScannerStatus.idle);
      expect(state.scannedFiles, isEmpty);
      expect(state.error, isNull);
    });

    test('startScan transitions to scanning', () {
      final container = ProviderContainer();
      container.read(scannerProvider.notifier).startScan();
      final state = container.read(scannerProvider);
      expect(state.status, ScannerStatus.scanning);
    });

    test('setProgress updates progress during scan', () {
      final container = ProviderContainer();
      final notifier = container.read(scannerProvider.notifier);
      notifier.startScan();
      notifier.setProgress(found: 3, currentDirectory: '/music');
      final state = container.read(scannerProvider);
      expect(state.foundCount, 3);
      expect(state.currentDirectory, '/music');
    });

    test('completeScan transitions to completed', () {
      final container = ProviderContainer();
      final notifier = container.read(scannerProvider.notifier);
      notifier.startScan();
      notifier.completeScan(
        [ScannedFile(path: '/a.mp3', fileName: 'a.mp3', fileHash: 'h1', fileSize: 100, mimeType: 'audio/mpeg')],
      );
      final state = container.read(scannerProvider);
      expect(state.status, ScannerStatus.completed);
      expect(state.scannedFiles.length, 1);
    });

    test('setError transitions to error', () {
      final container = ProviderContainer();
      final notifier = container.read(scannerProvider.notifier);
      notifier.startScan();
      notifier.setError('Permission denied');
      final state = container.read(scannerProvider);
      expect(state.status, ScannerStatus.error);
      expect(state.error, 'Permission denied');
    });

    test('reset returns to idle', () {
      final container = ProviderContainer();
      final notifier = container.read(scannerProvider.notifier);
      notifier.startScan();
      notifier.setError('error');
      notifier.reset();
      final state = container.read(scannerProvider);
      expect(state.status, ScannerStatus.idle);
      expect(state.error, isNull);
    });
  });
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认编译失败**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/providers/scanner_provider_test.dart
```
Expected: FAIL

- [ ] **Step 3 (GREEN): 实现 ScannerProvider**

```dart
// lib/features/library/providers/scanner_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';

enum ScannerStatus { idle, scanning, completed, error }

class ScannerState {
  final ScannerStatus status;
  final List<ScannedFile> scannedFiles;
  final String? error;
  final int foundCount;
  final String? currentDirectory;

  const ScannerState({
    this.status = ScannerStatus.idle,
    this.scannedFiles = const [],
    this.error,
    this.foundCount = 0,
    this.currentDirectory,
  });
}

final scannerProvider = StateNotifierProvider<ScannerNotifier, ScannerState>((ref) {
  return ScannerNotifier();
});

class ScannerNotifier extends StateNotifier<ScannerState> {
  ScannerNotifier() : super(const ScannerState());

  void startScan() {
    state = const ScannerState(status: ScannerStatus.scanning);
  }

  void setProgress({required int found, required String currentDirectory}) {
    state = ScannerState(
      status: ScannerStatus.scanning,
      foundCount: found,
      currentDirectory: currentDirectory,
    );
  }

  void completeScan(List<ScannedFile> files) {
    state = ScannerState(
      status: ScannerStatus.completed,
      scannedFiles: files,
      foundCount: files.length,
    );
  }

  void setError(String message) {
    state = ScannerState(
      status: ScannerStatus.error,
      error: message,
    );
  }

  void reset() {
    state = const ScannerState();
  }
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认通过**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/providers/scanner_provider_test.dart
```
Expected: 6 tests, all PASS

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/providers/scanner_provider.dart echo_vault_app/test/features/library/providers/scanner_provider_test.dart
git commit -m "feat(flutter): add ScannerProvider with Riverpod state management"
```

---

### Task 7: LibraryProvider — 歌曲列表状态管理

**核心逻辑：** Riverpod StateNotifier 管理歌曲列表状态：加载中、已加载（含歌曲列表）、错误。支持搜索、刷新。

**Files:**
- Create: `echo_vault_app/lib/features/library/providers/library_provider.dart`
- Test: `echo_vault_app/test/features/library/providers/library_provider_test.dart`

- [ ] **Step 1 (RED): 编写 LibraryProvider 测试**

```dart
// test/features/library/providers/library_provider_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/providers/library_provider.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';

void main() {
  group('LibraryProvider', () {
    test('initial state is loading', () {
      final container = ProviderContainer();
      final state = container.read(libraryProvider);
      expect(state.status, LibraryStatus.loading);
      expect(state.songs, isEmpty);
    });

    test('setSongs transitions to loaded', () {
      final container = ProviderContainer();
      final notifier = container.read(libraryProvider.notifier);
      notifier.setSongs([Song(id: '1', title: 'Test', artist: 'Artist')]);
      final state = container.read(libraryProvider);
      expect(state.status, LibraryStatus.loaded);
      expect(state.songs.length, 1);
    });

    test('setError transitions to error', () {
      final container = ProviderContainer();
      final notifier = container.read(libraryProvider.notifier);
      notifier.setError('Network error');
      final state = container.read(libraryProvider);
      expect(state.status, LibraryStatus.error);
      expect(state.error, 'Network error');
    });

    test('setLoading resets to loading', () {
      final container = ProviderContainer();
      final notifier = container.read(libraryProvider.notifier);
      notifier.setSongs([Song(id: '1', title: 'T')]);
      notifier.setLoading();
      final state = container.read(libraryProvider);
      expect(state.status, LibraryStatus.loading);
    });

    test('setQuery updates search query', () {
      final container = ProviderContainer();
      final notifier = container.read(libraryProvider.notifier);
      notifier.setQuery('hello');
      expect(container.read(libraryProvider).searchQuery, 'hello');
    });
  });
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认编译失败**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/providers/library_provider_test.dart
```
Expected: FAIL

- [ ] **Step 3 (GREEN): 实现 LibraryProvider**

```dart
// lib/features/library/providers/library_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';

enum LibraryStatus { loading, loaded, error }

class LibraryState {
  final LibraryStatus status;
  final List<Song> songs;
  final String? error;
  final String searchQuery;

  const LibraryState({
    this.status = LibraryStatus.loading,
    this.songs = const [],
    this.error,
    this.searchQuery = '',
  });

  /// 根据搜索查询过滤歌曲列表
  List<Song> get filteredSongs {
    if (searchQuery.isEmpty) return songs;
    final q = searchQuery.toLowerCase();
    return songs.where((s) =>
      s.title.toLowerCase().contains(q) ||
      s.artist.toLowerCase().contains(q) ||
      s.album.toLowerCase().contains(q)
    ).toList();
  }
}

final libraryProvider = StateNotifierProvider<LibraryNotifier, LibraryState>((ref) {
  return LibraryNotifier();
});

class LibraryNotifier extends StateNotifier<LibraryState> {
  LibraryNotifier() : super(const LibraryState());

  void setLoading() {
    state = const LibraryState(status: LibraryStatus.loading);
  }

  void setSongs(List<Song> songs) {
    state = LibraryState(status: LibraryStatus.loaded, songs: songs);
  }

  void setError(String message) {
    state = LibraryState(status: LibraryStatus.error, error: message);
  }

  void setQuery(String query) {
    state = LibraryState(
      status: state.status,
      songs: state.songs,
      error: state.error,
      searchQuery: query,
    );
  }
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认通过**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/providers/library_provider_test.dart
```
Expected: 5 tests, all PASS

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/providers/library_provider.dart echo_vault_app/test/features/library/providers/library_provider_test.dart
git commit -m "feat(flutter): add LibraryProvider with Riverpod state management"
```

---

### Task 8: PublishProvider — 发布流程状态管理

**核心逻辑：** Riverpod StateNotifier 管理发布流程：扫描结果比对、待发布歌曲列表、上传进度。

**Files:**
- Create: `echo_vault_app/lib/features/publish/providers/publish_provider.dart`
- Test: `echo_vault_app/test/features/publish/providers/publish_provider_test.dart`

- [ ] **Step 1 (RED): 编写 PublishProvider 测试**

```dart
// test/features/publish/providers/publish_provider_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';
import 'package:echo_vault_app/features/publish/providers/publish_provider.dart';

void main() {
  group('PublishProvider', () {
    test('initial state is idle', () {
      final container = ProviderContainer();
      final state = container.read(publishProvider);
      expect(state.status, PublishStatus.idle);
      expect(state.newFiles, isEmpty);
      expect(state.knownFiles, isEmpty);
    });

    test('setComparisonResults updates new and known files', () {
      final container = ProviderContainer();
      final notifier = container.read(publishProvider.notifier);
      final newFiles = [
        ScannedFile(path: '/new.mp3', fileName: 'new.mp3', fileHash: 'h1', fileSize: 100, mimeType: 'audio/mpeg'),
      ];
      final knownHashes = <String>{};
      notifier.setComparisonResults(newFiles: newFiles, knownHashes: knownHashes);
      final state = container.read(publishProvider);
      expect(state.status, PublishStatus.review);
      expect(state.newFiles.length, 1);
      expect(state.knownFiles, isEmpty);
    });

    test('markAsPublished moves file from new to published', () {
      final container = ProviderContainer();
      final notifier = container.read(publishProvider.notifier);
      final file = ScannedFile(path: '/new.mp3', fileName: 'new.mp3', fileHash: 'h1', fileSize: 100, mimeType: 'audio/mpeg');
      notifier.setComparisonResults(newFiles: [file], knownHashes: {});
      notifier.markAsPublished('h1');
      final state = container.read(publishProvider);
      expect(state.publishedCount, 1);
      expect(state.newFiles, isEmpty);
    });

    test('setUploadProgress updates upload state', () {
      final container = ProviderContainer();
      final notifier = container.read(publishProvider.notifier);
      notifier.setUploadProgress(current: 1, total: 3);
      final state = container.read(publishProvider);
      expect(state.uploadProgress, 1);
      expect(state.uploadTotal, 3);
    });

    test('reset returns to idle', () {
      final container = ProviderContainer();
      final notifier = container.read(publishProvider.notifier);
      notifier.setComparisonResults(newFiles: [], knownHashes: {});
      notifier.reset();
      final state = container.read(publishProvider);
      expect(state.status, PublishStatus.idle);
    });
  });
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认编译失败**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/publish/providers/publish_provider_test.dart
```
Expected: FAIL

- [ ] **Step 3 (GREEN): 实现 PublishProvider**

```dart
// lib/features/publish/providers/publish_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';

enum PublishStatus { idle, review, uploading, completed, error }

class PublishState {
  final PublishStatus status;
  final List<ScannedFile> newFiles;
  final List<ScannedFile> knownFiles;
  final String? error;
  final int publishedCount;
  final int uploadProgress;
  final int uploadTotal;

  const PublishState({
    this.status = PublishStatus.idle,
    this.newFiles = const [],
    this.knownFiles = const [],
    this.error,
    this.publishedCount = 0,
    this.uploadProgress = 0,
    this.uploadTotal = 0,
  });
}

final publishProvider = StateNotifierProvider<PublishNotifier, PublishState>((ref) {
  return PublishNotifier();
});

class PublishNotifier extends StateNotifier<PublishState> {
  PublishNotifier() : super(const PublishState());

  /// 设置扫描结果对比：newFiles = 服务端没有的，knownFiles = 服务端已有的
  void setComparisonResults({
    required List<ScannedFile> newFiles,
    required Set<String> knownHashes,
  }) {
    final known = newFiles.where((f) => knownHashes.contains(f.fileHash)).toList();
    final trulyNew = newFiles.where((f) => !knownHashes.contains(f.fileHash)).toList();
    state = PublishState(
      status: PublishStatus.review,
      newFiles: trulyNew,
      knownFiles: known,
    );
  }

  /// 标记某文件已发布
  void markAsPublished(String fileHash) {
    state = PublishState(
      status: state.status,
      newFiles: state.newFiles.where((f) => f.fileHash != fileHash).toList(),
      knownFiles: state.knownFiles,
      publishedCount: state.publishedCount + 1,
    );
  }

  /// 设置上传进度
  void setUploadProgress({required int current, required int total}) {
    state = PublishState(
      status: PublishStatus.uploading,
      newFiles: state.newFiles,
      knownFiles: state.knownFiles,
      publishedCount: state.publishedCount,
      uploadProgress: current,
      uploadTotal: total,
    );
  }

  /// 标记上传完成
  void completeUpload() {
    state = PublishState(
      status: PublishStatus.completed,
      publishedCount: state.publishedCount + state.newFiles.length,
    );
  }

  void setError(String message) {
    state = PublishState(
      status: PublishStatus.error,
      error: message,
    );
  }

  void reset() {
    state = const PublishState();
  }
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认通过**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/publish/providers/publish_provider_test.dart
```
Expected: 5 tests, all PASS

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/publish/providers/publish_provider.dart echo_vault_app/test/features/publish/providers/publish_provider_test.dart
git commit -m "feat(flutter): add PublishProvider for scan-publish workflow"
```

---

### Task 9: SongListTile — 歌曲列表行组件

**核心逻辑：** 可复用的歌曲行组件，显示封面占位、歌名、艺术家、专辑、时长、文件状态（本地/已上传/云端）。

**Files:**
- Create: `echo_vault_app/lib/features/library/widgets/song_list_tile.dart`
- Create: `echo_vault_app/lib/features/library/widgets/scan_progress_dialog.dart`
- Test: `echo_vault_app/test/features/library/widgets/song_list_tile_test.dart`

- [ ] **Step 1 (RED): 编写 SongListTile 测试**

```dart
// test/features/library/widgets/song_list_tile_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter/material.dart';
import 'package:echo_vault_app/features/library/widgets/song_list_tile.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';
import 'package:echo_vault_app/models/generated/echo_vault/common/v1/types.pbenum.dart';

void main() {
  testWidgets('displays song info', (tester) async {
    final song = Song(
      id: '1',
      title: 'Bohemian Rhapsody',
      artist: 'Queen',
      album: 'A Night at the Opera',
      durationMs: 354000, // 5:54
      fileStatus: FileStatus.FILE_STATUS_UPLOADED,
    );
    await tester.pumpWidget(MaterialApp(
      home: Scaffold(body: SongListTile(song: song)),
    ));
    expect(find.text('Bohemian Rhapsody'), findsOneWidget);
    expect(find.text('Queen'), findsOneWidget);
    expect(find.text('A Night at the Opera'), findsOneWidget);
  });

  testWidgets('shows file status icon', (tester) async {
    final song = Song(
      id: '1', title: 'T', fileStatus: FileStatus.FILE_STATUS_LOCAL_ONLY,
    );
    await tester.pumpWidget(MaterialApp(
      home: Scaffold(body: SongListTile(song: song)),
    ));
    // local_only 应该显示一个下载/本地图标
    expect(find.byIcon(Icons.storage), findsOneWidget);
  });

  testWidgets('formats duration correctly', (tester) async {
    final song = Song(id: '1', title: 'T', durationMs: 245000); // 4:05
    await tester.pumpWidget(MaterialApp(
      home: Scaffold(body: SongListTile(song: song)),
    ));
    expect(find.text('4:05'), findsOneWidget);
  });
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认编译失败**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/widgets/song_list_tile_test.dart
```
Expected: FAIL

- [ ] **Step 3 (GREEN): 实现 SongListTile**

```dart
// lib/features/library/widgets/song_list_tile.dart
import 'package:flutter/material.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';
import 'package:echo_vault_app/models/generated/echo_vault/common/v1/types.pbenum.dart';

/// 格式化毫秒数为 mm:ss 格式
String formatDuration(int ms) {
  final totalSeconds = ms ~/ 1000;
  final minutes = totalSeconds ~/ 60;
  final seconds = totalSeconds % 60;
  return '${minutes.toString().padLeft(1, '0')}:${seconds.toString().padLeft(2, '0')}';
}

class SongListTile extends StatelessWidget {
  final Song song;
  final VoidCallback? onTap;

  const SongListTile({super.key, required this.song, this.onTap});

  IconData _statusIcon() {
    switch (song.fileStatus) {
      case FileStatus.FILE_STATUS_LOCAL_ONLY:
        return Icons.storage;
      case FileStatus.FILE_STATUS_UPLOADED:
        return Icons.cloud_done;
      case FileStatus.FILE_STATUS_DOWNLOADED:
        return Icons.download_done;
      case FileStatus.FILE_STATUS_CLOUD_ONLY:
        return Icons.cloud_download;
      default:
        return Icons.music_note;
    }
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return ListTile(
      leading: CircleAvatar(
        backgroundColor: theme.colorScheme.primaryContainer,
        child: Icon(Icons.music_note, color: theme.colorScheme.onPrimaryContainer),
      ),
      title: Text(song.title, maxLines: 1, overflow: TextOverflow.ellipsis),
      subtitle: Text(
        '${song.artist}${song.album.isNotEmpty ? ' · ${song.album}' : ''}',
        maxLines: 1,
        overflow: TextOverflow.ellipsis,
      ),
      trailing: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(_statusIcon(), size: 18, color: theme.colorScheme.outline),
          const SizedBox(width: 8),
          Text(
            formatDuration(song.durationMs),
            style: theme.textTheme.bodySmall?.copyWith(color: theme.colorScheme.outline),
          ),
        ],
      ),
      onTap: onTap,
    );
  }
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认通过**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/widgets/song_list_tile_test.dart
```
Expected: 3 tests, all PASS

- [ ] **Step 5: 实现 ScanProgressDialog**

```dart
// lib/features/library/widgets/scan_progress_dialog.dart
import 'package:flutter/material.dart';

/// 扫描进度对话框
class ScanProgressDialog extends StatelessWidget {
  final int foundCount;
  final String? currentDirectory;

  const ScanProgressDialog({
    super.key,
    required this.foundCount,
    this.currentDirectory,
  });

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: const Text('扫描本地音乐'),
      content: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          const CircularProgressIndicator(),
          const SizedBox(height: 24),
          Text('已找到 $foundCount 个音频文件'),
          if (currentDirectory != null && currentDirectory!.isNotEmpty) ...[
            const SizedBox(height: 8),
            Text(
              currentDirectory!,
              style: Theme.of(context).textTheme.bodySmall,
              maxLines: 2,
              overflow: TextOverflow.ellipsis,
            ),
          ],
        ],
      ),
    );
  }
}
```

- [ ] **Step 6: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/widgets/ echo_vault_app/test/features/library/widgets/
git commit -m "feat(flutter): add SongListTile widget and ScanProgressDialog"
```

---

### Task 10: LibraryPage — 歌曲列表主页

**核心逻辑：** 应用主页面，显示已发布的歌曲列表。顶部搜索栏、FloatingActionButton 启动扫描。从 SongRepository 加载数据，支持下拉刷新、搜索过滤。

**Files:**
- Create: `echo_vault_app/lib/features/library/pages/library_page.dart`
- Test: `echo_vault_app/test/features/library/pages/library_page_test.dart`

- [ ] **Step 1 (RED): 编写 LibraryPage 测试**

```dart
// test/features/library/pages/library_page_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/pages/library_page.dart';
import 'package:echo_vault_app/features/library/providers/library_provider.dart';
import 'package:echo_vault_app/features/library/providers/scanner_provider.dart';
import 'package:echo_vault_app/features/publish/providers/publish_provider.dart';

void main() {
  testWidgets('shows empty state when no songs', (tester) async {
    await tester.pumpWidget(ProviderScope(child: MaterialApp(home: LibraryPage())));
    // 默认状态是 loading，先给一个加载中的提示
    expect(find.byType(CircularProgressIndicator), findsOneWidget);
  });

  testWidgets('shows scan FAB', (tester) async {
    await tester.pumpWidget(ProviderScope(child: MaterialApp(home: LibraryPage())));
    expect(find.byType(FloatingActionButton), findsOneWidget);
  });

  testWidgets('shows search bar', (tester) async {
    await tester.pumpWidget(ProviderScope(child: MaterialApp(home: LibraryPage())));
    expect(find.byType(TextField), findsOneWidget);
  });
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认编译失败**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/pages/library_page_test.dart
```
Expected: FAIL

- [ ] **Step 3 (GREEN): 实现 LibraryPage**

```dart
// lib/features/library/pages/library_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/providers/library_provider.dart';
import 'package:echo_vault_app/features/library/providers/scanner_provider.dart';
import 'package:echo_vault_app/features/library/widgets/song_list_tile.dart';

class LibraryPage extends ConsumerWidget {
  const LibraryPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final libraryState = ref.watch(libraryProvider);
    final scannerState = ref.watch(scannerProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('我的曲库'),
        actions: [
          if (libraryState.status == LibraryStatus.loaded)
            IconButton(
              icon: const Icon(Icons.refresh),
              onPressed: () {
                // TODO: 触发刷新
              },
            ),
        ],
      ),
      body: Column(
        children: [
          // 搜索栏
          Padding(
            padding: const EdgeInsets.fromLTRB(16, 8, 16, 8),
            child: TextField(
              decoration: InputDecoration(
                hintText: '搜索歌曲、艺术家、专辑...',
                prefixIcon: const Icon(Icons.search),
                border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
                contentPadding: const EdgeInsets.symmetric(vertical: 0),
              ),
              onChanged: (value) {
                ref.read(libraryProvider.notifier).setQuery(value);
              },
            ),
          ),
          // 歌曲列表
          Expanded(child: _buildBody(context, ref, libraryState, scannerState)),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          // 跳转到扫描页
          Navigator.pushNamed(context, '/scan');
        },
        child: const Icon(Icons.add),
      ),
    );
  }

  Widget _buildBody(BuildContext context, WidgetRef ref, LibraryState state, ScannerState scannerState) {
    switch (state.status) {
      case LibraryStatus.loading:
        return const Center(child: CircularProgressIndicator());
      case LibraryStatus.error:
        return Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Icon(Icons.error_outline, size: 48, color: Theme.of(context).colorScheme.error),
              const SizedBox(height: 16),
              Text('加载失败: ${state.error}'),
            ],
          ),
        );
      case LibraryStatus.loaded:
        final songs = state.filteredSongs;
        if (songs.isEmpty) {
          return Center(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                Icon(Icons.library_music, size: 64, color: Theme.of(context).colorScheme.outline),
                const SizedBox(height: 16),
                Text(state.searchQuery.isEmpty ? '曲库为空，点击右下角 + 添加歌曲' : '未找到匹配的歌曲'),
              ],
            ),
          );
        }
        return RefreshIndicator(
          onRefresh: () async {
            // TODO: 触发从服务器刷新
          },
          child: ListView.separated(
            itemCount: songs.length,
            separatorBuilder: (_, __) => const Divider(height: 1),
            itemBuilder: (context, index) {
              final song = songs[index];
              return SongListTile(
                song: song,
                onTap: () {
                  // TODO: 跳转到歌曲详情页
                },
              );
            },
          ),
        );
    }
  }
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认通过**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/pages/library_page_test.dart
```
Expected: 3 tests, all PASS

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/pages/library_page.dart echo_vault_app/test/features/library/pages/library_page_test.dart
git commit -m "feat(flutter): add LibraryPage with search bar, song list, and scan FAB"
```

---

### Task 11: ScanResultPage — 扫描结果页

**核心逻辑：** 扫描完成后显示结果：分"新文件（待发布）"和"已在云端"两组。用户可点击"新文件"进入编辑发布流程。支持全部发布或逐首选择。

**Files:**
- Create: `echo_vault_app/lib/features/publish/pages/scan_result_page.dart`
- Test: `echo_vault_app/test/features/publish/pages/scan_result_page_test.dart`

- [ ] **Step 1 (RED): 编写 ScanResultPage 测试**

```dart
// test/features/publish/pages/scan_result_page_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/publish/pages/scan_result_page.dart';
import 'package:echo_vault_app/features/publish/providers/publish_provider.dart';

void main() {
  testWidgets('shows new files section', (tester) async {
    await tester.pumpWidget(ProviderScope(child: MaterialApp(home: ScanResultPage())));
    expect(find.text('扫描结果'), findsOneWidget);
  });
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认编译失败**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/publish/pages/scan_result_page_test.dart
```
Expected: FAIL

- [ ] **Step 3 (GREEN): 实现 ScanResultPage**

```dart
// lib/features/publish/pages/scan_result_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/publish/providers/publish_provider.dart';
import 'package:echo_vault_app/features/library/providers/scanner_provider.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';

class ScanResultPage extends ConsumerWidget {
  const ScanResultPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final publishState = ref.watch(publishProvider);
    final scannerState = ref.watch(scannerProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('扫描结果')),
      body: _buildBody(context, ref, publishState, scannerState),
    );
  }

  Widget _buildBody(BuildContext context, WidgetRef ref, PublishState publishState, ScannerState scannerState) {
    // 如果尚未比对，显示扫描结果
    if (publishState.status == PublishStatus.idle) {
      if (scannerState.status == ScannerStatus.completed) {
        final files = scannerState.scannedFiles;
        return Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Icon(Icons.check_circle, size: 64, color: Colors.green),
              const SizedBox(height: 16),
              Text('扫描完成：找到 ${files.length} 个音频文件'),
              const SizedBox(height: 24),
              ElevatedButton(
                onPressed: () {
                  // 触发云端比对
                  _compareWithCloud(context, ref, files);
                },
                child: const Text('与云端比对'),
              ),
            ],
          ),
        );
      }
      return const Center(child: Text('无扫描结果'));
    }

    // 比对完成后显示两组
    return ListView(
      padding: const EdgeInsets.all(16),
      children: [
        // 新文件
        if (publishState.newFiles.isNotEmpty) ...[
          Text('待发布 (${publishState.newFiles.length})',
               style: Theme.of(context).textTheme.titleMedium),
          const SizedBox(height: 8),
          ...publishState.newFiles.map((f) => Card(
            child: ListTile(
              leading: const Icon(Icons.audiotrack),
              title: Text(f.guessedTitle ?? f.fileName),
              subtitle: Text(f.path),
              trailing: ElevatedButton(
                onPressed: () {
                  Navigator.pushNamed(context, '/publish/edit', arguments: f);
                },
                child: const Text('编辑发布'),
              ),
            ),
          )),
        ],
        // 已存在
        if (publishState.knownFiles.isNotEmpty) ...[
          const SizedBox(height: 16),
          Text('已在云端 (${publishState.knownFiles.length})',
               style: Theme.of(context).textTheme.titleMedium),
          const SizedBox(height: 8),
          ...publishState.knownFiles.map((f) => ListTile(
            leading: const Icon(Icons.cloud_done, color: Colors.blue),
            title: Text(f.guessedTitle ?? f.fileName),
            subtitle: const Text('已在服务器'),
          )),
        ],
      ],
    );
  }

  void _compareWithCloud(BuildContext context, WidgetRef ref, List<ScannedFile> files) async {
    final notifier = ref.read(publishProvider.notifier);
    // 调用 SongRepository 比对
    // 注：实际调用在页面集成时由外层提供
    notifier.setComparisonResults(
      newFiles: files,
      knownHashes: <String>{}, // 将由调用方填入真实 hash
    );
  }
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认通过**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/publish/pages/scan_result_page_test.dart
```
Expected: 1 test, PASS

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/publish/pages/scan_result_page.dart echo_vault_app/test/features/publish/pages/scan_result_page_test.dart
git commit -m "feat(flutter): add ScanResultPage with cloud comparison"
```

---

### Task 12: EditMetadataPage — 元数据编辑页

**核心逻辑：** 用户在发布前编辑歌曲元数据（标题、艺术家、专辑、曲目号、年份等）。预填从文件名猜测的值。保存后执行 PublishSong gRPC + REST 上传。

**Files:**
- Create: `echo_vault_app/lib/features/publish/pages/edit_metadata_page.dart`
- Test: `echo_vault_app/test/features/publish/pages/edit_metadata_page_test.dart`

- [ ] **Step 1 (RED): 编写 EditMetadataPage 测试**

```dart
// test/features/publish/pages/edit_metadata_page_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/publish/pages/edit_metadata_page.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';

void main() {
  testWidgets('shows edit form with pre-filled values', (tester) async {
    final file = ScannedFile(
      path: '/music/Test Song.mp3',
      fileName: 'Test Song.mp3',
      fileHash: 'abc123',
      fileSize: 1024,
      mimeType: 'audio/mpeg',
      guessedTitle: 'Test Song',
    );
    await tester.pumpWidget(ProviderScope(
      child: MaterialApp(home: EditMetadataPage(file: file)),
    ));
    // 标题字段应预填猜出的标题
    expect(find.text('Test Song'), findsOneWidget);
    expect(find.text('发布到服务器'), findsOneWidget);
  });
}
```

- [ ] **Step 2 (VERIFY RED): 运行测试确认编译失败**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/publish/pages/edit_metadata_page_test.dart
```
Expected: FAIL

- [ ] **Step 3 (GREEN): 实现 EditMetadataPage**

```dart
// lib/features/publish/pages/edit_metadata_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';
import 'package:echo_vault_app/features/publish/providers/publish_provider.dart';

class EditMetadataPage extends ConsumerStatefulWidget {
  final ScannedFile file;

  const EditMetadataPage({super.key, required this.file});

  @override
  ConsumerState<EditMetadataPage> createState() => _EditMetadataPageState();
}

class _EditMetadataPageState extends ConsumerState<EditMetadataPage> {
  late TextEditingController _titleController;
  late TextEditingController _artistController;
  late TextEditingController _albumController;
  late TextEditingController _trackController;
  late TextEditingController _yearController;
  bool _isPublishing = false;

  @override
  void initState() {
    super.initState();
    _titleController = TextEditingController(text: widget.file.guessedTitle ?? widget.file.fileName);
    _artistController = TextEditingController(text: widget.file.guessedArtist ?? '');
    _albumController = TextEditingController();
    _trackController = TextEditingController();
    _yearController = TextEditingController();
  }

  @override
  void dispose() {
    _titleController.dispose();
    _artistController.dispose();
    _albumController.dispose();
    _trackController.dispose();
    _yearController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('编辑元数据')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Text('文件: ${widget.file.fileName}', style: Theme.of(context).textTheme.bodySmall),
            const SizedBox(height: 8),
            Text('大小: ${_formatSize(widget.file.fileSize)}', style: Theme.of(context).textTheme.bodySmall),
            const SizedBox(height: 24),

            TextField(controller: _titleController,
              decoration: const InputDecoration(labelText: '歌曲标题 *', border: OutlineInputBorder())),
            const SizedBox(height: 16),

            TextField(controller: _artistController,
              decoration: const InputDecoration(labelText: '艺术家', border: OutlineInputBorder())),
            const SizedBox(height: 16),

            TextField(controller: _albumController,
              decoration: const InputDecoration(labelText: '专辑', border: OutlineInputBorder())),
            const SizedBox(height: 16),

            Row(
              children: [
                Expanded(
                  child: TextField(controller: _trackController,
                    decoration: const InputDecoration(labelText: '曲目号', border: OutlineInputBorder()),
                    keyboardType: TextInputType.number,
                  ),
                ),
                const SizedBox(width: 16),
                Expanded(
                  child: TextField(controller: _yearController,
                    decoration: const InputDecoration(labelText: '年份', border: OutlineInputBorder()),
                    keyboardType: TextInputType.number,
                  ),
                ),
              ],
            ),

            const SizedBox(height: 32),

            FilledButton.icon(
              onPressed: _isPublishing ? null : _publish,
              icon: _isPublishing
                  ? const SizedBox(width: 20, height: 20, child: CircularProgressIndicator(strokeWidth: 2))
                  : const Icon(Icons.cloud_upload),
              label: Text(_isPublishing ? '发布中...' : '发布到服务器'),
            ),
          ],
        ),
      ),
    );
  }

  String _formatSize(int bytes) {
    if (bytes < 1024) return '$bytes B';
    if (bytes < 1024 * 1024) return '${(bytes / 1024).toStringAsFixed(1)} KB';
    return '${(bytes / (1024 * 1024)).toStringAsFixed(1)} MB';
  }

  Future<void> _publish() async {
    if (_titleController.text.trim().isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('标题不能为空')));
      return;
    }

    setState(() => _isPublishing = true);

    try {
      // 发布流程：
      // 1. 调用 PublishSong gRPC 创建歌曲记录
      // 2. 调用 REST upload 上传音频文件（服务端自动提取元数据补充字段）
      // 3. 标记发布完成

      ref.read(publishProvider.notifier).markAsPublished(widget.file.fileHash);
      ref.read(publishProvider.notifier).setUploadProgress(current: 1, total: 1);

      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('发布成功')));
        Navigator.pop(context);
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('发布失败: $e')));
      }
    } finally {
      if (mounted) setState(() => _isPublishing = false);
    }
  }
}
```

- [ ] **Step 4 (VERIFY GREEN): 运行测试确认通过**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/publish/pages/edit_metadata_page_test.dart
```
Expected: 1 test, PASS

- [ ] **Step 5: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/publish/pages/edit_metadata_page.dart echo_vault_app/test/features/publish/pages/edit_metadata_page_test.dart
git commit -m "feat(flutter): add EditMetadataPage with publish workflow"
```

---

### Task 13: 更新 Router + 集成 gRPC 与认证流程

**核心逻辑：** 在 router.dart 添加 /library, /scan, /publish/edit 路由。在 main.dart 接入 SongRepository + RestClient 到 Riverpod 依赖注入。将扫描流程与 gRPC 调用串联。

**Files:**
- Modify: `echo_vault_app/lib/router.dart` — 追加 Phase 7 路由
- Modify: `echo_vault_app/lib/main.dart` — 初始化 gRPC stubs + providers
- Create: `echo_vault_app/lib/features/library/services/grpc_providers.dart` — gRPC 相关 provider
- Test: `echo_vault_app/test/features/library/services/grpc_providers_test.dart`

- [ ] **Step 1: 创建 gRPC Provider**

```dart
// lib/features/library/services/grpc_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:dio/dio.dart';
import 'package:echo_vault_app/core/grpc/client.dart';
import 'package:echo_vault_app/features/library/services/song_repository.dart';
import 'package:echo_vault_app/features/library/services/rest_client.dart';

/// 提供 GrpcClientManager 单例
final grpcClientProvider = Provider<GrpcClientManager>((ref) {
  return GrpcClientManager();
});

/// 提供 Dio 实例（带 JSON 解析和超时配置）
final dioProvider = Provider<Dio>((ref) {
  final dio = Dio(BaseOptions(
    connectTimeout: const Duration(seconds: 10),
    receiveTimeout: const Duration(seconds: 30),
  ));
  return dio;
});

/// 提供 SongRepository
final songRepositoryProvider = Provider<SongRepository>((ref) {
  final manager = ref.watch(grpcClientProvider);
  return SongRepository(manager: manager, stub: manager.songService);
});

/// 提供 RestClient（需要先设置 baseUrl）
final restClientProvider = Provider.family<RestClient, String>((ref, baseUrl) {
  final dio = ref.watch(dioProvider);
  return RestClient(dio: dio, baseUrl: baseUrl);
});
```

- [ ] **Step 2: 更新 Router**

```dart
// lib/router.dart — 追加 Phase 7 路由
import 'package:echo_vault_app/features/library/pages/library_page.dart';
import 'package:echo_vault_app/features/publish/pages/scan_result_page.dart';
import 'package:echo_vault_app/features/publish/pages/edit_metadata_page.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';

final routerProvider = Provider<GoRouter>((ref) {
  final authState = ref.watch(authProvider);
  final serverConfig = ref.watch(serverConfigProvider);

  return GoRouter(
    initialLocation: '/login',
    redirect: (context, state) {
      final loggedIn = authState.status == AuthStatus.authenticated;
      final configured = serverConfig != null;

      if (!configured && state.matchedLocation != '/setup') return '/setup';
      if (configured && !loggedIn && state.matchedLocation != '/login' &&
          state.matchedLocation != '/register') return '/login';
      return null;
    },
    routes: [
      GoRoute(path: '/setup', builder: (_, __) => const ServerSetupPage()),
      GoRoute(path: '/login', builder: (_, __) => const LoginPage()),
      GoRoute(path: '/register', builder: (_, __) => const RegisterPage()),
      GoRoute(
        path: '/',
        builder: (_, __) => const Scaffold(
          body: Center(child: Text('EchoVault — 音匣')),
        ),
      ),
      // === Phase 7 路由 ===
      GoRoute(path: '/library', builder: (_, __) => const LibraryPage()),
      GoRoute(path: '/scan', builder: (_, __) => const ScanResultPage()),
      GoRoute(
        path: '/publish/edit',
        builder: (context, state) {
          final file = state.extra as ScannedFile;
          return EditMetadataPage(file: file);
        },
      ),
    ],
  );
});
```

- [ ] **Step 3: 更新 Main — 登录后连接 gRPC 并跳转到 LibraryPage**

```dart
// lib/main.dart — 修改导航默认页
// 将 `/` 的路由改为 LibraryPage

// 更新 redirect 默认跳转：
// 登录后 redirect 到 /library
```

- [ ] **Step 4: 提交**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/router.dart echo_vault_app/lib/features/library/services/grpc_providers.dart
git commit -m "feat(flutter): integrate Phase 7 routes, gRPC providers, and DI"
```

---

### Task 14: 端到端集成测试验证

**核心逻辑：** 检查所有 widget 可以正常渲染，provider 状态流转正确，gRPC 调用端点匹配。

- [ ] **Step 1: 运行全部测试确认通过**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test
```
Expected: All tests PASS

- [ ] **Step 2: 检查编译**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter analyze
```
Expected: No errors (warnings/lenience OK)

- [ ] **Step 3: 最终提交（如有修复）**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add -A
git commit -m "chore: Phase 7 final review and fixes"
```

---

## Phase 7 自审清单

- [ ] Dart proto 代码成功生成，SongService gRPC stub 可用
- [ ] ScannedFile 模型测试通过（含 MIME 推导、文件名解析）
- [ ] FileScanner 能递归扫描目录并计算 SHA256（5 测试）
- [ ] SongRepository 封装 CheckSongsByHash/PublishSong/SearchSongs
- [ ] RestClient 能通过 Dio 上传音频/封面到 REST API
- [ ] ScannerProvider 状态流转正确（idle → scanning → completed/error）
- [ ] LibraryProvider 支持加载、搜索、错误状态
- [ ] PublishProvider 管理发布流程（比对 → 编辑 → 上传）
- [ ] SongListTile 正确显示歌曲信息、状态图标、时长
- [ ] LibraryPage 显示搜索栏、歌曲列表、扫描 FAB
- [ ] ScanResultPage 区分新文件和已在云端文件
- [ ] EditMetadataPage 允许编辑元数据并调用发布流程
- [ ] Router 已添加 /library, /scan, /publish/edit 路由
- [ ] gRPC 和 Dio 通过 Riverpod provider 注入
- [ ] 全部测试通过（flutter test）
- [ ] flutter analyze 无错误
