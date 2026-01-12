# Flutter FragmentShader 完全指南（二）：API 详解

> 本文档深入讲解 Flutter FragmentShader 的核心 API，包括资源加载、参数传递、生命周期管理等关键知识点。

---

## 一、FragmentProgram 类详解

### 1.1 什么是 FragmentProgram

`FragmentProgram` 是 Shader 程序的编译产物，代表一个已加载到 GPU 的着色器程序。它是创建 `FragmentShader` 实例的工厂。

```plain
.frag 文件 ──impellerc──→ .iplr 资源包 ──加载──→ FragmentProgram ──创建──→ FragmentShader
```

### 1.2 生命周期

```plain
┌─────────────────────────────────────────────────────────────────────────┐
│                        FragmentProgram 生命周期                          │
├─────────────────────────────────────────────────────────────────────────┤
│  1. 构建阶段                                                             │
│     flutter build/run 时，impellerc 将 .frag 编译为 .iplr 资源包         │
│     .iplr 包含 Impeller 和 Skia 多后端代码                               │
│                                                                         │
│  2. 加载阶段                                                             │
│     FragmentProgram.fromAsset() 读取 .iplr 并由引擎选择合适的后端         │
│     引擎在 _shaderRegistry 中全局缓存实例，同一 asset 复用缓存            │
│                                                                         │
│  3. 使用阶段                                                             │
│     program.fragmentShader() 创建可配置的 FragmentShader 实例            │
│                                                                         │
│  4. 释放阶段                                                             │
│     FragmentProgram 由引擎持有直到进程退出                               │
│     FragmentShader 需要手动调用 dispose() 释放 uniform/sampler 资源      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 加载方式

**从 Asset 加载（唯一方式）**

```dart
// 异步加载打包在应用中的 Shader
final program = await FragmentProgram.fromAsset('shaders/gradient.frag');
```

> 📌 **说明**：Flutter 3.35 只支持通过 `fromAsset` 加载 `.iplr` 资源包，该资源包由 `impellerc` 在构建时生成。引擎会根据当前渲染后端（Impeller Metal/Vulkan/GLES 或 Skia）自动选择合适的 runtime stage。目前没有公开的 `fromBytes` API 用于动态加载网络下载的 Shader。

### 1.4 缓存策略

> ⚠️ **重要**：Flutter 引擎会在 `_shaderRegistry` 中自动缓存 `FragmentProgram` 实例。同一 asset 路径多次调用 `fromAsset` 会立即返回缓存对象，不会重复加载或构建 GPU 程序。缓存实例由引擎持有直到进程退出。

**应用层无需手动缓存 `FragmentProgram`**，但如果需要复用 `Future` 或等待逻辑，可以在应用层做额外封装：

```dart
// ⚠️ 注意：虽然引擎会缓存 FragmentProgram，但在 build 中反复调用 fromAsset 会创建多个 Future
class SuboptimalShaderWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return FutureBuilder<FragmentProgram>(
      // 每次 build 都创建新的 Future（虽然底层 Program 是缓存的）
      future: FragmentProgram.fromAsset('shaders/wave.frag'),
      builder: (context, snapshot) {
        // ...
      },
    );
  }
}
```

```dart
// ✅ 推荐：在 initState 中缓存 Future
class GoodShaderWidget extends StatefulWidget {
  @override
  State<GoodShaderWidget> createState() => _GoodShaderWidgetState();
}

class _GoodShaderWidgetState extends State<GoodShaderWidget> {
  late final Future<FragmentProgram> _programFuture;

  @override
  void initState() {
    super.initState();
    // 缓存 Future，避免重复创建异步操作
    _programFuture = FragmentProgram.fromAsset('shaders/wave.frag');
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<FragmentProgram>(
      future: _programFuture,  // 复用缓存的 Future
      builder: (context, snapshot) {
        // ...
      },
    );
  }
}
```

**应用层 Future 缓存方案（可选）**

```dart
/// Shader Future 缓存管理器（用于复用 Future，非必需）
class ShaderCache {
  ShaderCache._();
  static final instance = ShaderCache._();

