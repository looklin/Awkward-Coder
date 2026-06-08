---
title: C# SkiaSharp 实战指南——从入门到高阶图形渲染
slug: csharp-skiasharp-practice
description: >-
  全面掌握 SkiaSharp —— Google Skia 图形引擎的 .NET 绑定库。从基础绘图、文本渲染、图像处理到高性能渲染优化，覆盖 Avalonia、MAUI、Blazor 等多平台集成场景。
tags:
  - technical
added: "June 08 2026"
---

# C# SkiaSharp 实战指南——从入门到高阶图形渲染

> 想跨平台画 2D 图形？SkiaSharp 是目前 .NET 生态中最成熟、性能最强的选择。Chrome、Firefox、Flutter、Android 的图形引擎都是它。本文从原理到实战，带你彻底掌握。

---

## 一、Skia 是什么？为什么选 SkiaSharp？

### 1.1 Skia 的身世

**Skia** 是 Google 开源的 2D 图形库，用 C++ 编写。它是：

- **Chrome / Chromium** 的渲染引擎
- **Android** 的系统图形库
- **Flutter** 的底层绘制引擎
- **Firefox** 的 Canvas 后端之一
- **Fuchsia OS** 的官方图形栈

简单说：**全球几十亿台设备每天都在用 Skia 渲染图形**。

### 1.2 SkiaSharp 是什么？

**SkiaSharp** 是 Mono 团队维护的 Skia .NET 绑定库。它通过 P/Invoke 调用原生 Skia 库，让你在 C# 中用面向对象的 API 进行 2D 图形绘制。

```
┌──────────────────────────────────────┐
│         你的 C# 代码                   │
│  using var canvas = new SKCanvas(...) │
│  canvas.DrawCircle(...)              │
├──────────────────────────────────────┤
│         SkiaSharp (C# 绑定)           │
│  P/Invoke → 原生 Skia 库              │
├──────────────────────────────────────┤
│     Skia (C++) + 平台后端              │
│  Windows: DirectWrite/GDI             │
│  macOS:   CoreGraphics                │
│  Linux:   FreeType/X11/Wayland        │
│  Android: AndroidBitmap/FontMgr       │
└──────────────────────────────────────┘
```

### 1.3 为什么不用 GDI+ / System.Drawing？

| 特性 | System.Drawing | SkiaSharp |
|------|---------------|-----------|
| **跨平台** | ❌ Windows only (.NET Framework)，Linux 需要 libgdiplus（功能残缺） | ✅ 原生支持 Windows / macOS / Linux / iOS / Android |
| **维护状态** | ⚠️ 官方标记为 "Windows only"，不推荐在新项目中使用 | ✅ 活跃维护（.NET Foundation） |
| **抗锯齿** | 基础 | 优秀（GPU 加速可用） |
| **文本渲染** | GDI 文本，质量一般 | 支持 FreeType / DirectWrite，质量高 |
| **图像格式** | 有限 | PNG / JPEG / WebP / BMP / GIF / ICO / KTX 等 |
| **性能** | 一般 | 优秀（可选 GPU 加速） |
| **生态集成** | 老旧 | Avalonia / MAUI / Blazor / Eto 等现代 UI 框架首选 |

> 微软官方建议：新项目应使用 SkiaSharp、ImageSharp 或 MAUI Graphics，而非 System.Drawing。

---

## 二、快速上手

### 2.1 安装

```bash
dotnet add package SkiaSharp
dotnet add package SkiaSharp.NativeAssets.Linux  # Linux 环境需要这个
```

### 2.2 Hello Triangle

```csharp
using SkiaSharp;

// 创建一个 400x300 的位图
using var bitmap = new SKBitmap(400, 300);
using var canvas = new SKCanvas(bitmap);

// 清屏为白色
canvas.Clear(SKColors.White);

// 创建画笔
using var paint = new SKPaint
{
    Color = SKColors.Blue,
    IsAntialias = true,  // 开启抗锯齿
    StrokeWidth = 4,
    Style = SKPaintStyle.Stroke  // 描边模式（Fill 为填充）
};

// 画一个三角形
var path = new SKPath();
path.MoveTo(200, 50);   // 顶点
path.LineTo(100, 250);  // 左下角
path.LineTo(300, 250);  // 右下角
path.Close();

canvas.DrawPath(path, paint);

// 保存为 PNG
using var image = SKImage.FromBitmap(bitmap);
using var data = image.Encode(SKEncodedImageFormat.Png, 100);
using var stream = File.OpenWrite("triangle.png");
data.SaveTo(stream);
```

