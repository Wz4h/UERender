# UE 渲染系统 07：从 Pixel Shader 认识 Shader

## 一、本节为什么现在讲 Shader

前面已经完成：

```text
3D Model
↓
Triangle
↓
Projection
↓
Rasterization
↓
Fragment
↓
Interpolation
```

到这里，Fragment 已经获得了后续计算需要的一部分数据，例如 UV、Normal、Vertex Color、Screen Position、Depth 等。

那么接下来就出现新的问题：

> 这些数据拿到以后，GPU 到底怎么使用？

这就是 Shader 要解决的问题。

---

# 二、先从 Pixel Shader 开始

现在先不急着定义“Shader 是什么”。

最自然的问题是：

> Rasterization 之后产生了 Fragment，谁来处理这些 Fragment？

答案是：

# Pixel Shader

在 Direct3D / Unreal Engine 语境中通常叫 Pixel Shader。

在 OpenGL / Vulkan 等资料中，也经常看到 Fragment Shader。

两者描述的是非常接近的管线阶段。

---

# 三、Pixel Shader 到底做什么

先用最简单的一句话：

> Pixel Shader 是运行在 GPU 像素阶段的一段程序，它根据当前 Fragment 的输入数据，计算当前绘制需要的像素阶段输出。

可以把它看成：

```text
Output = PixelShader(Input)
```

Fragment 可能提供：

```text
UV
Normal
Vertex Color
Depth
Screen Position
...
```

Pixel Shader 使用这些输入进行计算。

---

# 四、最简单的 Pixel Shader

例如：

```hlsl
float4 MainPS() : SV_Target
{
    return float4(1, 0, 0, 1);
}
```

它不读取 UV、Normal 或 Texture，只做一件事：

```text
输出红色
```

于是所有进入这段 Pixel Shader 的 Fragment 都得到相同输出。

这说明：

> Pixel Shader 本质上就是一段 GPU 程序。

---

# 五、同一个 Pixel Shader，为什么画面上每个位置可以不同？

因为：

> 执行的程序可以相同，但每个 Fragment 的输入数据不同。

例如三个顶点：

```text
A → UV(0,0)
B → UV(1,0)
C → UV(0,1)
```

经过 Rasterization 和 Interpolation 后，不同 Fragment 可以得到：

```text
Fragment A → UV(0.1, 0.2)
Fragment B → UV(0.4, 0.5)
Fragment C → UV(0.8, 0.7)
```

现在 Pixel Shader：

```hlsl
float4 MainPS(float2 UV : TEXCOORD0) : SV_Target
{
    return float4(UV.x, UV.y, 0, 1);
}
```

程序相同，但输入不同，所以输出不同。

因此：

> 同一个 Shader，并不意味着所有 Fragment 输出相同。
>
> 程序相同 + 输入不同 = 输出不同。

---

# 六、UV 为什么最终能够用来贴 Texture

Pixel Shader 最经典的用途之一，就是使用 UV 去采样 Texture。

例如：

```hlsl
Texture2D MyTexture;
SamplerState MySampler;

float4 MainPS(float2 UV : TEXCOORD0) : SV_Target
{
    return MyTexture.Sample(MySampler, UV);
}
```

流程：

```text
Vertex 保存 UV
↓
Rasterization
↓
Interpolation
↓
Fragment 得到自己的 UV
↓
Pixel Shader
↓
Texture.Sample(UV)
↓
得到对应位置的颜色
```

这就是 UV 插值真正被使用的地方之一。

---

# 七、Normal 为什么也会进入 Pixel Shader

不同 Fragment 可以拥有不同的插值 Normal。

Pixel Shader 可以使用：

```text
Normal
+
Light Direction
```

计算受光关系。

虽然所有 Fragment 执行相同代码，但因为 Normal 不同，最终亮度不同。

所以仍然是：

```text
相同程序
+
不同输入
=
不同输出
```

---

# 八、Pixel Shader 的输入从哪里来

Pixel Shader 的输入不能全部归结为“顶点数据”。

可以先分成三类。

## 1. 从 Vertex 传递并插值得到的数据

例如：

```text
UV
Normal
Vertex Color
Tangent
```

流程：

```text
Vertex Attribute
↓
Vertex Shader 输出
↓
Rasterization / Interpolation
↓
Fragment
↓
Pixel Shader
```

