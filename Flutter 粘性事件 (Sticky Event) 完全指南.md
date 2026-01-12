> 本文档面向中级 Flutter 开发者，详细介绍粘性事件的概念、5 种实现方案及选型建议。
>

---

## 一、概述
### 1.1 什么是粘性事件？
粘性事件（Sticky Event）是一种特殊的事件机制，其核心特性是**事件具有"记忆"能力**：

+ **先发送后监听**：当监听器注册时，能立即收到最后一次发送的事件
+ **先监听后发送**：正常接收后续发送的事件

这与普通事件的区别在于：普通事件如果在监听器注册前发送，监听器将永远无法收到该事件。

### 1.2 为什么需要粘性事件？
在 Flutter 开发中，经常遇到以下场景：

1. **页面间数据传递**：A 页面发送事件后跳转到 B 页面，B 页面需要接收该事件
2. **异步初始化**：App 启动时加载配置，后续页面需要获取配置结果
3. **状态同步**：新打开的页面需要立即获取当前登录状态、主题设置等

如果使用普通事件，由于监听器注册时机晚于事件发送，会导致事件丢失。

### 1.3 典型应用场景
| 场景 | 说明 |
| --- | --- |
| 登录状态广播 | 登录成功后，所有页面需要感知登录状态 |
| 全局配置分发 | App 配置加载完成后，各模块获取配置 |
| 跨页面通信 | 支付结果页返回后，订单页刷新状态 |
| 主题/语言切换 | 切换后所有页面立即响应 |


---

## 二、方案详解
### 2.1 方案一：RxDart BehaviorSubject（推荐）
#### 原理
`BehaviorSubject` 是 RxDart 提供的特殊 Subject，它会：

+ 缓存最新发送的值
+ 新订阅者订阅时立即收到缓存的最新值
+ 后续值正常推送给所有订阅者

```plain
时间线：
  发送事件 A ──────────────────────────────►
                    │
                    ▼ 订阅者1 注册（立即收到 A）
                    │
  发送事件 B ───────┼──────────────────────►
                    │         │
                    ▼         ▼ 订阅者2 注册（立即收到 B）
              收到 B      收到 B
```

#### 依赖安装
```yaml
# pubspec.yaml
dependencies:
  rxdart: ^0.27.7
```

#### 代码实现
```dart
import 'dart:async';
import 'package:rxdart/rxdart.dart';

/// 基于 BehaviorSubject 的粘性事件管理器
///
/// 注意：此实现显式区分"未发布过值"和"值为 null"两种状态，
/// 避免使用 null 作为"无事件"的替代造成语义混淆。
class RxStickyEvent<T> {
  final BehaviorSubject<T> _subject;

  /// 创建带初始值的粘性事件（推荐）
  RxStickyEvent.seeded(T initialValue)
      : _subject = BehaviorSubject<T>.seeded(initialValue);

  /// 创建无初始值的粘性事件
  ///
  /// 警告：新订阅者在首次 emit 前不会收到任何值。
  /// 如果 T 可为 null，请使用 [RxStickyEvent.seeded] 并传入明确的初始值。
  RxStickyEvent() : _subject = BehaviorSubject<T>();

  /// 发送事件
  void emit(T event) {
    _subject.add(event);
  }

  /// 监听事件，新监听者立即收到最新值（如果有）
  ///
  /// 重要：返回的 StreamSubscription 必须在不再需要时调用 cancel()，
  /// 否则会导致内存泄漏。
  StreamSubscription<T> listen(
    void Function(T event) onData, {
    Function? onError,
    void Function()? onDone,
    bool? cancelOnError,
  }) {
    return _subject.listen(
      onData,
      onError: onError,
      onDone: onDone,
      cancelOnError: cancelOnError,
    );
  }

  /// 获取当前值（可能为 null，仅当 T 可为 null 或未发布过值时）
  T? get valueOrNull => _subject.valueOrNull;

  /// 获取当前值（无值时抛 ValueStreamError）
  T get value => _subject.value;

  /// 是否已发布过值
  bool get hasValue => _subject.hasValue;

  /// 获取只读 Stream 用于 StreamBuilder
  ///
  /// 注意：不要暴露 _subject 本身，避免外部误用 sink 写入。
  Stream<T> get stream => _subject.stream;

  /// 释放资源
  ///
  /// 调用后此实例不可再使用。所有订阅者会收到 onDone 回调。
  void dispose() {
    _subject.close();
  }
}
```

