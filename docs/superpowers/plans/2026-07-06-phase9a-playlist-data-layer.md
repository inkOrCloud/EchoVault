# Phase 9a: 歌单数据层实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现歌单功能的数据层，包括 gRPC 仓储、Riverpod 状态管理和数据模型封装

**Architecture:** 基于现有的 SongRepository 模式，创建 PlaylistRepository 处理 gRPC 调用，使用 Riverpod 进行状态管理，封装 Playlist/PlaylistSong 模型

**Tech Stack:** Flutter, Riverpod, gRPC, protobuf, drift

---

## 文件结构

### 创建文件
- `lib/features/playlist/services/playlist_repository.dart` - gRPC 歌单仓储
- `lib/features/playlist/providers/playlist_provider.dart` - 歌单状态管理
- `lib/features/playlist/providers/playlist_song_provider.dart` - 歌单内歌曲状态管理
- `lib/features/playlist/models/playlist_model.dart` - 歌单元数据模型
- `lib/features/playlist/models/playlist_song_model.dart` - 歌单歌曲模型

### 测试文件
- `test/features/playlist/services/playlist_repository_test.dart` - 仓储测试
- `test/features/playlist/providers/playlist_provider_test.dart` - 仓储测试
- `test/features/playlist/providers/playlist_song_provider_test.dart` - 歌单歌曲测试
- `test/features/playlist/models/playlist_model_test.dart` - 模型测试

---

## Task 1: 歌单元数据模型

**Files:**
- Create: `lib/features/playlist/models/playlist_model.dart`
- Create: `lib/features/playlist/models/playlist_song_model.dart`
- Create: `test/features/playlist/models/playlist_model_test.dart`

- [ ] **Step 1: 创建 Playlist 模型**

```dart
// lib/features/playlist/models/playlist_model.dart
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';
import 'package:echo_vault_app/models/generated/google/protobuf/timestamp.pb.dart';

class PlaylistModel {
  final String id;
  final String name;
  final String? description;
  final String? coverUrl;
  final PlaylistType type;
  final String ownerId;
  final bool isPublic;
  final int songCount;
  final int version;
  final DateTime createdAt;
  final DateTime updatedAt;

  const PlaylistModel({
    required this.id,
    required this.name,
    this.description,
    this.coverUrl,
    this.type = PlaylistType.TYPE_UNSPECIFIED,
    required this.ownerId,
    this.isPublic = false,
    this.songCount = 0,
    this.version = 0,
    required this.createdAt,
    required this.updatedAt,
  });

  factory PlaylistModel.fromProto(Playlist proto) {
    return PlaylistModel(
      id: proto.id,
      name: proto.name,
      description: proto.description.isEmpty ? null : proto.description,
      coverUrl: proto.coverUrl.isEmpty ? null : proto.coverUrl,
      type: proto.type,
      ownerId: proto.ownerId,
      isPublic: proto.isPublic,
      songCount: proto.songCount,
      version: proto.version.toInt(),
      createdAt: proto.createdAt.toDateTime(),
      updatedAt: proto.updatedAt.toDateTime(),
    );
  }

  Playlist toProto() {
    return Playlist(
      id: id,
      name: name,
      description: description ?? '',
      coverUrl: coverUrl ?? '',
      type: type,
      ownerId: ownerId,
      isPublic: isPublic,
      songCount: songCount,
      version: version,
      createdAt: Timestamp.fromDateTime(createdAt),
      updatedAt: Timestamp.fromDateTime(updatedAt),
    );
  }

  PlaylistModel copyWith({
    String? id,
    String? name,
    String? description,
    String? coverUrl,
    PlaylistType? type,
    String? ownerId,
    bool? isPublic,
    int? songCount,
    int? version,
    DateTime? createdAt,
    DateTime? updatedAt,
  }) {
    return PlaylistModel(
      id: id ?? this.id,
      name: name ?? this.name,
      description: description ?? this.description,
      coverUrl: coverUrl ?? this.coverUrl,
      type: type ?? this.type,
      ownerId: ownerId ?? this.ownerId,
      isPublic: isPublic ?? this.isPublic,
      songCount: songCount ?? this.songCount,
      version: version ?? this.version,
      createdAt: createdAt ?? this.createdAt,
      updatedAt: updatedAt ?? this.updatedAt,
    );
  }
}
```