  final _cache = <String, Future<FragmentProgram>>{};

  Future<FragmentProgram> load(String assetPath) {
    return _cache.putIfAbsent(
      assetPath,
      () => FragmentProgram.fromAsset(assetPath),
    );
  }

  void clear() {
    _cache.clear();
  }
}

// 使用
final program = await ShaderCache.instance.load('shaders/wave.frag');
```

---

## 二、FragmentShader 类详解

### 2.1 创建 FragmentShader

```dart
final program = await FragmentProgram.fromAsset('shaders/gradient.frag');
final shader = program.fragmentShader();
```

**关键特性：**
- 同一个 `FragmentProgram` 可以创建多个 `FragmentShader` 实例
- 每个实例维护独立的 Uniform 参数
- 底层 GPU 程序是共享的

### 2.2 复用策略

```dart
// ❌ 错误示例：每帧创建新的 FragmentShader
class BadPainter extends CustomPainter {
  final FragmentProgram program;

  @override
  void paint(Canvas canvas, Size size) {
    // 每次 paint 都创建新实例，浪费资源！
    final shader = program.fragmentShader();
    shader.setFloat(0, size.width);
    // ...
  }
}
```

```dart
// ✅ 正确示例：复用 FragmentShader 实例
class GoodPainter extends CustomPainter {
  final FragmentShader shader;  // 外部传入，复用
  final double time;

  GoodPainter({required this.shader, required this.time});

  @override
  void paint(Canvas canvas, Size size) {
    // 只更新参数，不创建新实例
    shader
      ..setFloat(0, size.width)
      ..setFloat(1, size.height)
      ..setFloat(2, time);

    canvas.drawRect(Offset.zero & size, Paint()..shader = shader);
  }

  @override
  bool shouldRepaint(covariant GoodPainter oldDelegate) {
    return oldDelegate.shader != shader || oldDelegate.time != time;
  }

  // 注意：shader 由外部传入，生命周期由父级 State 管理，此处不调用 dispose()
}
```

### 2.3 生命周期管理

> ⚠️ **重要**：`FragmentShader` 提供 `dispose()` 方法用于释放 uniform 和 sampler buffer。必须在不再使用时调用 `dispose()` 以避免 GPU 内存泄漏。

```dart
class _ShaderWidgetState extends State<ShaderWidget> {
  FragmentShader? _shader;

  @override
  void initState() {
    super.initState();
    _loadShader();
  }

  Future<void> _loadShader() async {
    final program = await FragmentProgram.fromAsset('shaders/effect.frag');
    if (!mounted) return;
    setState(() => _shader = program.fragmentShader());
  }

  @override
  void dispose() {
    // 释放 uniform/sampler buffer
    _shader?.dispose();
    _shader = null;
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_shader == null) {
      return const Center(child: CircularProgressIndicator());
    }
    return CustomPaint(painter: _ShaderPainter(_shader!));
  }
}
```

---

## 三、Uniform 参数传递

### 3.1 setFloat 索引规则

Uniform 参数通过 `setFloat(index, value)` 设置，索引从 0 开始，按 GLSL 中的声明顺序排列。

**GLSL 声明：**
```glsl
uniform float u_time;        // 索引 0
uniform vec2 u_resolution;   // 索引 1, 2
uniform vec3 u_color;        // 索引 3, 4, 5
uniform vec4 u_rect;         // 索引 6, 7, 8, 9
```

**Dart 设置：**
```dart
shader
  ..setFloat(0, time)           // u_time
  ..setFloat(1, size.width)     // u_resolution.x
  ..setFloat(2, size.height)    // u_resolution.y
  ..setFloat(3, 1.0)            // u_color.r
  ..setFloat(4, 0.5)            // u_color.g
  ..setFloat(5, 0.0)            // u_color.b
  ..setFloat(6, rect.left)      // u_rect.x
  ..setFloat(7, rect.top)       // u_rect.y
  ..setFloat(8, rect.width)     // u_rect.z
  ..setFloat(9, rect.height);   // u_rect.w
