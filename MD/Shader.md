# UE 渲染系统：从顶点数据到单个 Graphics Pipeline 完成

> 本文承接前面的三角形、投影、光栅化、Sample 与 Fragment。
>
> 这一阶段的目标不是马上进入 UE 的 Base Pass、GBuffer 或延迟渲染，而是先把**一次最基本的图形管线流程**完整走通。

核心主线：

```text
Vertex Data
↓
Vertex Shader
↓
Primitive / Triangle
↓
Rasterization
↓
Fragment
↓
Interpolation
↓
Pixel Shader
↓
Depth / Stencil / Blend
↓
Render Target
```

到这里，我们第一次拥有了一条相对完整的 **Graphics Pipeline（图形渲染管线）**。

下一阶段再问：

> 一帧画面为什么不是只执行一次这样的流程，而是需要很多 Pass？

---

# 一、从光栅化之后的问题开始

前面已经知道：

```text
屏幕上的 Triangle
↓
Rasterization
↓
判断 Triangle 覆盖哪些 Sample
↓
产生 Fragment
```

但这时候 Fragment 还面临一个问题：

> Fragment 后面要参与纹理、光照等计算，它需要的 UV、Normal、Color 等数据从哪里来？

答案要从 Vertex 开始。

---

# 二、Vertex 不只是一个位置

一个 Vertex 并不只有 Position，它通常还会携带一组 **Vertex Attributes（顶点属性）**：

```text
Vertex
{
    Position
    UV
    Normal
    Tangent
    VertexColor
    ...
}
```

可以把 Vertex 理解成：

> **模型几何中的一个顶点，同时也是一组数据的载体。**

## 1. Position

Position 表示顶点在某个坐标空间中的位置。

模型资产里的 Position 通常首先是 Local Space / Model Space，之后可以继续经过变换：

```text
Local Space
↓
World Space
↓
View Space
↓
Clip Space
↓
最终用于光栅化的位置
```

Position 决定模型的几何形状。UE 中很多效果本质上也是修改顶点位置，例如 World Position Offset、草随风摆动、水面波浪和程序化顶点变形。

## 2. UV

UV 解决：

> **模型表面的某个位置，应该去二维 Texture 的哪里取数据？**

例如：

```text
Vertex A → UV(0,0)
Vertex B → UV(1,0)
Vertex C → UV(0,1)
```

它建立了 3D 模型表面和 2D Texture 之间的对应关系。

UV 可以用于采样 Base Color、Normal Map、Roughness、Metallic、AO、Emissive、Mask 等纹理。

## 3. Normal

Normal 表示：

> **这个表面朝哪个方向。**

Shader 要判断一个位置如何受光，需要知道光从哪里来，以及表面朝哪里。

低面数 Sphere 虽然仍由很多平 Triangle 组成，但如果 Vertex Normal 是平滑变化的，并进一步插值到 Triangle 内部，光照也可以连续变化，因此视觉上会更平滑。

## 4. Tangent

Tangent 表示沿着模型表面延伸的一个方向。

```text
        Normal
          ↑
          |
──────────┼────→ Tangent
```

Tangent 常和 Bitangent、Normal 一起建立 Tangent Space，帮助 Normal Map 中记录的方向正确映射到模型表面。

## 5. Vertex Color

Vertex Color 通常包含 RGBA 四个通道。它可以真的表示颜色，也经常作为四个控制数据通道。

例如：

```text
R → 苔藓强度
G → 泥土强度
B → 湿润程度
A → 额外 Mask
```

或者用 R 控制树叶是否参与风摆。

所以：

> **Vertex Color 不一定只是显示颜色，也经常是传给 Shader 的控制数据。**

---

# 三、为什么数据存在 Vertex 上，而不是提前存在 Fragment 上？

Vertex 属于模型，是稳定数据；Fragment 是运行时产生的。

同一个 Triangle：

```text
离 Camera 很远
→ 可能只覆盖很少 Sample

靠近 Camera
→ 可能覆盖大量 Sample
```

因此 Fragment 的数量和位置会随着 Camera、分辨率和 Triangle 的屏幕覆盖范围变化。