- [ ] **Step 2: 创建 PlaylistSong 模型**

```dart
// lib/features/playlist/models/playlist_song_model.dart
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';

class PlaylistSongModel {
  final String id;
  final String playlistId;
  final String songId;
  final int sortOrder;
  final DateTime addedAt;

  const PlaylistSongModel({
    required this.id,
    required this.playlistId,
    required this.songId,
    this.sortOrder = 0,
    required this.addedAt,
  });

  factory PlaylistSongModel.fromProto(PlaylistSong proto) {
    return PlaylistSongModel(
      id: proto.id,
      playlistId: proto.playlistId,
      songId: proto.songId,
      sortOrder: proto.sortOrder,
      addedAt: proto.addedAt.toDateTime(),
    );
  }

  PlaylistSong toProto() {
    return PlaylistSong(
      id: id,
      playlistId: playlistId,
      songId: songId,
      sortOrder: sortOrder,
      addedAt: Timestamp.fromDateTime(addedAt),
    );
  }
}
```

- [ ] **Step 3: 创建模型测试**

```dart
// test/features/playlist/models/playlist_model_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:echo_vault_app/features/playlist/models/playlist_model.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';
import 'package:echo_vault_app/models/generated/google/protobuf/timestamp.pb.dart';

void main() {
  test('PlaylistModel fromProto', () {
    final now = DateTime.now();
    final proto = Playlist(
      id: 'p1',
      name: 'Test Playlist',
      description: 'A test playlist',
      type: PlaylistType.TYPE_NORMAL,
      ownerId: 'u1',
      isPublic: true,
      songCount: 5,
      version: 1,
      createdAt: Timestamp.fromDateTime(now),
      updatedAt: Timestamp.fromDateTime(now),
    );

    final model = PlaylistModel.fromProto(proto);
    expect(model.id, 'p1');
    expect(model.name, 'Test Playlist');
    expect(model.description, 'A test playlist');
    expect(model.type, PlaylistType.TYPE_NORMAL);
    expect(model.ownerId, 'u1');
    expect(model.isPublic, isTrue);
    expect(model.songCount, 5);
  });

  test('PlaylistModel toProto', () {
    final now = DateTime.now();
    final model = PlaylistModel(
      id: 'p1',
      name: 'Test Playlist',
      ownerId: 'u1',
      createdAt: now,
      updatedAt: now,
    );

    final proto = model.toProto();
    expect(proto.id, 'p1');
    expect(proto.name, 'Test Playlist');
    expect(proto.ownerId, 'u1');
  });
}
```

- [ ] **Step 4: 运行测试验证**

Run: `flutter test test/features/playlist/models/playlist_model_test.dart`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add lib/features/playlist/models/ test/features/playlist/models/
git commit -m "feat(playlist): add PlaylistModel and PlaylistSongModel"
```

---

## Task 2: gRPC 歌单仓储

**Files:**
- Create: `lib/features/playlist/services/playlist_repository.dart`
- Create: `test/features/playlist/services/playlist_repository_test.dart`

- [ ] **Step 1: 创建 PlaylistService 接口和实现**

```dart
// lib/features/playlist/services/playlist_repository.dart
import 'package:fixnum/fixnum.dart';
import 'package:grpc/grpc.dart';
import 'package:echo_vault_app/core/grpc/client.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pbgrpc.dart';
import 'package:echo_vault_app/models/generated/echo_vault/common/v1/types.pb.dart';

