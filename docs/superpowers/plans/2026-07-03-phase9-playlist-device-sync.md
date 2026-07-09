# Phase 9：歌单 + 设备管理 + 同步状态 实现计划

> **致智能代理：** 必须使用子技能：推荐使用 superpowers:subagent-driven-development 或 superpowers:executing-plans 逐步执行本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 为 Flutter 客户端实现歌单管理、多设备管理和同步状态界面。

**架构：** 采用基于功能的架构，遵循现有模式。每个功能（歌单、设备、同步）在 `lib/features/` 下获得自己的目录，包含页面、提供者和组件。使用 Riverpod 进行状态管理，gRPC 进行服务器通信。

**技术栈：** Flutter, Dart, Riverpod, gRPC, protobuf, go_router

---

## 文件结构

```
echo_vault_app/lib/
├── features/
│   ├── playlist/
│   │   ├── pages/
│   │   │   ├── playlist_list_page.dart      # 歌单列表页
│   │   │   └── playlist_detail_page.dart    # 歌单详情页（含歌曲列表）
│   │   ├── providers/
│   │   │   └── playlist_provider.dart       # 歌单状态管理
│   │   └── widgets/
│   │       ├── playlist_tile.dart           # 歌单项组件
│   │       └── add_song_to_playlist_dialog.dart  # 添加歌曲对话框
│   ├── device/
│   │   ├── pages/
│   │   │   └── device_list_page.dart        # 设备列表页
│   │   ├── providers/
│   │   │   └── device_provider.dart         # 设备状态管理
│   │   └── widgets/
│   │       └── device_tile.dart             # 设备项组件
│   └── sync/
│       ├── pages/
│       │   └── sync_status_page.dart        # 同步状态页
│       ├── providers/
│       │   └── sync_provider.dart           # 同步状态管理
│       └── widgets/
│           └── sync_status_indicator.dart   # 同步状态指示器
├── router.dart                              # 添加新路由
└── main.dart                                # 更新导航
```

---

## 任务 1：歌单提供者（Provider）

**文件：**
- 创建：`lib/features/playlist/providers/playlist_provider.dart`
- 测试：手动验证

- [ ] **步骤 1：创建歌单状态模型**

```dart
// lib/features/playlist/providers/playlist_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';

enum PlaylistStatus { initial, loading, loaded, error }

class PlaylistListState {
  final PlaylistStatus status;
  final List<Playlist> playlists;
  final String? error;

  const PlaylistListState({
    this.status = PlaylistStatus.initial,
    this.playlists = const [],
    this.error,
  });
}

final playlistListProvider = StateNotifierProvider<PlaylistListNotifier, PlaylistListState>((ref) {
  return PlaylistListNotifier();
});

class PlaylistListNotifier extends StateNotifier<PlaylistListState> {
  PlaylistListNotifier() : super(const PlaylistListState());

  void setLoading() => state = const PlaylistListState(status: PlaylistStatus.loading);
  void setPlaylists(List<Playlist> playlists) => state = PlaylistListState(status: PlaylistStatus.loaded, playlists: playlists);
  void setError(String message) => state = PlaylistListState(status: PlaylistStatus.error, error: message);
  void addPlaylist(Playlist playlist) {
    state = PlaylistListState(
      status: state.status,
      playlists: [...state.playlists, playlist],
    );
  }
  void removePlaylist(String id) {
    state = PlaylistListState(
      status: state.status,
      playlists: state.playlists.where((p) => p.id != id).toList(),
    );
  }
  void updatePlaylist(Playlist updated) {
    state = PlaylistListState(
      status: state.status,
      playlists: state.playlists.map((p) => p.id == updated.id ? updated : p).toList(),
    );
  }
}
```

- [ ] **步骤 2：运行分析器验证无错误**

运行：`cd echo_vault_app && flutter analyze lib/features/playlist/providers/playlist_provider.dart`
预期：未发现问题

- [ ] **步骤 3：提交**

```bash
git add lib/features/playlist/providers/playlist_provider.dart
git commit -m "feat(playlist): 添加歌单状态管理提供者"
```

---

## 任务 2：歌单仓储（gRPC 服务）

**文件：**
- 创建：`lib/features/playlist/services/playlist_repository.dart`
- 测试：手动验证

- [ ] **步骤 1：创建歌单仓储**