```

### 3.2 索引映射表

| GLSL 类型 | 占用 Slot 数 | setFloat 调用 |
|-----------|-------------|---------------|
| `float` | 1 | `setFloat(i, value)` |
| `vec2` | 2 | `setFloat(i, x)`, `setFloat(i+1, y)` |
| `vec3` | 3 | `setFloat(i, x)`, `setFloat(i+1, y)`, `setFloat(i+2, z)` |
| `vec4` | 4 | `setFloat(i, x)`, ..., `setFloat(i+3, w)` |
| `mat2` | 4 | 列主序，4 个 float |
| `mat3` | 9 | 列主序，9 个 float |
| `mat4` | 16 | 列主序，16 个 float |

### 3.3 批量设置 Uniform

```dart
import 'dart:typed_data';

// 使用 Float32List 批量设置
final uniforms = Float32List.fromList([
  size.width,   // 索引 0
  size.height,  // 索引 1
  time,         // 索引 2
  mouseX,       // 索引 3
  mouseY,       // 索引 4
]);

// 从索引 0 开始批量写入
for (var i = 0; i < uniforms.length; i++) {
  shader.setFloat(i, uniforms[i]);
}
```

### 3.4 setImageSampler 纹理采样

**GLSL 声明：**
```glsl
uniform sampler2D u_texture;   // 采样器索引 0
uniform sampler2D u_noise;     // 采样器索引 1
```

**Dart 设置：**
```dart
import 'dart:ui' as ui;
import 'package:flutter/services.dart';

// 加载图片
final imageBytes = await rootBundle.load('assets/texture.png');
final codec = await ui.instantiateImageCodec(imageBytes.buffer.asUint8List());
final frame = await codec.getNextFrame();
final ui.Image texture = frame.image;
codec.dispose(); // 释放解码器资源

// 绑定到 FragmentShader（直接传入 ui.Image）
shader.setImageSampler(0, texture);

// 在不需要时释放图片资源
// texture.dispose();
```

> ⚠️ **重要说明**：采样器索引（`setImageSampler`）独立于 `setFloat` 的浮点 Uniform 索引。它们同样按照 GLSL 声明顺序从 0 开始递增，但不会互相占用槽位。采样器数量上限取决于目标 GPU 和后端（Metal/Vulkan/GLES）的硬件限制。

> 💡 **性能提示**：`setImageSampler` 接受 `ui.Image` 参数，引擎内部会自动创建采样器。应缓存并复用 `ui.Image` 实例，在不再需要时调用 `ui.Image.dispose()` 释放 GPU 资源，避免显存泄漏。

**完整纹理采样示例：**

```glsl
// texture_effect.frag
#version 300 es
precision mediump float;
#include <flutter/runtime_effect.glsl>

uniform vec2 u_resolution;
uniform float u_time;
uniform sampler2D u_texture;

out vec4 fragColor;

void main() {
    vec2 uv = FlutterFragCoord().xy / u_resolution;

    // 添加波动效果
    vec2 distortedUV = uv + vec2(
        sin(uv.y * 10.0 + u_time) * 0.02,
        cos(uv.x * 10.0 + u_time) * 0.02
    );

    // 采样纹理
    vec4 texColor = texture(u_texture, distortedUV);

    fragColor = texColor;
}
```

```dart
// Dart 代码
class TextureShaderPainter extends CustomPainter {
  final FragmentShader shader;
  final ui.Image texture;
  final double time;

  TextureShaderPainter({
    required this.shader,
    required this.texture,
    required this.time,
  });

  @override
  void paint(Canvas canvas, Size size) {
    shader
      ..setFloat(0, size.width)
      ..setFloat(1, size.height)
      ..setFloat(2, time)
      ..setImageSampler(0, texture);

    canvas.drawRect(
      Offset.zero & size,
      Paint()..shader = shader,
    );
  }