/// gRPC PlaylistService 抽象接口（生产用 PlaylistServiceClient，测试用 mock）
abstract class PlaylistService {
  Future<CreatePlaylistResponse> createPlaylist(CreatePlaylistRequest request);
  Future<GetPlaylistResponse> getPlaylist(GetPlaylistRequest request);
  Future<ListPlaylistsResponse> listPlaylists(ListPlaylistsRequest request);
  Future<UpdatePlaylistResponse> updatePlaylist(UpdatePlaylistRequest request);
  Future<DeletePlaylistResponse> deletePlaylist(DeletePlaylistRequest request);
  Future<AddSongToPlaylistResponse> addSongToPlaylist(AddSongToPlaylistRequest request);
  Future<RemoveSongFromPlaylistResponse> removeSongFromPlaylist(RemoveSongFromPlaylistRequest request);
  Future<ReorderSongsResponse> reorderSongs(ReorderSongsRequest request);
}

/// 生产实现 — 直接使用生成的 PlaylistServiceClient
class ChannelPlaylistService implements PlaylistService {
  final PlaylistServiceClient _client;
  ChannelPlaylistService(ClientChannel channel) : _client = PlaylistServiceClient(channel);

  @override Future<CreatePlaylistResponse> createPlaylist(CreatePlaylistRequest r) => _client.createPlaylist(r);
  @override Future<GetPlaylistResponse> getPlaylist(GetPlaylistRequest r) => _client.getPlaylist(r);
  @override Future<ListPlaylistsResponse> listPlaylists(ListPlaylistsRequest r) => _client.listPlaylists(r);
  @override Future<UpdatePlaylistResponse> updatePlaylist(UpdatePlaylistRequest r) => _client.updatePlaylist(r);
  @override Future<DeletePlaylistResponse> deletePlaylist(DeletePlaylistRequest r) => _client.deletePlaylist(r);
  @override Future<AddSongToPlaylistResponse> addSongToPlaylist(AddSongToPlaylistRequest r) => _client.addSongToPlaylist(r);
  @override Future<RemoveSongFromPlaylistResponse> removeSongFromPlaylist(RemoveSongFromPlaylistRequest r) => _client.removeSongFromPlaylist(r);
  @override Future<ReorderSongsResponse> reorderSongs(ReorderSongsRequest r) => _client.reorderSongs(r);
}

class PlaylistRepositoryException implements Exception {
  final String message;
  final int? grpcCode;
  const PlaylistRepositoryException(this.message, {this.grpcCode});
  @override String toString() => 'PlaylistRepositoryException: $message${grpcCode != null ? " (code: $grpcCode)" : ""}';
}

class PlaylistRepository {
  final PlaylistService _svc;

  PlaylistRepository({required GrpcClientManager manager}) : _svc = ChannelPlaylistService(manager.channel);
  PlaylistRepository.withService(this._svc);

  Future<Playlist> createPlaylist({
    required String name,
    String? description,
    bool isPublic = false,
  }) async {
    try {
      return (await _svc.createPlaylist(CreatePlaylistRequest(
        name: name,
        description: description ?? '',
        isPublic: isPublic,
      ))).playlist;
    } on GrpcError catch (e) { throw PlaylistRepositoryException(e.message ?? '', grpcCode: e.code); }
  }

  Future<Playlist> getPlaylist(String id) async {
    try {
      return (await _svc.getPlaylist(GetPlaylistRequest(id: id))).playlist;
    } on GrpcError catch (e) { throw PlaylistRepositoryException(e.message ?? '', grpcCode: e.code); }
  }

  Future<List<Playlist>> listPlaylists({int pageSize = 20}) async {
    try {
      return (await _svc.listPlaylists(ListPlaylistsRequest(
        pagination: PaginationRequest(pageSize: pageSize),
      ))).playlists;
    } on GrpcError catch (e) { throw PlaylistRepositoryException(e.message ?? '', grpcCode: e.code); }
  }

