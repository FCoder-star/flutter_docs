# Flutter FragmentShader 完全指南（三）：工具与调试

> 本文档详细讲解 Flutter Shader 的编译流程、Impeller 与 Skia 的深度对比、性能调试技巧以及跨平台兼容性处理。

---

## 一、Shader 编译流程

### 1.1 编译流程概览

```plain
┌─────────────────────────────────────────────────────────────────────────┐
│                        Shader 编译流程                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  开发阶段                                                                │
│  ┌──────────┐                                                           │
│  │ .frag    │ ─── 源代码（GLSL）                                         │
│  └────┬─────┘                                                           │
│       │                                                                 │
│       ▼                                                                 │
│  ┌──────────┐                                                           │
│  │ impellerc│ ─── Flutter 内置编译器                                     │
│  └────┬─────┘                                                           │
│       │                                                                 │
│       ▼                                                                 │
│  ┌──────────┐                                                           │
│  │ .spirv   │ ─── 中间字节码                                             │
│  └────┬─────┘                                                           │
│       │                                                                 │
│  构建阶段                                                                │
│       │                                                                 │
│       ├────────────┬────────────┬────────────┐                          │
│       ▼            ▼            ▼            ▼                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                     │
│  │ Metal   │  │ Vulkan  │  │ OpenGL  │  │ WebGL   │                     │
│  │ (iOS)   │  │(Android)│  │  (ES)   │  │ (Web)   │                     │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 pubspec.yaml 配置

```yaml
flutter:
  shaders:
    - shaders/gradient.frag
    - shaders/blur.frag
    - shaders/ripple.frag
```

当运行 `flutter build` 或 `flutter run` 时，Flutter 会自动：
1. 检测 `shaders` 列表中的文件
2. 使用 `impellerc` 编译为目标平台格式
3. 将编译产物打包到应用中

### 1.3 手动编译（高级用法）

> 📌 **重要提示**：Flutter SDK 会把 `impellerc` 二进制放在 `<FLUTTER_HOME>/bin/cache/artifacts/engine/<host-platform>/impellerc[.exe]`，例如 `darwin-x64`、`darwin-arm64`、`linux-x64` 或 `windows-x64`。配置好 `FLUTTER_HOME` 后即可直接调用。

```bash
# 查看 impellerc 帮助（macOS/Linux 示例）
$FLUTTER_HOME/bin/cache/artifacts/engine/darwin-x64/impellerc --help

# 如需频繁调用，可先设置变量
IMP=$FLUTTER_HOME/bin/cache/artifacts/engine/darwin-x64/impellerc

# Windows 请改用：
# set IMP=%FLUTTER_HOME%\bin\cache\artifacts\engine\windows-x64\impellerc.exe

# 编译为 Metal（iOS）
$IMP \
  --input shaders/effect.frag \
  --input-type frag \
  --metal-ios \
  --sl build/effect_ios.msl

# 编译为 Metal（macOS）
$IMP \
  --input shaders/effect.frag \
  --input-type frag \
  --metal-desktop \
  --sl build/effect_macos.msl

# 编译为 Vulkan SPIR-V（Android）
$IMP \
  --input shaders/effect.frag \
  --input-type frag \
  --vulkan \
  --spirv build/effect.spirv

# 编译为 OpenGL ES（Android 回退）
$IMP \
  --input shaders/effect.frag \
  --input-type frag \
  --opengl-es \
  --sl build/effect.gles

# 编译为 Flutter 运行时格式（.iplr，包含多平台）
$IMP \
  --input shaders/effect.frag \
  --input-type frag \
  --iplr \
  --runtime-stage-metal \
  --runtime-stage-gles3 \
  --sl build/effect.iplr
```

### 1.4 编译错误排查

```bash
# 查看详细编译日志
flutter run --verbose 2>&1 | grep -i shader

