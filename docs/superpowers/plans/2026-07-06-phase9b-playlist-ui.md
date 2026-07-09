# Phase 9b: Playlist UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement complete playlist UI including list page, detail page, and reusable widgets for EchoVault Flutter app.

**Architecture:** Follow existing patterns in library feature - use Riverpod for state management, ConsumerWidget for pages, and StatelessWidget for reusable widgets. UI follows Material Design 3 with consistent theming.

**Tech Stack:** Flutter, Riverpod, Material Design 3, existing playlist providers/repository

---

## File Structure

```
echo_vault_app/lib/features/playlist/
├── models/
│   ├── playlist_model.dart (existing)
│   └── playlist_song_model.dart (existing)
├── providers/
│   ├── playlist_provider.dart (existing)
│   └── playlist_song_provider.dart (existing)
├── services/
│   └── playlist_repository.dart (existing)
├── pages/
│   ├── playlist_list_page.dart (NEW)
│   └── playlist_detail_page.dart (NEW)
└── widgets/
    ├── playlist_tile.dart (NEW)
    └── add_song_dialog.dart (NEW)

echo_vault_app/test/features/playlist/
├── pages/
│   ├── playlist_list_page_test.dart (NEW)
│   └── playlist_detail_page_test.dart (NEW)
└── widgets/
    ├── playlist_tile_test.dart (NEW)
    └── add_song_dialog_test.dart (NEW)
```

---

## Task 1: PlaylistTile Widget

**Files:**
- Create: `echo_vault_app/lib/features/playlist/widgets/playlist_tile.dart`
- Test: `echo_vault_app/test/features/playlist/widgets/playlist_tile_test.dart`

- [ ] **Step 1: Write failing test for PlaylistTile**

```dart
// echo_vault_app/test/features/playlist/widgets/playlist_tile_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:echo_vault_app/features/playlist/widgets/playlist_tile.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';

void main() {
  testWidgets('PlaylistTile displays playlist info', (tester) async {
    final playlist = Playlist(
      id: 'p1',
      name: 'My Playlist',
      description: 'Test description',
      songCount: 10,
    );

    await tester.pumpWidget(
      MaterialApp(
        home: Scaffold(
          body: PlaylistTile(
            playlist: playlist,
            onTap: () {},
          ),
        ),
      ),
    );

    expect(find.text('My Playlist'), findsOneWidget);
    expect(find.text('10 首歌曲'), findsOneWidget);
    expect(find.byIcon(Icons.playlist_play), findsOneWidget);
  });

  testWidgets('PlaylistTile shows default cover when no coverUrl', (tester) async {
    final playlist = Playlist(
      id: 'p1',
      name: 'Test',
      coverUrl: '',
    );

    await tester.pumpWidget(
      MaterialApp(
        home: Scaffold(
          body: PlaylistTile(
            playlist: playlist,
            onTap: () {},
          ),
        ),
      ),
    );

    expect(find.byIcon(Icons.music_note), findsOneWidget);
  });
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/ink/Desktop/develop/EchoVault/echo_vault_app && flutter test test/features/playlist/widgets/playlist_tile_test.dart`
Expected: FAIL with "PlaylistTile not found"

- [ ] **Step 3: Implement PlaylistTile widget**