  Future<Playlist> updatePlaylist({
    required String id,
    String? name,
    String? description,
    bool? isPublic,
  }) async {
    try {
      return (await _svc.updatePlaylist(UpdatePlaylistRequest(
        id: id,
        name: name ?? '',
        description: description ?? '',
        isPublic: isPublic ?? false,
      ))).playlist;
    } on GrpcError catch (e) { throw PlaylistRepositoryException(e.message ?? '', grpcCode: e.code); }
  }

  Future<void> deletePlaylist(String id) async {
    try {
      await _svc.deletePlaylist(DeletePlaylistRequest(id: id));
    } on GrpcError catch (e) { throw PlaylistRepositoryException(e.message ?? '', grpcCode: e.code); }
  }

  Future<PlaylistSong> addSongToPlaylist({
    required String playlistId,
    required String songId,
    int sortOrder = 0,
  }) async {
    try {
      return (await _svc.addSongToPlaylist(AddSongToPlaylistRequest(
        playlistId: playlistId,
        songId: songId,
        sortOrder: sortOrder,
      ))).playlistSong;
    } on GrpcError catch (e) { throw PlaylistRepositoryException(e.message ?? '', grpcCode: e.code); }
  }

  Future<void> removeSongFromPlaylist({
    required String playlistId,
    required String songId,
  }) async {
    try {
      await _svc.removeSongFromPlaylist(RemoveSongFromPlaylistRequest(
        playlistId: playlistId,
        songId: songId,
      ));
    } on GrpcError catch (e) { throw PlaylistRepositoryException(e.message ?? '', grpcCode: e.code); }
  }

  Future<void> reorderSongs({
    required String playlistId,
    required List<String> songIds,
  }) async {
    try {
      await _svc.reorderSongs(ReorderSongsRequest(
        playlistId: playlistId,
        songIds: songIds,
      ));
    } on GrpcError catch (e) { throw PlaylistRepositoryException(e.message ?? '', grpcCode: e.code); }
  }
}
```

- [ ] **Step 2: 创建测试 mock 和测试**

```dart
// test/features/playlist/services/playlist_repository_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:grpc/grpc.dart';
import 'package:echo_vault_app/features/playlist/services/playlist_repository.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';

class MockPlaylistService implements PlaylistService {
  final Map<String, Object Function()> _handlers = {};
  bool throwOnCall = false;
  void when(String m, Object Function() h) { _handlers[m] = h; }

  @override Future<CreatePlaylistResponse> createPlaylist(CreatePlaylistRequest r) {
    if (throwOnCall) throw GrpcError.unavailable('down');
    return Future.value(_handlers['createPlaylist']!() as CreatePlaylistResponse);
  }
  @override Future<GetPlaylistResponse> getPlaylist(GetPlaylistRequest r) =>
    Future.value(_handlers['getPlaylist']!() as GetPlaylistResponse);
  @override Future<ListPlaylistsResponse> listPlaylists(ListPlaylistsRequest r) =>
    Future.value(_handlers['listPlaylists']!() as ListPlaylistsResponse);
  @override Future<UpdatePlaylistResponse> updatePlaylist(UpdatePlaylistRequest r) =>
    Future.value(_handlers['updatePlaylist']!() as UpdatePlaylistResponse);
  @override Future<DeletePlaylistResponse> deletePlaylist(DeletePlaylistRequest r) =>
    Future.value(_handlers['deletePlaylist']!() as DeletePlaylistResponse);
  @override Future<AddSongToPlaylistResponse> addSongToPlaylist(AddSongToPlaylistRequest r) =>
    Future.value(_handlers['addSongToPlaylist']!() as AddSongToPlaylistResponse);
  @override Future<RemoveSongFromPlaylistResponse> removeSongFromPlaylist(RemoveSongFromPlaylistRequest r) =>
    Future.value(_handlers['removeSongFromPlaylist']!() as RemoveSongFromPlaylistResponse);
  @override Future<ReorderSongsResponse> reorderSongs(ReorderSongsRequest r) =>
    Future.value(_handlers['reorderSongs']!() as ReorderSongsResponse);
}

