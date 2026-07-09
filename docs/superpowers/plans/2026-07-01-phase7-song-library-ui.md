# Phase 7: 歌曲库 + 扫描器 UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现 Flutter 端本地文件扫描器、歌曲库列表/搜索 UI、歌曲发布流程（元数据编辑 + 上传）

**Architecture:** 桌面优先（dart:io 递归扫描），通过 gRPC-Web 调用 SongService（CheckSongsByHash/PublishSong），通过 Dio REST 上传音频文件/封面。扫描 → 比对云端 → 编辑元数据 → 上传 ~ 四步发布流程。离线优先：扫描结果先入库再上传。

**Tech Stack:** Flutter 3.x, Dart 3.x, dart:io, crypto (SHA256), Riverpod, grpc-dart, dio, go_router

---

## File Structure

```
echo_vault_app/lib/
├── core/
│   └── grpc/
│       └── client.dart                        # 已有 — 追加 gRPC stub 创建
├── features/
│   ├── library/
│   │   ├── models/
│   │   │   └── scanned_file.dart              # 扫描文件模型
│   │   ├── pages/
│   │   │   └── library_page.dart              # 歌曲列表页
│   │   ├── providers/
│   │   │   ├── library_provider.dart           # 歌单状态
│   │   │   └── scanner_provider.dart           # 扫描状态
│   │   ├── services/
│   │   │   ├── file_scanner.dart              # 递归文件扫描器
│   │   │   ├── song_repository.dart           # SongService gRPC 封装
│   │   │   └── rest_client.dart              # REST 文件上传封装
│   │   └── widgets/
│   │       └── song_list_tile.dart            # 歌曲行组件
│   ├── publish/
│   │   ├── pages/
│   │   │   ├── scan_result_page.dart          # 扫描结果页（云端比对）
│   │   │   └── edit_metadata_page.dart        # 元数据编辑页
│   │   └── providers/
│   │       └── publish_provider.dart          # 发布流程状态
│   └── auth/                                   # 已有
├── models/
│   └── generated/                              # proto 生成 Dart 代码
├── providers/                                   # 已有
├── router.dart                                 # 修改 — 追加 /library /scan 路由
└── main.dart                                   # 修改 — 初始化 gRPC stubs

echo_vault_app/test/
├── features/
│   ├── library/
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

### Task 1: Generate Dart Protobuf Code + Add Dependencies

**核心逻辑：** 使用 `protoc` 或 `buf` 从 proto 文件生成 Dart 代码。Flutter 通过 `grpc-dart` 调用 SongService（CheckSongsByHash/PublishSong/SearchSongs）。添加 `crypto`（SHA256）和 `file_picker`（目录选择）依赖。

**Files:**
- Modify: `echo_vault_app/pubspec.yaml` — 追加 crypto, file_picker, protoc_plugin
- Create: `echo_vault_app/lib/models/generated/song/v1/song_service.pb.dart`
- Create: `echo_vault_app/lib/models/generated/song/v1/song_service.pbgrpc.dart`
- Create: `echo_vault_app/lib/models/generated/common/v1/types.pb.dart`
- Create: `echo_vault_app/lib/models/generated/sync/v1/sync_service.pb.dart`
- Create: `echo_vault_app/lib/models/generated/user/v1/user_service.pb.dart`

- [ ] **Step 1: Add dependencies to pubspec.yaml**

```yaml
# echo_vault_app/pubspec.yaml — 在 dependencies 末尾追加:
  crypto: ^3.0.6
  file_picker: ^8.1.7
  # proto 生成代码依赖
  fixnum: ^1.1.1
  async: ^2.11.1

# 在 dev_dependencies 末尾追加:
  protoc_plugin: ^21.1.2
```

- [ ] **Step 2: Run flutter pub get**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter pub get
```
Expected: Resolving dependencies... all packages downloaded

- [ ] **Step 3: Install protoc-gen-dart and generate proto Dart code**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault
# 安装 protoc-gen-dart
dart pub global activate protoc_plugin

# 生成 Dart proto 代码
mkdir -p echo_vault_app/lib/models/generated
protoc --dart_out=echo_vault_app/lib/models/generated \
  -I proto \
  proto/echo_vault/common/v1/types.proto \
  proto/echo_vault/user/v1/user_service.proto \
  proto/echo_vault/song/v1/song_service.proto \
  proto/echo_vault/sync/v1/sync_service.proto \
  proto/echo_vault/playlist/v1/playlist_service.proto \
  proto/echo_vault/lyric/v1/lyric_service.proto
```
Expected: Files generated in `echo_vault_app/lib/models/generated/`

Note: If `protoc` is not installed, use an alternative approach — manually create minimal Dart gRPC stubs for the SongService messages used in Phase 7 (see Step 3b fallback).

- [ ] **Step 3b (Fallback if protoc unavailable): Create minimal SongService Dart stubs manually**

Create `echo_vault_app/lib/models/generated/echo_vault/song/v1/song_service.pb.dart`:

```dart
// Auto-generated — minimal stubs for SongService messages
// Regenerate with protoc --dart_out when protoc is available

import 'package:fixnum/fixnum.dart';
import 'package:protobuf/protobuf.dart';

class Song extends GeneratedMessage {
  static final BuilderInfo _i = BuilderInfo('Song')
    ..a<String>(1, 'id', PbFieldType.OS)
    ..a<String>(2, 'title', PbFieldType.OS)
    ..a<String>(3, 'artist', PbFieldType.OS)
    ..a<String>(4, 'album', PbFieldType.OS)
    ..a<String>(5, 'genre', PbFieldType.OS)
    ..a<Int32>(6, 'trackNumber', PbFieldType.O3)
    ..a<Int32>(7, 'discNumber', PbFieldType.O3)
    ..a<Int32>(8, 'durationMs', PbFieldType.O3)
    ..a<Int32>(9, 'year', PbFieldType.O3)
    ..a<String>(10, 'fileName', PbFieldType.OS)
    ..a<Int64>(11, 'fileSize', PbFieldType.O6)
    ..a<String>(12, 'fileHash', PbFieldType.OS)
    ..a<String>(13, 'mimeType', PbFieldType.OS)
    ..a<Int32>(14, 'bitrate', PbFieldType.O3)
    ..a<Int32>(15, 'sampleRate', PbFieldType.O3)
    ..a<Int32>(16, 'source', PbFieldType.O3)
    ..a<Int32>(17, 'fileStatus', PbFieldType.O3)
    ..a<String>(18, 'ownerId', PbFieldType.OS)
    ..a<Int64>(19, 'version', PbFieldType.O6)
    ..a<bool>(20, 'isDeleted', PbFieldType.OB)
    ..a<String>(23, 'coverUrl', PbFieldType.OS)
    ..a<String>(24, 'audioUrl', PbFieldType.OS);

  @override
  BuilderInfo get info_ => _i;

  String get id => $_getSZ(0);
  set id(String v) => $_setString(0, v);
  String get title => $_getSZ(1);
  set title(String v) => $_setString(1, v);
  String get artist => $_getSZ(2);
  set artist(String v) => $_setString(2, v);
  String get album => $_getSZ(3);
  set album(String v) => $_setString(3, v);
  String get genre => $_getSZ(4);
  set genre(String v) => $_setString(4, v);
  int get trackNumber => $_getI(5);
  set trackNumber(int v) => $_setInt(5, v);
  int get discNumber => $_getI(6);
  set discNumber(int v) => $_setInt(6, v);
  int get durationMs => $_getI(7);
  set durationMs(int v) => $_setInt(7, v);
  int get year => $_getI(8);
  set year(int v) => $_setInt(8, v);
  String get fileName => $_getSZ(9);
  set fileName(String v) => $_setString(9, v);
  Int64 get fileSize => $_getI64(10);
  set fileSize(Int64 v) => $_setInt64(10, v);
  String get fileHash => $_getSZ(11);
  set fileHash(String v) => $_setString(11, v);
  String get mimeType => $_getSZ(12);
  set mimeType(String v) => $_setString(12, v);
  String get coverUrl => $_getSZ(18);
  set coverUrl(String v) => $_setString(18, v);
  String get audioUrl => $_getSZ(19);
  set audioUrl(String v) => $_setString(19, v);
}

class CheckSongsByHashRequest extends GeneratedMessage {
  static final BuilderInfo _i = BuilderInfo('CheckSongsByHashRequest')
    ..a<String>(1, 'deviceId', PbFieldType.OS)
    ..p<String>(2, 'fileHashes', PbFieldType.PS);
  @override BuilderInfo get info_ => _i;
  String get deviceId => $_getSZ(0);
  set deviceId(String v) => $_setString(0, v);
  List<String> get fileHashes => $_getList(1);
}

class CheckSongsByHashResponse extends GeneratedMessage {
  static final BuilderInfo _i = BuilderInfo('CheckSongsByHashResponse')
    ..pc<CheckSongsByHashResult>(1, 'results', PbFieldType.PM, CheckSongsByHashResult());
  @override BuilderInfo get info_ => _i;
  List<CheckSongsByHashResult> get results => $_getList(0);
}

class CheckSongsByHashResult extends GeneratedMessage {
  static final BuilderInfo _i = BuilderInfo('CheckSongsByHashResult')
    ..a<String>(1, 'fileHash', PbFieldType.OS)
    ..a<bool>(2, 'exists', PbFieldType.OB)
    ..a<Song>(3, 'song', PbFieldType.OM, Song());
  @override BuilderInfo get info_ => _i;
  String get fileHash => $_getSZ(0);
  set fileHash(String v) => $_setString(0, v);
  bool get exists => $_getB(1);
  set exists(bool v) => $_setBool(1, v);
  Song? get song => $_getN(2);
  set song(Song? v) => $_setN(2, v);
}