# 导出中间表示用于调试
$IMP --dump-ir --input shaders/effect.frag
```

**常见编译错误：**

| 错误信息 | 原因 | 解决方案 |
|---------|------|----------|
| `Shader compilation failed` | GLSL 语法错误 | 检查版本声明和语法 |
| `Unknown identifier` | 使用了不支持的函数 | 查阅 GLSL ES 3.0 规范 |
| `Precision not specified` | 缺少精度声明 | 添加 `precision mediump float;` |

---

## 二、Impeller vs Skia 深度对比

### 2.1 架构差异

```plain
┌─────────────────────────────────────────────────────────────────────────┐
│                           Skia 架构                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Flutter Widget ──→ RenderObject ──→ Skia ──→ GPU Backend              │
│                                        │                                │
│                                        ├──→ OpenGL                      │
│                                        ├──→ Metal                       │
│                                        ├──→ Vulkan                      │
│                                        └──→ Direct3D                    │
│                                                                         │
│  特点：通用 2D 图形库，运行时 JIT 编译 Shader                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         Impeller 架构                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Flutter Widget ──→ RenderObject ──→ Impeller ──→ GPU Backend          │
│                                          │                              │
│                                          ├──→ Metal (iOS/macOS)         │
│                                          ├──→ Vulkan (Android)          │
│                                          └──→ OpenGL ES (回退)          │
│                                                                         │
│  特点：Flutter 专属渲染器，AOT 预编译 Shader                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心差异对比

| 特性 | Skia | Impeller |
|------|------|----------|
| **设计目标** | 通用 2D 图形库 | Flutter 专属渲染器 |
| **Shader 编译** | 运行时 JIT | 构建时 AOT |
| **首帧性能** | 可能卡顿（需编译） | 流畅（已预编译） |
| **平台支持** | 广泛 | iOS/Android 为主 |
| **成熟度** | 非常成熟 | 持续完善中 |
| **自定义 Shader** | 完全支持 | 支持（有限制） |

### 2.3 Shader 语法差异

**Skia 模式（传统）：**

```glsl
#version 300 es
precision mediump float;

uniform vec2 u_resolution;
uniform float u_time;

out vec4 fragColor;

void main() {
    vec2 uv = gl_FragCoord.xy / u_resolution;
    fragColor = vec4(uv, 0.5 + 0.5 * sin(u_time), 1.0);
}
```

**Impeller 模式（推荐）：**

```glsl
#include <flutter/runtime_effect.glsl>
precision mediump float;

uniform vec2 u_resolution;
uniform float u_time;

out vec4 fragColor;

void main() {
    // 使用 FlutterFragCoord() 替代 gl_FragCoord
    vec2 uv = FlutterFragCoord().xy / u_resolution;
    fragColor = vec4(uv, 0.5 + 0.5 * sin(u_time), 1.0);
}
```

### 2.4 Impeller 的限制

> ⚠️ **注意**：使用 Impeller 时需要遵守以下限制

| 限制 | 说明 | 替代方案 |
|------|------|----------|
| 禁止递归函数 | GPU 不支持递归 | 使用循环展开 |
| 禁止动态数组长度 | 编译时必须确定 | 使用固定大小数组 |
| 纹理采样限制 | 最多 8 个采样器 | 合并纹理或分 pass |
| 精度声明 | 必须显式声明 | 添加 precision 语句 |

### 2.5 性能对比

```plain
首帧渲染时间对比（示例数据）：

Skia（无缓存）：
├── Shader 编译：~50-200ms
├── Pipeline 创建：~20-50ms
└── 首帧总计：~100-300ms

Impeller（预编译）：
├── Shader 加载：~5-10ms
├── Pipeline 创建：~5-10ms
└── 首帧总计：~15-30ms

稳定帧率（60fps 目标）：
├── Skia：需要 SkSL 缓存才能稳定
└── Impeller：开箱即用稳定
```

### 2.6 迁移检查清单

从 Skia 迁移到 Impeller 时，检查以下项目：

```dart
// 1. 检查 Shader 语法兼容性
// ❌ Skia 特有语法
gl_FragCoord.xy

// ✅ Impeller 兼容语法
FlutterFragCoord().xy
```

```yaml
# 2. 确保 pubspec.yaml 正确配置
flutter:
  shaders:
    - shaders/effect.frag
```

```bash
# 3. 测试 Impeller 模式
flutter run --enable-impeller

# 4. 检查特定功能支持
# 某些 Skia 特性在 Impeller 中可能尚未实现
```

---

## 三、Shader 预热（Warm-up）策略

### 3.1 为什么需要预热

