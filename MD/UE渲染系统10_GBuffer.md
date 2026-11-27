# UE 渲染系统 10：GBuffer

## 一、从延迟渲染引出 GBuffer

上一节学习了两种渲染路径：Forward Rendering（前向渲染）和 Deferred Rendering（延迟渲染）。

在典型的前向渲染中，表面属性的计算与光照计算在几何绘制阶段完成。而延迟渲染将两者拆开，先计算并保存表面属性，再在后续阶段计算光照。

```text
Forward：
Fragment → 表面属性计算 → 光照计算 → Scene Color

Deferred：
Fragment → 表面属性计算 → 保存表面属性
                                ↓
                            光照计算 → Scene Color
```

既然光照要延后计算，那么当前计算出来的表面属性就必须保存下来，否则后续光照阶段将无法获得这些数据。

这就是 GBuffer（Geometry Buffer）产生的原因。

**GBuffer 是延迟渲染中用于保存屏幕可见表面属性的一组缓冲区。**

本节以 UE 常见的桌面延迟渲染路径为例，重点研究这些数据从哪里来、如何保存，以及如何被后续 Pass 使用。

## 二、GBuffer 保存什么？

假设一个红色金属球经过投影和光栅化，在渲染图像的 `(100, 200)` 位置产生了一个片元。

为了描述这个片元对应的表面，可以使用以下属性：

| 属性 | 中文 | 含义 | 示例值 |
|---|---|---|---|
| BaseColor | 基础颜色 | 表面的基础颜色属性 | (1, 0, 0) |
| Normal | 法线 | 表面的方向 | (0, 0, 1) |
| Roughness | 粗糙度 | 描述表面粗糙程度 | 0.3 |
| Metallic | 金属度 | 描述表面的金属特性 | 1.0 |
| Specular | 高光属性 | 参与描述非金属表面的镜面反射特性 | 0.5 |

这些属性描述的是表面本身，而不是光照计算完成之后的颜色。

具体属性如何影响光照，将在后续光照与 PBR 课程中学习。本节不需要掌握它们的计算公式。

## 三、表面属性从哪里来？

前面学习顶点属性时，已经知道模型的顶点通常包含 Position、Normal、UV、Tangent、Vertex Color 等数据。

但 Roughness、Metallic、BaseColor 等属性通常不是模型顶点中预先保存好的最终数值，而是由材质定义计算方式，在着色过程中求出。

### 1. 使用常量或材质参数

例如在 UE Material Editor 中设置：

```text
Constant3Vector (1, 0, 0) → Base Color
Scalar (1.0)             → Metallic
Scalar (0.3)             → Roughness
```

使用这个材质的表面，可以得到相同的基础颜色、金属度和粗糙度。

### 2. 使用纹理

如果希望同一个表面的不同位置拥有不同的粗糙度，就可以使用 Roughness Texture。

```text
顶点 UV
   ↓
光栅化与插值
   ↓
当前片元的 UV
   ↓
Pixel Shader 根据 UV 采样纹理
   ↓
得到当前片元的 Roughness
```

例如不同片元采样后得到：

```text
Fragment A → Roughness = 0.2
Fragment B → Roughness = 0.5
Fragment C → Roughness = 0.9
```

同一个材质就能够在不同位置产生不同的表面属性。

### 3. 使用其他数据参与计算

材质还可以利用顶点颜色、插值法线、数学运算等数据计算表面属性。例如将 Vertex Color 的 R 通道作为粗糙度的控制权重。

因此，需要区分三个层次：

- 模型顶点、纹理、材质参数：提供原始数据。
- Material：定义如何利用这些数据计算表面属性。
- Pixel Shader：在像素阶段执行相应的计算，得到当前片元的具体属性。

**GBuffer 保存的不是材质图、模型顶点或 Shader 程序，而是表面属性经过计算之后的结果。**

## 四、GBuffer 与 Render Target 的关系

前面学习 Pixel Shader 时，已经知道它可以产生输出，并写入 Render Target（渲染目标）。

Render Target 不仅可以保存颜色，也可以保存其他数值。例如一个 RGBA 格式的渲染目标，每个位置拥有四个数据通道。这些通道并不一定要代表红、绿、蓝和透明度，也可以用来保存法线、粗糙度等数据。

现在假设需要保存：

```text
BaseColor：3 个分量
Normal：   3 个分量
Roughness：1 个分量
Metallic： 1 个分量
Specular： 1 个分量
```