```dart
// echo_vault_app/lib/features/playlist/widgets/playlist_tile.dart
import 'package:flutter/material.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';

class PlaylistTile extends StatelessWidget {
  final Playlist playlist;
  final VoidCallback? onTap;
  final VoidCallback? onLongPress;

  const PlaylistTile({
    super.key,
    required this.playlist,
    this.onTap,
    this.onLongPress,
  });

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return ListTile(
      leading: _buildCover(theme),
      title: Text(
        playlist.name,
        maxLines: 1,
        overflow: TextOverflow.ellipsis,
      ),
      subtitle: Text(
        '${playlist.songCount} 首歌曲',
        style: theme.textTheme.bodySmall?.copyWith(
          color: theme.colorScheme.outline,
        ),
      ),
      trailing: IconButton(
        icon: const Icon(Icons.more_vert),
        onPressed: () => _showOptions(context),
      ),
      onTap: onTap,
      onLongPress: onLongPress,
    );
  }

  Widget _buildCover(ThemeData theme) {
    if (playlist.coverUrl.isNotEmpty) {
      return ClipRRect(
        borderRadius: BorderRadius.circular(8),
        child: Image.network(
          playlist.coverUrl,
          width: 56,
          height: 56,
          fit: BoxFit.cover,
          errorBuilder: (_, __, ___) => _buildDefaultCover(theme),
        ),
      );
    }
    return _buildDefaultCover(theme);
  }

  Widget _buildDefaultCover(ThemeData theme) {
    return Container(
      width: 56,
      height: 56,
      decoration: BoxDecoration(
        color: theme.colorScheme.primaryContainer,
        borderRadius: BorderRadius.circular(8),
      ),
      child: Icon(
        Icons.music_note,
        color: theme.colorScheme.onPrimaryContainer,
      ),
    );
  }

  void _showOptions(BuildContext context) {
    showModalBottomSheet(
      context: context,
      builder: (context) => SafeArea(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            ListTile(
              leading: const Icon(Icons.edit),
              title: const Text('编辑歌单'),
              onTap: () {
                Navigator.pop(context);
                // TODO: Implement edit
              },
            ),
            ListTile(
              leading: const Icon(Icons.delete, color: Colors.red),
              title: const Text('删除歌单', style: TextStyle(color: Colors.red)),
              onTap: () {
                Navigator.pop(context);
                // TODO: Implement delete
              },
            ),
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/ink/Desktop/develop/EchoVault/echo_vault_app && flutter test test/features/playlist/widgets/playlist_tile_test.dart`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/playlist/widgets/playlist_tile.dart
git add echo_vault_app/test/features/playlist/widgets/playlist_tile_test.dart
git commit -m "feat(playlist): add PlaylistTile widget with cover and song count display"
```

---

## Task 2: PlaylistListPage

**Files:**
- Create: `echo_vault_app/lib/features/playlist/pages/playlist_list_page.dart`
- Test: `echo_vault_app/test/features/playlist/pages/playlist_list_page_test.dart`

- [ ] **Step 1: Write failing test for PlaylistListPage**

```dart
// echo_vault_app/test/features/playlist/pages/playlist_list_page_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/playlist/pages/playlist_list_page.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_provider.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';

class MockPlaylistRepository implements PlaylistRepository {
  List<Playlist> playlists = [];

  @override
  Future<Playlist> createPlaylist({required String name, String? description, bool isPublic = false}) async {
    final playlist = Playlist(id: 'p${playlists.length + 1}', name: name, description: description ?? '');
    playlists.add(playlist);
    return playlist;
  }

  @override
  Future<void> deletePlaylist(String id) async {
    playlists.removeWhere((p) => p.id == id);
  }

  @override
  Future<List<Playlist>> listPlaylists({int pageSize = 20}) async => playlists;

  @override
  Future<Playlist> getPlaylist(String id) async => throw UnimplementedError();

  @override
  Future<Playlist> updatePlaylist({required String id, String? name, String? description, bool? isPublic}) async {
    throw UnimplementedError();
  }

  @override
  Future<PlaylistSong> addSongToPlaylist({required String playlistId, required String songId, int position = 0}) async => throw UnimplementedError();

  @override
  Future<void> removeSongFromPlaylist({required String playlistId, required String songId}) async => throw UnimplementedError();

  @override
  Future<void> reorderSongs({required String playlistId, required List<String> songIds}) async => throw UnimplementedError();

  @override
  Future<List<PlaylistSong>> listPlaylistSongs({required String playlistId, int pageSize = 100}) async => throw UnimplementedError();
}