class PublishSongRequest extends GeneratedMessage {
  static final BuilderInfo _i = BuilderInfo('PublishSongRequest')
    ..a<String>(1, 'fileHash', PbFieldType.OS)
    ..a<String>(2, 'title', PbFieldType.OS)
    ..a<String>(3, 'artist', PbFieldType.OS)
    ..a<String>(4, 'album', PbFieldType.OS)
    ..a<String>(5, 'genre', PbFieldType.OS)
    ..a<Int32>(6, 'trackNumber', PbFieldType.O3)
    ..a<Int32>(7, 'discNumber', PbFieldType.O3)
    ..a<Int32>(8, 'year', PbFieldType.O3)
    ..a<String>(9, 'fileName', PbFieldType.OS)
    ..a<Int64>(10, 'fileSize', PbFieldType.O6)
    ..a<String>(11, 'mimeType', PbFieldType.OS);
  @override BuilderInfo get info_ => _i;
  String get fileHash => $_getSZ(0);
  set fileHash(String v) => $_setString(0, v);
  String get title => $_getSZ(1);
  set title(String v) => $_setString(1, v);
  String get artist => $_getSZ(2);
  set artist(String v) => $_setString(2, v);
  String get album => $_getSZ(3);
  set album(String v) => $_setString(3, v);
  String get genre => $_getSZ(4);
  set genre(String v) => $_setString(4, v);
  String get fileName => $_getSZ(8);
  set fileName(String v) => $_setString(8, v);
  Int64 get fileSize => $_getI64(9);
  set fileSize(Int64 v) => $_setInt64(9, v);
  String get mimeType => $_getSZ(10);
  set mimeType(String v) => $_setString(10, v);
}

class PublishSongResponse extends GeneratedMessage {
  static final BuilderInfo _i = BuilderInfo('PublishSongResponse')
    ..a<Song>(1, 'song', PbFieldType.OM, Song());
  @override BuilderInfo get info_ => _i;
  Song? get song => $_getN(0);
  set song(Song? v) => $_setN(0, v);
}

class SearchSongsRequest extends GeneratedMessage {
  static final BuilderInfo _i = BuilderInfo('SearchSongsRequest')
    ..a<String>(1, 'query', PbFieldType.OS);
  @override BuilderInfo get info_ => _i;
  String get query => $_getSZ(0);
  set query(String v) => $_setString(0, v);
}

class SearchSongsResponse extends GeneratedMessage {
  static final BuilderInfo _i = BuilderInfo('SearchSongsResponse')
    ..pc<Song>(1, 'songs', PbFieldType.PM, Song());
  @override BuilderInfo get info_ => _i;
  List<Song> get songs => $_getList(0);
}

class ListSongsRequest extends GeneratedMessage {
  static final BuilderInfo _i = BuilderInfo('ListSongsRequest');
  @override BuilderInfo get info_ => _i;
}

class ListSongsResponse extends GeneratedMessage {
  static final BuilderInfo _i = BuilderInfo('ListSongsResponse')
    ..pc<Song>(1, 'songs', PbFieldType.PM, Song());
  @override BuilderInfo get info_ => _i;
  List<Song> get songs => $_getList(0);
}

class GetSongRequest extends GeneratedMessage {
  static final BuilderInfo _i = BuilderInfo('GetSongRequest')
    ..a<String>(1, 'id', PbFieldType.OS);
  @override BuilderInfo get info_ => _i;
  String get id => $_getSZ(0);
  set id(String v) => $_setString(0, v);
}

class GetSongResponse extends GeneratedMessage {
  static final BuilderInfo _i = BuilderInfo('GetSongResponse')
    ..a<Song>(1, 'song', PbFieldType.OM, Song());
  @override BuilderInfo get info_ => _i;
  Song? get song => $_getN(0);
  set song(Song? v) => $_setN(0, v);
}
```

Create `echo_vault_app/lib/models/generated/echo_vault/song/v1/song_service.pbgrpc.dart`:

```dart
// Auto-generated — minimal gRPC stubs for SongService
import 'package:grpc/grpc.dart';
import 'song_service.pb.dart';

class SongServiceClient extends ServiceClient {
  static final _$service = ServiceDescriptor(
    'echo_vault.song.v1.SongService',
    methods: [
      MethodDescriptor(
        'CheckSongsByHash',
        'echo_vault.song.v1.CheckSongsByHashRequest',
        'echo_vault.song.v1.CheckSongsByHashResponse',
        false, false,
        (Object d) => CheckSongsByHashRequest()..mergeFromBuffer(d as List<int>),
        (CheckSongsByHashResponse m) => m.writeToBuffer(),
      ),
      MethodDescriptor(
        'PublishSong',
        'echo_vault.song.v1.PublishSongRequest',
        'echo_vault.song.v1.PublishSongResponse',
        false, false,
        (Object d) => PublishSongRequest()..mergeFromBuffer(d as List<int>),
        (PublishSongResponse m) => m.writeToBuffer(),
      ),
      MethodDescriptor(
        'SearchSongs',
        'echo_vault.song.v1.SearchSongsRequest',
        'echo_vault.song.v1.SearchSongsResponse',
        false, false,
        (Object d) => SearchSongsRequest()..mergeFromBuffer(d as List<int>),
        (SearchSongsResponse m) => m.writeToBuffer(),
      ),
      MethodDescriptor(
        'ListSongs',
        'echo_vault.song.v1.ListSongsRequest',
        'echo_vault.song.v1.ListSongsResponse',
        false, false,
        (Object d) => ListSongsRequest()..mergeFromBuffer(d as List<int>),
        (ListSongsResponse m) => m.writeToBuffer(),
      ),
      MethodDescriptor(
        'GetSong',
        'echo_vault.song.v1.GetSongRequest',
        'echo_vault.song.v1.GetSongResponse',
        false, false,
        (Object d) => GetSongRequest()..mergeFromBuffer(d as List<int>),
        (GetSongResponse m) => m.writeToBuffer(),
      ),
    ],
  );

  SongServiceClient(ClientChannel channel, {CallOptions? options})
      : super(channel, options: options, service: _$service);

  Future<CheckSongsByHashResponse> checkSongsByHash(
      CheckSongsByHashRequest request, {CallOptions? options}) async {
    return $createUnaryCall(_$service.methods[0], request, options: options);
  }

  Future<PublishSongResponse> publishSong(
      PublishSongRequest request, {CallOptions? options}) async {
    return $createUnaryCall(_$service.methods[1], request, options: options);
  }

  Future<SearchSongsResponse> searchSongs(
      SearchSongsRequest request, {CallOptions? options}) async {
    return $createUnaryCall(_$service.methods[2], request, options: options);
  }

  Future<ListSongsResponse> listSongs(
      ListSongsRequest request, {CallOptions? options}) async {
    return $createUnaryCall(_$service.methods[3], request, options: options);
  }

  Future<GetSongResponse> getSong(
      GetSongRequest request, {CallOptions? options}) async {
    return $createUnaryCall(_$service.methods[4], request, options: options);
  }
}
```

- [ ] **Step 4: Commit protobuf generation**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/pubspec.yaml echo_vault_app/lib/models/generated/
git commit -m "feat(flutter): add Dart protobuf stubs for SongService + crypto/file_picker deps"
```

---

### Task 2: ScannedFile Model + File Scanner

**核心逻辑：** `FileScanner` 使用 `dart:io` 递归扫描目录中的音频文件（MP3/FLAC/OGG/M4A/WAV/AAC/WMA/Opus）。对每个文件计算 SHA256 哈希、读取文件大小、提取文件名和扩展名。返回 `ScannedFile` 列表。桌面平台优先，Web 使用 `file_picker` 选择文件。

**Files:**
- Create: `echo_vault_app/lib/features/library/models/scanned_file.dart`
- Create: `echo_vault_app/lib/features/library/services/file_scanner.dart`
- Create: `echo_vault_app/test/features/library/services/file_scanner_test.dart`

- [ ] **Step 1 (RED): 编写 ScannedFile 模型 + FileScanner 测试**

```dart
// lib/features/library/models/scanned_file.dart
class ScannedFile {
  final String path;
  final String fileName;
  final String extension;
  final String fileHash; // SHA256 hex
  final int fileSize;    // bytes

  const ScannedFile({
    required this.path,
    required this.fileName,
    required this.extension,
    required this.fileHash,
    required this.fileSize,
  });

  /// 从 dart:io File 系统创建
  factory ScannedFile.fromFileSystem(String fullPath, String hash) {
    final uri = Uri.file(fullPath);
    final name = uri.pathSegments.last;
    final ext = name.contains('.')
        ? '.${name.split('.').last.toLowerCase()}'
        : '';
    final size = 0; // 实际值在 scanner 中填充
    return ScannedFile(
      path: fullPath,
      fileName: name,
      extension: ext,
      fileHash: hash,
      fileSize: size,
    );
  }

  Map<String, dynamic> toJson() => {
        'path': path,
        'fileName': fileName,
        'extension': extension,
        'fileHash': fileHash,
        'fileSize': fileSize,
      };

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is ScannedFile &&
          runtimeType == other.runtimeType &&
          fileHash == other.fileHash;

  @override
  int get hashCode => fileHash.hashCode;
}
```

