# Phase 9c：设备管理实现计划

> **目标：** 为 Flutter 客户端实现多设备管理功能，包括设备列表、移除设备、更新设备名称。
>
> **架构：** 遵循 Phase 9a 建立的重构模式。设备相关的 gRPC 方法（ListDevices/RemoveDevice/UpdateDevice）归属于 `UserService`，因此创建专属的 `DeviceService` 抽象层来封装这些调用。使用 Riverpod 进行状态管理，遵循 `Service → Repository → Provider → Widget` 的分层模式。
>
> **技术栈：** Flutter, Dart, Riverpod, gRPC, protobuf, go_router

---

## 文件结构

```
echo_vault_app/lib/
├── features/
│   └── device/                          # 新建
│       ├── services/
│       │   └── device_repository.dart   # gRPC 设备仓储
│       ├── providers/
│       │   └── device_provider.dart     # 设备状态管理
│       ├── pages/
│       │   └── device_list_page.dart    # 设备列表页
│       └── widgets/
│           └── device_tile.dart         # 设备项组件

echo_vault_app/test/
└── features/
    └── device/                          # 新建
        ├── services/
        │   └── device_repository_test.dart
        ├── providers/
        │   └── device_provider_test.dart
        ├── pages/
        │   └── device_list_page_test.dart
        └── widgets/
            └── device_tile_test.dart
```

---

## Task 1：设备数据模型 + gRPC 抽象层

### DeviceService 抽象接口

遵循 `SongService` / `PlaylistService` 的既有模式。因为设备 RPC 实际在 `UserServiceClient` 上，我们需要一个抽象来隔离：

```dart
/// DeviceService 抽象 — 封装 UserService 中的设备相关 RPC
abstract class DeviceService {
  Future<ListDevicesResponse> listDevices(ListDevicesRequest request);
  Future<RemoveDeviceResponse> removeDevice(RemoveDeviceRequest request);
  Future<UpdateDeviceResponse> updateDevice(UpdateDeviceRequest request);
}
```

```
 创建文件：
   lib/features/device/services/device_repository.dart
```

**包含：**
- `DeviceService` 抽象接口 — 3 个方法（listDevices / removeDevice / updateDevice）
- `ChannelDeviceService` — 包装 `UserServiceClient` 的生产实现
- `DeviceRepositoryException` — 异常类
- `DeviceRepository` — 对外暴露的业务接口
  - 构造函数：`DeviceRepository({required GrpcClientManager manager})` / `DeviceRepository.withService(this._svc)`
  - 方法：
    - `Future<List<Device>> listDevices()`
    - `Future<void> removeDevice(String deviceId)`
    - `Future<Device> updateDevice({required String deviceId, required String deviceName})`

**注意：** 设备的 proto type 是 `UserServiceClient` 的生成类型 `Device`（有 deviceId/deviceName/platform/osVersion/clientVersion/lastSyncAt/registeredAt）。直接使用生成的 `Device` proto 对象，不做额外封装。

**测试文件：**
```
 创建文件：
   test/features/device/services/device_repository_test.dart
```

测试用例：
- `listDevices` — 正常返回设备列表
- `removeDevice` — 成功移除
- `updateDevice` — 成功更新名称
- `listDevices throws on error` — GrpcError 转为 DeviceRepositoryException
- `removeDevice throws on error` — 同上
- `updateDevice throws on error` — 同上

---

## Task 2：设备状态管理 Provider

```
 创建文件：
   lib/features/device/providers/device_provider.dart
```

**设计：**

```dart
enum DeviceStatus { loading, loaded, error }

class DeviceState {
  final DeviceStatus status;
  final List<Device> devices;
  final String? error;
}
```

**提供者：**

```dart
// 可被覆盖的 provider（用于测试注入 mock）
final deviceRepositoryProvider = Provider<DeviceRepository>((ref) {
  throw UnimplementedError('DeviceRepository must be overridden');
});

final deviceProvider = StateNotifierProvider<DeviceNotifier, DeviceState>((ref) {
  return DeviceNotifier(ref.read(deviceRepositoryProvider));
});
```

**DeviceNotifier 方法：**
- `loadDevices()` — 加载设备列表，设置 loading/loaded/error 状态
- `removeDevice(String deviceId)` — 移除设备，成功后从 state 中删除
- `updateDevice(String deviceId, String deviceName)` — 更新设备名称，成功后更新 state 中的对应设备

**测试文件：**
```
 创建文件：
   test/features/device/providers/device_provider_test.dart
```

测试用例：
- `loadDevices` — 加载设备列表，状态变为 loaded，devices 有值
- `loadDevices error` — 出错时状态变为 error
- `removeDevice` — 移除后列表减少
- `updateDevice` — 更新后 name 变更
- 状态转换验证（initial → loading → loaded/error）

---

## Task 3：DeviceTile Widget