void main() {
  testWidgets('PlaylistListPage shows empty state', (tester) async {
    final mockRepo = MockPlaylistRepository();

    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          playlistRepositoryProvider.overrideWithValue(mockRepo),
        ],
        child: const MaterialApp(
          home: PlaylistListPage(),
        ),
      ),
    );

    await tester.pumpAndSettle();

    expect(find.text('我的歌单'), findsOneWidget);
    expect(find.text('暂无歌单，点击 + 创建'), findsOneWidget);
  });

  testWidgets('PlaylistListPage shows playlists', (tester) async {
    final mockRepo = MockPlaylistRepository();
    mockRepo.playlists = [
      Playlist(id: 'p1', name: 'Rock Music', songCount: 10),
      Playlist(id: 'p2', name: 'Pop Music', songCount: 20),
    ];

    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          playlistRepositoryProvider.overrideWithValue(mockRepo),
        ],
        child: const MaterialApp(
          home: PlaylistListPage(),
        ),
      ),
    );

    await tester.pumpAndSettle();

    expect(find.text('Rock Music'), findsOneWidget);
    expect(find.text('Pop Music'), findsOneWidget);
  });
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/ink/Desktop/develop/EchoVault/echo_vault_app && flutter test test/features/playlist/pages/playlist_list_page_test.dart`
Expected: FAIL with "PlaylistListPage not found"

- [ ] **Step 3: Implement PlaylistListPage**

```dart
// echo_vault_app/lib/features/playlist/pages/playlist_list_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
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
    Future.microtask(() => ref.read(playlistProvider.notifier).loadPlaylists());
  }

  @override
  Widget build(BuildContext context) {
    final state = ref.watch(playlistProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('我的歌单'),
        actions: [
          IconButton(
            icon: const Icon(Icons.search),
            onPressed: () => _showSearch(context),
          ),
        ],
      ),
      body: _buildBody(context, state),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showCreateDialog(context),
        child: const Icon(Icons.add),
      ),
    );
  }

  Widget _buildBody(BuildContext context, PlaylistState state) {
    switch (state.status) {
      case PlaylistStatus.loading:
        return const Center(child: CircularProgressIndicator());
      case PlaylistStatus.error:
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
      case PlaylistStatus.loaded:
        final playlists = state.filteredPlaylists;
        if (playlists.isEmpty) {
          return Center(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                Icon(Icons.playlist_play, size: 64, color: Theme.of(context).colorScheme.outline),
                const SizedBox(height: 16),
                Text(
                  state.searchQuery.isEmpty ? '暂无歌单，点击 + 创建' : '未找到匹配的歌单',
                ),
              ],
            ),
          );
        }
        return RefreshIndicator(
          onRefresh: () async => ref.read(playlistProvider.notifier).loadPlaylists(),
          child: ListView.separated(
            itemCount: playlists.length,
            separatorBuilder: (_, __) => const Divider(height: 1),
            itemBuilder: (_, i) => PlaylistTile(
              playlist: playlists[i],
              onTap: () => Navigator.pushNamed(context, '/playlist/${playlists[i].id}'),
            ),
          ),
        );
    }
  }

  void _showSearch(BuildContext context) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('搜索歌单'),
        content: TextField(
          autofocus: true,
          decoration: const InputDecoration(
            hintText: '输入歌单名称...',
            prefixIcon: Icon(Icons.search),
          ),
          onChanged: (v) => ref.read(playlistProvider.notifier).setQuery(v),
        ),
        actions: [
          TextButton(
            onPressed: () {
              ref.read(playlistProvider.notifier).setQuery('');
              Navigator.pop(context);
            },
            child: const Text('清除'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('完成'),
          ),
        ],
      ),
    );
  }

  void _showCreateDialog(BuildContext context) {
    final nameController = TextEditingController();
    final descController = TextEditingController();

    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('创建歌单'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(
              controller: nameController,
              decoration: const InputDecoration(
                labelText: '歌单名称',
                hintText: '输入歌单名称',
              ),
              autofocus: true,
            ),
            const SizedBox(height: 16),
            TextField(
              controller: descController,
              decoration: const InputDecoration(
                labelText: '描述（可选）',
                hintText: '输入歌单描述',
              ),
              maxLines: 2,
            ),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('取消'),
          ),
          FilledButton(
            onPressed: () async {
              if (nameController.text.trim().isNotEmpty) {
                await ref.read(playlistProvider.notifier).createPlaylist(
                  name: nameController.text.trim(),
                  description: descController.text.trim().isNotEmpty ? descController.text.trim() : null,
                );
                if (context.mounted) Navigator.pop(context);
              }
            },
            child: const Text('创建'),
          ),
        ],
      ),
    );
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/ink/Desktop/develop/EchoVault/echo_vault_app && flutter test test/features/playlist/pages/playlist_list_page_test.dart`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/playlist/pages/playlist_list_page.dart
git add echo_vault_app/test/features/playlist/pages/playlist_list_page_test.dart
git commit -m "feat(playlist): add PlaylistListPage with create, search, and list display"
```