### 2.3 核心概念速查

| 概念 | 说明 | 类比 |
|------|------|------|
| `SKCanvas` | 画布，所有绘制操作的入口 | HTML Canvas / GDI Graphics |
| `SKBitmap` | 位图（像素级操作） | BufferedImage |
| `SKImage` | 不可变图像（推荐用于渲染） | — |
| `SKPaint` | 画笔（颜色、粗细、字体等） | Pen + Brush |
| `SKPath` | 路径（任意形状） | GraphicsPath |
| `SKSurface` | 渲染表面（高级封装） | — |
| `SKShader` | 着色器（渐变、图案填充） | LinearGradientBrush |

---

## 三、基础绘图 API

### 3.1 基本形状

```csharp
void DrawShapes(SKCanvas canvas)
{
    using var paint = new SKPaint { IsAntialias = true };

    // 矩形
    paint.Color = SKColors.Red;
    canvas.DrawRect(10, 10, 100, 60, paint);

    // 圆角矩形
    paint.Color = SKColors.Green;
    canvas.DrawRoundRect(130, 10, 100, 60, 15, 15, paint);

    // 圆形
    paint.Color = SKColors.Blue;
    canvas.DrawCircle(300, 40, 30, paint);

    // 椭圆
    paint.Color = SKColors.Orange;
    canvas.DrawOval(new SKRect(10, 90, 120, 150), paint);

    // 圆角（胶囊形状）
    paint.Color = SKColors.Purple;
    canvas.DrawRoundRect(140, 90, 100, 60, 30, 30, paint);

    // 线段
    paint.Color = SKColors.Black;
    paint.StrokeWidth = 3;
    paint.Style = SKPaintStyle.Stroke;
    canvas.DrawLine(260, 90, 360, 150, paint);
}
```

### 3.2 路径（SKPath）—— 万能形状

```csharp
void DrawStar(SKCanvas canvas, float cx, float cy, float radius)
{
    using var paint = new SKPaint
    {
        Color = SKColors.Gold,
        IsAntialias = true,
        Style = SKPaintStyle.Fill
    };

    var path = new SKPath();
    int points = 5;
    float innerRadius = radius * 0.4f;

    for (int i = 0; i < points * 2; i++)
    {
        float r = (i % 2 == 0) ? radius : innerRadius;
        float angle = (float)(i * Math.PI / points - Math.PI / 2);
        float x = cx + r * (float)Math.Cos(angle);
        float y = cy + r * (float)Math.Sin(angle);

        if (i == 0) path.MoveTo(x, y);
        else path.LineTo(x, y);
    }
    path.Close();

    canvas.DrawPath(path, paint);
}
```

### 3.3 贝塞尔曲线

```csharp
void DrawCurves(SKCanvas canvas)
{
    using var paint = new SKPaint
    {
        Color = SKColors.Teal,
        IsAntialias = true,
        StrokeWidth = 3,
        Style = SKPaintStyle.Stroke
    };

    // 二次贝塞尔曲线
    var path1 = new SKPath();
    path1.MoveTo(20, 50);
    path1.QuadTo(100, 120, 180, 50);  // 一个控制点
    canvas.DrawPath(path1, paint);

    // 三次贝塞尔曲线
    var path2 = new SKPath();
    path2.MoveTo(20, 120);
    path2.CubicTo(60, 30, 140, 180, 180, 120);  // 两个控制点
    canvas.DrawPath(path2, paint);

    // 弧线
    paint.Color = SKColors.Crimson;
    canvas.DrawArc(
        new SKRect(20, 150, 180, 280),
        startAngle: 0,
        sweepAngle: 270,
        useCenter: true,
        paint
    );
}
```

---

## 四、渐变与着色器