  @override
  bool shouldRepaint(covariant TextureShaderPainter old) =>
      old.shader != shader || old.texture != texture || old.time != time;
}
```

---

## 四、与 CustomPainter 集成

### 4.1 基本模式

```dart
class ShaderPainter extends CustomPainter {
  final FragmentShader shader;
  final double time;
  final Color color;

  ShaderPainter({
    required this.shader,
    required this.time,
    required this.color,
  });

  @override
  void paint(Canvas canvas, Size size) {
    // 1. 设置所有 Uniform 参数
    shader
      ..setFloat(0, size.width)
      ..setFloat(1, size.height)
      ..setFloat(2, time)
      ..setFloat(3, color.red / 255.0)
      ..setFloat(4, color.green / 255.0)
      ..setFloat(5, color.blue / 255.0);

    // 2. 创建 Paint 并应用 Shader
    final paint = Paint()..shader = shader;

    // 3. 绘制
    canvas.drawRect(Offset.zero & size, paint);
  }

  @override
  bool shouldRepaint(covariant ShaderPainter oldDelegate) {
    return oldDelegate.shader != shader ||
           oldDelegate.time != time ||
           oldDelegate.color != color;
  }
}
```

### 4.2 完整示例：交互式 Shader

```dart
import 'dart:ui' as ui;
import 'package:flutter/material.dart';
import 'package:flutter/scheduler.dart';

class InteractiveShaderDemo extends StatefulWidget {
  const InteractiveShaderDemo({super.key});

  @override
  State<InteractiveShaderDemo> createState() => _InteractiveShaderDemoState();
}

class _InteractiveShaderDemoState extends State<InteractiveShaderDemo>
    with SingleTickerProviderStateMixin {
  ui.FragmentShader? _shader;
  late final Ticker _ticker;
  double _time = 0;
  Offset _mousePosition = Offset.zero;

  @override
  void initState() {
    super.initState();
    _loadShader();
    _ticker = createTicker((elapsed) {
      setState(() => _time = elapsed.inMilliseconds / 1000.0);
    })..start();
  }

  Future<void> _loadShader() async {
    final program = await ui.FragmentProgram.fromAsset('shaders/interactive.frag');
    if (mounted) {
      setState(() => _shader = program.fragmentShader());
    }
  }

  @override
  void dispose() {
    _ticker.dispose();
    _shader?.dispose();
    _shader = null;
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_shader == null) {
      return const Center(child: CircularProgressIndicator());
    }

    return MouseRegion(
      onHover: (event) {
        setState(() => _mousePosition = event.localPosition);
      },
      child: SizedBox.expand(
        child: CustomPaint(
          painter: InteractivePainter(
            shader: _shader!,
            time: _time,
            mousePosition: _mousePosition,
          ),
        ),
      ),
    );
  }
}

class InteractivePainter extends CustomPainter {
  final ui.FragmentShader shader;
  final double time;
  final Offset mousePosition;

  InteractivePainter({
    required this.shader,
    required this.time,
    required this.mousePosition,
  });

  @override
  void paint(Canvas canvas, Size size) {
    shader
      ..setFloat(0, size.width)       // u_resolution.x
      ..setFloat(1, size.height)      // u_resolution.y
      ..setFloat(2, time)             // u_time
      ..setFloat(3, mousePosition.dx) // u_mouse.x
      ..setFloat(4, mousePosition.dy);// u_mouse.y

    canvas.drawRect(
      Offset.zero & size,
      Paint()..shader = shader,
    );
  }

  @override
  bool shouldRepaint(covariant InteractivePainter old) =>
      old.shader != shader ||
      old.time != time ||
      old.mousePosition != mousePosition;
}
```

**配套 Shader（shaders/interactive.frag）：**

```glsl
#version 300 es
precision mediump float;
#include <flutter/runtime_effect.glsl>