void main() {
  late MockPlaylistService mock;
  late PlaylistRepository repo;
  setUp(() { mock = MockPlaylistService(); repo = PlaylistRepository.withService(mock); });

  test('createPlaylist', () async {
    mock.when('createPlaylist', () => CreatePlaylistResponse(playlist: Playlist(id: 'p1', name: 'Test')));
    final p = await repo.createPlaylist(name: 'Test');
    expect(p.id, 'p1');
    expect(p.name, 'Test');
  });

  test('listPlaylists', () async {
    mock.when('listPlaylists', () => ListPlaylistsResponse(playlists: [
      Playlist(id: 'p1', name: 'Playlist 1'),
      Playlist(id: 'p2', name: 'Playlist 2'),
    ]));
    final playlists = await repo.listPlaylists();
    expect(playlists.length, 2);
  });

  test('throws on error', () async {
    mock.throwOnCall = true;
    expect(() => repo.createPlaylist(name: 'Test'), throwsA(isA<PlaylistRepositoryException>()));
  });

  test('addSongToPlaylist', () async {
    mock.when('addSongToPlaylist', () => AddSongToPlaylistResponse(
      playlistSong: PlaylistSong(id: 'ps1', playlistId: 'p1', songId: 's1'),
    ));
    final ps = await repo.addSongToPlaylist(playlistId: 'p1', songId: 's1');
    expect(ps.id, 'ps1');
  });
}
```

- [ ] **Step 3: 运行测试验证**

Run: `flutter test test/features/playlist/services/playlist_repository_test.dart`
Expected: PASS

- [ ] **Step 4: 提交**

```bash
git add lib/features/playlist/services/ test/features/playlist/services/
git commit -m "feat(playlist): add PlaylistRepository with gRPC service"
```

---

## Task 3: 歌单状态管理 (PlaylistProvider)

**Files:**
- Create: `lib/features/playlist/providers/playlist_provider.dart`
- Create: `test/features/playlist/providers/playlist_provider_test.dart`

- [ ] **Step 1: 创建 PlaylistProvider**

```dart
// lib/features/playlist/providers/playlist_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/playlist/services/playlist_repository.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';

enum PlaylistStatus { loading, loaded, error }

class PlaylistState {
  final PlaylistStatus status;
  final List<Playlist> playlists;
  final String? error;
  final String searchQuery;
  const PlaylistState({this.status = PlaylistStatus.loading, this.playlists = const [], this.error, this.searchQuery = ''});
  List<Playlist> get filteredPlaylists {
    if (searchQuery.isEmpty) return playlists;
    final q = searchQuery.toLowerCase();
    return playlists.where((p) => p.name.toLowerCase().contains(q) || (p.description.toLowerCase().contains(q))).toList();
  }
}

final playlistRepositoryProvider = Provider<PlaylistRepository>((ref) {
  throw UnimplementedError('PlaylistRepository must be overridden');
});

final playlistProvider = StateNotifierProvider<PlaylistNotifier, PlaylistState>((ref) {
  return PlaylistNotifier(ref.read(playlistRepositoryProvider));
});

class PlaylistNotifier extends StateNotifier<PlaylistState> {
  final PlaylistRepository _repo;
  PlaylistNotifier(this._repo) : super(const PlaylistState());

  Future<void> loadPlaylists() async {
    state = const PlaylistState(status: PlaylistStatus.loading);
    try {
      final playlists = await _repo.listPlaylists();
      state = PlaylistState(status: PlaylistStatus.loaded, playlists: playlists);
    } catch (e) {
      state = PlaylistState(status: PlaylistStatus.error, error: e.toString());
    }
  }

