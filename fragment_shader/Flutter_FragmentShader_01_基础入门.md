# Flutter FragmentShader 完全指南（一）：基础入门

> 本文档面向中级 Flutter 开发者，假设你已熟悉 CustomPainter 和基本的 Flutter 动画，但尚未接触过 Shader 编程。

> ⚠️ **版本要求**：本教程使用 Flutter 3.10+ 的推荐写法（`runtime_effect.glsl` 和 `FlutterFragCoord()`）。如果使用更早版本，请参考文档末尾的兼容性说明。

---

## 一、FragmentShader 概述

### 1.1 什么是 FragmentShader

FragmentShader（片段着色器）是运行在 GPU 上的小程序，负责计算屏幕上每个像素的最终颜色。与 CPU 逐像素串行计算不同，GPU 可以同时并行处理数百万个像素，这使得复杂的视觉效果能够实时运行。

```plain
传统 CPU 绘制：
  像素1 → 像素2 → 像素3 → ... → 像素N （串行）

GPU FragmentShader：
  像素1 ─┐
  像素2 ─┼─→ 同时计算 → 输出
  像素3 ─┤
  ...   ─┘
```

### 1.2 为什么在 Flutter 中使用 FragmentShader

| 场景 | CPU 绘制 | FragmentShader |
|------|----------|----------------|
| 简单图形 | 足够快 | 过度设计 |
| 复杂渐变 | 可能卡顿 | 流畅 |
| 实时滤镜 | 困难 | 轻松实现 |
| 动态噪声/粒子 | 几乎不可能 | 标准做法 |
| 后期处理效果 | 性能差 | 高效 |

### 1.3 典型应用场景

- **动态背景**：流动渐变、噪声纹理、星空效果
- **图片滤镜**：色彩调整、模糊、锐化、风格化
- **UI 特效**：玻璃态、霓虹发光、波纹动画
- **数据可视化**：热力图、等高线、流场
- **游戏效果**：光照、阴影、粒子系统

---

## 二、GPU 渲染管线基础

### 2.1 渲染管线概览

GPU 渲染管线是将 3D/2D 数据转换为屏幕像素的流水线：

```plain
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  顶点数据    │ → │ Vertex      │ → │  光栅化      │ → │ Fragment    │ → 屏幕
│  (Vertices) │    │ Shader      │    │ (Rasterize) │    │ Shader      │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                        ↑                                      ↑
                   处理顶点位置                            计算像素颜色
```

### 2.2 Vertex Shader vs Fragment Shader

| 特性 | Vertex Shader（顶点着色器） | Fragment Shader（片段着色器） |
|------|---------------------------|------------------------------|
| 执行时机 | 每个顶点执行一次 | 每个像素执行一次 |
| 主要职责 | 变换顶点位置 | 计算像素颜色 |
| 输入 | 顶点坐标、法线、UV | 插值后的顶点数据 |
| 输出 | 裁剪空间坐标 | RGBA 颜色值 |
| Flutter 支持 | 不直接支持 | 完全支持 |

> 📌 **注意**：Flutter 目前只支持自定义 Fragment Shader，Vertex Shader 由引擎内部处理。这对于 2D UI 效果已经足够。

### 2.3 Flutter 中的 Shader 执行流程

```plain
┌──────────────────────────────────────────────────────────────────┐
│                        Flutter 应用                              │
├──────────────────────────────────────────────────────────────────┤
│  1. Dart 代码加载 .frag 文件                                      │
│  2. 创建 FragmentShader 实例                                      │
│  3. 设置 Uniform 参数（尺寸、时间、颜色等）                         │
│  4. 将 Shader 应用到 Paint                                        │
│  5. 使用 Canvas 绘制图形                                          │
├──────────────────────────────────────────────────────────────────┤
│                        GPU 执行                                   │
├──────────────────────────────────────────────────────────────────┤
│  6. GPU 对每个像素并行执行 Fragment Shader                         │
│  7. 输出最终颜色到帧缓冲                                          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 三、坐标系统与颜色空间

### 3.1 Flutter Canvas 坐标系

Flutter 使用左上角为原点的坐标系：

```plain
(0,0) ────────────────→ X
  │
  │    Flutter Canvas
  │    坐标系
  │
  ↓
  Y