即使使用 Impeller，首次使用 Shader 时仍需要：
- 加载 Shader 资源
- 创建 GPU Pipeline
- 分配 GPU 内存

预热可以将这些操作提前到应用启动时完成。

### 3.2 基本预热实现

```dart
import 'dart:ui' as ui;
import 'package:flutter/material.dart';

/// Shader 预热管理器
class ShaderWarmUpManager {
  static final instance = ShaderWarmUpManager._();
  ShaderWarmUpManager._();

  final Map<String, ui.FragmentShader> _shaders = {};
  bool _isWarmedUp = false;

  /// 预热所有 Shader
  Future<void> warmUp() async {
    if (_isWarmedUp) return;

    final shaderPaths = [
      'shaders/gradient.frag',
      'shaders/blur.frag',
      'shaders/ripple.frag',
    ];

    for (final path in shaderPaths) {
      try {
        final program = await ui.FragmentProgram.fromAsset(path);
        final shader = program.fragmentShader();

        // 执行一次虚拟绘制，触发 Pipeline 创建
        _performDummyDraw(shader);

        _shaders[path] = shader;
      } catch (e) {
        debugPrint('Failed to warm up shader: $path, error: $e');
      }
    }

    _isWarmedUp = true;
  }

  /// 获取预热后的 Shader
  ui.FragmentShader? getShader(String path) => _shaders[path];

  /// 执行虚拟绘制
  void _performDummyDraw(ui.FragmentShader shader) {
    final recorder = ui.PictureRecorder();
    final canvas = Canvas(recorder);

    shader
      ..setFloat(0, 100)
      ..setFloat(1, 100)
      ..setFloat(2, 0);

    canvas.drawRect(
      const Rect.fromLTWH(0, 0, 100, 100),
      Paint()..shader = shader,
    );

    final picture = recorder.endRecording();
    picture.dispose();
  }
}

// 在 main.dart 中使用
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // 预热 Shader
  await ShaderWarmUpManager.instance.warmUp();

  runApp(const MyApp());
}
```

> 👍 **Flutter 内置方案**：除了自定义管理器，Flutter 还提供了内置的 `ShaderWarmUp` 钩子来在首帧前执行一次 GPU 绘制：
>
> ```dart
> import 'dart:ui' as ui;
> import 'package:flutter/material.dart';
> import 'package:flutter/painting.dart';
>
> class FragmentShaderWarmUp extends ShaderWarmUp {
>   FragmentShaderWarmUp(this.shader);
>
>   final ui.FragmentShader shader;
>
>   @override
>   void warmUpOnCanvas(Canvas canvas) {
>     shader
>       ..setFloat(0, 256)
>       ..setFloat(1, 256)
>       ..setFloat(2, 0);
>
>     canvas.drawRect(
>       const Rect.fromLTWH(0, 0, 256, 256),
>       Paint()..shader = shader,
>     );
>   }
> }
>
> void main() async {
>   WidgetsFlutterBinding.ensureInitialized();
>   final program = await ui.FragmentProgram.fromAsset('shaders/gradient.frag');
>   final shader = program.fragmentShader();
>   PaintingBinding.instance.shaderWarmUp = FragmentShaderWarmUp(shader);
>   runApp(const MyApp());
> }
> ```
>
> 内置 warm-up 会在渲染第一帧之前自动执行一次，无需手动绘制到 `PictureRecorder`。

**两种方案对比：**

| 特性 | 自定义 ShaderManager | 内置 ShaderWarmUp |
|------|---------------------|-------------------|
| **用途** | 长期缓存 + 全局复用 + 生命周期管理 | 首帧卡顿优化 |
| **适用场景** | 多 Shader、需要缓存资源 | 单个或少量 Shader 预热 |
| **集成复杂度** | 需要自己维护 | 极简，与渲染管线对齐 |
| **跨页面复用** | 支持 | 不支持 |
| **推荐策略** | 需要复用 Shader、共享纹理或集中加载 | 仅需规避首帧卡顿 |

> 💡 **最佳实践**：两者并不冲突。可以先用 Manager 加载/缓存，再把缓存的 Shader 交给 `ShaderWarmUp` 进行一次预绘。

### 3.3 SkSL 缓存（Skia 模式）

对于使用 Skia 的场景，可以收集和打包 SkSL 缓存：