  Future<Playlist> createPlaylist({
    required String name,
    String? description,
    bool isPublic = false,
  }) async {
    try {
      final playlist = await _repo.createPlaylist(
        name: name,
        description: description,
        isPublic: isPublic,
      );
      state = PlaylistState(
        status: PlaylistStatus.loaded,
        playlists: [...state.playlists, playlist],
      );
      return playlist;
    } catch (e) {
      state = PlaylistState(status: PlaylistStatus.error, error: e.toString());
      rethrow;
    }
  }

  Future<void> deletePlaylist(String id) async {
    try {
      await _repo.deletePlaylist(id);
      state = PlaylistState(
        status: PlaylistStatus.loaded,
        playlists: state.playlists.where((p) => p.id != id).toList(),
      );
    } catch (e) {
      state = PlaylistState(status: PlaylistStatus.error, error: e.toString());
      rethrow;
    }
  }

  void setQuery(String q) {
    state = PlaylistState(
      status: state.status,
      playlists: state.playlists,
      error: state.error,
      searchQuery: q,
    );
  }
}
```

- [ ] **Step 2: 创建测试**

```dart
// test/features/playlist/providers/playlist_provider_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_provider.dart';
import 'package:echo_vault_app/features/playlist/services/playlist_repository.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';

class MockPlaylistRepository implements PlaylistRepository {
  List<Playlist> playlists = [];
  bool shouldThrow = false;

  @override Future<Playlist> createPlaylist({required String name, String? description, bool isPublic = false}) async {
    if (shouldThrow) throw Exception('error');
    final playlist = Playlist(id: 'p${playlists.length + 1}', name: name, description: description ?? '');
    playlists.add(playlist);
    return playlist;
  }

  @override Future<void> deletePlaylist(String id) async {
    if (shouldThrow) throw Exception('error');
    playlists.removeWhere((p) => p.id == id);
  }

  @override Future<List<Playlist>> listPlaylists({int pageSize = 20}) async {
    if (shouldThrow) throw Exception('error');
    return playlists;
  }

  @override Future<Playlist> getPlaylist(String id) async => throw UnimplementedError();
  @override Future<Playlist> updatePlaylist({required String id, String? name, String? description, bool? isPublic}) async => throw UnimplementedError();
  @override Future<PlaylistSong> addSongToPlaylist({required String playlistId, required String songId, int sortOrder = 0}) async => throw UnimplementedError();
  @override Future<void> removeSongFromPlaylist({required String playlistId, required String songId}) async => throw UnimplementedError();
  @override Future<void> reorderSongs({required String playlistId, required List<String> songIds}) async => throw UnimplementedError();
}

void main() {
  late MockPlaylistRepository mockRepo;
  late ProviderContainer container;

  setUp(() {
    mockRepo = MockPlaylistRepository();
    container = ProviderContainer(
      overrides: [
        playlistRepositoryProvider.overrideWithValue(mockRepo),
      ],
    );
  });

  tearDown(() => container.dispose());

  test('loadPlaylists', () async {
    mockRepo.playlists = [Playlist(id: 'p1', name: 'Test')];
    await container.read(playlistProvider.notifier).loadPlaylists();
    expect(container.read(playlistProvider).playlists.length, 1);
  });

  test('createPlaylist', () async {
    await container.read(playlistProvider.notifier).createPlaylist(name: 'New Playlist');
    expect(container.read(playlistProvider).playlists.length, 1);
    expect(container.read(playlistProvider).playlists.first.name, 'New Playlist');
  });

  test('deletePlaylist', () async {
    mockRepo.playlists = [Playlist(id: 'p1', name: 'Test')];
    await container.read(playlistProvider.notifier).loadPlaylists();
    await container.read(playlistProvider.notifier).deletePlaylist('p1');
    expect(container.read(playlistProvider).playlists.length, 0);
  });

  test('setQuery', () {
    mockRepo.playlists = [
      Playlist(id: 'p1', name: 'Rock Music'),
      Playlist(id: 'p2', name: 'Pop Music'),
    ];
    container.read(playlistProvider.notifier).loadPlaylists();
    container.read(playlistProvider.notifier).setQuery('rock');
    expect(container.read(playlistProvider).filteredPlaylists.length, 1);
  });
}
```

- [ ] **Step 3: 运行测试验证**

Run: `flutter test test/features/playlist/providers/playlist_provider_test.dart`
Expected: PASS

- [ ] **Step 4: 提交**

```bash
git add lib/features/playlist/providers/ test/features/playlist/providers/
git commit -m "feat(playlist): add PlaylistProvider with state management"
```

---

## Task 4: 歌单内歌曲状态管理 (PlaylistSongProvider)

**Files:**
- Create: `lib/features/playlist/providers/playlist_song_provider.dart`
- Create: `test/features/playlist/providers/playlist_song_provider_test.dart`

- [ ] **Step 1: 创建 PlaylistSongProvider**

```dart
// lib/features/playlist/providers/playlist_song_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/playlist/services/playlist_repository.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';