```

- 原点 `(0, 0)` 在左上角
- X 轴向右为正
- Y 轴向下为正
- 单位是逻辑像素（logical pixels）

### 3.2 Shader 中的坐标处理

在 Fragment Shader 中，Flutter 提供了 `FlutterFragCoord()` 函数来获取当前像素坐标：

```glsl
// FlutterFragCoord().xy 返回经过坐标系校正的像素坐标
// 它会自动处理不同渲染后端（Impeller/Skia）的坐标差异

// 归一化到 0-1 范围
vec2 uv = FlutterFragCoord().xy / u_resolution;

// 居中并保持宽高比
vec2 centered = (uv - 0.5) * vec2(u_resolution.x / u_resolution.y, 1.0);
```

> 💡 **推荐做法**：在 Shader 顶部添加 `#include <flutter/runtime_effect.glsl>` 后，使用 `FlutterFragCoord()` 替代原生的 `gl_FragCoord`。这样可以避免在不同渲染后端（Impeller、Skia）之间出现坐标系不一致的问题。
>
> ⚠️ **平台支持**：FragmentShader 目前支持 Android、iOS、macOS、Windows、Linux 平台，**Web 平台尚未支持**。

### 3.3 常用坐标变换

```glsl
// 1. 归一化坐标 (0 到 1)
vec2 uv = FlutterFragCoord().xy / u_resolution;

// 2. 居中坐标 (-0.5 到 0.5)
vec2 centered = uv - 0.5;

// 3. NDC 坐标 (-1 到 1)
vec2 ndc = uv * 2.0 - 1.0;

// 4. 保持宽高比的居中坐标
float aspect = u_resolution.x / u_resolution.y;
vec2 aspectCorrected = (uv - 0.5) * vec2(aspect, 1.0);
```

### 3.4 颜色空间

Fragment Shader 输出的颜色是 `vec4(r, g, b, a)`，每个分量范围 0.0 到 1.0：

```glsl
// 红色
vec4 red = vec4(1.0, 0.0, 0.0, 1.0);

// 半透明蓝色
vec4 semiBlue = vec4(0.0, 0.0, 1.0, 0.5);

// 从 Flutter Color 转换（假设 Color(0xFF3498DB)）
// R: 0x34 = 52  → 52/255 = 0.204
// G: 0x98 = 152 → 152/255 = 0.596
// B: 0xDB = 219 → 219/255 = 0.859
vec4 flutterBlue = vec4(0.204, 0.596, 0.859, 1.0);
```

> ⚠️ **警告**：Shader 中的颜色计算默认在线性空间进行，而屏幕显示使用 sRGB。对于精确的颜色混合，可能需要进行 gamma 校正：
> ```glsl
> // sRGB 转线性
> vec3 linear = pow(srgb, vec3(2.2));
> // 线性转 sRGB
> vec3 srgb = pow(linear, vec3(1.0/2.2));
> ```

---

## 四、GLSL 着色器语言入门

### 4.1 基本结构

Flutter 使用的 GLSL 版本为 OpenGL ES 3.0，推荐使用 Flutter 提供的运行时头文件：

```glsl
#include <flutter/runtime_effect.glsl>
precision mediump float;

// Uniform 变量（从 Dart 传入）
uniform vec2 u_resolution;
uniform float u_time;

// 输出颜色
out vec4 fragColor;

void main() {
    // 计算像素颜色
    fragColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```

> ✅ **最佳实践**：`#include <flutter/runtime_effect.glsl>` 统一了 Impeller 和 Skia 后端的 Shader 运行时，提供了 `FlutterFragCoord()`、`FlutterMain()` 等辅助函数。强烈推荐在所有 Flutter Shader 顶部包含它。Web 平台目前不支持 FragmentShader。