uniform vec2 u_resolution;
uniform float u_time;
uniform vec2 u_mouse;

out vec4 fragColor;

void main() {
    vec2 uv = FlutterFragCoord().xy / u_resolution;
    vec2 mouse = u_mouse / u_resolution;

    // 计算到鼠标的距离
    float dist = distance(uv, mouse);

    // 创建涟漪效果
    float ripple = sin(dist * 30.0 - u_time * 5.0) * 0.5 + 0.5;
    ripple *= smoothstep(0.5, 0.0, dist);

    // 背景渐变
    vec3 bg = mix(
        vec3(0.1, 0.1, 0.2),
        vec3(0.2, 0.1, 0.3),
        uv.y
    );

    // 涟漪颜色
    vec3 rippleColor = vec3(0.3, 0.6, 1.0);

    vec3 finalColor = mix(bg, rippleColor, ripple);
    fragColor = vec4(finalColor, 1.0);
}
```

---

## 五、与 ShaderMask 集成

### 5.1 基本用法

`ShaderMask` 可以将 Shader 效果应用到子 Widget 上：

```dart
class ShaderMaskDemo extends StatelessWidget {
  final ui.FragmentShader shader;

  const ShaderMaskDemo({super.key, required this.shader});

  @override
  Widget build(BuildContext context) {
    return ShaderMask(
      blendMode: BlendMode.srcATop,
      shaderCallback: (Rect bounds) {
        // 设置 Uniform
        shader
          ..setFloat(0, bounds.width)
          ..setFloat(1, bounds.height);
        return shader;
      },
      child: Container(
        width: 200,
        height: 200,
        color: Colors.white,
        child: const Center(
          child: Text(
            'Shader',
            style: TextStyle(fontSize: 48, fontWeight: FontWeight.bold),
          ),
        ),
      ),
    );
  }
}
```

### 5.2 动画 ShaderMask

```dart
import 'dart:ui' as ui;
import 'package:flutter/material.dart';
import 'package:flutter/scheduler.dart';

class AnimatedShaderMask extends StatefulWidget {
  const AnimatedShaderMask({super.key});

  @override
  State<AnimatedShaderMask> createState() => _AnimatedShaderMaskState();
}

class _AnimatedShaderMaskState extends State<AnimatedShaderMask>
    with SingleTickerProviderStateMixin {
  ui.FragmentShader? _shader;
  late final Ticker _ticker;
  double _time = 0;

  @override
  void initState() {
    super.initState();
    _loadShader();
    _ticker = createTicker((elapsed) {
      setState(() => _time = elapsed.inMilliseconds / 1000.0);
    })..start();
  }

  Future<void> _loadShader() async {
    final program = await ui.FragmentProgram.fromAsset('shaders/rainbow.frag');
    if (mounted) {
      setState(() => _shader = program.fragmentShader());
    }
  }

  @override
  void dispose() {
    _ticker.dispose();
    _shader?.dispose();
    _shader = null;
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_shader == null) {
      return const SizedBox.shrink();
    }

    return ShaderMask(
      blendMode: BlendMode.srcIn,
      shaderCallback: (Rect bounds) {
        _shader!
          ..setFloat(0, bounds.width)
          ..setFloat(1, bounds.height)
          ..setFloat(2, _time);
        return _shader!;
      },
      child: const Text(
        'Rainbow Text',
        style: TextStyle(
          fontSize: 64,
          fontWeight: FontWeight.bold,
          color: Colors.white,
        ),
      ),
    );
  }
}
```

### 5.3 BlendMode 选择

| BlendMode | 效果 | 适用场景 |
|-----------|------|----------|
| `srcIn` | Shader 只显示在子 Widget 不透明区域 | 文字、图标着色 |
| `srcATop` | Shader 覆盖在子 Widget 上 | 叠加效果 |
| `dstIn` | 子 Widget 只显示在 Shader 不透明区域 | 遮罩裁剪 |
| `multiply` | 颜色相乘 | 暗化效果 |
| `screen` | 颜色滤色 | 亮化效果 |

---

## 六、常见错误与排查

### 6.1 资源加载错误

```plain
错误：Unable to load asset: shaders/effect.frag
```

**排查步骤：**
1. 检查 `pubspec.yaml` 配置
2. 确认文件路径正确
3. 运行 `flutter pub get`
4. 重启应用（热重载不会更新 Shader）

```yaml
# pubspec.yaml
flutter:
  shaders:
    - shaders/effect.frag