enum PlaylistSongStatus { loading, loaded, error }

class PlaylistSongState {
  final PlaylistSongStatus status;
  final List<PlaylistSong> playlistSongs;
  final List<Song> songs;
  final String? error;
  const PlaylistSongState({
    this.status = PlaylistSongStatus.loading,
    this.playlistSongs = const [],
    this.songs = const [],
    this.error,
  });
}

final playlistSongProvider = StateNotifierProvider.family<PlaylistSongNotifier, PlaylistSongState, String>((ref, playlistId) {
  return PlaylistSongNotifier(ref.read(playlistRepositoryProvider), playlistId);
});

class PlaylistSongNotifier extends StateNotifier<PlaylistSongState> {
  final PlaylistRepository _repo;
  final String playlistId;

  PlaylistSongNotifier(this._repo, this.playlistId) : super(const PlaylistSongState());

  Future<void> addSong(String songId) async {
    try {
      final playlistSong = await _repo.addSongToPlaylist(
        playlistId: playlistId,
        songId: songId,
      );
      state = PlaylistSongState(
        status: PlaylistSongStatus.loaded,
        playlistSongs: [...state.playlistSongs, playlistSong],
        songs: state.songs,
      );
    } catch (e) {
      state = PlaylistSongState(
        status: PlaylistSongStatus.error,
        playlistSongs: state.playlistSongs,
        songs: state.songs,
        error: e.toString(),
      );
      rethrow;
    }
  }

  Future<void> removeSong(String songId) async {
    try {
      await _repo.removeSongFromPlaylist(
        playlistId: playlistId,
        songId: songId,
      );
      state = PlaylistSongState(
        status: PlaylistSongStatus.loaded,
        playlistSongs: state.playlistSongs.where((ps) => ps.songId != songId).toList(),
        songs: state.songs,
      );
    } catch (e) {
      state = PlaylistSongState(
        status: PlaylistSongStatus.error,
        playlistSongs: state.playlistSongs,
        songs: state.songs,
        error: e.toString(),
      );
      rethrow;
    }
  }

  Future<void> reorderSongs(List<String> songIds) async {
    try {
      await _repo.reorderSongs(
        playlistId: playlistId,
        songIds: songIds,
      );
      // 重新排序本地状态
      final reordered = songIds.map((id) => state.playlistSongs.firstWhere((ps) => ps.songId == id)).toList();
      state = PlaylistSongState(
        status: PlaylistSongStatus.loaded,
        playlistSongs: reordered,
        songs: state.songs,
      );
    } catch (e) {
      state = PlaylistSongState(
        status: PlaylistSongStatus.error,
        playlistSongs: state.playlistSongs,
        songs: state.songs,
        error: e.toString(),
      );
      rethrow;
    }
  }
}
```

- [ ] **Step 2: 创建测试**

```dart
// test/features/playlist/providers/playlist_song_provider_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_song_provider.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_provider.dart';
import 'package:echo_vault_app/features/playlist/services/playlist_repository.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';