```dart
// lib/features/playlist/services/playlist_repository.dart
import 'package:grpc/grpc.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pbgrpc.dart';
import 'package:echo_vault_app/core/grpc/client.dart';
import 'package:echo_vault_app/models/generated/echo_vault/common/v1/types.pb.dart' as common;

class PlaylistRepository {
  final GrpcClientManager _clientManager;

  PlaylistRepository(this._clientManager);

  PlaylistServiceClient get _client => PlaylistServiceClient(_clientManager.channel);

  Future<List<Playlist>> listPlaylists({int limit = 50, int offset = 0}) async {
    final request = ListPlaylistsRequest()
      ..pagination = (common.PaginationRequest()
        ..limit = limit
        ..offset = offset);
    final response = await _client.listPlaylists(request);
    return response.playlists.toList();
  }

  Future<Playlist> createPlaylist({
    required String name,
    String description = '',
    bool isPublic = false,
  }) async {
    final request = CreatePlaylistRequest()
      ..name = name
      ..description = description
      ..isPublic = isPublic;
    final response = await _client.createPlaylist(request);
    return response.playlist;
  }

  Future<Playlist> getPlaylist(String id) async {
    final request = GetPlaylistRequest()..id = id;
    final response = await _client.getPlaylist(request);
    return response.playlist;
  }

  Future<Playlist> updatePlaylist({
    required String id,
    String? name,
    String? description,
    bool? isPublic,
  }) async {
    final request = UpdatePlaylistRequest()..id = id;
    if (name != null) request.name = name;
    if (description != null) request.description = description;
    if (isPublic != null) request.isPublic = isPublic;
    final response = await _client.updatePlaylist(request);
    return response.playlist;
  }

  Future<void> deletePlaylist(String id) async {
    final request = DeletePlaylistRequest()..id = id;
    await _client.deletePlaylist(request);
  }

  Future<List<PlaylistSong>> listPlaylistSongs(String playlistId) async {
    final request = ListPlaylistSongsRequest()..playlistId = playlistId;
    final response = await _client.listPlaylistSongs(request);
    return response.songs.toList();
  }

  Future<PlaylistSong> addSong({
    required String playlistId,
    required String songId,
    int position = -1,
  }) async {
    final request = AddSongRequest()
      ..playlistId = playlistId
      ..songId = songId
      ..position = position;
    final response = await _client.addSong(request);
    return response.playlistSong;
  }

  Future<void> removeSong({
    required String playlistId,
    required String songId,
  }) async {
    final request = RemoveSongRequest()
      ..playlistId = playlistId
      ..songId = songId;
    await _client.removeSong(request);
  }

  Future<void> reorderSongs({
    required String playlistId,
    required List<String> songIds,
  }) async {
    final request = ReorderSongsRequest()
      ..playlistId = playlistId
      ..songIds.addAll(songIds);
    await _client.reorderSongs(request);
  }
}
```

- [ ] **步骤 2：运行分析器**

运行：`cd echo_vault_app && flutter analyze lib/features/playlist/services/playlist_repository.dart`
预期：未发现问题

- [ ] **步骤 3：提交**

```bash
git add lib/features/playlist/services/playlist_repository.dart
git commit -m "feat(playlist): 添加歌单 gRPC 仓储"
```

---

## 任务 3：歌单列表页

**文件：**
- 创建：`lib/features/playlist/pages/playlist_list_page.dart`
- 创建：`lib/features/playlist/widgets/playlist_tile.dart`
- 测试：手动验证

- [ ] **步骤 1：创建歌单项组件**

```dart
// lib/features/playlist/widgets/playlist_tile.dart
import 'package:flutter/material.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';

class PlaylistTile extends StatelessWidget {
  final Playlist playlist;
  final VoidCallback? onTap;
  final VoidCallback? onDelete;

  const PlaylistTile({
    super.key,
    required this.playlist,
    this.onTap,
    this.onDelete,
  });

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: CircleAvatar(
        child: Text(playlist.name.isNotEmpty ? playlist.name[0].toUpperCase() : '?'),
      ),
      title: Text(playlist.name),
      subtitle: Text('${playlist.songCount} 首歌曲'),
      trailing: PopupMenuButton<String>(
        onSelected: (value) {
          if (value == 'delete' && onDelete != null) {
            onDelete!();
          }
        },
        itemBuilder: (context) => [
          const PopupMenuItem(value: 'delete', child: Text('删除')),
        ],
      ),
      onTap: onTap,
    );
  }
}
```