### 4.1 线性渐变

```csharp
void DrawLinearGradient(SKCanvas canvas)
{
    using var paint = new SKPaint();
    paint.Shader = SKShader.CreateLinearGradient(
        new SKPoint(0, 0),
        new SKPoint(300, 200),
        new[] { SKColors.Red, SKColors.Yellow, SKColors.Blue },
        null,  // 颜色分布（null = 均匀分布）
        SKShaderTileMode.Clamp
    );

    canvas.DrawRect(0, 0, 300, 200, paint);
}
```

### 4.2 径向渐变

```csharp
void DrawRadialGradient(SKCanvas canvas)
{
    using var paint = new SKPaint();
    paint.Shader = SKShader.CreateRadialGradient(
        new SKPoint(150, 100),  // 中心
        120,                     // 半径
        new[]
        {
            SKColors.White.WithAlpha(255),
            SKColors.White.WithAlpha(0)
        },
        new[] { 0f, 1f },
        SKShaderTileMode.Clamp
    );

    canvas.DrawRect(0, 0, 300, 200, paint);
}
```

### 4.3 混合着色器

```csharp
void DrawMixedShader(SKCanvas canvas)
{
    using var paint = new SKPaint();

    using var linear = SKShader.CreateLinearGradient(
        SKPoint.Empty, new SKPoint(200, 0),
        new[] { SKColors.Red, SKColors.Blue }, null, SKShaderTileMode.Clamp
    );
    using var radial = SKShader.CreateRadialGradient(
        new SKPoint(100, 100), 80,
        new[] { SKColors.Yellow.WithAlpha(128), SKColors.Transparent },
        null, SKShaderTileMode.Clamp
    );

    // 混合两种渐变
    paint.Shader = SKShader.CreateCompose(linear, radial);
    canvas.DrawRect(0, 0, 200, 200, paint);
}
```

---

## 五、文本渲染

### 5.1 基础文本

```csharp
void DrawText(SKCanvas canvas)
{
    using var paint = new SKPaint
    {
        IsAntialias = true,
        TextSize = 24,
        Color = SKColors.Black,
        Typeface = SKTypeface.Default
    };

    // 简单绘制
    canvas.DrawText("Hello SkiaSharp!", 10, 40, paint);

    // 精确对齐绘制
    canvas.DrawText("居中文本", 150, 80, SKTextAlign.Center, paint);

    // 多行文本
    canvas.DrawText("第一行", 10, 120, paint);
    canvas.DrawText("第二行", 10, 150, paint);
}
```

### 5.2 加载自定义字体

```csharp
void DrawCustomFont(SKCanvas canvas)
{
    // 从文件加载
    using var typeface = SKTypeface.FromFile("./fonts/NotoSansSC-Regular.ttf");

    using var paint = new SKPaint
    {
        IsAntialias = true,
        TextSize = 32,
        Color = SKColors.Black,
        Typeface = typeface
    };

    canvas.DrawText("自定义字体渲染", 10, 50, paint);
}
```

### 5.3 文本布局（高级）

```csharp
void DrawRichText(SKCanvas canvas, string text, float maxWidth)
{
    using var paint = new SKPaint
    {
        IsAntialias = true,
        TextSize = 16,
        Color = SKColors.Black,
        Typeface = SKTypeface.Default
    };

    // 简单自动换行
    float y = 20;
    float lineHeight = paint.FontMetrics.Descent - paint.FontMetrics.Ascent;
    string[] words = text.Split(' ');
    string line = "";

    foreach (var word in words)
    {
        string testLine = string.IsNullOrEmpty(line) ? word : line + " " + word;
        float width = paint.MeasureText(testLine);

        if (width > maxWidth && !string.IsNullOrEmpty(line))
        {
            canvas.DrawText(line, 10, y, paint);
            y += lineHeight;
            line = word;
        }
        else
        {
            line = testLine;
        }
    }
    if (!string.IsNullOrEmpty(line))
    {
        canvas.DrawText(line, 10, y, paint);
    }
}
```

> 如果需要更复杂的文本布局（HTML 风格富文本），可以考虑 `HarfBuzzSharp`（SkiaSharp 的同源项目），或者使用 `Topten.RichTextKit` 库。