> 📌 **兼容性说明**：如果需要兼容 Flutter 3.10 之前的版本，可以使用 `#version 300 es` 替代 include，但需要手动处理坐标系差异。

### 4.2 数据类型

| 类型 | 说明 | 示例 |
|------|------|------|
| `float` | 浮点数 | `float x = 1.0;` |
| `int` | 整数 | `int i = 42;` |
| `bool` | 布尔值 | `bool flag = true;` |
| `vec2` | 2D 向量 | `vec2 pos = vec2(1.0, 2.0);` |
| `vec3` | 3D 向量/RGB | `vec3 color = vec3(1.0, 0.5, 0.0);` |
| `vec4` | 4D 向量/RGBA | `vec4 rgba = vec4(1.0, 0.5, 0.0, 1.0);` |
| `mat2/3/4` | 矩阵 | `mat2 m = mat2(1.0);` |
| `sampler2D` | 2D 纹理采样器 | `uniform sampler2D u_texture;` |

### 4.3 向量操作（Swizzling）

GLSL 支持灵活的向量分量访问：

```glsl
vec4 color = vec4(1.0, 0.5, 0.3, 1.0);

// 访问单个分量
float r = color.r;  // 或 color.x 或 color[0]
float g = color.g;  // 或 color.y 或 color[1]

// 重组分量
vec3 rgb = color.rgb;
vec2 rg = color.rg;
vec3 bgr = color.bgr;  // 反转顺序
vec4 rrgg = color.rrgg;  // 重复分量
```

### 4.4 常用内置函数

```glsl
// 数学函数
float a = sin(x);           // 正弦
float b = cos(x);           // 余弦
float c = pow(x, 2.0);      // 幂运算
float sqrtValue = sqrt(x);  // 平方根
float e = abs(x);           // 绝对值
float f = floor(x);         // 向下取整
float g = fract(x);         // 小数部分
float h = mod(x, y);        // 取模

// 插值函数
float i = mix(a, b, t);     // 线性插值: a*(1-t) + b*t
float j = smoothstep(0.0, 1.0, x);  // 平滑插值
float k = step(edge, x);    // x < edge ? 0.0 : 1.0
float l = clamp(x, 0.0, 1.0);  // 限制范围

// 向量函数
vec2 va = vec2(1.0, 2.0);
vec2 vb = vec2(3.0, 4.0);
float len = length(va);      // 向量长度
vec2 n = normalize(va);      // 归一化
float dotValue = dot(va, vb); // 点积
float dist = distance(va, vb); // 两点距离
```

### 4.5 控制流

```glsl
// 条件语句
if (x > 0.5) {
    color = vec4(1.0, 0.0, 0.0, 1.0);
} else {
    color = vec4(0.0, 0.0, 1.0, 1.0);
}

// 循环（谨慎使用，影响性能）
for (int i = 0; i < 10; i++) {
    // ...
}

// 三元运算符
float result = condition ? valueA : valueB;
```

> ⚠️ **性能提示**：在 Shader 中应尽量避免分支（if/else）和循环，因为 GPU 的并行架构对分支不友好。优先使用 `mix`、`step`、`smoothstep` 等函数实现条件逻辑。

---

## 五、Flutter 中的第一个 Shader

### 5.1 项目结构

```
my_shader_app/
├── lib/
│   └── main.dart
├── shaders/
│   └── gradient.frag
└── pubspec.yaml
```

### 5.2 配置 pubspec.yaml

```yaml
flutter:
  shaders:
    - shaders/gradient.frag
```

### 5.3 编写 Shader 文件

创建 `shaders/gradient.frag`：