- [ ] **步骤 2：创建歌单列表页**

```dart
// lib/features/playlist/pages/playlist_list_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_provider.dart';
import 'package:echo_vault_app/features/playlist/widgets/playlist_tile.dart';

class PlaylistListPage extends ConsumerStatefulWidget {
  const PlaylistListPage({super.key});

  @override
  ConsumerState<PlaylistListPage> createState() => _PlaylistListPageState();
}

class _PlaylistListPageState extends ConsumerState<PlaylistListPage> {
  @override
  void initState() {
    super.initState();
    // TODO: 从仓储加载歌单
  }

  @override
  Widget build(BuildContext context) {
    final state = ref.watch(playlistListProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('我的歌单'),
        actions: [
          IconButton(
            icon: const Icon(Icons.add),
            onPressed: () => _showCreateDialog(context),
          ),
        ],
      ),
      body: _buildBody(state),
    );
  }

  Widget _buildBody(PlaylistListState state) {
    switch (state.status) {
      case PlaylistStatus.loading:
        return const Center(child: CircularProgressIndicator());
      case PlaylistStatus.error:
        return Center(child: Text('错误: ${state.error}'));
      case PlaylistStatus.loaded:
        if (state.playlists.isEmpty) {
          return const Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                Icon(Icons.queue_music, size: 64, color: Colors.grey),
                SizedBox(height: 16),
                Text('还没有歌单', style: TextStyle(color: Colors.grey)),
              ],
            ),
          );
        }
        return ListView.builder(
          itemCount: state.playlists.length,
          itemBuilder: (context, index) {
            final playlist = state.playlists[index];
            return PlaylistTile(
              playlist: playlist,
              onTap: () => context.push('/playlist/${playlist.id}'),
              onDelete: () => _deletePlaylist(playlist.id),
            );
          },
        );
      default:
        return const SizedBox.shrink();
    }
  }

  void _showCreateDialog(BuildContext context) {
    final nameController = TextEditingController();
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('创建歌单'),
        content: TextField(
          controller: nameController,
          decoration: const InputDecoration(
            hintText: '歌单名称',
            border: OutlineInputBorder(),
          ),
          autofocus: true,
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('取消'),
          ),
          FilledButton(
            onPressed: () {
              // TODO: 通过仓储创建歌单
              Navigator.pop(context);
            },
            child: const Text('创建'),
          ),
        ],
      ),
    );
  }

  void _deletePlaylist(String id) {
    // TODO: 通过仓储删除歌单
  }
}
```

- [ ] **步骤 3：运行分析器**

运行：`cd echo_vault_app && flutter analyze lib/features/playlist/`
预期：未发现问题

- [ ] **步骤 4：提交**

```bash
git add lib/features/playlist/
git commit -m "feat(playlist): 添加歌单列表页，支持创建/删除"
```

---

## 任务 4：歌单详情页

**文件：**
- 创建：`lib/features/playlist/pages/playlist_detail_page.dart`
- 测试：手动验证

- [ ] **步骤 1：创建歌单详情页**