```dart
// test/features/library/services/file_scanner_test.dart
import 'dart:convert';
import 'dart:io';
import 'package:flutter_test/flutter_test.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';
import 'package:echo_vault_app/features/library/services/file_scanner.dart';

void main() {
  late Directory tempDir;
  late FileScanner scanner;

  setUp(() {
    tempDir = Directory.systemTemp.createTempSync('echovault_test_');
    scanner = FileScanner();
  });

  tearDown(() {
    tempDir.deleteSync(recursive: true);
  });

  group('FileScanner', () {
    test('scanDirectory returns empty list for empty directory', () async {
      final files = await scanner.scanDirectory(tempDir.path);
      expect(files, isEmpty);
    });

    test('scanDirectory finds MP3 files', () async {
      // 创建测试音频文件
      final mp3File = File('${tempDir.path}/test.mp3');
      await mp3File.writeAsBytes(List.filled(1024, 0x42)); // 1KB dummy MP3

      final files = await scanner.scanDirectory(tempDir.path);
      expect(files.length, 1);
      expect(files.first.fileName, 'test.mp3');
      expect(files.first.fileHash, isNotEmpty);
    });

    test('scanDirectory skips non-audio files', () async {
      await File('${tempDir.path}/readme.txt').writeAsString('hello');
      await File('${tempDir.path}/image.png').writeAsBytes([1, 2, 3]);

      final files = await scanner.scanDirectory(tempDir.path);
      expect(files, isEmpty);
    });

    test('scanDirectory finds multiple audio formats', () async {
      for (final ext in ['.mp3', '.flac', '.ogg', '.m4a', '.wav', '.opus']) {
        await File('${tempDir.path}/song$ext')
            .writeAsBytes(List.filled(100, 0x42));
      }
      // Also create a non-audio file
      await File('${tempDir.path}/cover.jpg').writeAsBytes([1, 2, 3]);

      final files = await scanner.scanDirectory(tempDir.path);
      expect(files.length, 6);
    });

    test('scanDirectory searches recursively', () async {
      final subDir = Directory('${tempDir.path}/subfolder');
      subDir.createSync();
      await File('${subDir.path}/deep.mp3').writeAsBytes(List.filled(100, 0x42));
      await File('${tempDir.path}/root.mp3').writeAsBytes(List.filled(100, 0x42));

      final files = await scanner.scanDirectory(tempDir.path);
      expect(files.length, 2);
    });

    test('computeSha256 returns consistent hash', () async {
      final file = File('${tempDir.path}/test.mp3');
      await file.writeAsBytes([0x01, 0x02, 0x03]);

      final hash1 = await scanner.computeSha256(file.path);
      final hash2 = await scanner.computeSha256(file.path);
      expect(hash1, hash2);
      expect(hash1.length, 64); // SHA256 hex = 64 chars
    });

    test('computeSha256 different files have different hashes', () async {
      await File('${tempDir.path}/a.mp3').writeAsBytes([0x01]);
      await File('${tempDir.path}/b.mp3').writeAsBytes([0x02]);

      final hashA = await scanner.computeSha256('${tempDir.path}/a.mp3');
      final hashB = await scanner.computeSha256('${tempDir.path}/b.mp3');
      expect(hashA, isNot(equals(hashB)));
    });
  });
}
```

- [ ] **Step 2 (VERIFY RED): Run test to confirm compilation failure**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/services/file_scanner_test.dart 2>&1 | tail -20
```
Expected: Compilation error — `FileScanner` not found, or `crypto` import fails if package not added

- [ ] **Step 3 (GREEN): 实现 FileScanner**

```dart
// lib/features/library/services/file_scanner.dart
import 'dart:convert';
import 'dart:io';
import 'package:crypto/crypto.dart';
import '../models/scanned_file.dart';

class FileScanner {
  /// 支持的音频扩展名（小写，含点）
  static const Set<String> audioExtensions = {
    '.mp3', '.flac', '.ogg', '.m4a', '.mp4',
    '.wav', '.aac', '.wma', '.opus',
  };

  /// 递归扫描目录中的所有音频文件
  Future<List<ScannedFile>> scanDirectory(String dirPath) async {
    final dir = Directory(dirPath);
    if (!await dir.exists()) return [];

    final files = <ScannedFile>[];
    await for (final entity in dir.list(recursive: true, followLinks: false)) {
      if (entity is File && _isAudioFile(entity.path)) {
        final hash = await computeSha256(entity.path);
        final stat = await entity.stat();
        files.add(ScannedFile(
          path: entity.path,
          fileName: entity.uri.pathSegments.last,
          extension: _extension(entity.path),
          fileHash: hash,
          fileSize: stat.size,
        ));
      }
    }
    return files;
  }

  /// 计算文件的 SHA256 哈希
  Future<String> computeSha256(String filePath) async {
    final file = File(filePath);
    final bytes = await file.readAsBytes();
    final digest = sha256.convert(bytes);
    return digest.toString();
  }

  bool _isAudioFile(String path) {
    return audioExtensions.contains(_extension(path));
  }

  String _extension(String path) {
    final name = path.split(Platform.pathSeparator).last;
    final dotIndex = name.lastIndexOf('.');
    if (dotIndex == -1) return '';
    return name.substring(dotIndex).toLowerCase();
  }
}
```

- [ ] **Step 4 (VERIFY GREEN): Run test to confirm all pass**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/services/file_scanner_test.dart -v
```
Expected: 7 tests, all PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/models/scanned_file.dart \
       echo_vault_app/lib/features/library/services/file_scanner.dart \
       echo_vault_app/test/features/library/services/file_scanner_test.dart
git commit -m "feat(flutter): add FileScanner with SHA256 hash and recursive scan (TDD, 7 tests)"
```

---

### Task 3: SongRepository — gRPC SongService 封装

**核心逻辑：** `SongRepository` 封装 `SongServiceClient` 的 gRPC 调用：CheckSongsByHash（批量查云端）、PublishSong（发布歌曲）、SearchSongs/ListSongs（搜索和列表）。通过 `GrpcClientManager` 获取 channel 和 JWT metadata。

**Files:**
- Modify: `echo_vault_app/lib/core/grpc/client.dart` — 追加 createSongService 方法
- Create: `echo_vault_app/lib/features/library/services/song_repository.dart`
- Create: `echo_vault_app/test/features/library/services/song_repository_test.dart`

- [ ] **Step 1 (RED): 编写 SongRepository 测试**

```dart
// test/features/library/services/song_repository_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:echo_vault_app/core/grpc/client.dart';
import 'package:echo_vault_app/features/library/services/song_repository.dart';

void main() {
  group('SongRepository', () {
    late GrpcClientManager grpcManager;
    late SongRepository repository;

    setUp(() {
      grpcManager = GrpcClientManager();
      repository = SongRepository(grpcManager);
    });

    test('checkSongsByHash throws when not connected', () async {
      expect(
        () => repository.checkSongsByHash(deviceId: 'dev', fileHashes: ['abc']),
        throwsA(isA<Exception>()),
      );
    });

    test('publishSong throws when not connected', () async {
      expect(
        () => repository.publishSong(
          fileHash: 'abc',
          title: 'Test',
          artist: 'Me',
          album: '',
          genre: '',
          trackNumber: 0,
          discNumber: 0,
          year: 2024,
          fileName: 'test.mp3',
          fileSize: 1000,
          mimeType: 'audio/mpeg',
        ),
        throwsA(isA<Exception>()),
      );
    });

    test('searchSongs throws when not connected', () async {
      expect(
        () => repository.searchSongs(query: 'test'),
        throwsA(isA<Exception>()),
      );
    });

    test('listSongs returns empty list when not connected', () async {
      final songs = await repository.listSongs();
      expect(songs, isEmpty);
    });
  });
}
```

- [ ] **Step 2 (VERIFY RED)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/services/song_repository_test.dart 2>&1 | tail -20
```
Expected: Compilation error — `SongRepository` not found

- [ ] **Step 3 (GREEN): 实现 SongRepository**

```dart
// lib/features/library/services/song_repository.dart
import 'dart:math';
import 'package:echo_vault_app/core/grpc/client.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pbgrpc.dart';

class SongRepository {
  final GrpcClientManager _grpcManager;
  SongServiceClient? _client;

  SongRepository(this._grpcManager);

  SongServiceClient _getClient() {
    if (_client == null) {
      _client = SongServiceClient(
        _grpcManager.channel,
        options: CallOptions(metadata: _grpcManager.metadata),
      );
    }
    return _client!;
  }

  /// 批量查询文件哈希在服务端是否存在
  Future<List<CheckSongsByHashResult>> checkSongsByHash({
    required String deviceId,
    required List<String> fileHashes,
  }) async {
    final client = _getClient();
    final request = CheckSongsByHashRequest()
      ..deviceId = deviceId
      ..fileHashes.addAll(fileHashes);
    final response = await client.checkSongsByHash(request);
    return response.results;
  }

  /// 发布新歌曲到服务端
  Future<Song> publishSong({
    required String fileHash,
    required String title,
    required String artist,
    required String album,
    required String genre,
    required int trackNumber,
    required int discNumber,
    required int year,
    required String fileName,
    required int fileSize,
    required String mimeType,
  }) async {
    final client = _getClient();
    final request = PublishSongRequest()
      ..fileHash = fileHash
      ..title = title
      ..artist = artist
      ..album = album
      ..genre = genre
      ..trackNumber = trackNumber
      ..discNumber = discNumber
      ..year = year
      ..fileName = fileName
      ..fileSize = Int64(fileSize)
      ..mimeType = mimeType;
    final response = await client.publishSong(request);
    if (response.song == null) {
      throw Exception('PublishSong returned null');
    }
    return response.song!;
  }

  /// 搜索歌曲
  Future<List<Song>> searchSongs({required String query}) async {
    final client = _getClient();
    final request = SearchSongsRequest()..query = query;
    final response = await client.searchSongs(request);
    return response.songs;
  }

  /// 获取歌曲列表
  Future<List<Song>> listSongs() async {
    try {
      final client = _getClient();
      final request = ListSongsRequest();
      final response = await client.listSongs(request);
      return response.songs;
    } catch (_) {
      return [];
    }
  }

  /// 获取单首歌曲
  Future<Song?> getSong(String id) async {
    try {
      final client = _getClient();
      final request = GetSongRequest()..id = id;
      final response = await client.getSong(request);
      return response.song;
    } catch (_) {
      return null;
    }
  }
}
```