---

## 六、图像处理

### 6.1 加载和保存

```csharp
// 加载图像
using var stream = File.OpenRead("input.jpg");
using var bitmap = SKBitmap.Decode(stream);

// 缩放
using var resized = bitmap.Resize(
    new SKImageInfo(200, 200),
    SKFilterQuality.High  // Low / Medium / High
);

// 保存
using var image = SKImage.FromBitmap(resized);
using var data = image.Encode(SKEncodedImageFormat.WebP, 85);
using var output = File.OpenWrite("output.webp");
data.SaveTo(output);
```

### 6.2 像素级操作

```csharp
void InvertColors(SKBitmap bitmap)
{
    // 获取像素数据的直接访问
    var pixels = bitmap.Pixels;

    for (int y = 0; y < bitmap.Height; y++)
    {
        for (int x = 0; x < bitmap.Width; x++)
        {
            var color = bitmap.GetPixel(x, y);
            var inverted = new SKColor(
                (byte)(255 - color.Red),
                (byte)(255 - color.Green),
                (byte)(255 - color.Blue),
                color.Alpha
            );
            bitmap.SetPixel(x, y, inverted);
        }
    }
}
```

### 6.3 高性能像素操作

```csharp
// GetPixel / SetPixel 很慢！大量像素操作应该用 unsafe 指针
void InvertColorsFast(SKBitmap bitmap)
{
    bitmap.LockPixels();
    var ptr = bitmap.GetPixels();
    int length = bitmap.Width * bitmap.Height;

    unsafe
    {
        byte* pixels = (byte*)ptr;
        for (int i = 0; i < length; i++)
        {
            int offset = i * 4;  // BGRA 格式
            pixels[offset]     = (byte)(255 - pixels[offset]);     // B
            pixels[offset + 1] = (byte)(255 - pixels[offset + 1]); // G
            pixels[offset + 2] = (byte)(255 - pixels[offset + 2]); // R
            // offset + 3 是 Alpha，不变
        }
    }
    bitmap.UnlockPixels();
}
```

> **性能对比：** 对 1920×1080 的图像做反色处理，`GetPixel/SetPixel` 需要约 2 秒，指针操作只需约 5 毫秒——**400 倍差距**。

### 6.4 图像滤镜（ImageFilter）

```csharp
SKImage ApplyBlur(SKImage image)
{
    using var paint = new SKPaint();
    paint.ImageFilter = SKImageFilter.CreateBlur(5, 5);

    using var surface = SKSurface.Create(new SKImageInfo(image.Width, image.Height));
    using var canvas = surface.Canvas;
    canvas.DrawImage(image, 0, 0, paint);

    return surface.Snapshot();
}

SKImage ApplyDropShadow(SKImage image)
{
    using var paint = new SKPaint();
    paint.ImageFilter = SKImageFilter.CreateDropShadow(
        dx: 5, dy: 5,
        sigmaX: 3, sigmaY: 3,
        color: SKColors.Black.WithAlpha(128)
    );

    using var surface = SKSurface.Create(new SKImageInfo(image.Width, image.Height));
    using var canvas = surface.Canvas;
    canvas.DrawImage(image, 0, 0, paint);

    return surface.Snapshot();
}
```

### 6.5 常用滤镜速查

```csharp
// 高斯模糊
SKImageFilter.CreateBlur(sigmaX, sigmaY)

// 颜色矩阵（调色、灰度、反色等）
SKImageFilter.CreateColorMatrix(float[] matrix)

// 位移
SKImageFilter.CreateOffset(dx, dy)

// 混合模式
SKImageFilter.CreateBlendMode(SKBlendMode.SrcOver, foregroundFilter, backgroundFilter)

// 组合
SKImageFilter.CreateCompose(outer, inner)
```

---

## 七、变换与坐标系

### 7.1 变换矩阵