```

### 6.2 索引越界错误

```plain
错误：RangeError (index): Invalid value
```

**原因：** `setFloat` 索引与 GLSL 声明不匹配

```dart
// ❌ 错误：vec2 占用 2 个索引，但只设置了 1 个
shader.setFloat(0, size.width);
shader.setFloat(1, time);  // 应该是索引 2

// ✅ 正确
shader.setFloat(0, size.width);   // u_resolution.x
shader.setFloat(1, size.height);  // u_resolution.y
shader.setFloat(2, time);         // u_time
```

### 6.3 黑屏或透明

**可能原因：**
1. Uniform 未设置或设置错误
2. Shader 输出 alpha 为 0
3. 坐标计算错误

**调试方法：**
```glsl
// 在 Shader 中输出调试颜色
void main() {
    // 测试：输出红色确认 Shader 正在执行
    fragColor = vec4(1.0, 0.0, 0.0, 1.0);
    return;

    // 正常代码...
}
```

### 6.4 纹理显示异常

```dart
// ❌ 错误：在 Shader 仍在使用时释放纹理
ui.Image? _image;
ui.FragmentShader? _shader;

void dispose() {
  _image?.dispose();   // 如果 _shader 还在绘制中使用此图片，会导致渲染错误
  _shader?.dispose();
  super.dispose();
}

// ✅ 正确：先释放 Shader，再释放纹理
void dispose() {
  _ticker?.dispose();  // 先停止动画，避免继续触发绘制
  _shader?.dispose();  // 释放 FragmentShader（它引用了 _image）
  _image?.dispose();   // 最后释放纹理资源
  super.dispose();
}
```

> ⚠️ **资源释放顺序**：应先释放使用资源的对象（FragmentShader），再释放被引用的资源（ui.Image）。如果 Shader 仍在绘制过程中使用已释放的图片，会导致渲染错误或崩溃。

### 6.5 性能问题排查

```dart
// 使用 shouldRepaint 避免不必要的重绘
@override
bool shouldRepaint(covariant MyPainter oldDelegate) {
  // 比较所有依赖项，包括 shader 实例
  return oldDelegate.shader != shader ||
         oldDelegate.time != time ||
         oldDelegate.color != color;
}
```

---

## 七、API 速查表

### FragmentProgram

| 方法 | 说明 |
|------|------|
| `fromAsset(String path)` | 从 Asset 加载 Shader（唯一方式） |
| `fragmentShader()` | 创建 FragmentShader 实例 |

### FragmentShader

| 方法 | 说明 |
|------|------|
| `setFloat(int index, double value)` | 设置 float/vecN 参数 |
| `setImageSampler(int index, Image image)` | 设置纹理采样器 |
| `dispose()` | 释放 uniform/sampler buffer |

### 常用 Uniform 类型索引

```plain
uniform float a;      // 索引: 0
uniform vec2 b;       // 索引: 1, 2
uniform vec3 c;       // 索引: 3, 4, 5
uniform vec4 d;       // 索引: 6, 7, 8, 9
uniform sampler2D e;  // 采样器索引: 0 (独立计数)
uniform sampler2D f;  // 采样器索引: 1
```

---

## 八、下一步

在下一篇文档中，我们将学习：
- Shader 编译流程与工具链
- Impeller vs Skia 深度对比
- 性能分析与调试技巧
- 跨平台兼容性处理

---

_文档版本：1.1 | 最后更新：2025-01-06_