```glsl
#include <flutter/runtime_effect.glsl>
precision mediump float;

// Uniform 参数（从 Dart 传入）
uniform vec2 u_resolution;  // 画布尺寸
uniform float u_time;       // 动画时间

// 输出颜色
out vec4 fragColor;

void main() {
    // 1. 计算归一化坐标 (0-1)
    // 使用 FlutterFragCoord() 获取经过坐标系校正的像素坐标
    vec2 uv = FlutterFragCoord().xy / u_resolution;

    // 2. 创建动态渐变
    // 使用时间让颜色流动
    float wave = sin(uv.x * 6.28 + u_time) * 0.5 + 0.5;

    // 3. 混合两种颜色
    vec3 color1 = vec3(0.1, 0.4, 0.8);  // 蓝色
    vec3 color2 = vec3(0.9, 0.3, 0.5);  // 粉色
    vec3 finalColor = mix(color1, color2, wave * uv.y);

    // 4. 输出最终颜色
    fragColor = vec4(finalColor, 1.0);
}
```

### 5.4 Dart 代码实现

创建 `lib/main.dart`：

```dart
import 'dart:ui' as ui;
import 'package:flutter/material.dart';
import 'package:flutter/scheduler.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Shader Demo',
      theme: ThemeData.dark(),
      home: const ShaderDemo(),
    );
  }
}

class ShaderDemo extends StatefulWidget {
  const ShaderDemo({super.key});

  @override
  State<ShaderDemo> createState() => _ShaderDemoState();
}

class _ShaderDemoState extends State<ShaderDemo>
    with SingleTickerProviderStateMixin {
  // Shader 相关
  ui.FragmentShader? _shader;
  bool _isLoading = true;
  String? _error;

  // 动画相关
  late final Ticker _ticker;
  double _time = 0.0;

  @override
  void initState() {
    super.initState();
    _loadShader();
    _startAnimation();
  }

  /// 加载 Shader
  Future<void> _loadShader() async {
    try {
      final program = await ui.FragmentProgram.fromAsset('shaders/gradient.frag');
      if (!mounted) return;
      setState(() {
        _shader = program.fragmentShader();
        _isLoading = false;
      });
    } catch (e) {
      if (!mounted) return;
      setState(() {
        _error = e.toString();
        _isLoading = false;
      });
    }
  }

  /// 启动动画
  void _startAnimation() {
    _ticker = createTicker((elapsed) {
      setState(() {
        _time = elapsed.inMilliseconds / 1000.0;
      });
    })..start();
  }

  @override
  void dispose() {
    _ticker.dispose();
    _shader?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('FragmentShader Demo'),
      ),
      body: Center(
        child: _buildContent(),
      ),
    );
  }

  Widget _buildContent() {
    if (_isLoading) {
      return const CircularProgressIndicator();
    }

    if (_error != null) {
      return Text(
        'Error: $_error',
        style: const TextStyle(color: Colors.red),
      );
    }

    return ClipRRect(
      borderRadius: BorderRadius.circular(16),
      child: CustomPaint(
        painter: GradientShaderPainter(
          shader: _shader!,
          time: _time,
        ),
        size: const Size(300, 300),
      ),
    );
  }
}

/// 自定义 Painter，使用 Shader 绘制
class GradientShaderPainter extends CustomPainter {
  GradientShaderPainter({
    required this.shader,
    required this.time,
  });

  final ui.FragmentShader shader;
  final double time;

  @override
  void paint(Canvas canvas, Size size) {
    // 设置 Uniform 参数
    // 注意：参数顺序必须与 .frag 文件中的声明顺序一致
    shader
      ..setFloat(0, size.width)   // u_resolution.x
      ..setFloat(1, size.height)  // u_resolution.y
      ..setFloat(2, time);        // u_time

    // 创建 Paint 并应用 Shader
    final paint = Paint()..shader = shader;

    // 绘制矩形
    canvas.drawRect(
      Rect.fromLTWH(0, 0, size.width, size.height),
      paint,
    );
  }

  @override
  bool shouldRepaint(covariant GradientShaderPainter oldDelegate) {
    // 时间变化时需要重绘
    return oldDelegate.time != time;
  }
}
```

### 5.5 运行效果

运行应用后，你将看到一个 300x300 的区域，显示蓝色到粉色的动态渐变效果，颜色会随时间流动变化。

### 5.6 代码解析

