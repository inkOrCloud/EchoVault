# Phase 9e：集成与路由实现计划

> **目标：** 将 Phase 9c（设备管理）和 Phase 9d（同步状态）集成到应用导航中，添加路由和导航抽屉，并编写集成测试验证端到端流程。
>
> **当前状态：**
> - 路由已有：`/setup`, `/login`, `/register`, `/library`, `/scan`, `/publish/edit`, `/player`, `/playlists`, `/playlist/:id`
> - 缺失路由：`/devices`, `/sync`
> - 导航：AppDrawer 已在重构中被移除（原在 main.dart 中）
> - 主应用：干净的 `MaterialApp.router` 配置

---

## 文件更改

| 操作 | 文件 |
|:---|:---|
| **修改** | `lib/router.dart` — 添加 `/devices` 和 `/sync` 路由 |
| **创建** | `lib/features/navigation/app_drawer.dart` — 独立导航抽屉组件 |
| **修改** | `lib/main.dart` — 应用内集成导航（可选，如各页面已有 Scaffold 则每页各自集成） |
| **创建** | `test/features/navigation/app_drawer_test.dart` — 导航抽屉测试 |
| **创建** | `test/features/navigation/navigation_flow_test.dart` — 集成测试 |

---

## Task 1: 路由更新

**修改文件：** `lib/router.dart`

添加两个新路由：
- `/devices` → `DeviceListPage`
- `/sync` → `SyncStatusPage`

同时保持现有路由结构不变。路由顺序建议统一放在现有 route 列表末尾。

**注意：** 需要确保 import 路径与现有 style 一致。

路由列表最终状态：
```dart
routes: [
  GoRoute(path: '/setup', ...),
  GoRoute(path: '/login', ...),
  GoRoute(path: '/register', ...),
  GoRoute(path: '/library', ...),
  GoRoute(path: '/scan', ...),
  GoRoute(path: '/publish/edit', ...),
  GoRoute(path: '/player', ...),
  GoRoute(path: '/playlists', ...),
  GoRoute(path: '/playlist/:id', ...),
  GoRoute(path: '/devices', ...),    // 新增
  GoRoute(path: '/sync', ...),       // 新增
],
```

**验证：**
- `dart analyze lib/router.dart` — 无问题
- 无单独测试，集成测试覆盖

---

## Task 2: AppDrawer 导航抽屉

**创建文件：** `lib/features/navigation/app_drawer.dart`

### 设计

遵循 Material 3 + Material You 风格，与原先的 AppDrawer 设计一致但更简洁：

```
┌──────────────────────────────┐
│  🎵 EchoVault — 音匣         │
│     你的私有音乐库            │
├──────────────────────────────┤
│  📚 我的曲库        /library │
│  📋 我的歌单        /playlists│
│  💻 设备管理        /devices  │
│  🔄 同步状态        /sync     │
├──────────────────────────────┤
│  🚪 退出登录                  │
└──────────────────────────────┘
```

### 实现

```dart
class AppDrawer extends ConsumerWidget {
  const AppDrawer({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 使用 GoRouterState 获取当前路径以高亮选中项
    final currentPath = GoRouterState.of(context).matchedLocation;

    // Drawer items: 曲库 / 歌单 / 设备管理 / 同步状态
    // Header: 应用名称 + 图标
    // Footer: 退出登录
  }
}
```

### 导航项

| 图标 | 标题 | 路由 |
|:---|:---|:---:|
| `Icons.library_music` | 我的曲库 | `/library` |
| `Icons.queue_music` | 我的歌单 | `/playlists` |
| `Icons.devices` | 设备管理 | `/devices` |
| `Icons.sync` | 同步状态 | `/sync` |

- 当前激活的路由高亮显示（使用 `primaryContainer` 背景）
- 点击后关闭 Drawer 并导航到目标路由

### 集成方式

在每个需要导航的页面（LibraryPage / PlaylistListPage / SyncStatusPage / DeviceListPage）的 Scaffold 中添加 `drawer:` 参数：

```dart
Scaffold(
  drawer: const AppDrawer(),
  ...
)
```

或统一在路由 builder 中包裹。选择在 **各页面直接添加 drawer** 的方式，保持最小改动。

---

## Task 3：页面集成 — 添加 Drawer

**修改的页面：**

| 页面 | 添加内容 |
|:---|:---|
| `LibraryPage` | `drawer: const AppDrawer()` |
| `PlaylistListPage` | `drawer: const AppDrawer()` |
| `PlaylistDetailPage` | `drawer: const AppDrawer()` |
| `DeviceListPage` | `drawer: const AppDrawer()` |
| `SyncStatusPage` | `drawer: const AppDrawer()` |

注意：auth 页（登录/注册）和 setup 页不需要 drawer。

---

## Task 4：集成测试

### AppDrawer 测试 (`test/features/navigation/app_drawer_test.dart`)

| 测试 | 说明 |
|:---|:---|
| 显示所有导航项 | 4 个菜单项文本都存在 |
| 当前激活项高亮 | 在 `/library` 页面打开 drawer，曲库项高亮 |
| 点击导航项跳转 | 点击"设备管理"后跳转到 `/devices` |
| 显示退出登录按钮 | 底部有退出登录 |
| 点击退出登录触发登出 | 调用 authProvider 的 logout |

### 导航流程集成测试 (`test/features/navigation/navigation_flow_test.dart`)

端到端模拟完整导航流程：

| 步骤 | 操作 |
|:---|:---|
| 1 | 创建带 mock auth + mock 所有 repository 的 ProviderContainer |
| 2 | 从 `/library` 开始 |
| 3 | 打开 Drawer |
| 4 | 导航到"我的歌单" |
| 5 | 导航到"设备管理" |
| 6 | 导航到"同步状态" |
| 7 | 导航回"我的曲库" |
| 验证 | 所有页面正常渲染，路由正确切换 |

---

## 预计工作量

| Task | 文件 | 测试数 | 预估时间 |
|:---:|:---|:---:|:---:|
| Task 1: 路由更新 | 1 (router.dart) | — | 10min |
| Task 2: AppDrawer | 1 (lib) | — | 30min |
| Task 3: 页面集成 | 5 (pages) | — | 20min |
| Task 4: 集成测试 | 2 (test) | 8 | 1h |
| **总计** | **9** | **~8** | **2h** |

---

## 依赖关系

```
router.dart  ──→  DeviceListPage, SyncStatusPage  (import)
AppDrawer    ──→  router.dart (GoRouterState)
Pages        ──→  AppDrawer
Tests        ──→  AppDrawer + all Providers (mock)
```