在不压缩、不进行特殊编码的简单存储模型下，一个 RGBA 渲染目标显然不足以保存全部数据。

因此，典型的延迟渲染会使用多个 Render Target 来承载这些表面属性。这些用于保存表面属性的渲染目标共同构成 GBuffer。

```text
GBuffer
├── Render Target A
├── Render Target B
└── Render Target C ...
```

Render Target 是资源在绘制过程中承担的输出角色，GBuffer 则是按照数据用途命名的一组资源。

**不是所有 Render Target 都属于 GBuffer。** 例如保存场景颜色的 Scene Color、保存泛光结果的目标，都可以是 Render Target，但它们并不因此成为 GBuffer。

## 五、如何一次输出多个 Render Target？

如果需要三个 Render Target，是否必须将场景重新绘制三遍？

不需要。GPU 支持 MRT（Multiple Render Targets，多重渲染目标），允许一次绘制中的 Pixel Shader 同时输出到多个绑定好的颜色渲染目标。

下面是简化的 HLSL 示意代码：

```hlsl
struct PSOutput
{
    float4 Target0 : SV_Target0;
    float4 Target1 : SV_Target1;
    float4 Target2 : SV_Target2;
};

PSOutput MainPS()
{
    PSOutput Output;
    Output.Target0 = float4(1.0, 0.0, 0.0, 0.0);
    Output.Target1 = float4(0.5, 0.5, 1.0, 0.3);
    Output.Target2 = float4(1.0, 0.5, 0.0, 0.0);
    return Output;
}
```

这里的三个输出分别对应三个已绑定的颜色渲染目标。实际写入还受到覆盖关系、深度测试、写入状态等条件影响。

这段代码只用于说明 MRT 的基本机制，不是 UE Base Pass 的真实 Shader 源码。

### 数据打包（Packing）

不同属性也不必各自占用一张 Render Target，可以利用同一张目标的不同通道进行保存。

例如以下是一种假想的布局：

| 渲染目标 | R | G | B | A |
|---|---|---|---|---|
| Target 0 | BaseColor.R | BaseColor.G | BaseColor.B | 未使用 |
| Target 1 | 编码后的 Normal.X | 编码后的 Normal.Y | 编码后的 Normal.Z | Roughness |
| Target 2 | Metallic | Specular | 未使用 | 未使用 |

这里以将法线分量从 -1～1 映射到 0～1 为编码示例。实际 UE 可以采用不同的格式、编码和打包方式。

**上表不是 UE 实际 GBuffer 通道布局。** 本节只需要理解多个输出目标与通道打包的关系，不需要记忆具体版本的内部格式。

## 六、为什么 GBuffer 是二维数据？

GBuffer 不需要保存整个三维模型，因为经过投影和光栅化后，片元已经对应到渲染图像中的位置。

假设使用与渲染图像对应的 GBuffer 资源，并采用每像素一个采样点的简化模型：

```text
三维模型
   ↓
投影与光栅化
   ↓
Fragment (100, 200)
   ↓
Pixel Shader 计算表面属性
   ↓
写入各个 GBuffer 对应的位置 (100, 200)
```

不同 GBuffer 资源在同一个位置保存的数据，共同描述同一个可见表面。

例如：

| 图像位置 | BaseColor | Normal | Roughness | Metallic |
|---|---|---|---|---|
| (100, 200) | 红色 | 方向 A | 0.3 | 1.0 |
| (101, 200) | 红色 | 方向 B | 0.4 | 1.0 |
| (102, 200) | 灰色 | 方向 C | 0.8 | 0.0 |

因此，后续 Pass 可以通过图像坐标读取同一个位置的多个属性。

这里的图像位置指内部渲染分辨率对应的位置，不一定与显示器最终输出分辨率完全相同。

GBuffer 只记录当前视角下经过可见性判断后留下来的表面信息，不是场景中全部三维几何的数据副本。

### 与 Depth Buffer 的区别

同一个位置可能被多个片元覆盖。通过深度测试，最终留下来的不透明表面才会成为对应位置的 GBuffer 数据。

在 UE 中，Depth 通常保存在独立的深度资源中，与 GBuffer 一起为后续渲染提供信息，而不是必须占用某张普通的 GBuffer 颜色渲染目标。

深度测试的具体机制将在下一节详细学习。

## 七、GBuffer 如何连接 Base Pass 与 Lighting？