```bash
# 1. 收集 SkSL（在真机上运行）
flutter run --profile --cache-sksl --purge-persistent-cache

# 2. 操作应用，触发所有 Shader 路径

# 3. 按 M 键导出 SkSL
# 输出：flutter_01.sksl.json

# 4. 构建时打包 SkSL
flutter build apk --bundle-sksl-path flutter_01.sksl.json
flutter build ios --bundle-sksl-path flutter_01.sksl.json
```

### 3.4 预热时机选择

| 时机 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 应用启动时 | 后续使用流畅 | 增加启动时间 | Shader 数量少 |
| 闪屏页期间 | 用户无感知 | 需要闪屏页 | 大多数应用 |
| 首次进入页面 | 按需加载 | 首次可能卡顿 | Shader 数量多 |
| 后台预加载 | 不阻塞 UI | 实现复杂 | 高性能要求 |

---

## 四、性能分析与调试

### 4.1 Flutter DevTools

```bash
# 启动 profile 模式并输出 DevTools 链接
flutter run --profile --trace-startup

# 另开终端启动 DevTools（Flutter 3.13+ 已内置命令）
flutter devtools
```

**使用步骤：**

1. 在 DevTools 首页选择 "Connect to a Running App"，输入 `flutter run` 打印的 VM Service URL
2. 打开 **Performance** 面板，点击 Record，操作应用触发 Shader，再停止录制
3. 检查 UI/Raster/GPU 三条时间线：`Raster > 16 ms` 通常表示 GPU 侧瓶颈；若看到 `ShaderCompile` 事件可定位到具体 `.frag`

**Performance 面板关键指标：**

```plain
┌─────────────────────────────────────────────────────────────────────────┐
│                        DevTools Performance                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Frame Timeline:                                                        │
│  ├── UI Thread: Widget 构建、布局计算                                    │
│  ├── Raster Thread: GPU 绘制、Shader 执行                               │
│  └── GPU Thread: 实际 GPU 命令执行                                       │
│                                                                         │
│  关注点：                                                                │
│  - Raster 时间 > 16ms 表示 GPU 瓶颈                                      │
│  - Shader 编译会在 Raster 中显示为长条                                   │
│  - ShaderCompile 事件可定位到具体的 .frag 文件                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 命令行调试

```bash
# Skia 模式调试
flutter run --profile --trace-skia

# Impeller 模式调试（iOS/macOS/Android）
flutter run --profile --enable-impeller --trace-systrace

# Skia 模式调试（对比用）
flutter run --profile --no-enable-impeller --trace-skia
```

> 📌 **注意**：Flutter Tool 不支持 `--impeller-print-shaders` 或 `--impeller-backend` 等标志。要在 Android 上指定 Impeller 后端，需在 `AndroidManifest.xml` 中配置：
> ```xml
> <meta-data
>     android:name="io.flutter.embedding.android.ImpellerBackend"
>     android:value="opengles" />
> ```
> iOS/macOS 始终使用 Metal 后端。

### 4.3 Android 性能分析

```bash
# 查看渲染时间分布
adb shell dumpsys gfxinfo <package_name>

# 使用 Perfetto（Android 11+，推荐）
adb shell perfetto -o /data/misc/perfetto/trace.pftrace -t 10s \
    gfx,view,frame,hwui,sched,ftrace/idle
adb pull /data/misc/perfetto/trace.pftrace trace.pftrace

# Android GPU Inspector (AGI)
agi trace --app <package_name> --enable-gpu