#### 使用示例
```dart
// 1. 创建粘性事件
final loginEvent = RxStickyEvent<UserInfo>();

// 2. 先发送事件
loginEvent.emit(UserInfo(name: 'John', id: 1));

// 3. 后注册监听（立即收到 UserInfo）
final subscription = loginEvent.listen((user) {
  print('收到用户信息: ${user.name}');
});

// 4. 在 Widget 中使用 StreamBuilder
StreamBuilder<UserInfo>(
  stream: loginEvent.stream,
  builder: (context, snapshot) {
    if (snapshot.hasData) {
      return Text('欢迎, ${snapshot.data!.name}');
    }
    return Text('未登录');
  },
)

// 5. 页面销毁时取消订阅
@override
void dispose() {
  subscription.cancel();
  super.dispose();
}
```

#### 优点
+ **功能强大**：支持 map、where、debounce、combineLatest 等丰富操作符
+ **类型安全**：泛型支持，编译期类型检查
+ **Flutter 集成好**：与 StreamBuilder 无缝配合
+ **多订阅者支持**：广播模式，多个监听者同时接收

#### 缺点
+ **需要外部依赖**：必须引入 rxdart 包
+ **学习成本**：需要理解响应式编程概念
+ **资源管理**：需要手动 dispose 和取消订阅

#### 适用场景
+ 已在项目中使用 RxDart
+ 需要复杂的流操作（合并、过滤、防抖等）
+ UI 状态管理（登录状态、购物车、主题等）
+ 需要与 StreamBuilder 配合使用

#### 补充：RxDart 扩展操作符
除了直接使用 `BehaviorSubject`，RxDart 还提供了便捷的扩展操作符：

```dart
// 方式一：shareValue - 将普通 Stream 转为粘性流
final stickyStream = myStream.shareValue();

// 方式二：shareReplay - 缓存最近 N 个值
final replayStream = myStream.shareReplay(maxSize: 1);

// 方式三：startWith - 为流添加初始值
final streamWithDefault = myStream.startWith(defaultValue);
```

这些操作符可以避免手写粘性回放逻辑，推荐在已有 Stream 基础上使用。

---

### 2.2 方案二：自定义 Sticky EventBus
#### 原理
通过 `Map<Type, dynamic>` 缓存每种事件类型的最新值，结合 `StreamController.broadcast()` 实现多订阅者广播。新监听者注册时，先检查缓存并立即回调。

> **⚠️**** 注意**：此方案适合简单场景。对于复杂需求，推荐使用 RxDart 的 `BehaviorSubject` 或 `shareReplay(maxSize: 1)`，它们提供更可靠的时序保证。
>

#### 代码实现
```dart
import 'dart:async';

/// 自定义粘性事件总线（单例模式）
///
/// 设计说明：
/// - 使用 Map<Type, dynamic> 按事件类型缓存最新值
/// - 粘性回放采用同步方式，确保时序可预测
/// - 全局单例，生命周期与应用一致
class StickyEventBus {
  StickyEventBus._();
  static final StickyEventBus _instance = StickyEventBus._();
  static StickyEventBus get instance => _instance;

  final _controller = StreamController.broadcast();
  final Map<Type, dynamic> _stickyEvents = {};

  /// 发送粘性事件（缓存最新值）
  void emitSticky<T>(T event) {
    _stickyEvents[T] = event;
    _controller.add(event);
  }

  /// 发送普通事件（不缓存）
  void emit<T>(T event) {
    _controller.add(event);
  }

  /// 监听指定类型事件
  ///
  /// 粘性回放策略：同步立即回调缓存值，确保时序可预测。
  /// 如果需要异步回放，请使用 [onAsync] 方法。
  StreamSubscription<T> on<T>(void Function(T event) callback) {
    // 先订阅流，确保不会错过订阅期间发送的事件
    final subscription = _controller.stream
        .where((event) => event is T)
        .cast<T>()
        .listen(callback);

    // 同步回放缓存的粘性事件（时序可预测）
    if (_stickyEvents.containsKey(T)) {
      callback(_stickyEvents[T] as T);
    }

    return subscription;
  }

  /// 异步监听指定类型事件（粘性回放延迟到下一微任务）
  ///
  /// 适用于需要在当前同步代码执行完毕后再处理粘性事件的场景。
  /// 注意：可能导致事件顺序与发送顺序不一致。
  StreamSubscription<T> onAsync<T>(void Function(T event) callback) {
    final subscription = _controller.stream
        .where((event) => event is T)
        .cast<T>()
        .listen(callback);

    if (_stickyEvents.containsKey(T)) {
      scheduleMicrotask(() {
        // 再次检查，防止在微任务执行前事件被移除
        if (_stickyEvents.containsKey(T)) {
          callback(_stickyEvents[T] as T);
        }
      });
    }

    return subscription;
  }

  /// 获取粘性事件（不监听）
  T? getStickyEvent<T>() => _stickyEvents[T] as T?;

  /// 移除指定类型的粘性事件
  void removeStickyEvent<T>() {
    _stickyEvents.remove(T);
  }

  /// 清空所有粘性事件
  void clearStickyEvents() {
    _stickyEvents.clear();
  }

  /// 释放资源（通常不需要调用，除非要重建 EventBus）
  void dispose() {
    _controller.close();
    _stickyEvents.clear();
  }
}
```