Modify `echo_vault_app/lib/core/grpc/client.dart` — 目前已有 `GrpcClientManager`，无需额外修改（`channel` 和 `metadata` 已暴露）。如果 metadata 尚未公开为 getter，确保修改：

```dart
// lib/core/grpc/client.dart — 确认这些 getter 存在
class GrpcClientManager {
  // ...existing code...

  /// 当前 JWT metadata
  Map<String, String> get metadata => _accessToken != null && _accessToken!.isNotEmpty
      ? {'authorization': 'Bearer $_accessToken'}
      : {};

  /// gRPC channel
  ClientChannel get channel => _channel;
}
```

- [ ] **Step 4 (VERIFY GREEN)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/services/song_repository_test.dart -v
```
Expected: 4 tests, all PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/services/song_repository.dart \
       echo_vault_app/test/features/library/services/song_repository_test.dart
git commit -m "feat(flutter): add SongRepository with gRPC wrappers (TDD, 4 tests)"
```

---

### Task 4: RestClient — Dio 文件上传封装

**核心逻辑：** `RestClient` 使用 Dio 调用后端 REST 文件服务。支持上传音频文件（自动提取元数据 → 补充字段）、上传封面、下载封面/音频。从 `ServerConfig` 读取 REST 地址。

**Files:**
- Create: `echo_vault_app/lib/features/library/services/rest_client.dart`
- Create: `echo_vault_app/test/features/library/services/rest_client_test.dart`

- [ ] **Step 1 (RED): 编写 RestClient 测试**

```dart
// test/features/library/services/rest_client_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:echo_vault_app/features/library/services/rest_client.dart';

void main() {
  late RestClient restClient;

  setUp(() {
    restClient = RestClient();
  });

  test('constructor creates instance', () {
    expect(restClient, isNotNull);
  });

  test('uploadAudio throws without server config', () async {
    // 无服务器配置时应抛出异常
    expect(
      () => restClient.uploadAudio(
        songId: 'test-id',
        filePath: '/tmp/test.mp3',
        fileName: 'test.mp3',
      ),
      throwsException,
    );
  });

  test('uploadCover throws without server config', () async {
    expect(
      () => restClient.uploadCover(
        songId: 'test-id',
        filePath: '/tmp/cover.jpg',
      ),
      throwsException,
    );
  });

  test('buildAudioUrl returns expected URL', () {
    final url = restClient.buildAudioUrl(songId: 'abc-123', baseUrl: 'http://localhost:9091');
    expect(url, 'http://localhost:9091/api/v1/files/download/audio/abc-123');
  });

  test('buildCoverUrl returns expected URL', () {
    final url = restClient.buildCoverUrl(songId: 'abc-123', baseUrl: 'http://localhost:9091');
    expect(url, 'http://localhost:9091/api/v1/files/download/cover/abc-123');
  });
}
```

- [ ] **Step 2 (VERIFY RED)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/services/rest_client_test.dart 2>&1 | tail -20
```
Expected: Compilation error

- [ ] **Step 3 (GREEN): 实现 RestClient**

```dart
// lib/features/library/services/rest_client.dart
import 'dart:io';
import 'package:dio/dio.dart';
import 'package:echo_vault_app/core/config/server_config.dart';

class RestClient {
  Dio? _dio;

  Dio _getDio() {
    if (_dio == null) {
      _dio = Dio(BaseOptions(
        connectTimeout: const Duration(seconds: 30),
        receiveTimeout: const Duration(seconds: 60),
        sendTimeout: const Duration(seconds: 120),
      ));
    }
    return _dio!;
  }

  Future<String> _getBaseUrl() async {
    final config = await ServerConfig.getConfig();
    if (config.host.isEmpty) {
      throw Exception('Server not configured');
    }
    return 'http://${config.host}:${config.restPort}';
  }

  /// 上传音频文件到服务器
  /// 服务器会自动解析元数据并补充 Song 空字段 + 保存嵌入封面
  Future<void> uploadAudio({
    required String songId,
    required String filePath,
    required String fileName,
  }) async {
    final baseUrl = await _getBaseUrl();
    final dio = _getDio();
    final formData = FormData.fromMap({
      'file': await MultipartFile.fromFile(filePath, filename: fileName),
    });
    await dio.post(
      '$baseUrl/api/v1/files/upload',
      queryParameters: {'type': 'audio', 'song_id': songId},
      data: formData,
    );
  }

  /// 上传封面图片
  Future<void> uploadCover({
    required String songId,
    required String filePath,
  }) async {
    final baseUrl = await _getBaseUrl();
    final dio = _getDio();
    final formData = FormData.fromMap({
      'file': await MultipartFile.fromFile(filePath),
    });
    await dio.post(
      '$baseUrl/api/v1/files/upload',
      queryParameters: {'type': 'cover', 'song_id': songId},
      data: formData,
    );
  }

  /// 构建音频下载 URL（给 Song.audioUrl 字段使用）
  String buildAudioUrl({required String songId, required String baseUrl}) {
    return '$baseUrl/api/v1/files/download/audio/$songId';
  }

  /// 构建封面下载 URL（给 Song.coverUrl 字段使用）
  String buildCoverUrl({required String songId, required String baseUrl}) {
    return '$baseUrl/api/v1/files/download/cover/$songId';
  }
}
```

- [ ] **Step 4 (VERIFY GREEN)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/services/rest_client_test.dart -v
```
Expected: 5 tests, all PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/services/rest_client.dart \
       echo_vault_app/test/features/library/services/rest_client_test.dart
git commit -m "feat(flutter): add RestClient with Dio file upload/download (TDD, 5 tests)"
```

---

### Task 5: ScannerProvider — 扫描流程状态管理

**核心逻辑：** `ScannerNotifier` 管理扫描流程状态：空闲 → 扫描中 → 完成。扫描完成后持有 `ScannedFile` 列表，并调用 `SongRepository.checkSongsByHash` 比对云端，标记哪些已存在。

**Files:**
- Create: `echo_vault_app/lib/features/library/providers/scanner_provider.dart`
- Create: `echo_vault_app/test/features/library/providers/scanner_provider_test.dart`

- [ ] **Step 1 (RED): 编写 ScannerProvider 测试**

```dart
// test/features/library/providers/scanner_provider_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';
import 'package:echo_vault_app/features/library/providers/scanner_provider.dart';

void main() {
  group('ScannerNotifier', () {
    late ProviderContainer container;

    setUp(() {
      container = ProviderContainer();
    });

    tearDown(() {
      container.dispose();
    });

    test('initial state is idle', () {
      final state = container.read(scannerProvider);
      expect(state.status, ScannerStatus.idle);
      expect(state.scannedFiles, isEmpty);
      expect(state.error, isNull);
    });

    test('setScannedFiles updates files', () {
      final notifier = container.read(scannerProvider.notifier);
      final files = [
        ScannedFile(
          path: '/tmp/a.mp3', fileName: 'a.mp3',
          extension: '.mp3', fileHash: 'abc', fileSize: 1000,
        ),
      ];
      notifier.setScannedFiles(files);
      final state = container.read(scannerProvider);
      expect(state.scannedFiles.length, 1);
      expect(state.status, ScannerStatus.completed);
    });

    test('reset clears state', () {
      final notifier = container.read(scannerProvider.notifier);
      notifier.setScannedFiles([
        ScannedFile(
          path: '/tmp/a.mp3', fileName: 'a.mp3',
          extension: '.mp3', fileHash: 'abc', fileSize: 1000,
        ),
      ]);
      notifier.reset();
      final state = container.read(scannerProvider);
      expect(state.scannedFiles, isEmpty);
      expect(state.status, ScannerStatus.idle);
    });

    test('setError captures error message', () {
      final notifier = container.read(scannerProvider.notifier);
      notifier.setError('Permission denied');
      final state = container.read(scannerProvider);
      expect(state.error, 'Permission denied');
      expect(state.status, ScannerStatus.error);
    });
  });
}
```

- [ ] **Step 2 (VERIFY RED)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/providers/scanner_provider_test.dart 2>&1 | tail -20
```
Expected: Compilation error — `ScannerStatus`, `ScannerState`, `scannerProvider` not found

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

  const ScannerState({
    this.status = ScannerStatus.idle,
    this.scannedFiles = const [],
    this.error,
  });
}

class ScannerNotifier extends StateNotifier<ScannerState> {
  ScannerNotifier() : super(const ScannerState());

  void setScanning() {
    state = const ScannerState(status: ScannerStatus.scanning);
  }

  void setScannedFiles(List<ScannedFile> files) {
    state = ScannerState(
      status: ScannerStatus.completed,
      scannedFiles: files,
    );
  }

  void setError(String message) {
    state = ScannerState(status: ScannerStatus.error, error: message);
  }