---

## Task 3: AddSongDialog Widget

**Files:**
- Create: `echo_vault_app/lib/features/playlist/widgets/add_song_dialog.dart`
- Test: `echo_vault_app/test/features/playlist/widgets/add_song_dialog_test.dart`

- [ ] **Step 1: Write failing test for AddSongDialog**

```dart
// echo_vault_app/test/features/playlist/widgets/add_song_dialog_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/playlist/widgets/add_song_dialog.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_provider.dart';
import 'package:echo_vault_app/features/library/providers/library_provider.dart';
import 'package:echo_vault_app/models/generated/echo_vault/song/v1/song_service.pb.dart';

void main() {
  testWidgets('AddSongDialog shows search field and song list', (tester) async {
    await tester.pumpWidget(
      ProviderScope(
        child: MaterialApp(
          home: Scaffold(
            body: AddSongDialog(playlistId: 'p1'),
          ),
        ),
      ),
    );

    expect(find.text('添加歌曲'), findsOneWidget);
    expect(find.byType(TextField), findsOneWidget);
  });
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/ink/Desktop/develop/EchoVault/echo_vault_app && flutter test test/features/playlist/widgets/add_song_dialog_test.dart`
Expected: FAIL with "AddSongDialog not found"

- [ ] **Step 3: Implement AddSongDialog**