#### 使用示例
```dart
// 定义事件类
class LoginSuccessEvent {
  final String userId;
  final String userName;
  LoginSuccessEvent(this.userId, this.userName);
}

class LogoutEvent {}

// 1. 发送粘性事件
StickyEventBus.instance.emitSticky(
  LoginSuccessEvent('001', 'John'),
);

// 2. 后续页面监听（立即收到缓存的事件）
final subscription = StickyEventBus.instance.on<LoginSuccessEvent>((event) {
  print('用户已登录: ${event.userName}');
});

// 3. 主动获取粘性事件（不监听后续变化）
final cachedEvent = StickyEventBus.instance.getStickyEvent<LoginSuccessEvent>();
if (cachedEvent != null) {
  print('缓存的用户: ${cachedEvent.userName}');
}

// 4. 登出时移除粘性事件
StickyEventBus.instance.removeStickyEvent<LoginSuccessEvent>();
StickyEventBus.instance.emit(LogoutEvent());

// 5. 取消订阅
subscription.cancel();
```

#### 优点
+ **无外部依赖**：纯 Dart 实现
+ **使用简单**：类似 Android EventBus 的 API 风格
+ **按类型区分**：不同事件类型独立缓存
+ **灵活控制**：可选择发送粘性或普通事件

#### 缺点
+ **功能有限**：不支持流操作符
+ **类型安全较弱**：运行时类型检查
+ **单例限制**：全局共享可能导致事件污染

#### 适用场景
+ 不想引入额外依赖
+ 简单的跨页面/组件通信
+ 熟悉 Android EventBus 的开发者
+ 事件类型明确且数量有限的场景

---

### 2.3 方案三：ValueNotifier
#### 原理
`ValueNotifier` 是 Flutter 内置的轻量级可观察对象，它持有一个值并在值变化时通知监听者。由于它始终持有当前值，新监听者可以立即读取。

> **⚠️**** 语义说明**：`ValueNotifier` 本质是**状态持有器**，而非事件流。它与真正的粘性事件有以下区别：
>
> + 新监听者不会自动收到"回放"，需要主动读取 `value`
> + 相同值赋值不会触发通知（去重机制）
> + 更适合"状态共享"而非"事件广播"场景
>

#### 代码实现
```dart
import 'package:flutter/foundation.dart';

/// 基于 ValueNotifier 的状态持有器
///
/// 注意：这不是严格意义上的"粘性事件"，而是状态共享机制。
/// - 新监听者需要主动读取 value，不会自动回放
/// - 相同值赋值不触发通知
class StickyValueNotifier<T> extends ValueNotifier<T?> {
  StickyValueNotifier([T? initialValue]) : super(initialValue);

  /// 更新值（相同值不触发通知）
  void emit(T event) {
    value = event;
  }

  /// 是否有值
  bool get hasValue => value != null;

  /// 清除值
  void clear() {
    value = null;
  }
}
```