# GPU 渲染模式条（老版本调试）
# 在开发者选项中启用 "GPU 渲染模式分析"
```

> 💡 **工具选择**：对旧版 Android SDK 若仍需 `systrace.py`，需从 `platform-tools/systrace` 目录手动安装；新项目更推荐 Perfetto。Perfetto trace 可在 https://ui.perfetto.dev 中打开分析。

### 4.4 iOS 性能分析

```bash
# 使用 Instruments
# 1. 打开 Xcode → Product → Profile
# 2. 选择 "Metal System Trace" 或 "GPU"
# 3. 分析 Shader 执行时间
```

### 4.5 常见性能问题与排查路径

| 问题 | 症状 | 调试步骤 | 解决方案 |
|------|------|----------|----------|
| Shader 编译卡顿 | 首次进入页面掉帧 | DevTools Timeline 出现 `ShaderCompile` | 启用 `ShaderWarmUp` 或 SkSL 缓存 |
| 过度绘制 | FPS 持续 <60 | 打开 `PerformanceOverlay` 查看 Raster >16ms | 简化层级、使用 `RepaintBoundary` |
| Uniform 更新频繁 | GPU 等待 CPU | DevTools GPU 线程出现 `waitForUniforms` | 缓存 uniform 值，仅在变化时调用 `setFloat` |
| 纹理过大 | 内存激增、OOM | Android Studio Memory Profiler | 预加载缩略图、及时 `dispose()` ImageShader |
| 复杂循环 | Profile 模式掉帧 | Perfetto Trace 显示 Fragment 超时 | 改用查表/纹理噪声替代循环 |

> 💡 **实时监控**：可在调试构建中启用 `WidgetsBinding.instance.platformDispatcher.performanceOverlayEnabled = true;`，实时观察 UI/Raster 时间与 `Shader compilations` 计数。

**实际案例：动态背景 Shader 卡顿排查**

```dart
// 问题：首次进入页面时出现明显卡顿
// 步骤1：使用 DevTools Performance 录制
// 发现：Raster 线程出现 ShaderCompile 事件，耗时 120ms

// 步骤2：实施 ShaderWarmUp
class MyShaderWarmUp extends ShaderWarmUp {
  MyShaderWarmUp(this.shader);
  final ui.FragmentShader shader;

  @override
  void warmUpOnCanvas(Canvas canvas) {
    shader
      ..setFloat(0, 256)
      ..setFloat(1, 256)
      ..setFloat(2, 0);
    canvas.drawRect(
      const Rect.fromLTWH(0, 0, 256, 256),
      Paint()..shader = shader,
    );
  }
}

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final program = await ui.FragmentProgram.fromAsset('shaders/background.frag');
  PaintingBinding.instance.shaderWarmUp = MyShaderWarmUp(program.fragmentShader());
  runApp(const MyApp());
}

// 结果：首帧卡顿消失，Raster 时间降至 8ms
```

### 4.6 调试 Shader 代码

> 💡 **前提**：以下示例基于 Flutter 3.10+ 的 `FlutterFragCoord()` 和 `runtime_effect.glsl`。

```glsl
#include <flutter/runtime_effect.glsl>
precision mediump float;

uniform vec2 u_resolution;

out vec4 fragColor;

// 方法1：输出调试颜色
void main() {
    vec2 uv = FlutterFragCoord().xy / u_resolution;

    // 调试：显示 UV 坐标
    fragColor = vec4(uv, 0.0, 1.0);
    return;  // 临时返回

    // 正常代码...
}
```

```glsl
#include <flutter/runtime_effect.glsl>
precision mediump float;

uniform vec2 u_resolution;

out vec4 fragColor;

// 方法2：分段调试
void main() {
    vec2 uv = FlutterFragCoord().xy / u_resolution;

    // 调试特定区域
    if (uv.x < 0.5) {
        fragColor = vec4(1.0, 0.0, 0.0, 1.0);  // 左半红色
        return;
    }

    // 正常代码...
}
```

```glsl
#include <flutter/runtime_effect.glsl>
precision mediump float;

uniform vec2 u_resolution;

out vec4 fragColor;

// 方法3：数值可视化
float someCalculation() {
    // 示例计算
    return sin(FlutterFragCoord().x * 0.1);
}

void main() {
    float value = someCalculation();

    // 将数值映射到颜色
    // 负值显示蓝色，正值显示红色
    fragColor = vec4(
        max(0.0, value),
        0.0,
        max(0.0, -value),
        1.0
    );
}
```

### 4.7 基准测试与回归检测

**导出 Timeline 数据：**

```bash
# 运行集成测试并输出 timeline
flutter drive --profile --trace-startup \
  --target=test_driver/perf.dart \
  --driver=test_driver/perf_test.dart

# 导出 timeline json
flutter run --profile --trace-startup --timeline-streams=Compiler,Shader \
  --timeline-file=timeline.json