```dart
// echo_vault_app/lib/features/playlist/widgets/add_song_dialog.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/providers/library_provider.dart';
import 'package:echo_vault_app/features/library/widgets/song_list_tile.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_song_provider.dart';

class AddSongDialog extends ConsumerStatefulWidget {
  final String playlistId;

  const AddSongDialog({super.key, required this.playlistId});

  @override
  ConsumerState<AddSongDialog> createState() => _AddSongDialogState();
}

class _AddSongDialogState extends ConsumerState<AddSongDialog> {
  String _searchQuery = '';

  @override
  Widget build(BuildContext context) {
    final libraryState = ref.watch(libraryProvider);
    final playlistSongs = ref.watch(playlistSongProvider(widget.playlistId));

    // Filter songs that are not already in the playlist
    final existingSongIds = playlistSongs.playlistSongs.map((ps) => ps.songId).toSet();
    final availableSongs = libraryState.songs
        .where((s) => !existingSongIds.contains(s.id))
        .where((s) {
          if (_searchQuery.isEmpty) return true;
          final q = _searchQuery.toLowerCase();
          return s.title.toLowerCase().contains(q) ||
                 s.artist.toLowerCase().contains(q) ||
                 s.album.toLowerCase().contains(q);
        })
        .toList();

    return Dialog(
      child: ConstrainedBox(
        constraints: BoxConstraints(
          maxHeight: MediaQuery.of(context).size.height * 0.7,
          maxWidth: MediaQuery.of(context).size.width * 0.9,
        ),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Padding(
              padding: const EdgeInsets.all(16),
              child: Row(
                children: [
                  const Expanded(
                    child: Text(
                      '添加歌曲',
                      style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
                    ),
                  ),
                  IconButton(
                    icon: const Icon(Icons.close),
                    onPressed: () => Navigator.pop(context),
                  ),
                ],
              ),
            ),
            Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16),
              child: TextField(
                decoration: InputDecoration(
                  hintText: '搜索歌曲...',
                  prefixIcon: const Icon(Icons.search),
                  border: OutlineInputBorder(
                    borderRadius: BorderRadius.circular(12),
                  ),
                ),
                onChanged: (v) => setState(() => _searchQuery = v),
              ),
            ),
            const SizedBox(height: 8),
            Expanded(
              child: availableSongs.isEmpty
                  ? Center(
                      child: Text(
                        _searchQuery.isEmpty ? '曲库为空' : '未找到匹配的歌曲',
                      ),
                    )
                  : ListView.builder(
                      itemCount: availableSongs.length,
                      itemBuilder: (_, i) {
                        final song = availableSongs[i];
                        return SongListTile(
                          song: song,
                          onTap: () async {
                            await ref
                                .read(playlistSongProvider(widget.playlistId).notifier)
                                .addSong(song.id);
                            if (context.mounted) {
                              ScaffoldMessenger.of(context).showSnackBar(
                                SnackBar(content: Text('已添加: ${song.title}')),
                              );
                            }
                          },
                        );
                      },
                    ),
            ),
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/ink/Desktop/develop/EchoVault/echo_vault_app && flutter test test/features/playlist/widgets/add_song_dialog_test.dart`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/playlist/widgets/add_song_dialog.dart
git add echo_vault_app/test/features/playlist/widgets/add_song_dialog_test.dart
git commit -m "feat(playlist): add AddSongDialog with search and song selection"
```

---

## Task 4: PlaylistDetailPage

**Files:**
- Create: `echo_vault_app/lib/features/playlist/pages/playlist_detail_page.dart`
- Test: `echo_vault_app/test/features/playlist/pages/playlist_detail_page_test.dart`

- [ ] **Step 1: Write failing test for PlaylistDetailPage**

```dart
// echo_vault_app/test/features/playlist/pages/playlist_detail_page_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/playlist/pages/playlist_detail_page.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_provider.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_song_provider.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';

class MockPlaylistRepository implements PlaylistRepository {
  List<Playlist> playlists = [];
  List<PlaylistSong> playlistSongs = [];

  @override
  Future<Playlist> createPlaylist({required String name, String? description, bool isPublic = false}) async => throw UnimplementedError();

  @override
  Future<void> deletePlaylist(String id) async => throw UnimplementedError();

  @override
  Future<List<Playlist>> listPlaylists({int pageSize = 20}) async => playlists;

  @override
  Future<Playlist> getPlaylist(String id) async => playlists.firstWhere((p) => p.id == id);

  @override
  Future<Playlist> updatePlaylist({required String id, String? name, String? description, bool? isPublic}) async => throw UnimplementedError();

  @override
  Future<PlaylistSong> addSongToPlaylist({required String playlistId, required String songId, int position = 0}) async {
    final ps = PlaylistSong(playlistId: playlistId, songId: songId, position: playlistSongs.length);
    playlistSongs.add(ps);
    return ps;
  }

  @override
  Future<void> removeSongFromPlaylist({required String playlistId, required String songId}) async {
    playlistSongs.removeWhere((ps) => ps.playlistId == playlistId && ps.songId == songId);
  }

  @override
  Future<void> reorderSongs({required String playlistId, required List<String> songIds}) async => throw UnimplementedError();

  @override
  Future<List<PlaylistSong>> listPlaylistSongs({required String playlistId, int pageSize = 100}) async => 
      playlistSongs.where((ps) => ps.playlistId == playlistId).toList();
}