#### 使用示例
```dart
// 1. 创建全局实例
final themeNotifier = StickyValueNotifier<ThemeMode>(ThemeMode.light);

// 2. 发送事件
themeNotifier.emit(ThemeMode.dark);

// 3. 监听变化
themeNotifier.addListener(() {
  print('主题变更: ${themeNotifier.value}');
});

// 4. 直接读取当前值（粘性特性）
print('当前主题: ${themeNotifier.value}');

// 5. 在 Widget 中使用 ValueListenableBuilder
ValueListenableBuilder<ThemeMode?>(
  valueListenable: themeNotifier,
  builder: (context, theme, child) {
    return Text('当前主题: $theme');
  },
)
```

#### 优点
+ **Flutter 原生**：无需任何依赖
+ **轻量简单**：API 简洁易懂
+ **Widget 集成**：配合 ValueListenableBuilder 使用

#### 缺点
+ **单值限制**：每个实例只能持有一个值
+ **无流操作**：不支持 map、filter 等操作
+ **手动管理**：需要手动 removeListener

#### 适用场景
+ 简单的单值状态共享
+ 主题、语言等全局配置
+ 不需要复杂事件流的场景

---

### 2.4 方案四：Provider / Riverpod 状态管理
#### 原理
状态管理库天然具备"粘性"特性：状态始终存在于内存中，新的 Widget 订阅时立即获取当前状态值。本节以 Riverpod 2.x 为例（Provider 原理类似）。

> **📌**** 说明**：Riverpod 2.x 推荐使用 `Notifier` + `NotifierProvider`，而非旧版的 `StateNotifier` + `StateNotifierProvider`。
>

#### 依赖安装
```yaml
# pubspec.yaml
dependencies:
  flutter_riverpod: ^2.4.9
```

#### 代码实现
```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

// 定义状态类（不可变）
class AuthState {
  final bool isLoggedIn;
  final String? userId;
  final String? userName;

  const AuthState({
    this.isLoggedIn = false,
    this.userId,
    this.userName,
  });

  AuthState copyWith({bool? isLoggedIn, String? userId, String? userName}) {
    return AuthState(
      isLoggedIn: isLoggedIn ?? this.isLoggedIn,
      userId: userId ?? this.userId,
      userName: userName ?? this.userName,
    );
  }
}

// 使用 Notifier（Riverpod 2.x 推荐方式）
class AuthNotifier extends Notifier<AuthState> {
  @override
  AuthState build() {
    // 返回初始状态
    return const AuthState();
  }

  void login(String userId, String userName) {
    state = state.copyWith(
      isLoggedIn: true,
      userId: userId,
      userName: userName,
    );
  }

  void logout() {
    state = const AuthState();
  }
}

// 创建 Provider（新版 API）
final authProvider = NotifierProvider<AuthNotifier, AuthState>(
  AuthNotifier.new,
);
```

#### 使用示例
```dart
// 1. 在 App 根部包裹 ProviderScope
void main() {
  runApp(
    ProviderScope(
      child: MyApp(),
    ),
  );
}

// 2. 在任意页面发送事件（修改状态）
class LoginPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ElevatedButton(
      onPressed: () {
        ref.read(authProvider.notifier).login('001', 'John');
        Navigator.pushNamed(context, '/home');
      },
      child: Text('登录'),
    );
  }
}

// 3. 在其他页面监听（立即获取当前状态 - 粘性特性）
class HomePage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final authState = ref.watch(authProvider);

    return Text(
      authState.isLoggedIn
        ? '欢迎, ${authState.userName}'
        : '未登录',
    );
  }
}

// 4. 监听状态变化执行副作用
class ProfilePage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    ref.listen<AuthState>(authProvider, (previous, next) {
      if (previous?.isLoggedIn == true && !next.isLoggedIn) {
        Navigator.pushReplacementNamed(context, '/login');
      }
    });

    return Container();
  }
}
```

#### 优点
+ **完整解决方案**：状态管理 + 依赖注入一体化
+ **作用域控制**：可限定状态的生命周期范围
+ **自动重建**：状态变化时自动触发 Widget 重建
+ **社区生态好**：文档完善，插件丰富