## 2. 当前光栅化位置产生的数据

例如：

```text
Screen Position
Depth
```

## 3. 外部资源和运行时数据

例如：

```text
Texture
Sampler
Constant Buffer
Camera Position
Time
其他 Buffer
```

因此 Pixel Shader 可以使用的数据可以理解成：

```text
插值后的 Fragment 数据
+
当前屏幕阶段数据
+
外部资源 / 参数
```

---

# 九、Pixel Shader 输出什么

这里不能简单定义成：

> Pixel Shader 输出最终屏幕颜色。

因为后面学习 Render Pass 后会发现，不同绘制任务对 Pixel Shader 输出要求不同。

所以目前最稳妥的定义是：

> Pixel Shader 产生当前这一次绘制在像素阶段所需要的输出。

在最简单实验中，它可以输出 Color。

但现在不要把它永久定义成最终屏幕颜色。

---

# 十、Pixel Shader 并不等于“每个 Pixel 执行一次”

Pixel Shader 这个名字容易让人误以为：

```text
一个最终 Pixel
=
执行一次 Pixel Shader
```

这个说法不严谨。

同一个屏幕位置可能被多个 Triangle 覆盖，因此可能存在多个 Fragment。

后面还会经过：

```text
Depth Test
Blend
MSAA
```

所以更准确的说法是：

> 光栅化产生的 Fragment 会进入 Pixel Shader 阶段进行计算。

---

# 十一、既然有 Pixel Shader，那 Shader 到底是什么

现在可以开始抽象。

```text
Pixel Shader
=
运行在 Pixel Stage 的 GPU 程序
```

那其他阶段能不能写程序？

可以。

于是得到：

> Shader 是运行在 GPU 某个可编程阶段上的程序。

Pixel Shader 只是其中一种。

---

# 十二、回头看 Vertex Shader

在 Rasterization 之前，我们已经有 Vertex Shader：

```text
Vertex Data
↓
Vertex Shader
↓
Processed Vertex
↓
Primitive Assembly
↓
Rasterization
```

Vertex Shader 处理的是：

> 每个 Vertex 进入后续图形管线前的数据。

常见工作包括：

```text
处理 Vertex Position
进行坐标变换
进行顶点偏移
输出 UV / Normal / Color 等后续需要的数据
```

因此：

```text
Vertex Shader
→ 处理 Vertex

Pixel Shader
→ 处理 Rasterization 后的 Fragment 数据
```

---

# 十三、为什么叫“可编程图形管线”

整个 Graphics Pipeline 并不是所有步骤都允许开发者任意写程序。

它包含：

```text
可编程阶段
+
固定功能阶段
```

例如：

```text
Vertex Data
↓
Vertex Shader        ← 可编程
↓
Primitive Assembly  ← 固定功能为主
↓
Rasterization       ← 固定功能
↓
Interpolation       ← 硬件负责为主
↓
Pixel Shader         ← 可编程
↓
Depth / Stencil     ← 固定功能 + 状态配置
↓
Blend               ← 固定功能 + 状态配置
↓
Render Target
```

所以：

> 可编程图形管线并不是整条 Pipeline 都由开发者自己写。
>
> 真正开放给开发者编程的关键入口，就是不同 Shader Stage。

---

# 十四、为什么 Rasterization 不是 Shader

Rasterization 的任务是：

```text
Triangle
↓
判断覆盖哪些 Sample
↓
产生 Fragment
```

传统图形管线中，这通常由 GPU 固定功能硬件完成。

开发者可以配置 Cull Mode、Fill Mode、MSAA、Depth Bias 等，但一般不会自己写一个 rasterizeTriangle() 替换 GPU 原生 Rasterizer。

因此：

```text
Rasterization
→ 固定功能阶段

Shader
→ 可编程阶段
```

---

# 十五、Depth / Stencil / Blend 也是 Shader 吗

不是。

它们通常属于固定功能阶段。

开发者主要配置状态，例如：

```text
Depth Test On / Off
Compare Function
Depth Write
Blend Factor
Stencil Rule
```

GPU 固定功能单元执行这些规则。

---

# 十六、常见 Shader 类型

## Vertex Shader

处理 Vertex，位于 Rasterization 之前。

## Pixel Shader

处理 Rasterization 之后的 Fragment 数据。

## Geometry Shader

