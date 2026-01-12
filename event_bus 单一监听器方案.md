## 适用场景说明

**📌 重要前提**：`event_bus` 支持多个监听器是其**正常且常见的使用方式**。在大多数场景下，一个事件被多个组件监听是合理的需求（例如：用户登录事件可能需要同时更新 UI、刷新缓存、记录日志等）。

**本方案解决的特定场景**：
- 需要确保某个事件类型**全局只有一个处理逻辑**（如全局单例服务的状态管理）
- 避免因代码重构或页面重复进入导致的**意外重复订阅**
- 需要**精确控制**某个事件的监听器生命周期

**⚠️ 使用建议**：
- 如果你的业务需要多个组件独立响应同一事件，**请直接使用原生 `event_bus`**
- 只有在明确需要"单一监听器"约束时，才使用本方案

## 问题分析
`event_bus` 包基于 Dart Streams，**没有内置的单一监听器机制**。每次调用 `on<T>().listen()` 都会创建新的订阅。在**不需要多监听器的场景下**，这可能导致事件被意外多次处理。

## 推荐方案：单例服务 + 订阅状态管理
### 多事件类型版本
支持多种事件类型，每种类型只有一个监听器：

```dart
import 'dart:async';
import 'package:event_bus/event_bus.dart';

/// 单例事件总线服务（支持多事件类型）
///
/// ⚠️ 重要注意事项：
/// 1. 事件继承：如果 ChildEvent extends BaseEvent，发送 ChildEvent 时
///    BaseEvent 的监听器也会被触发（这是 event_bus 的默认行为）。
///    建议使用扁平的事件结构，避免事件继承。
/// 2. 生命周期管理：监听器闭包可能捕获 BuildContext/State 等短生命周期对象，
///    必须在对应作用域结束时调用 unlisten()，否则会导致内存泄漏或
///    "setState() called after dispose" 错误。
/// 3. 异步操作：所有生命周期方法（listen/unlisten/dispose）均为异步方法，
///    必须正确 await 以确保 StreamSubscription 完全取消后再进行后续操作。
class SingleListenerEventBus {
  SingleListenerEventBus._internal();
  static final SingleListenerEventBus _instance = SingleListenerEventBus._internal();
  factory SingleListenerEventBus() => _instance;

  final EventBus _eventBus = EventBus();
  final Map<Type, StreamSubscription> _subscriptions = {};
  final Map<Type, Future<void>> _pendingOperations = {};

  /// 注册监听器（每种事件类型只能有一个监听器）
  ///
  /// [handler] 事件处理函数
  /// [replaceExisting] 为 true 时，会替换已存在的监听器；为 false 时忽略重复注册
  ///
  /// 返回 true 表示注册成功，false 表示已存在监听器且未替换
  ///
  /// ⚠️ 注意：此方法为异步方法，必须 await 以确保旧订阅完全取消后再注册新监听器
  /// 内部使用串行化机制防止并发调用导致的竞态条件
  Future<bool> listen<T>(
    void Function(T event) handler, {
    bool replaceExisting = false,
  }) async {
    final type = T;

    // 同步捕获当前待处理的操作，然后创建新的 completer
    final previousOperation = _pendingOperations[type];
    final completer = Completer<void>();
    _pendingOperations[type] = completer.future;

    try {
      // 等待前一个操作完成（如果有）
      await previousOperation;

      final existing = _subscriptions[type];

      // 检查是否已存在监听器
      if (existing != null) {
        if (!replaceExisting) {
          return false; // 已存在监听器，不重复注册
        }
        // 替换模式：先完全取消旧订阅
        await existing.cancel();
        _subscriptions.remove(type);
      }

      // 注册新监听器
      _subscriptions[type] = _eventBus.on<T>().listen(handler);
      return true;
    } finally {
      completer.complete();
      // 只有当前操作是最后一个时才清理
      if (_pendingOperations[type] == completer.future) {
        _pendingOperations.remove(type);
      }
    }
  }

  /// 检查某事件类型是否已有监听器
  bool hasListener<T>() => _subscriptions.containsKey(T);

  /// 取消某事件类型的监听
  ///
  /// ⚠️ 注意：此方法为异步方法，必须 await 以确保订阅完全取消
  Future<void> unlisten<T>() async {
    final type = T;
    final existing = _subscriptions.remove(type);
    await existing?.cancel();
  }

  /// 发送事件
  void fire<T>(T event) {
    _eventBus.fire(event);
  }

  /// 取消所有监听并清理资源
  ///
  /// ⚠️ 注意：此方法为异步方法，使用 Future.wait 并发取消所有订阅以提高性能
  Future<void> dispose() async {
    await Future.wait(_subscriptions.values.map((sub) => sub.cancel()));
    _subscriptions.clear();
  }

  /// 用于单元测试：重置所有状态
  Future<void> reset() async {
    await dispose();
  }
}
```

