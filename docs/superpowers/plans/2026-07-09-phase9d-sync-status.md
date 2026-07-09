# Phase 9d：同步状态 UI 实现计划

> **目标：** 为 Flutter 客户端实现同步状态界面，包括同步状态指示器、同步状态详情页、以及基础的同步仓储抽象。
>
> **架构：** 遵循 Phase 9a/9c 建立的模式：`Service 抽象 → ChannelService → Repository → Provider (StateNotifier) → Widgets/Pages`。同步模块与歌单/设备不同之处在于 SyncService 包含流式 RPC（PullChanges/SubscribeChanges），为简化起见 Phase 9d 聚焦于 Unary RPC（PushChanges/AckChanges）和 UI 状态管理。
>
> **技术栈：** Flutter, Dart, Riverpod, gRPC, protobuf, go_router, shared_preferences

---

## 文件结构

```
echo_vault_app/lib/
├── features/
│   └── sync/                                # 新建
│       ├── services/
│       │   └── sync_repository.dart         # gRPC 同步仓储
│       ├── providers/
│       │   └── sync_provider.dart           # 同步状态管理 + lastSyncTime 持久化
│       ├── pages/
│       │   └── sync_status_page.dart        # 同步状态页
│       └── widgets/
│           └── sync_status_indicator.dart   # 紧凑型同步状态指示器

echo_vault_app/test/
└── features/
    └── sync/                                # 新建
        ├── services/
        │   └── sync_repository_test.dart
        ├── providers/
        │   └── sync_provider_test.dart
        ├── pages/
        │   └── sync_status_page_test.dart
        └── widgets/
            └── sync_status_indicator_test.dart
```

---

## Task 1：SyncRepository（gRPC 同步仓储）

### SyncService 抽象接口

```dart
abstract class SyncService {
  Future<PushChangesResponse> pushChanges(PushChangesRequest request);
  Future<AckChangesResponse> ackChanges(AckChangesRequest request);
}
```

流式 RPC（PullChanges/SubscribeChanges）暂不暴露，留给后续 Phase。

### ChannelSyncService

包装 `SyncServiceClient`，生产实现。

### SyncRepository

| 方法 | 说明 |
|:---|:---|
| `pushChanges({deviceId, lastPullVersion, changes})` | 提交本地变更 |
| `ackChanges({deviceId, ackedVersion})` | 确认已消费的变更 |

直接使用生成的 proto 类型（`PushChangesRequest`/`PushChangesResponse`/`AckChangesRequest`/`AckChangesResponse`/`SyncChange`），不做额外封装。

### 测试用例（6 项）

- `pushChanges returns response with server_version and accepted_count`
- `pushChanges throws on gRPC error` → `SyncRepositoryException`
- `ackChanges succeeds`
- `ackChanges throws on gRPC error` → `SyncRepositoryException`
- `pushChanges with changes list returns accepted count`
- `pushChanges with conflict info returns conflicts`

---

## Task 2：SyncProvider（同步状态管理）

### SyncStatus enum

```dart
enum SyncStatus { idle, syncing, completed, error }
```

### SyncState

```dart
class SyncState {
  final SyncStatus status;
  final int pendingChanges;
  final int syncedChanges;
  final String? error;
  final DateTime? lastSyncTime;
}
```

### SyncNotifier 方法

| 方法 | 行为 |
|:---|:---|
| `loadLastSyncTime()` | 从 SharedPreferences 读取上次同步时间（初始化） |
| `startSync()` | 设置为 syncing 状态，调用 `pushChanges`，成功后记录时间 |
| `setPendingChanges(int count)` | 仅更新待同步数 |
| `updateLastSyncTime()` | 更新同步时间并持久化到 SharedPreferences |
| `reset()` | 恢复到 idle 状态 |

**lastSyncTime 持久化：** 使用 `SharedPreferences` 存储键 `sync_last_time`（毫秒时间戳），`loadLastSyncTime()` 在 provider 初始化时读取。

### 可覆盖的 Provider

```dart
final syncRepositoryProvider = Provider<SyncRepository>((ref) {
  throw UnimplementedError('SyncRepository must be overridden');
});

final syncProvider = StateNotifierProvider<SyncNotifier, SyncState>((ref) {
  return SyncNotifier(
    repo: ref.read(syncRepositoryProvider),
    prefs: ref.read(sharedPrefsProvider),
  );
});
```

需要新增 `sharedPrefsProvider`（如果有现成就复用，否则新增）。

### 测试用例（6 项）

- `initial state is idle with zero changes`
- `startSync sets status to syncing then completed`
- `startSync sets error status on failure`
- `setPendingChanges updates pending count`
- `updateLastSyncTime updates lastSyncTime`
- `reset restores to idle`

---

## Task 3：SyncStatusIndicator Widget

紧凑的标签式组件，显示同步状态的小圆点/图标 + 文字。

| 状态 | 图标 | 文字 | 颜色 |
|:---|:---|:---|:---|
| idle | `Icons.sync` | "等待同步" | surfaceVariant |
| syncing | `CircularProgressIndicator` (12x12) | "同步中 (x/y)" | primaryContainer |
| completed | `Icons.check_circle` | "已同步" 或 "上次同步 HH:mm" | secondaryContainer |
| error | `Icons.error` | "同步失败" | errorContainer |

使用 `ConsumerWidget`，通过 `ref.watch(syncProvider)` 获取状态。

### 测试用例（6 项）

- `shows idle state with 等待同步 text`
- `shows syncing state with progress indicator`
- `shows completed state with check icon`
- `shows completed state with last sync time`
- `shows error state with 同步失败 text`
- `reacts to state changes`

---

## Task 4：SyncStatusPage

全屏页面展示同步详情。

### UI 设计

- **AppBar：** "同步状态" + 右上角同步按钮
- **Card 1：同步概览**
  - 行1：标题 "同步状态" + SyncStatusIndicator
  - 行2：待同步变更数
  - 行3：已同步变更数
  - 行4：上次同步时间（如有）
  - 行5：错误信息（如有）+ 重试按钮
- **Card 2：同步说明**
  - 离线修改自动缓存
  - 联网后自动同步
  - 冲突时服务端优先
  - 歌曲文件需手动上传
- **底部：** "立即同步" 按钮（仅 idle/completed/error 状态时可点击）

### 测试用例（6 项）

- `shows sync overview card with indicator`
- `shows pending and synced change counts`
- `displays last sync time when available`
- `displays error message when present`
- `sync button triggers startSync`
- `shows sync info card`

---

## 预计工作量

| Task | 文件数 | 测试数 | 预估时间 |
|:---:|:---:|:---:|:---:|
| Task 1: SyncRepository | 2 (lib + test) | 6 | 1h |
| Task 2: SyncProvider | 2 (lib + test) | 6 | 1h |
| Task 3: SyncStatusIndicator | 2 (lib + test) | 6 | 0.5h |
| Task 4: SyncStatusPage | 2 (lib + test) | 6 | 1h |
| **总计** | **8** | **24** | **3.5h** |

---

## 参考模式

### SyncRepository（仿 DeviceRepository）

```
SyncService (abstract) ← ChannelSyncService (wraps SyncServiceClient)
       ↓
SyncRepository (withService / with GrpcClientManager)
       ↓
SyncNotifier (StateNotifier) ← syncProvider (StateNotifierProvider)
       ↓
SyncStatusIndicator / SyncStatusPage (consume via ref.watch)
```

### SyncState 生命周期

```
idle → syncing → completed → idle
                    ↓
                 error → idle (重试)
```