```dart
// lib/features/playlist/pages/playlist_detail_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_provider.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';

class PlaylistDetailPage extends ConsumerStatefulWidget {
  final String playlistId;

  const PlaylistDetailPage({super.key, required this.playlistId});

  @override
  ConsumerState<PlaylistDetailPage> createState() => _PlaylistDetailPageState();
}

class _PlaylistDetailPageState extends ConsumerState<PlaylistDetailPage> {
  Playlist? _playlist;
  List<PlaylistSong> _songs = [];
  bool _loading = true;

  @override
  void initState() {
    super.initState();
    _loadPlaylist();
  }

  Future<void> _loadPlaylist() async {
    // TODO: 从仓储加载歌单和歌曲
    setState(() => _loading = false);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(_playlist?.name ?? '歌单详情'),
        actions: [
          if (_playlist != null)
            IconButton(
              icon: const Icon(Icons.edit),
              onPressed: () => _showEditDialog(context),
            ),
        ],
      ),
      body: _loading
          ? const Center(child: CircularProgressIndicator())
          : _songs.isEmpty
              ? const Center(
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      Icon(Icons.music_note, size: 64, color: Colors.grey),
                      SizedBox(height: 16),
                      Text('歌单是空的', style: TextStyle(color: Colors.grey)),
                    ],
                  ),
                )
              : _buildSongList(),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showAddSongDialog(context),
        child: const Icon(Icons.add),
      ),
    );
  }

  Widget _buildSongList() {
    return ReorderableListView.builder(
      itemCount: _songs.length,
      onReorder: (oldIndex, newIndex) => _reorderSongs(oldIndex, newIndex),
      itemBuilder: (context, index) {
        final playlistSong = _songs[index];
        final song = playlistSong.song;
        return ListTile(
          key: ValueKey(song.id),
          leading: CircleAvatar(child: Text('${index + 1}')),
          title: Text(song.title.isNotEmpty ? song.title : '未知标题'),
          subtitle: Text(song.artist.isNotEmpty ? song.artist : '未知艺术家'),
          trailing: IconButton(
            icon: const Icon(Icons.remove_circle_outline),
            onPressed: () => _removeSong(song.id),
          ),
        );
      },
    );
  }

  void _showEditDialog(BuildContext context) {
    // TODO: 实现编辑对话框
  }

  void _showAddSongDialog(BuildContext context) {
    // TODO: 实现添加歌曲对话框
  }

  void _reorderSongs(int oldIndex, int newIndex) {
    // TODO: 实现排序
  }

  void _removeSong(String songId) {
    // TODO: 实现移除歌曲
  }
}
```

- [ ] **步骤 2：运行分析器**

运行：`cd echo_vault_app && flutter analyze lib/features/playlist/pages/playlist_detail_page.dart`
预期：未发现问题

- [ ] **步骤 3：提交**

```bash
git add lib/features/playlist/pages/playlist_detail_page.dart
git commit -m "feat(playlist): 添加歌单详情页，含歌曲列表"
```

---

## 任务 5：设备提供者（Provider）

**文件：**
- 创建：`lib/features/device/providers/device_provider.dart`
- 测试：手动验证

- [ ] **步骤 1：创建设备状态模型**

```dart
// lib/features/device/providers/device_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/models/generated/echo_vault/user/v1/user_service.pb.dart';

enum DeviceStatus { initial, loading, loaded, error }

class DeviceListState {
  final DeviceStatus status;
  final List<Device> devices;
  final String? error;

  const DeviceListState({
    this.status = DeviceStatus.initial,
    this.devices = const [],
    this.error,
  });
}

final deviceListProvider = StateNotifierProvider<DeviceListNotifier, DeviceListState>((ref) {
  return DeviceListNotifier();
});

class DeviceListNotifier extends StateNotifier<DeviceListState> {
  DeviceListNotifier() : super(const DeviceListState());

  void setLoading() => state = const DeviceListState(status: DeviceStatus.loading);
  void setDevices(List<Device> devices) => state = DeviceListState(status: DeviceStatus.loaded, devices: devices);
  void setError(String message) => state = DeviceListState(status: DeviceStatus.error, error: message);
  void removeDevice(String deviceId) {
    state = DeviceListState(
      status: state.status,
      devices: state.devices.where((d) => d.deviceId != deviceId).toList(),
    );
  }
  void updateDevice(Device updated) {
    state = DeviceListState(
      status: state.status,
      devices: state.devices.map((d) => d.deviceId == updated.deviceId ? updated : d).toList(),
    );
  }
}
```

- [ ] **步骤 2：运行分析器**

运行：`cd echo_vault_app && flutter analyze lib/features/device/providers/device_provider.dart`
预期：未发现问题

- [ ] **步骤 3：提交**

```bash
git add lib/features/device/providers/device_provider.dart
git commit -m "feat(device): 添加设备状态管理提供者"
```

---

## 任务 6：设备仓储（gRPC 服务）

**文件：**
- 创建：`lib/features/device/services/device_repository.dart`
- 测试：手动验证

- [ ] **步骤 1：创建设备仓储**