  void reset() {
    state = const ScannerState();
  }
}

final scannerProvider = StateNotifierProvider<ScannerNotifier, ScannerState>((ref) {
  return ScannerNotifier();
});
```

- [ ] **Step 4 (VERIFY GREEN)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/providers/scanner_provider_test.dart -v
```
Expected: 4 tests, all PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/providers/scanner_provider.dart \
       echo_vault_app/test/features/library/providers/scanner_provider_test.dart
git commit -m "feat(flutter): add ScannerProvider state management (TDD, 4 tests)"
```

---

### Task 6: LibraryProvider — 歌库列表状态管理

**核心逻辑：** `LibraryNotifier` 管理歌曲列表状态：加载中 → 已完成。从 `SongRepository.listSongs()` 拉取云端歌曲，支持搜索（调用 `searchSongs`）、刷新列表。

**Files:**
- Create: `echo_vault_app/lib/features/library/providers/library_provider.dart`
- Create: `echo_vault_app/test/features/library/providers/library_provider_test.dart`

- [ ] **Step 1 (RED): 编写 LibraryProvider 测试**

```dart
// test/features/library/providers/library_provider_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/providers/library_provider.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';

void main() {
  group('LibraryNotifier', () {
    late ProviderContainer container;

    setUp(() {
      container = ProviderContainer();
    });

    tearDown(() {
      container.dispose();
    });

    test('initial state is idle', () {
      final state = container.read(libraryProvider);
      expect(state.status, LibraryStatus.idle);
      expect(state.songs, isEmpty);
      expect(state.searchQuery, '');
    });

    test('setSearchQuery updates query', () {
      final notifier = container.read(libraryProvider.notifier);
      notifier.setSearchQuery('summer');
      final state = container.read(libraryProvider);
      expect(state.searchQuery, 'summer');
    });

    test('setSearchQuery empty clears', () {
      final notifier = container.read(libraryProvider.notifier);
      notifier.setSearchQuery('old');
      notifier.setSearchQuery('');
      final state = container.read(libraryProvider);
      expect(state.searchQuery, '');
    });

    test('setSongs updates list', () {
      final notifier = container.read(libraryProvider.notifier);
      final song = Song()..id = '1'..title = 'Test';
      notifier.setSongs([song]);
      final state = container.read(libraryProvider);
      expect(state.songs.length, 1);
      expect(state.status, LibraryStatus.completed);
    });

    test('reset clears everything', () {
      final notifier = container.read(libraryProvider.notifier);
      final song = Song()..id = '1'..title = 'Test';
      notifier.setSongs([song]);
      notifier.setSearchQuery('test');
      notifier.reset();
      final state = container.read(libraryProvider);
      expect(state.songs, isEmpty);
      expect(state.searchQuery, '');
      expect(state.status, LibraryStatus.idle);
    });
  });
}
```

- [ ] **Step 2 (VERIFY RED)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/providers/library_provider_test.dart 2>&1 | tail -20
```
Expected: Compilation error

- [ ] **Step 3 (GREEN): 实现 LibraryProvider**

```dart
// lib/features/library/providers/library_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';

enum LibraryStatus { idle, loading, completed, error }

class LibraryState {
  final LibraryStatus status;
  final List<Song> songs;
  final String searchQuery;
  final String? error;

  const LibraryState({
    this.status = LibraryStatus.idle,
    this.songs = const [],
    this.searchQuery = '',
    this.error,
  });
}

class LibraryNotifier extends StateNotifier<LibraryState> {
  LibraryNotifier() : super(const LibraryState());

  void setLoading() {
    state = const LibraryState(status: LibraryStatus.loading);
  }

  void setSongs(List<Song> songs) {
    state = LibraryState(
      status: LibraryStatus.completed,
      songs: songs,
      searchQuery: state.searchQuery,
    );
  }

  void setSearchQuery(String query) {
    state = LibraryState(
      status: state.status,
      songs: state.songs,
      searchQuery: query,
    );
  }

  void setError(String message) {
    state = LibraryState(
      status: LibraryStatus.error,
      error: message,
      searchQuery: state.searchQuery,
    );
  }

  void reset() {
    state = const LibraryState();
  }
}

final libraryProvider = StateNotifierProvider<LibraryNotifier, LibraryState>((ref) {
  return LibraryNotifier();
});
```

- [ ] **Step 4 (VERIFY GREEN)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/providers/library_provider_test.dart -v
```
Expected: 5 tests, all PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/providers/library_provider.dart \
       echo_vault_app/test/features/library/providers/library_provider_test.dart
git commit -m "feat(flutter): add LibraryProvider state management (TDD, 5 tests)"
```

---

### Task 7: SongListTile Widget — 歌曲列表行组件

**核心逻辑：** 显示单首歌曲的封面缩略图、标题、艺术家、专辑。支持点击进入详情（后续 Phase 8 播放器），支持长按操作菜单。

**Files:**
- Create: `echo_vault_app/lib/features/library/widgets/song_list_tile.dart`
- Create: `echo_vault_app/test/features/library/widgets/song_list_tile_test.dart`

- [ ] **Step 1 (RED): 编写 SongListTile widget test**

```dart
// test/features/library/widgets/song_list_tile_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:echo_vault_app/features/library/widgets/song_list_tile.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';

void main() {
  testWidgets('renders song title and artist', (tester) async {
    final song = Song()
      ..id = '1'
      ..title = 'Summer Nights'
      ..artist = 'Test Artist'
      ..album = 'Summer Album';

    await tester.pumpWidget(MaterialApp(
      home: Scaffold(body: SongListTile(song: song, onTap: () {})),
    ));

    expect(find.text('Summer Nights'), findsOneWidget);
    expect(find.text('Test Artist'), findsOneWidget);
  });

  testWidgets('shows album when no artist', (tester) async {
    final song = Song()
      ..id = '2'
      ..title = 'Alone'
      ..album = 'Silent Album';

    await tester.pumpWidget(MaterialApp(
      home: Scaffold(body: SongListTile(song: song, onTap: () {})),
    ));

    expect(find.text('Alone'), findsOneWidget);
    expect(find.text('Silent Album'), findsOneWidget);
  });

  testWidgets('responds to tap', (tester) async {
    final song = Song()..id = '3'..title = 'Tap Me';
    bool tapped = false;

    await tester.pumpWidget(MaterialApp(
      home: Scaffold(body: SongListTile(
        song: song,
        onTap: () => tapped = true,
      )),
    ));

    await tester.tap(find.text('Tap Me'));
    expect(tapped, isTrue);
  });

  testWidgets('shows placeholder icon when no cover', (tester) async {
    final song = Song()..id = '4'..title = 'No Cover';

    await tester.pumpWidget(MaterialApp(
      home: Scaffold(body: SongListTile(song: song, onTap: () {})),
    ));

    expect(find.byIcon(Icons.music_note), findsOneWidget);
  });
}
```

- [ ] **Step 2 (VERIFY RED)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/widgets/song_list_tile_test.dart 2>&1 | tail -20
```
Expected: Compilation error

- [ ] **Step 3 (GREEN): 实现 SongListTile**

```dart
// lib/features/library/widgets/song_list_tile.dart
import 'package:flutter/material.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';

class SongListTile extends StatelessWidget {
  final Song song;
  final VoidCallback onTap;
  final VoidCallback? onLongPress;

  const SongListTile({
    super.key,
    required this.song,
    required this.onTap,
    this.onLongPress,
  });

  @override
  Widget build(BuildContext context) {
    final subtitle = song.artist.isNotEmpty ? song.artist : song.album;
    final theme = Theme.of(context);

    return ListTile(
      leading: CircleAvatar(
        backgroundColor: theme.colorScheme.surfaceContainerHighest,
        child: Icon(Icons.music_note, color: theme.colorScheme.primary),
      ),
      title: Text(
        song.title,
        maxLines: 1,
        overflow: TextOverflow.ellipsis,
        style: theme.textTheme.bodyLarge?.copyWith(fontWeight: FontWeight.w500),
      ),
      subtitle: subtitle.isNotEmpty
          ? Text(subtitle, maxLines: 1, overflow: TextOverflow.ellipsis)
          : null,
      trailing: const Icon(Icons.play_arrow, size: 20),
      onTap: onTap,
      onLongPress: onLongPress,
    );
  }
}
```

- [ ] **Step 4 (VERIFY GREEN)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/widgets/song_list_tile_test.dart -v
```
Expected: 4 tests, all PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/widgets/song_list_tile.dart \
       echo_vault_app/test/features/library/widgets/song_list_tile_test.dart
git commit -m "feat(flutter): add SongListTile widget (TDD, 4 widget tests)"
```

---

### Task 8: LibraryPage — 歌曲库主页

**核心逻辑：** LibraryPage 显示歌曲列表，支持搜索。顶部搜索栏 → 搜索框输入触发 `searchSongs` 或 `listSongs`。底部 FAB 按钮启动扫描流程。使用 `SongListTile` 渲染每行。

**Files:**
- Create: `echo_vault_app/lib/features/library/pages/library_page.dart`
- Create: `echo_vault_app/test/features/library/pages/library_page_test.dart`

- [ ] **Step 1 (RED): 编写 LibraryPage widget test**

```dart
// test/features/library/pages/library_page_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/pages/library_page.dart';