所以不能提前给模型存好所有 Fragment 数据，更合理的方式是：

```text
Vertex 保存稳定属性
↓
运行时产生 Fragment
↓
动态计算 Fragment 所需要的属性
```

这就引出了 Interpolation（插值）。

---

# 四、Sample 和 Fragment 不要混淆

Sample 可以理解为：

> **屏幕上的采样位置。**

Rasterization 需要判断 Triangle 是否覆盖某个 Sample。

所以：

```text
Sample
→ “在哪里”

Fragment
→ Triangle 在这里产生的一份候选渲染数据
```

Sample 本身不是模型顶点，因此不要简单说 Sample 自己拥有 UV、Normal 或 VertexColor。

更准确的是：

> 当 Triangle 覆盖 Sample 并产生 Fragment 后，Fragment 会得到后续阶段所需要的属性。

---

# 五、Interpolation：Fragment 的数据怎么来？

插值可以理解为：

> **已知几个位置的数据，计算中间位置应该得到什么数据。**

例如：

```text
A = 0
B = 100
```

P 如果正好在中间，就可以得到 50；如果更靠近 A，结果就更接近 A。

所以插值不是简单平均，而是根据当前位置分配权重。

三角形内部：

```text
        A
       / \
      / P \
     /     \
    B-------C
```

假设 P 对三个顶点的权重分别是：

```text
A = 0.5
B = 0.3
C = 0.2
```

那么 P 的某个属性可以理解成：

```text
P = A × 0.5 + B × 0.3 + C × 0.2
```

三个权重之和为 1。

---

# 六、三个最直观的插值例子

## 1. Vertex Color

```text
A = Red
B = Green
C = Blue
```

Triangle 内部会出现连续颜色变化：

```text
三个 Vertex Color
↓
Interpolation
↓
Fragment 获得自己的 Color
```

## 2. UV

```text
A → UV(0,0)
B → UV(1,0)
C → UV(0,1)
```

Triangle 内部某个 Fragment 经过插值后可能得到：

```text
UV = (0.37, 0.62)
```

后面的 Shader 就可以拿这个 UV 去 Texture 中采样。

## 3. Normal

三个 Vertex 可以拥有不同 Normal，Fragment 得到插值后的 Normal 后，表面受光方向就可以连续变化。

所以插值解决的本质问题是：

> **少量 Vertex 数据，如何变成大量 Fragment 所需要的数据？**

---

# 七、Fragment 有数据以后，谁来处理？

到这里，Fragment 已经可能拥有：

```text
UV
Normal
VertexColor
Screen Position
Depth
...
```

接下来就出现一个自然问题：

> **这些数据谁来使用？**

答案就是 Pixel Shader。

这一阶段先不要把 UE Material 引进来，因为课程还没有讲 Pass。现在只研究当前这一次绘制中的 Pixel Shader。

---

# 八、Pixel Shader 是什么？

可以先定义为：

> **Pixel Shader 是运行在 GPU 像素阶段的一段程序，它处理光栅化之后产生的 Fragment 数据，并产生当前绘制所需要的像素阶段输出。**

抽象成：

```text
Fragment Data
↓
Pixel Shader
↓
Output
```

或者：

```text
Output = PixelShader(Input)
```

---

# 九、不同 Fragment 的 Pixel Shader 一样吗？

需要区分“程序”和“输入数据”。

假设很多 Fragment 都执行同一个 Pixel Shader：

```text
OutputColor = VertexColor
```

但它们的输入不同：

```text
Fragment A → Color = (0.8, 0.1, 0.1)
Fragment B → Color = (0.3, 0.5, 0.2)
Fragment C → Color = (0.1, 0.2, 0.7)
```

所以：

```text
同一个程序
+
不同输入
=
不同输出
```

这也是 GPU 非常擅长的工作模式：让大量数据并行执行相似程序。

---

# 十、Pixel Shader 的最简单例子

例如：

```hlsl
float4 MainPS() : SV_Target
{
    return float4(1, 0, 0, 1);
}
```

这段 Pixel Shader 不关心 UV、Normal 或 Texture，直接输出红色。