```

**分析 Timeline：**

1. 将 `timeline.json` 导入 DevTools "Timeline" 页签比较历史版本
2. 结合 `package:metrics_center` 或 CI，将平均 Raster/GPU 时间写入基线，防止 Shader 变更退化

**使用 PerformanceOverlay Widget：**

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(
    MaterialApp(
      // 显示性能叠加层
      showPerformanceOverlay: true,
      home: const MyShaderPage(),
    ),
  );
}
```

**编程式性能监控：**

```dart
import 'dart:ui' as ui;
import 'package:flutter/scheduler.dart';

class ShaderPerformanceMonitor {
  final List<Duration> _frameTimes = [];

  void startMonitoring() {
    SchedulerBinding.instance.addTimingsCallback((timings) {
      for (final timing in timings) {
        final rasterTime = timing.rasterDuration;
        _frameTimes.add(rasterTime);

        // 警告：Raster 时间超过 16ms
        if (rasterTime.inMilliseconds > 16) {
          debugPrint('⚠️ Slow frame: ${rasterTime.inMilliseconds}ms');
        }
      }
    });
  }

  double get averageFrameTime {
    if (_frameTimes.isEmpty) return 0;
    final total = _frameTimes.fold<int>(
      0,
      (sum, duration) => sum + duration.inMicroseconds,
    );
    return total / _frameTimes.length / 1000; // 转换为毫秒
  }

  void reset() => _frameTimes.clear();
}

// 使用示例
void main() {
  final monitor = ShaderPerformanceMonitor();
  monitor.startMonitoring();

  runApp(const MyApp());
}
```

**性能基准示例：**

```dart
// test_driver/shader_perf_test.dart
import 'package:flutter_driver/flutter_driver.dart';
import 'package:test/test.dart';

void main() {
  group('Shader Performance', () {
    late FlutterDriver driver;

    setUpAll(() async {
      driver = await FlutterDriver.connect();
    });

    tearDownAll(() async {
      await driver.close();
    });

    test('Shader warm-up reduces first frame time', () async {
      // 启动应用并记录 timeline
      final timeline = await driver.traceAction(() async {
        await driver.tap(find.byValueKey('shader_page'));
        await Future.delayed(const Duration(seconds: 2));
      });

      // 分析 timeline
      final summary = TimelineSummary.summarize(timeline);

      // 断言：首帧 Raster 时间应小于 20ms
      expect(
        summary.summaryJson['average_frame_build_time_millis'],
        lessThan(20.0),
      );
    });
  });
}
```

---

## 五、跨平台兼容性

### 5.1 平台支持矩阵

| 平台 | 渲染器 | Shader 后端 | 状态 |
|------|--------|-------------|------|
| iOS | Impeller（唯一渲染器） | Metal | 稳定 |
| Android | Impeller（默认，可退回 Skia） | Vulkan/OpenGL ES | 稳定 |
| macOS | Skia（默认），Impeller（预览） | Metal | 预览 |
| Windows | Skia | OpenGL/ANGLE | 稳定 |
| Linux | Skia | OpenGL | 稳定 |
| Web | CanvasKit | WebGL | 稳定（不支持 FragmentShader） |

### 5.2 平台特定配置

**iOS（Info.plist）：**
```xml
<!-- iOS 已默认使用 Impeller，无需配置 -->
<!-- Flutter 3.16+ 已移除 Skia 后端 -->
```

**Android（AndroidManifest.xml）：**
```xml
<!-- Android 默认启用 Impeller，如需禁用可添加： -->
<meta-data
    android:name="io.flutter.embedding.android.EnableImpeller"
    android:value="false" />

<!-- 指定 Impeller 后端（可选）： -->
<meta-data
    android:name="io.flutter.embedding.android.ImpellerBackend"
    android:value="opengles" />
```

**macOS（Info.plist）：**
```xml
<!-- macOS 需要显式启用 Impeller（预览功能） -->
<key>FLTEnableImpeller</key>
<true/>
```

### 5.3 条件编译策略