```csharp
void DrawTransformed(SKCanvas canvas)
{
    using var paint = new SKPaint { Color = SKColors.Blue, IsAntialias = true };

    // 原始矩形
    canvas.DrawRect(0, 0, 50, 50, paint);

    // 保存当前状态
    canvas.Save();

    // 平移
    canvas.Translate(100, 0);
    paint.Color = SKColors.Red;
    canvas.DrawRect(0, 0, 50, 50, paint);

    // 旋转（在已平移的基础上再旋转 45 度）
    canvas.RotateDegrees(45, 25, 25);
    paint.Color = SKColors.Green;
    canvas.DrawRect(0, 0, 50, 50, paint);

    // 恢复
    canvas.Restore();

    // 缩放
    canvas.Save();
    canvas.Translate(250, 0);
    canvas.Scale(1.5f, 1.5f);
    paint.Color = SKColors.Orange;
    canvas.DrawRect(0, 0, 50, 50, paint);
    canvas.Restore();
}
```

### 7.2 裁剪区域

```csharp
void DrawClipped(SKCanvas canvas)
{
    using var paint = new SKPaint { IsAntialias = true };

    // 保存状态
    canvas.Save();

    // 设置圆形裁剪区域
    var clipPath = new SKPath();
    clipPath.AddCircle(150, 150, 100);
    canvas.ClipPath(clipPath);

    // 在裁剪区域内绘制 —— 超出部分被裁掉
    paint.Color = SKColors.Red;
    canvas.DrawRect(50, 50, 200, 200, paint);

    canvas.Restore();
}
```

---

## 八、多平台集成

### 8.1 Avalonia

```xml
<!-- 安装 Avalonia.Skia 包后自动支持，无需额外配置 -->
<!-- Avalonia 默认后端就是 SkiaSharp -->
```

Avalonia 的 `DrawingContext` 底层就是 Skia，你也可以直接用 `SKElement` 嵌入原生 SkiaSharp 画布：

```xml
<skia:SKElement x:Name="skiaCanvas"
                PaintSurface="OnPaintSurface" />
```

```csharp
void OnPaintSurface(object sender, SKPaintSurfaceEventArgs e)
{
    var canvas = e.Surface.Canvas;
    canvas.Clear(SKColors.White);

    using var paint = new SKPaint
    {
        Color = SKColors.Blue,
        TextSize = 24,
        IsAntialias = true
    };
    canvas.DrawText("Hello from SkiaSharp in Avalonia!", 20, 40, paint);
}
```

### 8.2 .NET MAUI

```xml
<!-- 安装 SkiaSharp.Views.Maui.Controls -->
<skia:SKCanvasView x:Name="canvasView"
                   PaintSurface="OnCanvasViewPaintSurface" />
```

```csharp
void OnCanvasViewPaintSurface(object sender, SKPaintSurfaceEventArgs e)
{
    var info = e.Info;
    var canvas = e.Surface.Canvas;

    canvas.Clear(SKColors.White);

    // 绘制自适应内容
    using var paint = new SKPaint
    {
        Color = SKColors.Blue,
        IsAntialias = true,
        Style = SKPaintStyle.Fill
    };

    float radius = Math.Min(info.Width, info.Height) * 0.3f;
    canvas.DrawCircle(info.Width / 2, info.Height / 2, radius, paint);
}
```

### 8.3 Blazor Server / WebAssembly

```razor
@using SkiaSharp.Views.Blazor

<SKCanvasView @ref="canvasView"
              Width="400" Height="300"
              PaintSurface="OnPaintSurface" />

@code {
    SKCanvasView canvasView;

    void OnPaintSurface(object sender, SKPaintSurfaceEventArgs e)
    {
        var canvas = e.Surface.Canvas;
        canvas.Clear(SKColors.White);

        using var paint = new SKPaint
        {
            Color = SKColors.Green,
            IsAntialias = true,
            TextSize = 28
        };
        canvas.DrawText("Hello Blazor + SkiaSharp!", 20, 50, paint);
    }
}
```

### 8.4 控制台 / 服务端（无头渲染）

SkiaSharp 完全可以在无 GUI 的服务器端运行，非常适合：

- 生成图表 / 报表图片
- 图片处理和缩略图
- 验证码生成
- 数据可视化导出