所以经过它处理的 Fragment 都会得到红色输出。

这说明：

> **Pixel Shader 本质上就是 GPU 上执行的一段程序。**

---

# 十一、把上一节 UV 插值接到 Pixel Shader

例如：

```hlsl
float4 MainPS(float2 UV : TEXCOORD0) : SV_Target
{
    return float4(UV.x, UV.y, 0, 1);
}
```

不同 Fragment 得到不同 UV：

```text
Fragment A → UV(0.1, 0.2)
Fragment B → UV(0.4, 0.5)
Fragment C → UV(0.8, 0.7)
```

它们执行同一个 Pixel Shader，但因为 UV 不同，所以输出颜色不同。

整个链条第一次完整串起来：

```text
Vertex UV
↓
Interpolation
↓
Fragment UV
↓
Pixel Shader
↓
输出结果
```

---

# 十二、Texture Sampling 例子

Pixel Shader 还可以使用 UV 去采样 Texture：

```hlsl
Texture2D MyTexture;
SamplerState MySampler;

float4 MainPS(float2 UV : TEXCOORD0) : SV_Target
{
    return MyTexture.Sample(MySampler, UV);
}
```

所以：

```text
同一段 Pixel Shader
+
不同 Fragment UV
=
不同 Texture Sample 结果
```

---

# 十三、Normal 也可以被 Pixel Shader 使用

不同 Fragment 可以拥有不同的插值 Normal。

Pixel Shader 可以根据：

```text
Normal
+
Light Direction
```

进行计算。

虽然执行的是同一套程序，但 Normal 不同，因此结果也不同。

---

# 十四、为什么叫 Pixel Shader？

Direct3D / UE 体系中经常叫 Pixel Shader；OpenGL / Vulkan 等资料中经常看到 Fragment Shader。

需要注意：

> **Fragment 不等于最终 Pixel。**

一个屏幕位置上可能有多个 Triangle 产生的 Fragment，后面还可能经过 Depth Test、Blend、MSAA 等处理。

所以不要死记“一个 Pixel 一定执行一次 Pixel Shader”。

更加稳妥的理解是：

> **光栅化产生 Fragment 后，这些像素阶段候选数据进入 Pixel Shader 进行计算。**

---

# 十五、Pixel Shader 输出什么？

这里不要定义成：

> Pixel Shader 永远输出最终屏幕颜色。

因为后面学习 Multi-Pass、Forward Rendering 和 Deferred Rendering 后，会发现不同 Pass 的输出可能不同。

现在应该使用更通用的定义：

> **Pixel Shader 输出当前这一次绘制在像素阶段需要产生的结果。**

在最简单实验里，它当然可以直接输出颜色：

```text
Pixel Shader
↓
RGBA
```

但现在先不要把这个结果永久定义成最终屏幕颜色。

只需要记住：

```text
Fragment
↓
Pixel Shader
↓
当前绘制需要的像素阶段输出
```

至于这个结果到底写到哪里、为什么以后不同 Pass 会输出不同东西，留到 Pass 章节解释。

---

# 十六、从 Pixel Shader 扩展到 Shader

Pixel Shader 只是 Shader 的一种。

回头看光栅化之前：Vertex 也要经过可编程处理，这就是 Vertex Shader。

最基本流程：

```text
Vertex Data
↓
Vertex Shader
↓
Triangle
↓
Rasterization
↓
Fragment
↓
Pixel Shader
```

于是可以抽象出 Shader 的通用概念：

> **Shader 是运行在 GPU 某个可编程阶段上的程序。**

不同 Shader 的区别主要在于：

```text
运行在什么阶段
处理什么数据
负责什么工作
```

常见类型包括：

- Vertex Shader：处理 Vertex
- Pixel Shader：处理光栅化后的像素阶段数据
- Geometry Shader：处理 Primitive
- Hull / Domain Shader：处理 Tessellation
- Compute Shader：进行通用 GPU 并行计算
- 更现代的 Mesh Shader 等

当前阶段重点只需要真正掌握 Vertex Shader 和 Pixel Shader。