#### 缺点
+ **依赖较重**：引入完整状态管理框架
+ **学习成本高**：需要理解 Provider/Riverpod 概念
+ **可能过度设计**：简单场景使用显得繁琐

#### 适用场景
+ 项目已使用 Provider/Riverpod
+ 需要完整的状态管理方案
+ 状态需要跨多个页面共享
+ 需要依赖注入能力

---

### 2.5 方案五：Future / Completer
#### 原理
`Future` 天然具备粘性特性：一旦 Future 完成，无论何时调用 `.then()` 或 `await`，都能立即获得结果。适合**一次性**粘性事件。

> **⚠️**** 重要**：此方案仅适用于"只发生一次"的事件。如需多次发送，请使用 BehaviorSubject 或 EventBus。
>

#### 代码实现
```dart
import 'dart:async';

/// 基于 Future 的一次性粘性事件
///
/// 设计原则：
/// - 只能发送一次，确保语义清晰
/// - 不提供 reset 方法，避免已等待的 Future 行为不一致
/// - 多次 await 同一 future 是安全的
class FutureStickyEvent<T> {
  Completer<T>? _completer;
  T? _value;
  bool _hasValue = false;

  /// 发送事件（只能发送一次）
  ///
  /// 如果已发送过，将抛出 [StateError]。
  void emit(T value) {
    if (_hasValue) {
      throw StateError('FutureStickyEvent 只能发送一次');
    }
    _value = value;
    _hasValue = true;
    _completer?.complete(value);
  }

  /// 获取事件值（支持先发后听、先听后发）
  ///
  /// 多次调用此 getter 是安全的，都会返回相同的结果。
  Future<T> get future {
    if (_hasValue) {
      return Future.value(_value);
    }
    _completer ??= Completer<T>();
    return _completer!.future;
  }

  /// 同步获取值（未发送时返回 null）
  T? get valueOrNull => _value;

  /// 是否已发送过值
  bool get hasValue => _hasValue;
}
```

#### 使用示例
```dart
// 1. 创建一次性粘性事件
final appInitEvent = FutureStickyEvent<AppConfig>();

// 场景 A：先发送后监听
appInitEvent.emit(AppConfig(apiUrl: 'https://api.example.com'));
appInitEvent.future.then((config) {
  print('配置已加载: ${config.apiUrl}'); // 立即执行
});

// 场景 B：先监听后发送
appInitEvent.future.then((config) {
  print('等待配置...'); // 等待 emit 后执行
});
// 稍后...
appInitEvent.emit(AppConfig(apiUrl: 'https://api.example.com'));

// 2. 使用 async/await
Future<void> initApp() async {
  final config = await appInitEvent.future;
  print('App 初始化完成: ${config.apiUrl}');
}

// 3. 在 Widget 中使用 FutureBuilder
FutureBuilder<AppConfig>(
  future: appInitEvent.future,
  builder: (context, snapshot) {
    if (snapshot.hasData) {
      return Text('API: ${snapshot.data!.apiUrl}');
    }
    return CircularProgressIndicator();
  },
)
```

#### 优点
+ **无依赖**：纯 Dart 实现
+ **语义清晰**：适合一次性异步结果
+ **简单直观**：async/await 友好

#### 缺点
+ **只能发送一次**：不适合重复事件
+ **无法取消**：监听后无法取消等待
+ **无多值支持**：只能持有单个结果

#### 适用场景
+ App 初始化完成事件
+ 一次性配置加载
+ 首次登录成功通知
+ 任何"只发生一次"的异步事件

---

## 三、方案对比与选型指南
### 3.1 特性对比表
| 特性 | BehaviorSubject | 自定义 EventBus | ValueNotifier | Riverpod | Future |
| --- | --- | --- | --- | --- | --- |
| 外部依赖 | rxdart | 无 | 无 | riverpod | 无 |
| 多次发送 | ✅ | ✅ | ✅ | ✅ | ❌ |
| 多订阅者 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 取消订阅 | ✅ | ✅ | ✅ | 自动 | ❌ |
| 流操作符 | ✅ | ❌ | ❌ | 有限 | ❌ |
| 自动回放 | ✅ | ✅ | ❌ | ❌ | ✅ |
| 时序可靠性 | 高 | 中 | - | 高 | 高 |
| Widget 集成 | StreamBuilder | 手动 | ValueListenableBuilder | Consumer | FutureBuilder |
| 学习成本 | 中 | 低 | 低 | 高 | 低 |