处理 Primitive，例如 Triangle，可以生成或修改图元。

## Hull Shader / Domain Shader

用于 Tessellation（细分）相关流程。

## Compute Shader

不一定属于：

```text
Vertex
↓
Rasterization
↓
Pixel
```

这条传统图形管线。

它可以：

```text
Buffer / Texture
↓
Compute Shader
↓
Buffer / Texture
```

常用于图像处理、粒子、模拟、GPU Culling 等。

---

# 十七、Shader 类型为什么这么多

本质原因是：

> GPU Pipeline 中不同阶段处理的数据和任务不同。

例如：

```text
Vertex Shader
→ 输入是 Vertex

Pixel Shader
→ 输入主要是 Fragment 相关数据

Geometry Shader
→ 输入是 Primitive

Compute Shader
→ 输入由程序自行组织
```

它们都是 GPU 程序，只是运行岗位不同。

---

# 十八、现在先不要把 Material 当成 Shader

UE Material 和 Shader 密切相关，但现在还没有学习 Render Pass。

因此这一节不要说：

```text
Material = Pixel Shader
```

也不要说：

```text
一个 Material = 一个 Shader
```

目前只需要知道：

> UE Material 是更高层的图形化描述系统，它会参与生成真正运行在 GPU 上的 Shader。

为什么同一个 Material 后面会生成多个 Shader，等 Render Pass 章节再讲。

---

# 十九、UE 中怎么观察 Pixel Shader 的概念

虽然这一节不正式讲 Material System，但可以借 UE Material Editor 做教学实验。

目的不是证明：

```text
Material 就是 Pixel Shader
```

而是：

> 借 Material Editor 快速观察像素阶段计算现象。

## 实验一：固定颜色

创建 Unlit Material：

```text
Constant3Vector (1,0,0)
↓
Emissive Color
```

观察整个表面为红色。

## 实验二：UV 直接变成颜色

```text
TextureCoordinate
↓
U → R
V → G
B = 0
↓
Emissive Color
```

观察表面形成连续渐变。

完整流程：

```text
Vertex 保存 UV
↓
Rasterization
↓
Interpolation
↓
不同 Fragment 得到不同 UV
↓
执行同一套像素阶段逻辑
↓
不同 UV 产生不同颜色
```

## 实验三：UV 采样 Texture

```text
TextureCoordinate
↓
Texture Sample
↓
Emissive Color
```

观察不同 Fragment 使用不同 UV，从 Texture 的不同位置取数据。

---

# 二十、本节完整流程

```text
Vertex Data
│
├─ Position
├─ UV
├─ Normal
├─ Vertex Color
└─ ...
      ↓
Vertex Shader
      ↓
Primitive Assembly
      ↓
Triangle
      ↓
Rasterization
      ↓
Fragment
      ↓
Interpolation
      ↓
Fragment Data
│
├─ UV
├─ Normal
├─ Vertex Color
└─ ...
      ↓
Pixel Shader
      ↓
Pixel Stage Output
      ↓
Depth / Stencil
      ↓
Blend
      ↓
Render Target
```

这就是一条典型的 Graphics Pipeline。

---

# 二十一、本节真正要记住的五句话

1. Shader 是运行在 GPU 可编程阶段上的程序。
2. Pixel Shader 处理 Rasterization 之后的 Fragment 数据。
3. 大量 Fragment 可以执行同一段 Pixel Shader，但因为输入不同，输出也可以不同。
4. Vertex Shader 和 Pixel Shader 是传统 Graphics Pipeline 中最核心的两个可编程阶段。
5. Pixel Shader 的输出不是永远等于最终屏幕颜色，它输出的是当前绘制在像素阶段所需要的结果。

---

# 二十二、下一节

到目前为止，我们仍然只研究：

> 一次 Graphics Pipeline 内部发生什么。

还没有正式解释：

```text
为什么一帧需要多次渲染？
为什么同一个 Material 会生成不同 Shader？
为什么有 Base Pass / Shadow Pass？
为什么 Pixel Shader 输出会变化？
```

这些问题都指向下一阶段：

# Render Pass / Multi-Pass Rendering

下一节从一个问题开始：

> 既然一条 Graphics Pipeline 已经能把几何画到 Render Target，为什么现代游戏的一帧还需要很多次这样的 GPU 工作？