void main() {
  testWidgets('PlaylistDetailPage shows playlist name and empty state', (tester) async {
    final mockRepo = MockPlaylistRepository();
    mockRepo.playlists = [Playlist(id: 'p1', name: 'Test Playlist', songCount: 0)];

    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          playlistRepositoryProvider.overrideWithValue(mockRepo),
        ],
        child: const MaterialApp(
          home: PlaylistDetailPage(playlistId: 'p1'),
        ),
      ),
    );

    await tester.pumpAndSettle();

    expect(find.text('Test Playlist'), findsOneWidget);
    expect(find.text('暂无歌曲，点击 + 添加'), findsOneWidget);
  });
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /home/ink/Desktop/develop/EchoVault/echo_vault_app && flutter test test/features/playlist/pages/playlist_detail_page_test.dart`
Expected: FAIL with "PlaylistDetailPage not found"

- [ ] **Step 3: Implement PlaylistDetailPage**

```dart
// echo_vault_app/lib/features/playlist/pages/playlist_detail_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/library/providers/library_provider.dart';
import 'package:echo_vault_app/features/library/widgets/song_list_tile.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_provider.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_song_provider.dart';
import 'package:echo_vault_app/features/playlist/widgets/add_song_dialog.dart';

class PlaylistDetailPage extends ConsumerStatefulWidget {
  final String playlistId;

  const PlaylistDetailPage({super.key, required this.playlistId});

  @override
  ConsumerState<PlaylistDetailPage> createState() => _PlaylistDetailPageState();
}

class _PlaylistDetailPageState extends ConsumerState<PlaylistDetailPage> {
  @override
  void initState() {
    super.initState();
    Future.microtask(() {
      ref.read(playlistSongProvider(widget.playlistId).notifier).loadSongs();
    });
  }