### 3.2 选型决策树
```plain
需要粘性事件？
    │
    ├─ 事件只发送一次？
    │       │
    │       └─ 是 → Future/Completer
    │
    └─ 事件可多次发送？
            │
            ├─ 项目已用状态管理库？
            │       │
            │       └─ 是 → Provider/Riverpod
            │
            ├─ 需要可靠的时序保证？
            │       │
            │       └─ 是 → RxDart BehaviorSubject（推荐）
            │
            ├─ 只是简单单值状态共享？
            │       │
            │       └─ 是 → ValueNotifier
            │
            └─ 无依赖要求 + 简单场景？
                    │
                    └─ 是 → 自定义 EventBus
```

---

### 3.3 场景推荐
| 场景 | 推荐方案 | 说明 |
| --- | --- | --- |
| **通用粘性事件（首选）** | BehaviorSubject | 时序可靠，功能完整 |
| 无依赖要求 + 简单通信 | 自定义 EventBus | 注意时序问题 |
| 简单单值状态共享 | ValueNotifier | 非严格粘性事件 |
| 已使用状态管理库 | Provider/Riverpod | 状态拉取模式 |
| 一次性初始化事件 | Future/Completer | 语义清晰 |


---

## 四、最佳实践与注意事项
### 4.1 生命周期管理
```dart
class MyPage extends StatefulWidget {
  @override
  _MyPageState createState() => _MyPageState();
}

class _MyPageState extends State<MyPage> {
  StreamSubscription? _subscription;

  @override
  void initState() {
    super.initState();
    // 注册监听
    _subscription = stickyEvent.listen((event) {
      // 处理事件
    });
  }

  @override
  void dispose() {
    // 务必取消订阅，避免内存泄漏
    _subscription?.cancel();
    super.dispose();
  }
}
```

### 4.2 避免内存泄漏
**常见问题**：在 Widget dispose 后仍然调用 setState

```dart
// 错误示例
_subscription = event.listen((data) {
  setState(() {  // Widget 已销毁时会报错
    _data = data;
  });
});

// 正确示例
_subscription = event.listen((data) {
  if (mounted) {  // 检查 Widget 是否仍然挂载
    setState(() {
      _data = data;
    });
  }
});
```

### 4.3 粘性事件的清理时机
```dart
// 登出时清理用户相关的粘性事件
void logout() {
  StickyEventBus.instance.removeStickyEvent<UserInfoEvent>();
  StickyEventBus.instance.removeStickyEvent<CartEvent>();
  // 发送登出事件
  StickyEventBus.instance.emit(LogoutEvent());
}
```

### 4.4 避免事件污染
单例 EventBus 的粘性事件是全局共享的，需注意：

+ 及时清理不再需要的粘性事件
+ 考虑使用作用域隔离（如 Riverpod 的 ProviderScope）
+ 敏感数据不要长期缓存在粘性事件中

### 4.5 调试技巧
```dart
// 在 EventBus 中添加日志
void emitSticky<T>(T event) {
  debugPrint('[StickyEventBus] 发送粘性事件: ${T.toString()}');
  _stickyEvents[T] = event;
  _controller.add(event);
}
```

---

## 五、总结
| 需求场景 | 首选方案 | 备注 |
| --- | --- | --- |
| **通用粘性事件** | BehaviorSubject | 时序可靠，功能完整 |
| 无依赖 + 简单通信 | 自定义 EventBus | 注意时序问题 |
| 无依赖 + 单值状态 | ValueNotifier | 非严格粘性事件 |
| 完整状态管理 | Riverpod | 状态拉取模式 |
| 一次性事件 | Future/Completer | 语义清晰 |


选择合适的方案取决于项目规模、团队熟悉度和具体需求。

**推荐**：对于大多数需要粘性事件的场景，**BehaviorSubject** 是首选方案，它提供可靠的时序保证和丰富的流操作能力。如果不想引入外部依赖，可以使用**自定义 EventBus**，但需注意其时序限制。

---

_文档版本：1.1 | 最后更新：2025-12-28_