```dart
// lib/features/device/services/device_repository.dart
import 'package:grpc/grpc.dart';
import 'package:echo_vault_app/models/generated/echo_vault/user/v1/user_service.pb.dart';
import 'package:echo_vault_app/models/generated/echo_vault/user/v1/user_service.pbgrpc.dart';
import 'package:echo_vault_app/core/grpc/client.dart';

class DeviceRepository {
  final GrpcClientManager _clientManager;

  DeviceRepository(this._clientManager);

  UserServiceClient get _client => UserServiceClient(_clientManager.channel);

  Future<List<Device>> listDevices() async {
    final response = await _client.listDevices(ListDevicesRequest());
    return response.devices.toList();
  }

  Future<Device> registerDevice({
    required String deviceId,
    required String deviceName,
    required String platform,
    String osVersion = '',
    String clientVersion = '',
  }) async {
    final request = RegisterDeviceRequest()
      ..deviceId = deviceId
      ..deviceName = deviceName
      ..platform = platform
      ..osVersion = osVersion
      ..clientVersion = clientVersion;
    final response = await _client.registerDevice(request);
    return response.device;
  }

  Future<void> removeDevice(String deviceId) async {
    final request = RemoveDeviceRequest()..deviceId = deviceId;
    await _client.removeDevice(request);
  }

  Future<Device> updateDevice({
    required String deviceId,
    required String deviceName,
  }) async {
    final request = UpdateDeviceRequest()
      ..deviceId = deviceId
      ..deviceName = deviceName;
    final response = await _client.updateDevice(request);
    return response.device;
  }
}
```

- [ ] **步骤 2：运行分析器**

运行：`cd echo_vault_app && flutter analyze lib/features/device/services/device_repository.dart`
预期：未发现问题

- [ ] **步骤 3：提交**

```bash
git add lib/features/device/services/device_repository.dart
git commit -m "feat(device): 添加设备 gRPC 仓储"
```

---

## 任务 7：设备列表页

**文件：**
- 创建：`lib/features/device/pages/device_list_page.dart`
- 创建：`lib/features/device/widgets/device_tile.dart`
- 测试：手动验证

- [ ] **步骤 1：创建设备项组件**

```dart
// lib/features/device/widgets/device_tile.dart
import 'package:flutter/material.dart';
import 'package:echo_vault_app/models/generated/echo_vault/user/v1/user_service.pb.dart';
import 'package:intl/intl.dart';

class DeviceTile extends StatelessWidget {
  final Device device;
  final bool isCurrentDevice;
  final VoidCallback? onDelete;

  const DeviceTile({
    super.key,
    required this.device,
    this.isCurrentDevice = false,
    this.onDelete,
  });

  IconData _getPlatformIcon() {
    switch (device.platform.toLowerCase()) {
      case 'windows':
        return Icons.desktop_windows;
      case 'macos':
        return Icons.laptop_mac;
      case 'linux':
        return Icons.computer;
      case 'android':
        return Icons.android;
      case 'ios':
        return Icons.phone_iphone;
      default:
        return Icons.device_unknown;
    }
  }

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: Icon(_getPlatformIcon()),
      title: Row(
        children: [
          Expanded(child: Text(device.deviceName)),
          if (isCurrentDevice)
            Container(
              padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
              decoration: BoxDecoration(
                color: Theme.of(context).colorScheme.primaryContainer,
                borderRadius: BorderRadius.circular(12),
              ),
              child: Text(
                '当前设备',
                style: TextStyle(
                  fontSize: 12,
                  color: Theme.of(context).colorScheme.onPrimaryContainer,
                ),
              ),
            ),
        ],
      ),
      subtitle: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text('${device.platform} • ${device.osVersion}'),
          if (device.hasLastSyncAt())
            Text(
              '上次同步: ${DateFormat('yyyy-MM-dd HH:mm').format(device.lastSyncAt.toDateTime())}',
              style: Theme.of(context).textTheme.bodySmall,
            ),
        ],
      ),
      trailing: isCurrentDevice
          ? null
          : IconButton(
              icon: const Icon(Icons.delete_outline),
              onPressed: onDelete,
            ),
    );
  }
}
```

- [ ] **步骤 2：创建设备列表页**