```
 创建文件：
   lib/features/device/widgets/device_tile.dart
```

**Props：**
- `Device device` — 设备 proto 对象
- `VoidCallback? onRename` — 重命名回调
- `VoidCallback? onRemove` — 移除回调

**UI 设计：**
- 左侧：平台图标（根据 platform 返回对应的 IconData）
  - `windows` → Icons.desktop_windows
  - `macos` → Icons.desktop_mac
  - `linux` → Icons.computer
  - `android` → Icons.phone_android
  - `ios` → Icons.phone_iphone
  - `web` → Icons.language
  - 默认 → Icons.devices
- 标题：`deviceName`
- 副标题：最后同步时间（格式化显示，如"最后同步：2026-07-09 14:30"），若无同步则显示"尚未同步"
- 右侧：更多操作按钮（PopupMenuButton 或 IconButton）
  - "重命名" → 触发 onRename
  - "移除设备" → 触发 onRemove
- 使用 ListTile 构建，适配 Material You 主题

**测试文件：**
```
 创建文件：
   test/features/device/widgets/device_tile_test.dart
```

测试用例：
- 渲染设备名称和平台图标
- 显示最后同步时间
- 显示"尚未同步"当 lastSyncAt 为空
- onRename 回调触发
- onRemove 回调触发

---

## Task 4：DeviceListPage

```
 创建文件：
   lib/features/device/pages/device_list_page.dart
```

**UI 设计：**
- AppBar: "设备管理"
- 主体：
  - 状态为 loading：显示 CircularProgressIndicator
  - 状态为 error：显示错误提示 + 重试按钮
  - 状态为 loaded + 空列表：显示空状态提示（"暂无已注册设备"）
  - 状态为 loaded + 有数据：ListView 展示 DeviceTile 列表
- 每个 DeviceTile 的操作：
  - 重命名：弹出对话框（TextField + 确认/取消）
  - 移除：弹出确认对话框（"确定要移除设备 xxx 吗？"）
- 右上角：刷新按钮
- 在 `initState` 或首次 build 时自动加载设备列表

**测试文件：**
```
 创建文件：
   test/features/device/pages/device_list_page_test.dart
```

测试用例：
- 显示加载指示器
- 显示设备列表
- 显示空状态提示
- 显示错误状态 + 重试按钮
- 重命名对话框交互
- 移除确认对话框交互

---

## Task 5：集成与验证

**步骤：**

1. 在 `lib/features/library/services/grpc_providers.dart` 中添加 `deviceRepositoryProvider` 的真实注入

```dart
final deviceRepositoryProvider = Provider<DeviceRepository>((ref) {
  return DeviceRepository(manager: ref.watch(grpcClientProvider));
});
```

注意：需要和 Task 2 中的 `deviceRepositoryProvider` 声明保持一致。建议将 provider 声明统一放在 `device_provider.dart` 中（Playlist 模式），或放在 `grpc_providers.dart` 中（Song 模式）。**选择 Playlist 模式**：provider 声明在 `device_provider.dart` 中，测试通过 override 注入 mock。

2. 运行静态分析：
```bash
cd echo_vault_app && dart analyze lib/features/device/
```

3. 运行测试：
```bash
cd echo_vault_app && flutter test test/features/device/
```

4. 提交代码：
```bash
git add lib/features/device/ test/features/device/
git commit -m "feat(device): Phase 9c 设备管理 — 仓储/Provider/UI/测试"
```

---

## 预计工作量

| Task | 文件数 | 测试数 | 预估时间 |
|:---:|:---:|:---:|:---:|
| Task 1: DeviceService + Repository | 2 (lib + test) | 6 | 1h |
| Task 2: DeviceProvider | 2 (lib + test) | 5 | 0.5h |
| Task 3: DeviceTile Widget | 2 (lib + test) | 5 | 0.5h |
| Task 4: DeviceListPage | 2 (lib + test) | 6 | 1h |
| Task 5: 集成验证 | — | — | 0.5h |
| **总计** | **8** | **22** | **3.5h** |

---

## 参考模式

### Phase 9a PlaylistRepository 模式对照

```
PlaylistService (abstract) ← ChannelPlaylistService (wraps PlaylistServiceClient)
       ↓
PlaylistRepository (withService / with GrpcClientManager)
       ↓
PlaylistNotifier (StateNotifier) ← playlistProvider (StateNotifierProvider)
       ↓
Pages / Widgets (consume via ref.watch)
```

### Phase 9c DeviceRepository 模式

```
DeviceService (abstract) ← ChannelDeviceService (wraps UserServiceClient)
       ↓
DeviceRepository (withService / with GrpcClientManager)
       ↓
DeviceNotifier (StateNotifier) ← deviceProvider (StateNotifierProvider)
       ↓
DeviceListPage (consume via ref.watch)
```