**Shader 文件关键点：**

1. `#include <flutter/runtime_effect.glsl>` - 引入 Flutter 运行时代码，获得 `FlutterFragCoord()` 等工具（若仍需 `#version 300 es`，确保它位于文件最前）
2. `precision mediump float` - 设置浮点精度（移动端必需）
3. `uniform` - 声明从 Dart 传入的参数
4. `FlutterFragCoord()` - 获取经过 Flutter 坐标系校正的像素坐标
5. `fragColor` - 输出变量，最终像素颜色

**Dart 代码关键点：**

1. `FragmentProgram.fromAsset()` - 异步加载 Shader
2. `program.fragmentShader()` - 创建 Shader 实例
3. `shader.setFloat(index, value)` - 设置 Uniform 参数
4. `Paint()..shader = shader` - 将 Shader 应用到画笔
5. `Ticker` - 驱动动画，每帧更新时间

> ⚠️ **重要**：`setFloat` 的索引必须与 Shader 中 uniform 声明的顺序一致。`u_resolution` 是 `vec2`，占用索引 0 和 1；`u_time` 是 `float`，占用索引 2。

---

## 六、常见问题与排查

### 6.1 Shader 加载失败

```dart
// 错误：Shader asset not found
// 解决：检查 pubspec.yaml 配置和文件路径
flutter:
  shaders:
    - shaders/gradient.frag  # 确保路径正确
```

### 6.2 编译错误

```plain
// 错误：Shader compilation failed
// 常见原因：
// 1. GLSL 语法错误
// 2. 版本声明缺失或错误
// 3. 精度声明缺失（移动端）
```

### 6.3 显示黑屏或异常颜色

```dart
// 检查 Uniform 参数顺序
shader
  ..setFloat(0, size.width)   // 索引 0
  ..setFloat(1, size.height)  // 索引 1
  ..setFloat(2, time);        // 索引 2

// 确保与 Shader 中的声明顺序一致：
// uniform vec2 u_resolution;  // 占用 0, 1
// uniform float u_time;       // 占用 2
```

### 6.4 性能问题

```glsl
// ❌ 避免在 Shader 中使用复杂循环
for (int i = 0; i < 1000; i++) { ... }

// ✅ 优化：减少迭代次数或使用数学公式替代
```

---

## 七、练习题

1. **修改颜色**：将渐变颜色改为绿色到黄色
2. **改变方向**：让渐变从左到右变为从上到下
3. **添加参数**：从 Dart 传入自定义颜色值
4. **圆形渐变**：实现从中心向外的径向渐变

---

## 八、下一步

在下一篇文档中，我们将深入学习：
- FragmentProgram 与 FragmentShader API 详解
- Uniform 参数的高级用法
- 纹理采样（sampler2D）
- 与 ShaderMask 的集成

---

## 九、版本兼容性说明

### Flutter 3.10+ (推荐)

本教程使用的写法适用于 Flutter 3.10 及更高版本：

```glsl
#include <flutter/runtime_effect.glsl>
precision mediump float;

uniform vec2 u_resolution;
out vec4 fragColor;

void main() {
    vec2 uv = FlutterFragCoord().xy / u_resolution;
    fragColor = vec4(uv, 0.0, 1.0);
}
```

### Flutter 3.10 之前的版本

如果需要兼容更早版本，使用以下写法：

```glsl
#version 300 es
precision mediump float;

uniform vec2 u_resolution;
out vec4 fragColor;

void main() {
    vec2 uv = gl_FragCoord.xy / u_resolution;
    // 注意：可能需要手动处理 Y 轴翻转
    fragColor = vec4(uv, 0.0, 1.0);
}
```

> ⚠️ **注意**：使用 `gl_FragCoord` 时，在 Impeller 和 Skia 渲染后端之间可能会出现坐标系方向不一致的问题。建议升级到 Flutter 3.10+ 以使用统一的 `FlutterFragCoord()` API。

---

_文档版本：1.1 | 最后更新：2025-01-06_