```csharp
// 纯服务端生成统计图
using var bitmap = new SKBitmap(800, 400);
using var canvas = new SKCanvas(bitmap);
canvas.Clear(SKColors.White);

// ... 绘制图表 ...

using var image = SKImage.FromBitmap(bitmap);
using var data = image.Encode(SKEncodedImageFormat.Png, 100);

// 返回给前端
return File(data.ToArray(), "image/png");
```

---

## 九、性能优化

### 9.1 对象复用

```csharp
// ❌ 每次绘制都创建新对象 —— 频繁 GC
void OnPaint(SKCanvas canvas)
{
    for (int i = 0; i < 1000; i++)
    {
        using var paint = new SKPaint { Color = GetRandomColor() };
        canvas.DrawCircle(GetRandomX(), GetRandomY(), 5, paint);
    }
}

// ✅ 复用 Paint 对象
private readonly SKPaint _circlePaint = new SKPaint { IsAntialias = true };

void OnPaintOptimized(SKCanvas canvas)
{
    for (int i = 0; i < 1000; i++)
    {
        _circlePaint.Color = GetRandomColor();
        canvas.DrawCircle(GetRandomX(), GetRandomY(), 5, _circlePaint);
    }
}
```

### 9.2 图像缓存

```csharp
// 预渲染复杂内容到 SKImage，之后直接 DrawImage
private SKImage? _cachedImage;

void OnPaint(SKCanvas canvas, SKImageInfo info)
{
    if (_cachedImage == null)
    {
        // 离屏渲染
        using var surface = SKSurface.Create(info);
        var offscreenCanvas = surface.Canvas;

        // ... 绘制复杂内容 ...
        DrawComplexScene(offscreenCanvas);

        _cachedImage = surface.Snapshot();
    }

    // 直接绘制缓存图像 —— 极快
    canvas.DrawImage(_cachedImage, 0, 0);
}
```

### 9.3 按需渲染（脏区域）

```csharp
private bool _isDirty = true;
private SKImage? _renderedImage;

void OnPaint(SKCanvas canvas, SKImageInfo info)
{
    if (_isDirty)
    {
        using var surface = SKSurface.Create(info);
        DrawScene(surface.Canvas);
        _renderedImage?.Dispose();
        _renderedImage = surface.Snapshot();
        _isDirty = false;
    }

    if (_renderedImage != null)
    {
        canvas.DrawImage(_renderedImage, 0, 0);
    }
}

void Invalidate() => _isDirty = true;
```

### 9.4 GPU 加速渲染

对于需要高帧率的场景（动画、游戏），使用 GPU 后端：

```csharp
// GRContext 管理 GPU 上下文
using var context = GRContext.Create(GRBackend.OpenGL);

using var glInterface = GRGlInterface.Create();
using var context = GRContext.Create(GRBackend.OpenGL, glInterface);

// 创建使用 GPU 的 Surface
using var surface = SKSurface.Create(
    context,
    false,  // 不需要 stencil
    new SKImageInfo(800, 600, SKColorType.Rgba8888, SKAlphaType.Premul)
);

var canvas = surface.Canvas;
```

> 在 Avalonia 和 MAUI 中，SkiaSharp 的 GPU 加速通常是自动启用的。

### 9.5 性能基准参考

| 操作 | 方法 | 耗时（10000 次） |
|------|------|:---:|
| 画圆 | DrawCircle | ~2ms |
| 画路径 | DrawPath (100 点) | ~5ms |
| 文本 | DrawText | ~3ms |
| 图像 | DrawImage (1000×1000) | ~1ms |
| 像素读取 | GetPixel (1000×1000) | ~2000ms ⚠️ |
| 像素读取 | 指针 (1000×1000) | ~5ms ✅ |

---

## 十、实战：生成数据可视化图表