void main() {
  testWidgets('shows empty state when no songs', (tester) async {
    await tester.pumpWidget(ProviderScope(
      child: MaterialApp(home: LibraryPage()),
    ));
    await tester.pumpAndSettle();

    expect(find.text('歌曲库'), findsOneWidget);
    expect(find.byIcon(Icons.search), findsOneWidget);
    expect(find.byType(FloatingActionButton), findsOneWidget);
  });

  testWidgets('search field exists and can type', (tester) async {
    await tester.pumpWidget(ProviderScope(
      child: MaterialApp(home: LibraryPage()),
    ));
    await tester.pumpAndSettle();

    final searchField = find.byType(TextField);
    expect(searchField, findsOneWidget);

    await tester.enterText(searchField, 'summer');
    await tester.pumpAndSettle();

    expect(find.text('summer'), findsOneWidget);
  });
}
```

- [ ] **Step 2 (VERIFY RED)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/pages/library_page_test.dart 2>&1 | tail -20
```
Expected: Compilation error

- [ ] **Step 3 (GREEN): 实现 LibraryPage**

```dart
// lib/features/library/pages/library_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:echo_vault_app/providers/auth_provider.dart';
import 'package:echo_vault_app/features/library/providers/library_provider.dart';
import 'package:echo_vault_app/features/library/widgets/song_list_tile.dart';

class LibraryPage extends ConsumerWidget {
  const LibraryPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final libraryState = ref.watch(libraryProvider);
    final authState = ref.watch(authProvider);
    final theme = Theme.of(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text('歌曲库'),
        actions: [
          if (authState.status == AuthStatus.authenticated)
            IconButton(
              icon: const Icon(Icons.logout),
              tooltip: '退出登录',
              onPressed: () {
                ref.read(authProvider.notifier).logout();
                context.go('/login');
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
                hintText: '搜索歌曲、艺术家...',
                prefixIcon: const Icon(Icons.search),
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(12),
                ),
                filled: true,
                fillColor: theme.colorScheme.surfaceContainerLow,
              ),
              onChanged: (query) {
                ref.read(libraryProvider.notifier).setSearchQuery(query);
              },
            ),
          ),
          // 歌曲列表
          Expanded(
            child: libraryState.status == LibraryStatus.loading
                ? const Center(child: CircularProgressIndicator())
                : libraryState.songs.isEmpty
                    ? _buildEmptyState(theme)
                    : ListView.builder(
                        itemCount: libraryState.songs.length,
                        itemBuilder: (context, index) {
                          final song = libraryState.songs[index];
                          return SongListTile(
                            song: song,
                            onTap: () {
                              // Phase 8: 播放器
                            },
                          );
                        },
                      ),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          // Phase 7 Task 9: 跳转到扫描页
          context.push('/scan');
        },
        child: const Icon(Icons.add),
      ),
    );
  }

  Widget _buildEmptyState(ThemeData theme) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(Icons.library_music, size: 64, color: theme.colorScheme.outline),
          const SizedBox(height: 16),
          Text('还没有歌曲', style: theme.textTheme.titleMedium),
          const SizedBox(height: 8),
          Text('点击右下角 + 按钮扫描本地音乐文件',
              style: theme.textTheme.bodyMedium?.copyWith(
                color: theme.colorScheme.outline,
              )),
        ],
      ),
    );
  }
}
```

- [ ] **Step 4 (VERIFY GREEN)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/library/pages/library_page_test.dart -v
```
Expected: 2 tests, all PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/library/pages/library_page.dart \
       echo_vault_app/test/features/library/pages/library_page_test.dart
git commit -m "feat(flutter): add LibraryPage with search bar (TDD, 2 widget tests)"
```

---

### Task 9: ScanResultPage — 扫描结果 + 云端比对

**核心逻辑：** 扫描完成后显示结果列表，每首歌曲标注"云端已有 ✅"或"新文件 ⬆️"。用户勾选要上传的歌曲，点击"发布选中歌曲"进入元数据编辑。比对通过 `SongRepository.checkSongsByHash` 完成。

**Files:**
- Create: `echo_vault_app/lib/features/publish/pages/scan_result_page.dart`
- Create: `echo_vault_app/test/features/publish/pages/scan_result_page_test.dart`

- [ ] **Step 1 (RED): 编写 ScanResultPage test**

```dart
// test/features/publish/pages/scan_result_page_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/publish/pages/scan_result_page.dart';

void main() {
  testWidgets('shows scanner idle state', (tester) async {
    await tester.pumpWidget(ProviderScope(
      child: const MaterialApp(home: ScanResultPage()),
    ));
    await tester.pumpAndSettle();

    expect(find.text('扫描本地音乐'), findsOneWidget);
    expect(find.text('选择文件夹'), findsOneWidget);
  });

  testWidgets('has select folder button', (tester) async {
    await tester.pumpWidget(ProviderScope(
      child: const MaterialApp(home: ScanResultPage()),
    ));
    await tester.pumpAndSettle();

    expect(find.byType(ElevatedButton), findsAtLeast(1));
  });
}
```

- [ ] **Step 2 (VERIFY RED)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/publish/pages/scan_result_page_test.dart 2>&1 | tail -20
```
Expected: Compilation error

- [ ] **Step 3 (GREEN): 实现 ScanResultPage**

```dart
// lib/features/publish/pages/scan_result_page.dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:file_picker/file_picker.dart';
import 'package:echo_vault_app/features/library/providers/scanner_provider.dart';
import 'package:echo_vault_app/features/library/services/file_scanner.dart';

class ScanResultPage extends ConsumerWidget {
  const ScanResultPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final scanState = ref.watch(scannerProvider);
    final theme = Theme.of(context);

    return Scaffold(
      appBar: AppBar(title: const Text('扫描本地音乐')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            // 扫描状态指示
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Column(
                  children: [
                    Icon(
                      scanState.status == ScannerStatus.completed
                          ? Icons.check_circle
                          : scanState.status == ScannerStatus.error
                              ? Icons.error
                              : Icons.folder_open,
                      size: 48,
                      color: scanState.status == ScannerStatus.completed
                          ? Colors.green
                          : scanState.status == ScannerStatus.error
                              ? Colors.red
                              : theme.colorScheme.primary,
                    ),
                    const SizedBox(height: 12),
                    Text(
                      scanState.status == ScannerStatus.completed
                          ? '找到 ${scanState.scannedFiles.length} 首歌曲'
                          : scanState.status == ScannerStatus.error
                              ? scanState.error ?? '扫描出错'
                              : scanState.status == ScannerStatus.scanning
                                  ? '正在扫描...'
                                  : '选择音乐文件夹',
                      style: theme.textTheme.titleMedium,
                    ),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 16),
            // 文件列表预览
            Expanded(
              child: scanState.scannedFiles.isEmpty
                  ? Center(
                      child: Text(
                        scanState.status == ScannerStatus.idle
                            ? '点击下方按钮选择音乐文件夹'
                            : '未找到音频文件',
                        style: theme.textTheme.bodyMedium?.copyWith(
                          color: theme.colorScheme.outline,
                        ),
                      ),
                    )
                  : ListView.builder(
                      itemCount: scanState.scannedFiles.length,
                      itemBuilder: (context, index) {
                        final file = scanState.scannedFiles[index];
                        return ListTile(
                          leading: const Icon(Icons.audiotrack),
                          title: Text(file.fileName),
                          subtitle: Text(
                            '${(file.fileSize / 1024 / 1024).toStringAsFixed(1)} MB',
                          ),
                          trailing: Text(
                            file.extension,
                            style: TextStyle(color: theme.colorScheme.primary),
                          ),
                        );
                      },
                    ),
            ),
            const SizedBox(height: 16),
            // 操作按钮
            SizedBox(
              width: double.infinity,
              child: ElevatedButton.icon(
                onPressed: () => _pickFolder(context, ref),
                icon: const Icon(Icons.folder_open),
                label: Text(
                  scanState.scannedFiles.isEmpty
                      ? '选择文件夹'
                      : '重新选择文件夹',
                ),
              ),
            ),
            if (scanState.scannedFiles.isNotEmpty) ...[
              const SizedBox(height: 8),
              SizedBox(
                width: double.infinity,
                child: FilledButton.icon(
                  onPressed: () {
                    // 跳转到元数据编辑页（Task 10）
                    context.push('/publish', extra: scanState.scannedFiles);
                  },
                  icon: const Icon(Icons.cloud_upload),
                  label: Text('发布 ${scanState.scannedFiles.length} 首歌曲'),
                ),
              ),
            ],
          ],
        ),
      ),
    );
  }

  Future<void> _pickFolder(BuildContext context, WidgetRef ref) async {
    final result = await FilePicker.platform.getDirectoryPath();
    if (result == null) return;

    final notifier = ref.read(scannerProvider.notifier);
    notifier.reset();
    notifier.setScanning();

    try {
      final scanner = FileScanner();
      final files = await scanner.scanDirectory(result);
      notifier.setScannedFiles(files);
    } catch (e) {
      notifier.setError(e.toString());
    }
  }
}
```

- [ ] **Step 4 (VERIFY GREEN)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/publish/pages/scan_result_page_test.dart -v
```
Expected: 2 tests, all PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/publish/pages/scan_result_page.dart \
       echo_vault_app/test/features/publish/pages/scan_result_page_test.dart
git commit -m "feat(flutter): add ScanResultPage with file picker (TDD, 2 widget tests)"
```

---

### Task 10: EditMetadataPage — 元数据编辑页

**核心逻辑：** 用户发布前可编辑歌曲元数据（标题、艺术家、专辑、流派、年份）。支持逐首编辑或批量应用（如统一艺术家名）。编辑后调用 `PublishSong` gRPC + REST 上传音频文件。

**Files:**
- Create: `echo_vault_app/lib/features/publish/pages/edit_metadata_page.dart`
- Create: `echo_vault_app/test/features/publish/pages/edit_metadata_page_test.dart`

- [ ] **Step 1 (RED): 编写 EditMetadataPage test**

```dart
// test/features/publish/pages/edit_metadata_page_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/publish/pages/edit_metadata_page.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';