```dart
// lib/features/device/pages/device_list_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/device/providers/device_provider.dart';
import 'package:echo_vault_app/features/device/widgets/device_tile.dart';

class DeviceListPage extends ConsumerStatefulWidget {
  const DeviceListPage({super.key});

  @override
  ConsumerState<DeviceListPage> createState() => _DeviceListPageState();
}

class _DeviceListPageState extends ConsumerState<DeviceListPage> {
  @override
  void initState() {
    super.initState();
    // TODO: 从仓储加载设备列表
  }

  @override
  Widget build(BuildContext context) {
    final state = ref.watch(deviceListProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('设备管理')),
      body: _buildBody(state),
    );
  }

  Widget _buildBody(DeviceListState state) {
    switch (state.status) {
      case DeviceStatus.loading:
        return const Center(child: CircularProgressIndicator());
      case DeviceStatus.error:
        return Center(child: Text('错误: ${state.error}'));
      case DeviceStatus.loaded:
        if (state.devices.isEmpty) {
          return const Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                Icon(Icons.devices, size: 64, color: Colors.grey),
                SizedBox(height: 16),
                Text('没有已注册的设备', style: TextStyle(color: Colors.grey)),
              ],
            ),
          );
        }
        return ListView.builder(
          itemCount: state.devices.length,
          itemBuilder: (context, index) {
            final device = state.devices[index];
            return DeviceTile(
              device: device,
              isCurrentDevice: false, // TODO: 检测当前设备
              onDelete: () => _removeDevice(device.deviceId),
            );
          },
        );
      default:
        return const SizedBox.shrink();
    }
  }

  void _removeDevice(String deviceId) {
    // TODO: 实现移除设备
  }
}
```

- [ ] **步骤 3：运行分析器**

运行：`cd echo_vault_app && flutter analyze lib/features/device/`
预期：未发现问题

- [ ] **步骤 4：添加 intl 依赖**

运行：`cd echo_vault_app && flutter pub add intl`

- [ ] **步骤 5：提交**

```bash
git add lib/features/device/
git add pubspec.yaml pubspec.lock
git commit -m "feat(device): 添加设备列表页，含平台图标"
```

---

## 任务 8：同步提供者（Provider）

**文件：**
- 创建：`lib/features/sync/providers/sync_provider.dart`
- 测试：手动验证

- [ ] **步骤 1：创建同步状态模型**

```dart
// lib/features/sync/providers/sync_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

enum SyncStatus { idle, syncing, error, completed }

class SyncState {
  final SyncStatus status;
  final int pendingChanges;
  final int syncedChanges;
  final String? error;
  final DateTime? lastSyncTime;

  const SyncState({
    this.status = SyncStatus.idle,
    this.pendingChanges = 0,
    this.syncedChanges = 0,
    this.error,
    this.lastSyncTime,
  });
}

final syncProvider = StateNotifierProvider<SyncNotifier, SyncState>((ref) {
  return SyncNotifier();
});

class SyncNotifier extends StateNotifier<SyncState> {
  SyncNotifier() : super(const SyncState());

  void setIdle() => state = const SyncState();
  void setSyncing({int pending = 0, int synced = 0}) {
    state = SyncState(
      status: SyncStatus.syncing,
      pendingChanges: pending,
      syncedChanges: synced,
      lastSyncTime: state.lastSyncTime,
    );
  }
  void setCompleted() {
    state = SyncState(
      status: SyncStatus.completed,
      lastSyncTime: DateTime.now(),
    );
  }
  void setError(String message) {
    state = SyncState(
      status: SyncStatus.error,
      error: message,
      lastSyncTime: state.lastSyncTime,
    );
  }
  void setPendingChanges(int count) {
    state = SyncState(
      status: state.status,
      pendingChanges: count,
      lastSyncTime: state.lastSyncTime,
    );
  }
}
```

- [ ] **步骤 2：运行分析器**

运行：`cd echo_vault_app && flutter analyze lib/features/sync/providers/sync_provider.dart`
预期：未发现问题

- [ ] **步骤 3：提交**

```bash
git add lib/features/sync/providers/sync_provider.dart
git commit -m "feat(sync): 添加同步状态管理提供者"
```

---

## 任务 9：同步状态页

**文件：**
- 创建：`lib/features/sync/pages/sync_status_page.dart`
- 创建：`lib/features/sync/widgets/sync_status_indicator.dart`
- 测试：手动验证

- [ ] **步骤 1：创建同步状态指示器组件**