```dart
import 'dart:ui' as ui;
import 'package:flutter/foundation.dart';

/// 根据平台选择 Shader 变体
class PlatformShaderLoader {
  static Future<ui.FragmentProgram> load(String baseName) async {
    if (kIsWeb) {
      // Web 平台不支持 FragmentShader
      throw UnsupportedError(
        'FragmentShader is not supported on Web platform. '
        'Please use alternative rendering methods like Canvas or CSS effects.'
      );
    }

    String path;
    switch (defaultTargetPlatform) {
      case TargetPlatform.iOS:
      case TargetPlatform.macOS:
        // Apple 平台使用 Metal 优化版本
        path = 'shaders/metal/$baseName.frag';
        break;
      case TargetPlatform.android:
        // Android 使用通用版本
        path = 'shaders/android/$baseName.frag';
        break;
      default:
        // Desktop 使用 OpenGL 版本
        path = 'shaders/desktop/$baseName.frag';
    }

    return ui.FragmentProgram.fromAsset(path);
  }
}
```

### 5.4 兼容性 Shader 编写

```glsl
// compatible_effect.frag
// 兼容 Impeller 和 Skia 的写法

#version 300 es
precision mediump float;

uniform vec2 u_resolution;
uniform float u_time;

out vec4 fragColor;

void main() {
    // 使用 gl_FragCoord（两种模式都支持）
    vec2 uv = gl_FragCoord.xy / u_resolution;

    // 避免使用平台特定函数
    // 使用基础 GLSL ES 3.0 函数

    vec3 color = vec3(uv, 0.5 + 0.5 * sin(u_time));
    fragColor = vec4(color, 1.0);
}
```

### 5.5 Web 平台注意事项

```dart
// Web 平台 Shader 支持检测
class WebShaderSupport {
  static bool get isSupported {
    if (!kIsWeb) return true;

    // Web 平台需要 WebGL 2.0 支持
    // 可以通过 JavaScript 互操作检测
    return true; // 简化示例
  }

  static Widget buildWithFallback({
    required Widget shaderWidget,
    required Widget fallbackWidget,
  }) {
    if (isSupported) {
      return shaderWidget;
    }
    return fallbackWidget;
  }
}
```

### 5.6 低端设备回退

```dart
/// 设备能力检测与回退
class ShaderCapabilityManager {
  static bool _useShaders = true;

  /// 检测设备是否支持 Shader
  static Future<void> initialize() async {
    try {
      // 尝试加载一个简单的测试 Shader
      final program = await ui.FragmentProgram.fromAsset('shaders/test.frag');
      final shader = program.fragmentShader();

      // 执行测试绘制
      final recorder = ui.PictureRecorder();
      final canvas = Canvas(recorder);
      canvas.drawRect(
        const Rect.fromLTWH(0, 0, 10, 10),
        Paint()..shader = shader,
      );
      recorder.endRecording();

      _useShaders = true;
    } catch (e) {
      debugPrint('Shader not supported: $e');
      _useShaders = false;
    }
  }

  static bool get useShaders => _useShaders;
}

// 使用示例
class AdaptiveBackground extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    if (ShaderCapabilityManager.useShaders) {
      return ShaderBackground();
    }
    // 回退到普通渐变
    return Container(
      decoration: const BoxDecoration(
        gradient: LinearGradient(
          colors: [Colors.blue, Colors.purple],
        ),
      ),
    );
  }
}
```

---

## 六、调试命令速查

```bash
# 基本运行
flutter run                              # 默认模式
flutter run --profile                    # Profile 模式（性能分析）
flutter run --release                    # Release 模式

# Impeller 相关
flutter run --enable-impeller            # 启用 Impeller
flutter run --no-enable-impeller         # 禁用 Impeller
flutter run --trace-systrace             # 系统追踪（配合 Impeller）

# Skia 相关
flutter run --trace-skia                 # Skia 追踪
flutter run --cache-sksl                 # 收集 SkSL
flutter run --purge-persistent-cache     # 清除缓存

# 构建
flutter build apk --bundle-sksl-path x.json  # 打包 SkSL
flutter build ios --enable-impeller          # iOS Impeller 构建

# 调试
flutter logs                             # 查看日志
flutter analyze                          # 代码分析
```

---

## 七、下一步

在下一篇文档中，我们将通过实战案例学习：
- 动态渐变背景
- 图片滤镜效果
- 波纹/涟漪动画
- 玻璃态模糊效果
- 粒子特效

---

_文档版本：1.1 | 最后更新：2025-01-06_