void main() {
  final testFiles = [
    ScannedFile(
      path: '/tmp/song.mp3', fileName: 'song.mp3',
      extension: '.mp3', fileHash: 'abc123', fileSize: 1024,
    ),
  ];

  testWidgets('shows metadata form fields', (tester) async {
    await tester.pumpWidget(ProviderScope(
      child: MaterialApp(
        home: EditMetadataPage(scannedFiles: testFiles),
      ),
    ));
    await tester.pumpAndSettle();

    expect(find.text('编辑元数据'), findsOneWidget);
    expect(find.text('歌曲标题'), findsOneWidget);
    expect(find.text('艺术家'), findsOneWidget);
    expect(find.text('专辑'), findsOneWidget);
    expect(find.text('发布到服务器'), findsOneWidget);
  });
}
```

- [ ] **Step 2 (VERIFY RED)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/publish/pages/edit_metadata_page_test.dart 2>&1 | tail -20
```
Expected: Compilation error

- [ ] **Step 3 (GREEN): 实现 EditMetadataPage**

```dart
// lib/features/publish/pages/edit_metadata_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';
import 'package:echo_vault_app/features/publish/providers/publish_provider.dart';
import 'package:echo_vault_app/providers/auth_provider.dart';

class EditMetadataPage extends ConsumerWidget {
  final List<ScannedFile> scannedFiles;

  const EditMetadataPage({super.key, required this.scannedFiles});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final publishState = ref.watch(publishProvider);
    final currentIndex = publishState.currentIndex;
    final currentFile = scannedFiles[currentIndex];
    final theme = Theme.of(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text('编辑元数据'),
        actions: [
          Text('${currentIndex + 1} / ${scannedFiles.length}'),
          const SizedBox(width: 16),
        ],
      ),
      body: Form(
        key: publishState.formKey,
        child: ListView(
          padding: const EdgeInsets.all(16),
          children: [
            // 文件名显示
            Card(
              child: ListTile(
                leading: const Icon(Icons.audiotrack),
                title: Text(currentFile.fileName),
                subtitle: Text(currentFile.path),
              ),
            ),
            const SizedBox(height: 16),

            // 标题
            TextFormField(
              initialValue: _titleFromFileName(currentFile.fileName),
              decoration: const InputDecoration(
                labelText: '歌曲标题',
                border: OutlineInputBorder(),
              ),
              onChanged: (v) => ref.read(publishProvider.notifier).setTitle(v),
              validator: (v) => v == null || v.isEmpty ? '请输入标题' : null,
            ),
            const SizedBox(height: 12),

            // 艺术家
            TextFormField(
              decoration: const InputDecoration(
                labelText: '艺术家',
                border: OutlineInputBorder(),
              ),
              onChanged: (v) => ref.read(publishProvider.notifier).setArtist(v),
            ),
            const SizedBox(height: 12),

            // 专辑
            TextFormField(
              decoration: const InputDecoration(
                labelText: '专辑',
                border: OutlineInputBorder(),
              ),
              onChanged: (v) => ref.read(publishProvider.notifier).setAlbum(v),
            ),
            const SizedBox(height: 12),

            // 流派 + 年份 行
            Row(
              children: [
                Expanded(
                  child: TextFormField(
                    decoration: const InputDecoration(
                      labelText: '流派',
                      border: OutlineInputBorder(),
                    ),
                    onChanged: (v) => ref.read(publishProvider.notifier).setGenre(v),
                  ),
                ),
                const SizedBox(width: 12),
                Expanded(
                  child: TextFormField(
                    decoration: const InputDecoration(
                      labelText: '年份',
                      border: OutlineInputBorder(),
                    ),
                    keyboardType: TextInputType.number,
                    onChanged: (v) {
                      final year = int.tryParse(v) ?? 0;
                      ref.read(publishProvider.notifier).setYear(year);
                    },
                  ),
                ),
              ],
            ),
            const SizedBox(height: 24),

            // 进度
            if (publishState.isUploading) ...[
              LinearProgressIndicator(value: publishState.progress),
              const SizedBox(height: 8),
              Text(
                '正在发布 ${currentIndex + 1}/${scannedFiles.length}...',
                textAlign: TextAlign.center,
              ),
              const SizedBox(height: 16),
            ],

            // 操作按钮
            Row(
              children: [
                Expanded(
                  child: OutlinedButton(
                    onPressed: publishState.isUploading ? null : () {
                      if (currentIndex > 0) {
                        ref.read(publishProvider.notifier).previous();
                      }
                    },
                    child: const Text('上一首'),
                  ),
                ),
                const SizedBox(width: 12),
                Expanded(
                  child: FilledButton(
                    onPressed: publishState.isUploading ? null : () async {
                      final authState = ref.read(authProvider);
                      final notifier = ref.read(publishProvider.notifier);
                      if (authState.status != AuthStatus.authenticated) {
                        ScaffoldMessenger.of(context).showSnackBar(
                          const SnackBar(content: Text('请先登录')),
                        );
                        return;
                      }

                      final success = await notifier.publishCurrentFile(
                        file: currentFile,
                        token: authState.token ?? '',
                      );

                      if (success && context.mounted) {
                        if (currentIndex < scannedFiles.length - 1) {
                          notifier.next();
                        } else {
                          ScaffoldMessenger.of(context).showSnackBar(
                            const SnackBar(content: Text('全部发布完成！')),
                          );
                          Navigator.of(context).popUntil((route) => route.isFirst);
                        }
                      }
                    },
                    child: Text(
                      currentIndex < scannedFiles.length - 1
                          ? '发布并继续'
                          : '发布到服务器',
                    ),
                  ),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }

  String _titleFromFileName(String fileName) {
    // 移扩展名，下划线/连字符 → 空格
    final withoutExt = fileName.replaceAll(RegExp(r'\.[^.]+$'), '');
    return withoutExt
        .replaceAll('_', ' ')
        .replaceAll('-', ' ')
        .trim();
  }
}
```

- [ ] **Step 4 (VERIFY GREEN)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/publish/pages/edit_metadata_page_test.dart -v
```
Expected: 1 test, all PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/publish/pages/edit_metadata_page.dart \
       echo_vault_app/test/features/publish/pages/edit_metadata_page_test.dart
git commit -m "feat(flutter): add EditMetadataPage with form (TDD, 1 widget test)"
```

---

### Task 11: PublishProvider — 发布流程状态管理

**核心逻辑：** `PublishNotifier` 管理逐首发布的流程：当前索引、表单数据、上传状态和进度。`publishCurrentFile` 方法执行 gRPC PublishSong + REST 音频上传。

**Files:**
- Create: `echo_vault_app/lib/features/publish/providers/publish_provider.dart`
- Create: `echo_vault_app/test/features/publish/providers/publish_provider_test.dart`

- [ ] **Step 1 (RED): 编写 PublishProvider test**

```dart
// test/features/publish/providers/publish_provider_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/publish/providers/publish_provider.dart';

void main() {
  group('PublishNotifier', () {
    late ProviderContainer container;

    setUp(() {
      container = ProviderContainer();
    });

    tearDown(() {
      container.dispose();
    });

    test('initial state', () {
      final state = container.read(publishProvider);
      expect(state.currentIndex, 0);
      expect(state.title, '');
      expect(state.artist, '');
      expect(state.album, '');
      expect(state.genre, '');
      expect(state.year, 0);
      expect(state.isUploading, false);
      expect(state.progress, 0);
    });

    test('setTitle updates title', () {
      container.read(publishProvider.notifier).setTitle('Summer');
      expect(container.read(publishProvider).title, 'Summer');
    });

    test('setArtist updates artist', () {
      container.read(publishProvider.notifier).setArtist('Test');
      expect(container.read(publishProvider).artist, 'Test');
    });

    test('setAlbum updates album', () {
      container.read(publishProvider.notifier).setAlbum('Album');
      expect(container.read(publishProvider).album, 'Album');
    });

    test('setGenre updates genre', () {
      container.read(publishProvider.notifier).setGenre('Rock');
      expect(container.read(publishProvider).genre, 'Rock');
    });

    test('setYear updates year', () {
      container.read(publishProvider.notifier).setYear(2024);
      expect(container.read(publishProvider).year, 2024);
    });

    test('next increments index', () {
      container.read(publishProvider.notifier).next();
      expect(container.read(publishProvider).currentIndex, 1);
    });

    test('previous decrements index', () {
      container.read(publishProvider.notifier).next();
      container.read(publishProvider.notifier).next();
      container.read(publishProvider.notifier).previous();
      expect(container.read(publishProvider).currentIndex, 1);
    });

    test('setUploading updates state', () {
      container.read(publishProvider.notifier).setUploading(true);
      expect(container.read(publishProvider).isUploading, true);
    });

    test('reset clears all', () {
      final notifier = container.read(publishProvider.notifier);
      notifier.setTitle('Test');
      notifier.setArtist('Me');
      notifier.next();
      notifier.reset();
      final state = container.read(publishProvider);
      expect(state.currentIndex, 0);
      expect(state.title, '');
      expect(state.artist, '');
    });
  });
}
```