```dart
// lib/features/sync/widgets/sync_status_indicator.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/sync/providers/sync_provider.dart';
import 'package:intl/intl.dart';

class SyncStatusIndicator extends ConsumerWidget {
  const SyncStatusIndicator({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(syncProvider);

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
      decoration: BoxDecoration(
        color: _getStatusColor(context, state.status),
        borderRadius: BorderRadius.circular(16),
      ),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          _buildIcon(state.status),
          const SizedBox(width: 8),
          Text(
            _getStatusText(state),
            style: TextStyle(
              color: _getStatusTextColor(context, state.status),
              fontSize: 12,
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildIcon(SyncStatus status) {
    switch (status) {
      case SyncStatus.syncing:
        return const SizedBox(
          width: 12,
          height: 12,
          child: CircularProgressIndicator(strokeWidth: 2),
        );
      case SyncStatus.completed:
        return const Icon(Icons.check_circle, size: 12);
      case SyncStatus.error:
        return const Icon(Icons.error, size: 12);
      case SyncStatus.idle:
        return const Icon(Icons.sync, size: 12);
    }
  }

  String _getStatusText(SyncState state) {
    switch (state.status) {
      case SyncStatus.syncing:
        return '同步中 (${state.syncedChanges}/${state.pendingChanges})';
      case SyncStatus.completed:
        if (state.lastSyncTime != null) {
          return '上次同步: ${DateFormat('HH:mm').format(state.lastSyncTime!)}';
        }
        return '已同步';
      case SyncStatus.error:
        return '同步失败';
      case SyncStatus.idle:
        return '等待同步';
    }
  }

  Color _getStatusColor(BuildContext context, SyncStatus status) {
    switch (status) {
      case SyncStatus.syncing:
        return Theme.of(context).colorScheme.primaryContainer;
      case SyncStatus.completed:
        return Theme.of(context).colorScheme.secondaryContainer;
      case SyncStatus.error:
        return Theme.of(context).colorScheme.errorContainer;
      case SyncStatus.idle:
        return Theme.of(context).colorScheme.surfaceVariant;
    }
  }

  Color _getStatusTextColor(BuildContext context, SyncStatus status) {
    switch (status) {
      case SyncStatus.syncing:
        return Theme.of(context).colorScheme.onPrimaryContainer;
      case SyncStatus.completed:
        return Theme.of(context).colorScheme.onSecondaryContainer;
      case SyncStatus.error:
        return Theme.of(context).colorScheme.onErrorContainer;
      case SyncStatus.idle:
        return Theme.of(context).colorScheme.onSurfaceVariant;
    }
  }
}
```

- [ ] **步骤 2：创建同步状态页**

```dart
// lib/features/sync/pages/sync_status_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/sync/providers/sync_provider.dart';
import 'package:echo_vault_app/features/sync/widgets/sync_status_indicator.dart';
import 'package:intl/intl.dart';

class SyncStatusPage extends ConsumerWidget {
  const SyncStatusPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(syncProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('同步状态'),
        actions: [
          IconButton(
            icon: const Icon(Icons.sync),
            onPressed: () => _manualSync(ref),
          ),
        ],
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          Card(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Row(
                    mainAxisAlignment: MainAxisAlignment.spaceBetween,
                    children: [
                      const Text('同步状态', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
                      const SyncStatusIndicator(),
                    ],
                  ),
                  const SizedBox(height: 16),
                  _buildInfoRow('待同步变更', '${state.pendingChanges}'),
                  _buildInfoRow('已同步变更', '${state.syncedChanges}'),
                  if (state.lastSyncTime != null)
                    _buildInfoRow('上次同步', DateFormat('yyyy-MM-dd HH:mm:ss').format(state.lastSyncTime!)),
                  if (state.error != null)
                    Padding(
                      padding: const EdgeInsets.only(top: 8),
                      child: Text(
                        '错误: ${state.error}',
                        style: TextStyle(color: Theme.of(context).colorScheme.error),
                      ),
                    ),
                ],
              ),
            ),
          ),
          const SizedBox(height: 16),
          Card(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const Text('同步说明', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
                  const SizedBox(height: 8),
                  const Text('• 离线时修改会自动缓存'),
                  const Text('• 联网后自动同步到服务器'),
                  const Text('• 冲突时优先保留服务端数据'),
                  const Text('• 歌曲文件需要手动上传'),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildInfoRow(String label, String value) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(label, style: const TextStyle(color: Colors.grey)),
          Text(value),
        ],
      ),
    );
  }

  void _manualSync(WidgetRef ref) {
    // TODO: 触发手动同步
  }
}
```

- [ ] **步骤 3：运行分析器**

运行：`cd echo_vault_app && flutter analyze lib/features/sync/`
预期：未发现问题

- [ ] **步骤 4：提交**

```bash
git add lib/features/sync/
git commit -m "feat(sync): 添加同步状态页，支持手动同步"
```