## 事件类型定义示例
```dart
/// 用户登录事件
class UserLoginEvent {
  final String userId;
  final String username;
  UserLoginEvent(this.userId, this.username);
}

/// 用户登出事件
class UserLogoutEvent {
  final String userId;
  UserLogoutEvent(this.userId);
}

/// 网络状态变化事件
class NetworkStatusEvent {
  final bool isConnected;
  NetworkStatusEvent(this.isConnected);
}

/// 消息通知事件
class MessageEvent {
  final String title;
  final String content;
  MessageEvent(this.title, this.content);
}
```

## 使用方式
### 1. 在 main.dart 中注册全局监听器（仅限应用级事件）
```dart
Future<void> main() async {
  // 在 async main() 中 await 任何内容前，必须先初始化 Flutter 绑定
  // 避免 "ServicesBinding.defaultBinaryMessenger was accessed before the binding was initialized" 错误
  WidgetsFlutterBinding.ensureInitialized();

  final eventBus = SingleListenerEventBus();

  // 注册用户登录事件监听器（检查返回值以确保注册成功）
  final loginRegistered = await eventBus.listen<UserLoginEvent>((event) {
    print('用户登录: ${event.username}');
    // 处理登录逻辑
  });
  assert(loginRegistered, 'UserLoginEvent 监听器注册失败，可能已存在');

  // 注册用户登出事件监听器
  final logoutRegistered = await eventBus.listen<UserLogoutEvent>((event) {
    print('用户登出: ${event.userId}');
    // 处理登出逻辑
  });
  assert(logoutRegistered, 'UserLogoutEvent 监听器注册失败，可能已存在');

  // 注册网络状态事件监听器
  final networkRegistered = await eventBus.listen<NetworkStatusEvent>((event) {
    print('网络状态: ${event.isConnected ? "已连接" : "已断开"}');
    // 处理网络状态变化
  });
  assert(networkRegistered, 'NetworkStatusEvent 监听器注册失败，可能已存在');

  // 注册消息事件监听器
  final messageRegistered = await eventBus.listen<MessageEvent>((event) {
    print('收到消息: ${event.title}');
    // 显示通知等
  });
  assert(messageRegistered, 'MessageEvent 监听器注册失败，可能已存在');

  runApp(MyApp());
}
```

### 2. 在任意位置发送事件
```dart
// 登录页面
SingleListenerEventBus().fire(UserLoginEvent('123', 'zhangsan'));

// 设置页面
SingleListenerEventBus().fire(UserLogoutEvent('123'));

// 网络监听服务
SingleListenerEventBus().fire(NetworkStatusEvent(true));

// 推送服务
SingleListenerEventBus().fire(MessageEvent('新消息', '你有一条新的通知'));
```

### 3. 重复注册会被忽略
```dart
final eventBus = SingleListenerEventBus();

// 第一次注册 - 成功
await eventBus.listen<UserLoginEvent>((event) {
  print('处理器 A');
});

// 第二次注册 - 返回 false（已存在监听器）
final success = await eventBus.listen<UserLoginEvent>((event) {
  print('处理器 B');
});

if (!success) {
  print('警告: UserLoginEvent 已注册，未创建新的处理器');
}

// 发送事件 - 只会输出 "处理器 A"
eventBus.fire(UserLoginEvent('123', 'test'));
```

### 4. 替换已有监听器
```dart
// 强制替换已有监听器
await eventBus.listen<UserLoginEvent>(
  (event) {
    print('新的处理器');
  },
  replaceExisting: true,
);
```

### 5. 取消特定事件的监听
```dart
// 取消用户登录事件的监听
await SingleListenerEventBus().unlisten<UserLoginEvent>();

// 检查是否还有监听器
if (!SingleListenerEventBus().hasListener<UserLoginEvent>()) {
  print('UserLoginEvent 已无监听器');
}
```

## ⚠️ 生命周期管理与内存泄漏警告

### 两种使用场景

#### 1. 全局事件（应用级别）
适用于整个应用生命周期都需要监听的事件，如：
- 应用配置变更
- 远程推送通知
- 全局网络状态变化

**注册位置**：在 `main()` 函数中注册

**⚠️ 关键警告**：全局监听器的闭包**绝不能**捕获短生命周期对象，如：
- `BuildContext`
- `State` 对象
- `mounted` 属性
- 任何 Widget 相关的引用

否则会导致：
- 内存泄漏（Widget 销毁后仍被引用）
- "setState() called after dispose" 错误
- 应用崩溃或异常行为

#### 2. 局部事件（页面/模块级别）
适用于特定页面或模块的事件，如：
- 页面内的消息通知
- 模块内的状态同步
- 临时的业务逻辑事件

**注册位置**：在 `StatefulWidget` 的 `initState()` 中注册

**释放位置**：在 `State` 的 `dispose()` 中调用 `unlisten()`