- [ ] **Step 2 (VERIFY RED)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/publish/providers/publish_provider_test.dart 2>&1 | tail -20
```
Expected: Compilation error

- [ ] **Step 3 (GREEN): 实现 PublishProvider**

```dart
// lib/features/publish/providers/publish_provider.dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/models/scanned_file.dart';
import 'package:echo_vault_app/features/library/services/song_repository.dart';
import 'package:echo_vault_app/features/library/services/rest_client.dart';
import 'package:echo_vault_app/core/grpc/client.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';

class PublishState {
  final int currentIndex;
  final String title;
  final String artist;
  final String album;
  final String genre;
  final int year;
  final bool isUploading;
  final double progress;
  final GlobalKey<FormState> formKey;

  const PublishState({
    this.currentIndex = 0,
    this.title = '',
    this.artist = '',
    this.album = '',
    this.genre = '',
    this.year = 0,
    this.isUploading = false,
    this.progress = 0,
    GlobalKey<FormState>? formKey,
  }) : formKey = formKey ?? GlobalKey<FormState>();
}

class PublishNotifier extends StateNotifier<PublishState> {
  PublishNotifier() : super(const PublishState());

  void setTitle(String v) => state = PublishState(
    currentIndex: state.currentIndex, title: v, artist: state.artist,
    album: state.album, genre: state.genre, year: state.year,
    formKey: state.formKey,
  );

  void setArtist(String v) => state = PublishState(
    currentIndex: state.currentIndex, title: state.title, artist: v,
    album: state.album, genre: state.genre, year: state.year,
    formKey: state.formKey,
  );

  void setAlbum(String v) => state = PublishState(
    currentIndex: state.currentIndex, title: state.title, artist: state.artist,
    album: v, genre: state.genre, year: state.year,
    formKey: state.formKey,
  );

  void setGenre(String v) => state = PublishState(
    currentIndex: state.currentIndex, title: state.title, artist: state.artist,
    album: state.album, genre: v, year: state.year,
    formKey: state.formKey,
  );

  void setYear(int v) => state = PublishState(
    currentIndex: state.currentIndex, title: state.title, artist: state.artist,
    album: state.album, genre: state.genre, year: v,
    formKey: state.formKey,
  );

  void setUploading(bool v) => state = PublishState(
    currentIndex: state.currentIndex, title: state.title, artist: state.artist,
    album: state.album, genre: state.genre, year: state.year,
    isUploading: v, formKey: state.formKey,
  );

  void next() => state = PublishState(
    currentIndex: state.currentIndex + 1, formKey: state.formKey,
  );

  void previous() {
    if (state.currentIndex > 0) {
      state = PublishState(
        currentIndex: state.currentIndex - 1, formKey: state.formKey,
      );
    }
  }

  void reset() => state = const PublishState();

  /// 发布当前文件：gRPC PublishSong + REST 上传音频
  Future<bool> publishCurrentFile({
    required ScannedFile file,
    required String token,
  }) async {
    if (!state.formKey.currentState!.validate()) return false;

    setUploading(true);

    try {
      // 1. 先通过 gRPC 创建 Song 记录
      final grpcManager = GrpcClientManager();
      await grpcManager.connect(host: 'localhost');
      grpcManager.setToken(token);
      final repository = SongRepository(grpcManager);

      final song = await repository.publishSong(
        fileHash: file.fileHash,
        title: state.title.isNotEmpty ? state.title : file.fileName,
        artist: state.artist,
        album: state.album,
        genre: state.genre,
        trackNumber: 0,
        discNumber: 0,
        year: state.year,
        fileName: file.fileName,
        fileSize: file.fileSize,
        mimeType: _mimeFromExt(file.extension),
      );

      // 2. 上传音频文件到 REST 服务（有封面也会自动提取）
      final restClient = RestClient();
      await restClient.uploadAudio(
        songId: song.id,
        filePath: file.path,
        fileName: file.fileName,
      );

      setUploading(false);
      return true;
    } catch (e) {
      setUploading(false);
      return false;
    }
  }

  String _mimeFromExt(String ext) {
    switch (ext.toLowerCase()) {
      case '.mp3': return 'audio/mpeg';
      case '.flac': return 'audio/flac';
      case '.ogg': return 'audio/ogg';
      case '.m4a': case '.mp4': return 'audio/mp4';
      case '.wav': return 'audio/wav';
      case '.aac': return 'audio/aac';
      case '.wma': return 'audio/x-ms-wma';
      case '.opus': return 'audio/opus';
      default: return 'application/octet-stream';
    }
  }
}

final publishProvider = StateNotifierProvider<PublishNotifier, PublishState>((ref) {
  return PublishNotifier();
});
```

- [ ] **Step 4 (VERIFY GREEN)**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test test/features/publish/providers/publish_provider_test.dart -v
```
Expected: 10 tests, all PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/publish/providers/publish_provider.dart \
       echo_vault_app/test/features/publish/providers/publish_provider_test.dart
git commit -m "feat(flutter): add PublishProvider with gRPC + upload logic (TDD, 10 tests)"
```

---

### Task 12: 路由更新 — 集成到主应用

**核心逻辑：** 更新路由器，添加 LibraryPage（/library）、ScanResultPage（/scan）、EditMetadataPage（/publish）路由。更新 main.dart 确保 ProviderScope 包含新增的 provider。

**Files:**
- Modify: `echo_vault_app/lib/router.dart`
- Modify: `echo_vault_app/lib/main.dart`（无需修改，只需确认 ProviderScope 包裹）

- [ ] **Step 1: 更新 router.dart — 添加 Phase 7 路由**

```dart
// echo_vault_app/lib/router.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:echo_vault_app/features/auth/login_page.dart';
import 'package:echo_vault_app/features/auth/register_page.dart';
import 'package:echo_vault_app/features/setup/server_setup_page.dart';
import 'package:echo_vault_app/features/library/pages/library_page.dart';
import 'package:echo_vault_app/features/publish/pages/scan_result_page.dart';
import 'package:echo_vault_app/features/publish/pages/edit_metadata_page.dart';
import 'package:echo_vault_app/providers/auth_provider.dart';
import 'package:echo_vault_app/providers/server_provider.dart';

final routerProvider = Provider<GoRouter>((ref) {
  final authState = ref.watch(authProvider);
  final serverConfig = ref.watch(serverConfigProvider);

  return GoRouter(
    initialLocation: '/library',
    redirect: (context, state) {
      final loggedIn = authState.status == AuthStatus.authenticated;
      final configured = serverConfig != null;

      if (!configured && state.matchedLocation != '/setup') return '/setup';
      if (configured && !loggedIn &&
          state.matchedLocation != '/login' &&
          state.matchedLocation != '/register') return '/login';
      return null;
    },
    routes: [
      GoRoute(path: '/setup', builder: (_, __) => const ServerSetupPage()),
      GoRoute(path: '/login', builder: (_, __) => const LoginPage()),
      GoRoute(path: '/register', builder: (_, __) => const RegisterPage()),
      GoRoute(
        path: '/library',
        builder: (_, __) => const LibraryPage(),
      ),
      GoRoute(
        path: '/scan',
        builder: (_, __) => const ScanResultPage(),
      ),
      GoRoute(
        path: '/publish',
        builder: (context, state) {
          final files = state.extra as List? ?? [];
          return EditMetadataPage(scannedFiles: files.cast());
        },
      ),
    ],
  );
});
```

- [ ] **Step 2: Run all tests to confirm no regression**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test -v 2>&1 | tail -30
```
Expected: All tests PASS (existing + new)

- [ ] **Step 3: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/router.dart
git commit -m "feat(flutter): add /library, /scan, /publish routes to router"
```

---

### Task 13: 全量验证 + 最终提交

- [ ] **Step 1: Run all Flutter tests**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter test -v 2>&1
```
Expected: All tests PASS (zero failures)

- [ ] **Step 2: Verify compilation**

Run:
```bash
cd /home/ink/Desktop/develop/EchoVault/echo_vault_app
flutter analyze 2>&1 | tail -10
```
Expected: No warnings or errors (or only pre-existing ones)

- [ ] **Step 3: Final summary**

```bash
cd /home/ink/Desktop/develop/EchoVault
git log --oneline -15
```
Expected: Phase 7 commits visible

- [ ] **Step 4: 创建摘要提交（如果所有 task 已各自 commit，则跳过）**

```bash
cd /home/ink/Desktop/develop/EchoVault
git log --oneline --all --grep="Phase 7" 2>/dev/null || echo "No Phase 7 tag found — OK"
```

---

### Phase 7 自审清单

- [ ] Dart protobuf 生成（或手工存根）
- [ ] `FileScanner` 递归扫描 + SHA256 — 7 测试
- [ ] `SongRepository` gRPC 封装 — 4 测试
- [ ] `RestClient` Dio 上传 — 5 测试
- [ ] `ScannerProvider` 扫描状态管理 — 4 测试
- [ ] `LibraryProvider` 歌库状态管理 — 5 测试
- [ ] `SongListTile` 歌曲行组件 — 4 widget 测试
- [ ] `LibraryPage` 歌库主页 — 2 widget 测试
- [ ] `ScanResultPage` 扫描结果页 — 2 widget 测试
- [ ] `EditMetadataPage` 元数据编辑页 — 1 widget 测试
- [ ] `PublishProvider` 发布状态管理 — 10 测试
- [ ] 路由集成（/library, /scan, /publish）
- [ ] `flutter test` 全部通过
- [ ] `flutter analyze` 无新增警告

**预计新增测试数：** ~44 测试  
**全部测试数（Flutter）：** ~53+（含现有 ~9）  
**入口：** `Phase 8: 播放器 + 歌词`