  @override
  Widget build(BuildContext context) {
    final playlistState = ref.watch(playlistProvider);
    final songState = ref.watch(playlistSongProvider(widget.playlistId));
    final libraryState = ref.watch(libraryProvider);

    // Find playlist
    final playlist = playlistState.playlists
        .where((p) => p.id == widget.playlistId)
        .firstOrNull;

    if (playlist == null) {
      return Scaffold(
        appBar: AppBar(title: const Text('歌单')),
        body: const Center(child: Text('歌单不存在')),
      );
    }

    // Get actual songs from library
    final playlistSongIds = songState.playlistSongs.map((ps) => ps.songId).toSet();
    final songs = libraryState.songs.where((s) => playlistSongIds.contains(s.id)).toList();

    return Scaffold(
      appBar: AppBar(
        title: Text(playlist.name),
        actions: [
          IconButton(
            icon: const Icon(Icons.edit),
            onPressed: () => _showEditDialog(context, playlist),
          ),
          IconButton(
            icon: const Icon(Icons.delete),
            onPressed: () => _confirmDelete(context),
          ),
        ],
      ),
      body: _buildBody(context, songState, songs),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showAddSongDialog(context),
        child: const Icon(Icons.add),
      ),
    );
  }

  Widget _buildBody(BuildContext context, PlaylistSongState songState, List<dynamic> songs) {
    switch (songState.status) {
      case PlaylistSongStatus.loading:
        return const Center(child: CircularProgressIndicator());
      case PlaylistSongStatus.error:
        return Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Icon(Icons.error_outline, size: 48, color: Theme.of(context).colorScheme.error),
              const SizedBox(height: 16),
              Text('加载失败: ${songState.error}'),
            ],
          ),
        );
      case PlaylistSongStatus.loaded:
        if (songs.isEmpty) {
          return Center(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                Icon(Icons.playlist_add, size: 64, color: Theme.of(context).colorScheme.outline),
                const SizedBox(height: 16),
                const Text('暂无歌曲，点击 + 添加'),
              ],
            ),
          );
        }
        return ReorderableListView.builder(
          itemCount: songs.length,
          onReorder: (oldIndex, newIndex) {
            if (oldIndex < newIndex) newIndex -= 1;
            final songIds = songs.map((s) => s.id as String).toList();
            final item = songIds.removeAt(oldIndex);
            songIds.insert(newIndex, item);
            ref.read(playlistSongProvider(widget.playlistId).notifier).reorderSongs(songIds);
          },
          itemBuilder: (_, i) {
            final song = songs[i];
            return Dismissible(
              key: ValueKey(song.id),
              direction: DismissDirection.endToStart,
              background: Container(
                color: Colors.red,
                alignment: Alignment.centerRight,
                padding: const EdgeInsets.only(right: 16),
                child: const Icon(Icons.delete, color: Colors.white),
              ),
              onDismissed: (_) {
                ref.read(playlistSongProvider(widget.playlistId).notifier).removeSong(song.id);
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(content: Text('已移除: ${song.title}')),
                );
              },
              child: SongListTile(song: song),
            );
          },
        );
    }
  }

  void _showAddSongDialog(BuildContext context) {
    showDialog(
      context: context,
      builder: (context) => AddSongDialog(playlistId: widget.playlistId),
    );
  }

  void _showEditDialog(BuildContext context, dynamic playlist) {
    final nameController = TextEditingController(text: playlist.name);
    final descController = TextEditingController(text: playlist.description);

    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('编辑歌单'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(
              controller: nameController,
              decoration: const InputDecoration(labelText: '歌单名称'),
            ),
            const SizedBox(height: 16),
            TextField(
              controller: descController,
              decoration: const InputDecoration(labelText: '描述（可选）'),
              maxLines: 2,
            ),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('取消'),
          ),
          FilledButton(
            onPressed: () async {
              if (nameController.text.trim().isNotEmpty) {
                await ref.read(playlistProvider.notifier).updatePlaylist(
                  id: widget.playlistId,
                  name: nameController.text.trim(),
                  description: descController.text.trim().isNotEmpty ? descController.text.trim() : null,
                );
                if (context.mounted) Navigator.pop(context);
              }
            },
            child: const Text('保存'),
          ),
        ],
      ),
    );
  }

  void _confirmDelete(BuildContext context) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('删除歌单'),
        content: const Text('确定要删除这个歌单吗？此操作不可撤销。'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('取消'),
          ),
          FilledButton(
            style: FilledButton.styleFrom(backgroundColor: Colors.red),
            onPressed: () async {
              await ref.read(playlistProvider.notifier).deletePlaylist(widget.playlistId);
              if (context.mounted) {
                Navigator.pop(context);
                Navigator.pop(context);
              }
            },
            child: const Text('删除'),
          ),
        ],
      ),
    );
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /home/ink/Desktop/develop/EchoVault/echo_vault_app && flutter test test/features/playlist/pages/playlist_detail_page_test.dart`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/features/playlist/pages/playlist_detail_page.dart
git add echo_vault_app/test/features/playlist/pages/playlist_detail_page_test.dart
git commit -m "feat(playlist): add PlaylistDetailPage with song list, reorder, and management"
```

---

## Task 5: Integration Tests

**Files:**
- Create: `echo_vault_app/test/features/playlist/integration/playlist_flow_test.dart`

- [ ] **Step 1: Write integration test for complete playlist flow**