**⚠️ 重要限制**：由于本方案基于"每种事件类型只能有一个监听器"的设计，如果多个页面需要监听同一事件类型（如都监听 `MessageEvent`），必须采用以下策略之一：
1. **为每个页面定义独立的事件类型**（推荐）：如 `ProfileMessageEvent`、`HomeMessageEvent`
2. **在注册前先 unlisten**：确保旧监听器已释放
3. **使用 `replaceExisting: true`**：强制替换已有监听器（但会影响其他页面）

否则第二个页面的 `listen()` 会返回 `false`，监听器注册失败。

### 局部事件完整示例

```dart
import 'dart:async';
import 'package:flutter/material.dart';

// 为页面定义专属的事件类型，避免与全局事件冲突
class ProfilePageMessageEvent {
  final String title;
  final String content;
  ProfilePageMessageEvent(this.title, this.content);
}

class ProfilePage extends StatefulWidget {
  @override
  State<ProfilePage> createState() => _ProfilePageState();
}

class _ProfilePageState extends State<ProfilePage> {
  @override
  void initState() {
    super.initState();

    // 在 initState 中注册局部事件监听器
    // 使用 unawaited() 明确表示我们不等待注册完成
    // 添加 catchError 处理可能的异常，避免未捕获错误
    unawaited(_registerEventListeners().catchError((error, stackTrace) {
      debugPrint('注册事件监听器失败: $error');
    }));
  }

  Future<void> _registerEventListeners() async {
    final eventBus = SingleListenerEventBus();

    // 注册页面专属的消息事件监听器
    final success = await eventBus.listen<ProfilePageMessageEvent>((event) {
      // 使用 mounted 检查避免在 Widget 销毁后调用 setState
      if (!mounted) return;

      // 安全地使用 context 显示 SnackBar
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(event.title)),
      );
    });

    if (!success) {
      debugPrint('警告: ProfilePageMessageEvent 监听器已存在');
    }
  }

  @override
  void dispose() {
    // 在 dispose 中取消监听，防止内存泄漏
    // 使用 unawaited() 因为 dispose() 是同步方法
    // 添加 catchError 处理可能的异常
    unawaited(SingleListenerEventBus().unlisten<ProfilePageMessageEvent>().catchError((error) {
      debugPrint('取消监听失败: $error');
    }));
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('个人中心')),
      body: Center(child: Text('Profile Page')),
    );
  }
}
```

### 关于 unawaited() 的说明

当在同步方法（如 `initState()` 或 `dispose()`）中调用异步方法时，无法使用 `await`。此时应使用 `unawaited()` 函数明确表示我们有意不等待 Future 完成，以避免 `unawaited_futures` lint 警告。

```dart
import 'dart:async'; // 导入以使用 unawaited()

// 在同步方法中调用异步方法
@override
void dispose() {
  unawaited(SingleListenerEventBus().unlisten<MessageEvent>());
  super.dispose();
}
```

## 核心原理
1. **单例模式**：`SingleListenerEventBus()` 始终返回同一个实例
2. **类型映射**：`Map<Type, StreamSubscription>` 以事件类型为 key 存储订阅
3. **状态检查**：`_subscriptions.containsKey(type)` 确保每种类型只有一个监听器
4. **异步取消**：`replaceExisting` 参数支持替换已有监听器，通过 `await cancel()` 确保旧订阅完全释放后再注册新监听器，避免竞态条件

## API 速查
| 方法 | 返回值 | 说明 |
| --- | --- | --- |
| `listen<T>(handler)` | `Future<bool>` | 【异步】注册监听器（已存在则忽略），返回是否成功 |
| `listen<T>(handler, replaceExisting: true)` | `Future<bool>` | 【异步】注册监听器（已存在则替换，替换前 await 旧订阅取消） |
| `hasListener<T>()` | `bool` | 【同步】检查是否已有监听器 |
| `unlisten<T>()` | `Future<void>` | 【异步】取消某类型的监听 |
| `fire<T>(event)` | `void` | 【同步】发送事件 |
| `dispose()` | `Future<void>` | 【异步】取消所有监听（使用 Future.wait 并发取消） |
| `reset()` | `Future<void>` | 【异步】重置（用于测试） |


## 注意事项
1. **⚠️ 内存泄漏警告**：任何引用 `BuildContext`/`State` 的监听器都必须在对应作用域结束时调用 `unlisten()`，否则会导致内存泄漏和 "setState() called after dispose" 错误。全局监听器绝不能捕获短生命周期对象。
2. **单元测试**：在 `setUp` 或 `tearDown` 中 `await SingleListenerEventBus().reset()` 清理状态，防止用例之间互相影响
3. **依赖注入**：如果使用 get_it 等框架，可以将 `SingleListenerEventBus` 注册为单例，统一在注入容器中管理生命周期
4. **事件继承**：避免使用事件继承（如 `ChildEvent extends BaseEvent`），否则发送子事件时父事件监听器也会触发
5. **注册时机**：
   - **全局事件**：在 `main()` 中注册，确保监听器只依赖无状态服务
   - **局部事件**：在组件 `initState()` 中注册，在 `dispose()` 中释放
   - 无法 `await` 时请使用 `unawaited()` 明确说明意图