现在可以将前面几节课的概念串联起来。

```text
Mesh / Material
       ↓
     Base Pass
       ├─ Vertex Shader
       ├─ Rasterization / Interpolation
       └─ Pixel Shader：计算表面属性
                  ↓
          MRT：写入多个渲染目标
                  ↓
               GBuffer
                  ↓
     Lighting：读取表面属性与深度等数据
                  +
                光源数据
                  ↓
            计算光照贡献
                  ↓
              Scene Color
```

在典型的 UE 延迟渲染路径中，Base Pass 的核心职责之一就是生成 GBuffer。后续光照阶段读取这些数据及所需的其他资源，计算并累积光照贡献。

这样，光照阶段不需要重新执行生成这些表面属性的原始 Material Graph，也不需要重新光栅化原始场景几何来获取同一份属性。

GBuffer 因此成为表面属性计算与后续光照计算之间的数据桥梁。

注意，这里展示的是延迟渲染的核心数据流，并非 UE 一帧所有 Pass 的完整执行顺序。Base Pass 也可能写入其他资源或完成其他计算，不能将它的全部职责等同于写 GBuffer。

## 八、UE 实验：观察 GBuffer

本节最重要的实践，是直接观察材质属性在屏幕空间中的表现。

### 实验一：观察固定材质属性

1. 使用桌面 Deferred Rendering 路径的 UE 项目，放置一个 Sphere 和一个 Cube，并添加光源。
2. 创建一个普通 Opaque、Default Lit 材质。
3. 设置 Base Color 为红色，Metallic 为 1.0，Roughness 为 0.3。
4. 将材质应用到 Sphere。
5. 在编辑器视口的 View Mode 中找到 Buffer Visualization，分别查看 Base Color、World Normal、Roughness、Metallic 等模式。菜单名称与可用模式可能随版本和项目设置变化。
6. 在 Lit 模式与不同 Buffer Visualization 模式之间切换，比较结果。

预期观察：

| 可视化模式 | 观察重点 |
|---|---|
| Lit | 受光照及其他渲染效果影响的场景画面 |
| Base Color | 表面的基础颜色属性 |
| World Normal | 用颜色表示世界空间法线方向 |
| Roughness | 用灰度表示粗糙度数值 |
| Metallic | 用灰度表示金属度数值 |
| Scene Depth | 独立深度资源的可视化结果 |

World Normal 中的颜色不是材质颜色，而是方向数据的可视化。例如将 -1～1 的三个方向分量映射到 0～1 后，就可以用 RGB 显示。

Roughness 和 Metallic 是标量，因此可以使用黑白灰表达其数值。具体显示亮度可能受到可视化方式影响，不应直接将屏幕上的颜色当作底层存储位值。

### 实验二：修改材质属性

将 Sphere 材质的 Roughness 分别改为 0.1 和 0.9，在 Lit 和 Roughness 模式中观察差异。

然后只修改 Metallic，观察 Metallic 模式下对应区域的变化。

这个实验可以帮助理解：材质属性改变后，计算并保存的屏幕空间表面数据也会随之改变。

### 实验三：观察同一材质内的属性变化

使用一张黑白纹理连接 Roughness（作为数值数据使用时关闭纹理的 sRGB），保持其他属性不变。

观察球体不同区域在 Roughness 模式中的变化，并联系前面学习的 UV 插值和纹理采样。

需要注意，Buffer Visualization 展示的是渲染数据的可视化或解码结果，不代表每一种显示模式都对应一张独立的底层 Render Target。

## 九、本节总结

本节需要掌握以下关系：

| 概念 | 作用 |
|---|---|
| Material | 定义表面属性的计算方式 |
| Pixel Shader | 计算片元对应的表面属性 |
| Render Target | 承载绘制输出的目标 |
| MRT | 让一次绘制同时输出到多个颜色渲染目标 |
| GBuffer | 保存屏幕可见表面属性的一组资源 |
| Lighting | 读取表面数据及其他必要信息，计算光照 |

核心数据流：

```text
Material → Pixel Shader → MRT / GBuffer → Lighting
```

GBuffer 的重点不是记忆具体有几张纹理，而是理解：**延迟渲染如何把材质计算结果保存为能够被后续 Pass 读取的屏幕空间数据。**

下一节：Depth Buffer、Depth Test 与 Early-Z。研究同一个屏幕位置出现多个片元时，GPU 如何确定可见表面，以及如何减少不必要的着色计算。