```dart
// echo_vault_app/test/features/playlist/integration/playlist_flow_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:echo_vault_app/features/playlist/pages/playlist_list_page.dart';
import 'package:echo_vault_app/features/playlist/providers/playlist_provider.dart';
import 'package:echo_vault_app/models/generated/echo_vault/playlist/v1/playlist_service.pb.dart';

class MockPlaylistRepository implements PlaylistRepository {
  List<Playlist> playlists = [];
  int nextId = 1;

  @override
  Future<Playlist> createPlaylist({required String name, String? description, bool isPublic = false}) async {
    final playlist = Playlist(id: 'p$nextId', name: name, description: description ?? '');
    nextId++;
    playlists.add(playlist);
    return playlist;
  }

  @override
  Future<void> deletePlaylist(String id) async {
    playlists.removeWhere((p) => p.id == id);
  }

  @override
  Future<List<Playlist>> listPlaylists({int pageSize = 20}) async => playlists;

  @override
  Future<Playlist> getPlaylist(String id) async => playlists.firstWhere((p) => p.id == id);

  @override
  Future<Playlist> updatePlaylist({required String id, String? name, String? description, bool? isPublic}) async {
    final index = playlists.indexWhere((p) => p.id == id);
    if (index == -1) throw Exception('Not found');
    final existing = playlists[index];
    final updated = Playlist(
      id: existing.id,
      name: name ?? existing.name,
      description: description ?? existing.description,
    );
    playlists[index] = updated;
    return updated;
  }

  @override
  Future<PlaylistSong> addSongToPlaylist({required String playlistId, required String songId, int position = 0}) async => throw UnimplementedError();

  @override
  Future<void> removeSongFromPlaylist({required String playlistId, required String songId}) async => throw UnimplementedError();

  @override
  Future<void> reorderSongs({required String playlistId, required List<String> songIds}) async => throw UnimplementedError();

  @override
  Future<List<PlaylistSong>> listPlaylistSongs({required String playlistId, int pageSize = 100}) async => [];
}

void main() {
  testWidgets('Complete playlist flow: create and verify', (tester) async {
    final mockRepo = MockPlaylistRepository();

    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          playlistRepositoryProvider.overrideWithValue(mockRepo),
        ],
        child: const MaterialApp(
          home: PlaylistListPage(),
        ),
      ),
    );

    await tester.pumpAndSettle();

    // Empty state
    expect(find.text('暂无歌单，点击 + 创建'), findsOneWidget);

    // Create playlist
    await tester.tap(find.byIcon(Icons.add));
    await tester.pumpAndSettle();

    expect(find.text('创建歌单'), findsOneWidget);

    await tester.enterText(find.byType(TextField).first, 'My New Playlist');
    await tester.tap(find.text('创建'));
    await tester.pumpAndSettle();

    // Verify playlist appears
    expect(find.text('My New Playlist'), findsOneWidget);
    expect(find.text('0 首歌曲'), findsOneWidget);
  });
}
```

- [ ] **Step 2: Run integration test**

Run: `cd /home/ink/Desktop/develop/EchoVault/echo_vault_app && flutter test test/features/playlist/integration/playlist_flow_test.dart`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/test/features/playlist/integration/playlist_flow_test.dart
git commit -m "test(playlist): add integration test for playlist creation flow"
```

---

## Task 6: Update Routing

**Files:**
- Modify: `echo_vault_app/lib/app.dart` or router configuration

- [ ] **Step 1: Check existing router configuration**

Run: `find /home/ink/Desktop/develop/EchoVault/echo_vault_app/lib -name "*.dart" | xargs grep -l "GoRouter\|routes" | head -5`

- [ ] **Step 2: Add playlist routes to router**

Based on the existing router pattern, add:
- `/playlists` route for PlaylistListPage
- `/playlist/:id` route for PlaylistDetailPage

- [ ] **Step 3: Test routing works**

Run manual test or add routing test

- [ ] **Step 4: Commit**

```bash
cd /home/ink/Desktop/develop/EchoVault
git add echo_vault_app/lib/
git commit -m "feat(playlist): add playlist routes to app router"
```

---

## Summary

**Phase 9b Implementation Checklist:**

- [ ] PlaylistTile Widget (Task 1)
- [ ] PlaylistListPage (Task 2)
- [ ] AddSongDialog Widget (Task 3)
- [ ] PlaylistDetailPage (Task 4)
- [ ] Integration Tests (Task 5)
- [ ] Update Routing (Task 6)

**Estimated Time:** 3 days (as per design document)

**Test Coverage:** 10 tests total
- PlaylistTile: 2 tests
- PlaylistListPage: 2 tests
- AddSongDialog: 1 test
- PlaylistDetailPage: 1 test
- Integration: 1 test
- Existing provider tests: 3 tests (already in place)

---

## Execution Options

**Plan complete and saved to `docs/superpowers/plans/2026-07-06-phase9b-playlist-ui.md`. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?**