---

# 十七、整个图形管线哪些地方可以自己写？

传统图形管线不是所有步骤都能由开发者随便编程。

可以粗略理解成：

```text
Vertex Data
↓
Vertex Shader        ← 可编程
↓
Primitive Assembly  ← 固定功能为主
↓
Rasterization       ← 固定功能
↓
Interpolation       ← 硬件完成为主
↓
Pixel Shader         ← 可编程
↓
Depth / Stencil     ← 固定功能 + 状态配置
↓
Blend               ← 固定功能 + 状态配置
↓
Render Target
```

开发者真正可以编写的是 Shader 阶段。

Rasterization 通常不是开发者自己写一个 `RasterizeTriangle()`，而是 GPU 固定功能负责；开发者更多是配置 Cull Mode、Fill Mode、Depth State、Blend State、Stencil State、MSAA 等规则。

因此：

> **可编程图形管线并不是整条 Pipeline 都能任意编程，而是在若干关键阶段开放 Shader 作为可编程入口。**

---

# 十八、Pixel Shader 之后发生什么？

Pixel Shader 计算完之后，结果通常还不是直接送到屏幕。

逻辑上可以先理解成：

```text
Pixel Shader
↓
Depth / Stencil
↓
Blend
↓
Render Target
```

真实 GPU 为了性能可能存在 Early-Z 等优化，让某些测试提前发生。当前阶段先理解逻辑职责，不把硬件执行顺序绝对化。

---

# 十九、Depth Test

同一个屏幕位置可能被多个 Triangle 覆盖。

例如：

```text
Fragment A
Depth = 0.2

Fragment B
Depth = 0.8
```

Depth Test 解决：

> **覆盖同一位置的候选 Fragment，谁满足深度规则？**

所以：

```text
Rasterization
→ 谁覆盖了这里？

Depth Test
→ 哪些候选 Fragment 可以通过？
```

---

# 二十、Stencil Test

Stencil 可以理解为额外的一套屏幕级筛选规则。

它可以控制：

> **某些区域是否允许当前 Fragment 通过。**

以后可以用于局部效果、遮罩、特殊轮廓、Portal 等效果。

---

# 二十一、Blending

如果前景是半透明的，就不能简单让前面的 Fragment 把后面的结果完全覆盖掉。

Blending 决定：

> **当前 Fragment 输出如何和目标中已有的值组合。**

可以先这样区分：

```text
Depth / Stencil
→ 是否允许写

Blend
→ 怎么和已有结果组合
```

---

# 二十二、Render Target 是什么？

Render Target 可以先理解为：

> **GPU 当前允许把渲染结果写进去的一张二维图像资源。**

但：

> **Render Target 不一定就是最终送给显示器的图片。**

它也可能只是中间结果。

后面的 Multi-Pass 会大量出现：

```text
Pass A
↓
Render Target A

Pass B 读取 A
↓
Render Target B
```

因此现阶段只记：

```text
Render Target
=
当前渲染过程的输出目标
```

---

# 二十三、最终显示：Back Buffer 与 Present

如果一直追到最终显示，可以先粗略理解成：

```text
各种渲染结果
↓
最终画面
↓
Back Buffer
↓
Present
↓
显示系统
↓
Display
```

Back Buffer 可以先理解为 GPU 准备好的下一张最终显示画面。

Present 可以理解为：

> **把已经准备好的画面提交给显示系统。**

最后显示器看到的，又回到了第一节：

```text
二维 Pixel Grid
+
每个 Pixel 的颜色
```

于是整个学习流程第一次闭环。

---

# 二十四、这整套流程叫什么？

从模型数据到 Render Target：

```text
Vertex Data
↓
Vertex Shader
↓
Primitive Assembly
↓
Rasterization
↓
Fragment
↓
Interpolation
↓
Pixel Shader
↓
Depth / Stencil
↓
Blend
↓
Render Target
```

这一整套流程通常称为：

# Graphics Pipeline

中文可以叫：

# 图形管线 / 图形渲染管线

更加具体一点，我们目前讨论的是典型的：

> **实时光栅化图形管线。**