---

## 任务 10：更新路由

**文件：**
- 修改：`lib/router.dart`
- 测试：手动验证

- [ ] **步骤 1：在路由中添加新路由**

```dart
// lib/router.dart - 在顶部添加这些导入
import 'package:echo_vault_app/features/playlist/pages/playlist_list_page.dart';
import 'package:echo_vault_app/features/playlist/pages/playlist_detail_page.dart';
import 'package:echo_vault_app/features/device/pages/device_list_page.dart';
import 'package:echo_vault_app/features/sync/pages/sync_status_page.dart';

// 在 GoRouter 的 routes 列表中添加这些路由
GoRoute(
  path: '/playlists',
  builder: (_, __) => const PlaylistListPage(),
),
GoRoute(
  path: '/playlist/:id',
  builder: (context, state) {
    final id = state.pathParameters['id']!;
    return PlaylistDetailPage(playlistId: id);
  },
),
GoRoute(
  path: '/devices',
  builder: (_, __) => const DeviceListPage(),
),
GoRoute(
  path: '/sync',
  builder: (_, __) => const SyncStatusPage(),
),
```

- [ ] **步骤 2：运行分析器**

运行：`cd echo_vault_app && flutter analyze lib/router.dart`
预期：未发现问题

- [ ] **步骤 3：提交**

```bash
git add lib/router.dart
git commit -m "feat(router): 添加歌单、设备和同步路由"
```

---

## 任务 11：更新主导航

**文件：**
- 修改：`lib/main.dart`
- 测试：手动验证

- [ ] **步骤 1：添加导航抽屉或底部导航**

在主脚手架中添加导航抽屉，包含以下链接：
- 我的曲库（Library）
- 我的歌单（Playlists）
- 设备管理（Devices）
- 同步状态（Sync）

```dart
// 添加到 main.dart 或创建新的带导航的 shell 页面
Drawer(
  child: ListView(
    padding: EdgeInsets.zero,
    children: [
      DrawerHeader(
        decoration: BoxDecoration(
          color: Theme.of(context).colorScheme.primary,
        ),
        child: Text(
          '音匣 EchoVault',
          style: TextStyle(
            color: Theme.of(context).colorScheme.onPrimary,
            fontSize: 24,
          ),
        ),
      ),
      ListTile(
        leading: const Icon(Icons.library_music),
        title: const Text('我的曲库'),
        onTap: () => context.go('/library'),
      ),
      ListTile(
        leading: const Icon(Icons.queue_music),
        title: const Text('我的歌单'),
        onTap: () => context.go('/playlists'),
      ),
      ListTile(
        leading: const Icon(Icons.devices),
        title: const Text('设备管理'),
        onTap: () => context.go('/devices'),
      ),
      ListTile(
        leading: const Icon(Icons.sync),
        title: const Text('同步状态'),
        onTap: () => context.go('/sync'),
      ),
      const Divider(),
      ListTile(
        leading: const Icon(Icons.logout),
        title: const Text('退出登录'),
        onTap: () {
          // TODO: 退出登录
        },
      ),
    ],
  ),
)
```

- [ ] **步骤 2：运行分析器**

运行：`cd echo_vault_app && flutter analyze lib/main.dart`
预期：未发现问题

- [ ] **步骤 3：提交**

```bash
git add lib/main.dart
git commit -m "feat(navigation): 添加导航抽屉，包含所有功能模块"
```

---

## 任务 12：集成测试

**文件：**
- 测试：手动验证

- [ ] **步骤 1：运行完整应用分析**

运行：`cd echo_vault_app && flutter analyze`
预期：未发现问题

- [ ] **步骤 2：运行 Web 构建**

运行：`cd echo_vault_app && flutter build web`
预期：构建成功

- [ ] **步骤 3：最终提交**

```bash
git add -A
git commit -m "feat(phase9): 完成歌单、设备和同步界面实现"
```

---

## 总结

本计划实现：

1. **歌单管理**：创建、编辑、删除歌单；添加/移除/排序歌曲
2. **设备管理**：列出设备、显示平台图标、移除设备
3. **同步状态**：显示同步进度、手动触发同步、同步历史

所有任务遵循现有代码库模式：
- Riverpod 用于状态管理
- gRPC 用于服务器通信
- 基于功能的目录结构
- 一致的组件模式

**预计时间：** 4-6 小时