class MockPlaylistRepositoryForSongs implements PlaylistRepository {
  @override Future<PlaylistSong> addSongToPlaylist({required String playlistId, required String songId, int sortOrder = 0}) async {
    return PlaylistSong(id: 'ps1', playlistId: playlistId, songId: songId, sortOrder: sortOrder);
  }

  @override Future<void> removeSongFromPlaylist({required String playlistId, required String songId}) async {}

  @override Future<void> reorderSongs({required String playlistId, required List<String> songIds}) async {}

  @override Future<Playlist> createPlaylist({required String name, String? description, bool isPublic = false}) async => throw UnimplementedError();
  @override Future<void> deletePlaylist(String id) async => throw UnimplementedError();
  @override Future<Playlist> getPlaylist(String id) async => throw UnimplementedError();
  @override Future<List<Playlist>> listPlaylists({int pageSize = 20}) async => throw UnimplementedError();
  @override Future<Playlist> updatePlaylist({required String id, String? name, String? description, bool? isPublic}) async => throw UnimplementedError();
}

void main() {
  late MockPlaylistRepositoryForSongs mockRepo;
  late ProviderContainer container;

  setUp(() {
    mockRepo = MockPlaylistRepositoryForSongs();
    container = ProviderContainer(
      overrides: [
        playlistRepositoryProvider.overrideWithValue(mockRepo),
      ],
    );
  });

  tearDown(() => container.dispose());

  test('addSong', () async {
    await container.read(playlistSongProvider('p1').notifier).addSong('s1');
    expect(container.read(playlistSongProvider('p1')).playlistSongs.length, 1);
  });

  test('removeSong', () async {
    await container.read(playlistSongProvider('p1').notifier).addSong('s1');
    await container.read(playlistSongProvider('p1').notifier).removeSong('s1');
    expect(container.read(playlistSongProvider('p1')).playlistSongs.length, 0);
  });
}
```

- [ ] **Step 3: 运行测试验证**

Run: `flutter test test/features/playlist/providers/playlist_song_provider_test.dart`
Expected: PASS

- [ ] **Step 4: 提交**

```bash
git add lib/features/playlist/providers/playlist_song_provider.dart test/features/playlist/providers/playlist_song_provider_test.dart
git commit -m "feat(playlist): add PlaylistSongProvider for song management"
```

---

## Task 5: 集成测试和最终验证

**Files:**
- Modify: `lib/features/playlist/` (all files)

- [ ] **Step 1: 运行所有歌单相关测试**

Run: `flutter test test/features/playlist/`
Expected: All tests PASS

- [ ] **Step 2: 运行代码分析**

Run: `flutter analyze`
Expected: No issues found

- [ ] **Step 3: 提交最终变更**

```bash
git add -A
git commit -m "feat(playlist): complete Phase 9a - playlist data layer"
```

---

## 自检清单

**1. 规范覆盖：**
- ✅ PlaylistRepository - gRPC 歌单仓储（CRUD + 歌曲操作）
- ✅ PlaylistProvider - Riverpod 歌单状态管理
- ✅ PlaylistSongProvider - 歌单内歌曲状态管理
- ✅ 歌单元数据模型 - Playlist/PlaylistSong 模型封装

**2. 占位符扫描：**
- ✅ 无 TBD、TODO 或模糊描述
- ✅ 所有代码块完整

**3. 类型一致性：**
- ✅ 所有方法签名一致
- ✅ 模型属性与 protobuf 定义匹配

---

## 执行选项

**Plan complete and saved to `docs/superpowers/plans/2026-07-06-phase9a-playlist-data-layer.md`. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?**