---

# 二十五、为什么 Shader 直到现在才正式讲？

并不是 Shader 之前不存在。

Vertex Shader 早就在光栅化之前参与流程。

只是前面的课程先解决：

```text
Vertex 是什么
Triangle 是什么
Projection 是什么
Rasterization 是什么
Fragment 是什么
Interpolation 是什么
```

等这些输入和阶段都明白后，再统一展开 Shader，就不会只是在背 Shader 名字。

因此正确理解是：

> **Shader 不是现在才出现，而是现在才正式展开。**

---

# 二十六、这一阶段暂时不要深入 UE Material

UE Material 和 Shader 密切相关，但现在不适合直接说：

```text
Material = Pixel Shader
```

也不适合说：

```text
一个 Material = 一套固定 Shader
```

因为后面学习 Pass 后会发现，一个 Material 可能参与不同 Pass，并生成不同 Shader 版本。

当前阶段只需要知道：

> **UE Material 是更高层的渲染描述系统，它最终会参与生成 GPU 执行的 Shader。**

Material 与 Pass、Shader Permutation 的具体关系放到后面的 UE 渲染章节再讲。

---

# 二十七、单个 Graphics Pipeline 结束以后，下一个问题是什么？

现在我们第一次拥有了一个完整模型：

```text
输入几何数据
↓
Vertex Shader
↓
Rasterization
↓
Pixel Shader
↓
Depth / Blend
↓
Render Target
```

看起来似乎：

> 一帧画面执行一次这样的流程就够了。

但真实游戏画面还需要：

```text
阴影
光照
半透明
后处理
Bloom
Tone Mapping
UI
...
```

于是出现新的问题：

> **为什么不能一次 Graphics Pipeline 把所有事情全部完成？**

进一步：

> **如果一帧需要多次、不同目的的 GPU 工作，它们之间怎么组织？**

这就是下一节的主题：

# Multi-Pass Rendering

---

# 二十八、下一节预告：为什么需要多个 Pass？

下一节先不背 Base Pass、Shadow Pass、Lighting Pass 等 UE 名词，而是先回答三个问题。

## 1. 为什么一帧不能一次画完？

不同渲染任务可能：

- 需要不同输入
- 使用不同 Shader
- 从不同视角观察场景
- 产生不同输出
- 使用前一个阶段生成的结果

## 2. 一个 Pass 到底是什么？

Pass 不是：

```text
Pixel Shader 之后的一个固定阶段
```

而是：

> **为了完成一个明确渲染目标而组织起来的一次 GPU 工作阶段。**

有些 Pass 会重新经过：

```text
Vertex Shader
↓
Rasterization
↓
Pixel Shader
```

有些 Pass 则直接读取之前生成的 Texture / Buffer：

```text
Texture / Buffer
↓
Pixel Shader / Compute Shader
↓
新的结果
```

## 3. 多个 Pass 怎么连接？

一种非常重要的方式就是：

```text
Pass A
↓
Render Target / Buffer
↓
Pass B 读取
↓
新的 Render Target / Buffer
```

这会把已经学过的 Shader、Render Target、Texture、Depth 等概念重新串起来。

---

# 本阶段最终总结

这一阶段最应该掌握的不是名词数量，而是这条因果链：

```text
模型只有少量 Vertex
↓
Vertex 携带 Position / UV / Normal / Color 等属性
↓
Vertex Shader 处理顶点
↓
Triangle 被投影到屏幕
↓
Rasterization 判断覆盖哪些 Sample
↓
产生 Fragment
↓
Interpolation 为 Fragment 计算 UV / Normal / Color 等属性
↓
Pixel Shader 使用这些数据执行程序
↓
产生当前绘制所需要的像素阶段结果
↓
Depth / Stencil / Blend 决定如何保留与组合
↓
写入 Render Target
```

这一整条就是：

# Graphics Pipeline

而下一阶段开始研究：

```text
一个 Frame
↓
为什么需要多个 Graphics / Compute 工作阶段
↓
这些阶段如何通过中间结果连接
```

也就是：

# Render Pass / Multi-Pass Rendering