```csharp
public class ChartGenerator
{
    public SKImage GenerateLineChart(
        IEnumerable<float> data,
        int width = 800,
        int height = 400,
        float margin = 50)
    {
        var info = new SKImageInfo(width, height);
        using var surface = SKSurface.Create(info);
        var canvas = surface.Canvas;

        // 背景
        canvas.Clear(SKColors.White);

        // 图表区域
        float chartLeft = margin;
        float chartRight = width - margin;
        float chartTop = margin;
        float chartBottom = height - margin;
        float chartWidth = chartRight - chartLeft;
        float chartHeight = chartBottom - chartTop;

        // 网格线
        using var gridPaint = new SKPaint
        {
            Color = SKColors.LightGray,
            StrokeWidth = 1,
            Style = SKPaintStyle.Stroke
        };

        for (int i = 0; i <= 5; i++)
        {
            float y = chartTop + chartHeight * i / 5;
            canvas.DrawLine(chartLeft, y, chartRight, y, gridPaint);
        }

        // 数据线
        float min = data.Min();
        float max = data.Max();
        float range = max - min == 0 ? 1 : max - min;

        using var linePaint = new SKPaint
        {
            Color = SKColors.Blue,
            StrokeWidth = 2,
            IsAntialias = true,
            Style = SKPaintStyle.Stroke
        };

        var path = new SKPath();
        var points = data.ToList();

        for (int i = 0; i < points.Count; i++)
        {
            float x = chartLeft + chartWidth * i / (points.Count - 1);
            float y = chartBottom - chartHeight * (points[i] - min) / range;

            if (i == 0) path.MoveTo(x, y);
            else path.LineTo(x, y);
        }
        canvas.DrawPath(path, linePaint);

        // 数据点
        using var dotPaint = new SKPaint
        {
            Color = SKColors.Blue,
            IsAntialias = true,
            Style = SKPaintStyle.Fill
        };

        for (int i = 0; i < points.Count; i++)
        {
            float x = chartLeft + chartWidth * i / (points.Count - 1);
            float y = chartBottom - chartHeight * (points[i] - min) / range;
            canvas.DrawCircle(x, y, 4, dotPaint);
        }

        // 坐标轴
        using var axisPaint = new SKPaint
        {
            Color = SKColors.Black,
            StrokeWidth = 1.5f,
            Style = SKPaintStyle.Stroke
        };
        canvas.DrawLine(chartLeft, chartTop, chartLeft, chartBottom, axisPaint);
        canvas.DrawLine(chartLeft, chartBottom, chartRight, chartBottom, axisPaint);

        // Y 轴标签
        using var labelPaint = new SKPaint
        {
            TextSize = 12,
            Color = SKColors.Gray,
            IsAntialias = true,
            TextAlign = SKTextAlign.Right
        };

        for (int i = 0; i <= 5; i++)
        {
            float y = chartTop + chartHeight * i / 5;
            float value = max - range * i / 5;
            canvas.DrawText($"{value:F1}", chartLeft - 8, y + 4, labelPaint);
        }

        return surface.Snapshot();
    }
}
```

---

## 十一、生态延伸

SkiaSharp 不是孤立的，它有一系列"兄弟"库：

| 库 | 用途 |
|----|------|
| **SkiaSharp** | 2D 图形核心 |
| **HarfBuzzSharp** | 文本排版（OpenType 支持） |
| **Svg.Skia** | SVG 加载和渲染 |
| **QRCoder / ZXing.Net** | 配合 SkiaSharp 生成二维码 |
| **ImageSharp** | 纯 C# 图像处理（不依赖 Skia） |
| **Magick.NET** | ImageMagick 的 .NET 绑定（功能最强但最重） |

---

## 十二、总结

SkiaSharp 是 .NET 生态中 2D 图形处理的**事实标准**。它的优势可以总结为：

1. **跨平台一致性**：Windows / macOS / Linux / iOS / Android 表现一致
2. **高性能**：GPU 加速 + 原生 C++ 后端，性能碾压纯 C# 方案
3. **API 友好**：面向对象，和 Canvas / GDI 概念一一对应，学习成本低
4. **生态完善**：Avalonia / MAUI / Blazor 全部原生支持
5. **工业级质量**：Chrome / Android 同款引擎，经过几十亿设备验证

> 如果你在 .NET 项目中需要画图、生成图表、处理图像、或者做自定义 UI 渲染——选 SkiaSharp 准没错。

---

*本文基于 SkiaSharp 当前稳定版本编写。项目地址：[github.com/mono/SkiaSharp](https://github.com/mono/SkiaSharp)*
