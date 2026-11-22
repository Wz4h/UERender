# UE 渲染系统 08：Render Pass 全景

## 一、本节学习目标

前面已经学习了基础 Graphics Pipeline：

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

这条流程描述的是：**一次图形绘制内部，GPU 如何处理图形数据。**

但一帧游戏画面通常不只包含一次绘制任务。场景中可能同时有不透明物体、阴影、光照与反射、玻璃、烟雾、Bloom、景深、色调映射和 UI。这些任务的输入、处理方式和输出并不相同，因此一帧通常需要组织多个渲染任务，也就是多个 Pass。

本节不深入每个 Pass 的算法，而是先建立一张地图：

1. Pass 是什么？
2. 为什么一帧需要多个 Pass？
3. 常见 Pass 分别负责什么？
4. 不同 Pass 会产生什么类型的输出？
5. 如何在 UE 中观察实际的 GPU 渲染事件？

---

## 二、Pass 和 Graphics Pipeline 的区别

### Graphics Pipeline

Graphics Pipeline 描述一次图形绘制内部的处理阶段，例如：

```text
Vertex Shader
↓
Rasterization
↓
Pixel Shader
↓
Depth / Stencil
↓
Blend
↓
Render Target
```

### Render Pass

Render Pass 描述为了完成某个渲染目标而组织起来的一次 GPU 工作。

例如：处理场景主体表面、生成阴影所需的数据、处理半透明物体，或对已经生成的场景图像进行后处理。

因此可以先建立这样的层级：

```text
Frame
↓
多个 Pass
↓
每个 Pass 根据任务使用相应的 GPU 工作
    ├─ Graphics Pipeline
    └─ Compute 等其他 GPU 工作
```

**注意：不是每个 Pass 都会完整执行一遍传统 Graphics Pipeline。** 有些 Pass 需要重新处理几何，有些 Pass 直接读取已有的 Texture 或 Buffer 进行屏幕空间处理。

---

## 三、为什么一帧需要多个 Pass？

假设场景里有一个角色、一盏灯、一块玻璃，以及 Bloom 效果。这些内容最终组成一张画面，但它们需要解决的问题不同。

### 1. 绘制角色表面

需要处理角色的 Mesh、Vertex、Triangle 和 Material，通常会经过图形管线：

```text
Mesh
↓
Vertex Shader
↓
Rasterization
↓
Pixel Shader
↓
输出当前绘制需要的数据
```

### 2. 生成阴影

阴影需要判断光源照向某个位置时，光线是否被其他物体挡住。这通常需要从光源视角处理场景几何，而不是直接复用摄像机视角下的光栅化结果。

### 3. 处理玻璃

玻璃具有半透明特性。它通常需要结合已经存在的场景颜色进行混合，而不是像普通不透明物体那样简单覆盖。

### 4. 处理 Bloom

Bloom 处理的是已经生成的图像内容，主要关注画面中的高亮区域。它不需要重新处理角色的顶点和三角形。

因此，一帧画面更接近：

```text
场景
↓
Pass A
↓
中间结果
↓
Pass B
↓
中间结果
↓
Pass C
↓
...
↓
最终画面
```

这就是 Multi-Pass Rendering 的基本思想：

> **一帧画面由多个具有不同目标的渲染任务协同完成。**

---

## 四、如何理解一个 Pass 的输入和输出？

可以用一个通用模型理解：

```text
Pass
{
    Input
    Shader / GPU 工作
    Render State
    Output
}
```

每次看到一个 Pass，可以先问三个问题：

1. **它要解决什么问题？**
2. **它需要读取什么输入？**
3. **它处理后要留下什么结果？**

Pass 的输出不一定是可以直接显示的颜色图。它可能是颜色、深度、法线或材质属性、阴影相关数据、运动矢量、遮罩，以及其他 Texture / Buffer。

这些结果可能被后续 Pass 读取，也可能参与合成。

Pass 之间也不一定是简单的串行流水线：有些任务存在依赖，有些可以分支或独立执行，最后再汇合。

---

# 五、UE 中常见的 Pass / 渲染任务

下面介绍的是常见渲染任务类别，帮助建立概念地图。它们不是 UE 在所有项目、平台和渲染路径下都会严格执行的一张固定清单。

## 1. Depth PrePass：深度预处理

**任务：提前记录从摄像机视角看到的表面深度。**

- 典型输出：Depth Buffer。
- 主要用途：为后续深度测试、遮挡判断等工作提供深度信息，某些情况下也能减少后续不必要的像素着色计算。

可以理解成：先记录场景中各个屏幕位置上，离摄像机最近的表面有多远。

```text
场景几何
↓
从 Camera 视角处理
↓
记录深度
↓
Depth Buffer
```

并不是所有场景或配置都会执行独立的 Depth PrePass。

## 2. Shadow Depth Pass：阴影深度 Pass

**任务：从光源视角记录场景深度，为阴影判断提供依据。**

- 典型输出：光源视角下的深度数据，或阴影系统使用的相关数据。
- 主要用途：判断光线是否被场景中的物体遮挡。

它与摄像机视角下的深度不同：

```text
Camera View → 摄像机看到的深度
Light View  → 光源看到的深度
```

因为观察视角不同，通常需要重新处理相应的几何。

## 3. Base Pass：基础渲染 Pass

**任务：处理场景主体的基础表面。**

- 主要对象：通常包括 Opaque 和 Masked 几何。
- 输出：当前渲染路径下，后续渲染需要的基础场景数据。
- 作用：为后续光照及其他渲染阶段提供基础信息。

可以先把它理解为：处理场景主体，让 GPU 得到后续渲染所需要的表面信息。

Base Pass 的具体输出组织方式与渲染路径有关。后面学习 Forward Rendering 和 Deferred Rendering 时，再分别研究它们的差异。

## 4. Decal Pass：贴花 Pass

**任务：把额外的表面效果附加到已有场景表面上。**

常见用途：

- 墙面污渍
- 弹孔
- 划痕
- 地面积水
- 局部颜色或材质变化

输出可能体现为被修改的表面属性或相关颜色数据。具体实现和输出取决于贴花类型及渲染路径。

## 5. Lighting Pass：光照相关 Pass

**任务：计算光源对场景表面的影响。**

可能读取表面信息、光源信息、阴影信息和其他光照相关数据。

输出：光照贡献，并累积或合成到相应的场景颜色目标中。

后续会进一步学习直接光照、间接光照和不同光照模型。

需要注意：在不同渲染路径中，光照计算的组织方式不同，不应把 Lighting Pass 理解成所有 UE 渲染配置下都完全相同的独立步骤。

## 6. Reflection / GI：反射与全局光照相关阶段

**任务：处理环境反射和间接光照。**

常见效果：

- 物体反射周围环境
- 光线经过表面反弹后照亮其他区域
- 室内外环境产生间接光照

相关技术包括 Lumen、屏幕空间反射、Reflection Capture 等。

这些是完整的渲染技术或系统，内部可能由多个 GPU 阶段组成，不应简单地认为一种技术就等于一个 Pass。

## 7. Fog / Atmosphere：雾与大气相关阶段

**任务：计算雾和大气对场景的影响。**

常见效果包括距离雾、体积雾、天空与大气散射。输出会参与场景颜色的计算或合成。

## 8. Translucency Pass：半透明 Pass

**任务：处理玻璃、水、烟雾和粒子等半透明表面。**

半透明物体通常需要结合已有场景颜色进行混合。

常见对象：

- 玻璃
- 水面
- 烟雾
- 粒子
- 半透明特效

半透明的执行时机和具体路径可能不同，例如某些半透明效果会涉及景深前后不同的处理阶段。

## 9. Velocity Pass：运动矢量相关阶段

**任务：记录像素或表面在连续帧之间的运动信息。**

- 典型输出：Velocity / Motion Vector 数据。
- 主要用途：运动模糊、时间抗锯齿、TSR 等时间相关技术。

并不是所有项目都会有一个独立的 Velocity Pass。有些运动矢量数据可以在其他渲染阶段中生成。

## 10. Post Process Pass：后处理 Pass

**任务：对已经生成的场景图像进行进一步处理。**

常见效果：

- Bloom
- Depth of Field
- Motion Blur
- Anti-Aliasing
- Tone Mapping
- Color Grading
- Temporal Super Resolution

输入通常包括场景颜色，以及效果需要的深度、运动等辅助数据。

输出是处理后的图像，供后续阶段继续使用。

## 11. UI / Final Composition：界面与最终合成

**任务：把 UI 等内容合成到最终显示画面。**

例如血条、技能栏、小地图、菜单和 HUD。

UI 的具体合成路径与场景几何渲染 Pass 不完全相同。这里先把它作为整帧输出流程中的最后一类任务来认识。

---

## 六、把常见渲染任务放到一张图里

下面是帮助理解的简化示意图，**不是 UE 固定的执行顺序，也不表示每个项目都会执行全部这些任务**。

```text
Frame
│
├─ 场景准备 / 可见性判断
│
├─ Depth PrePass
│    └─ 深度数据
│
├─ Shadow Depth Pass
│    └─ 光源视角下的深度 / 阴影数据
│
├─ Base Pass
│    └─ 场景主体的基础表面数据
│
├─ Decal
│    └─ 贴花对表面产生的影响
│
├─ Lighting / Reflection / GI
│    └─ 光照、反射和间接光照贡献
│
├─ Fog / Atmosphere
│    └─ 雾与大气效果
│
├─ Translucency
│    └─ 半透明内容与场景的混合结果
│
├─ Velocity 相关阶段
│    └─ 运动矢量数据
│
├─ Post Processing
│    └─ Bloom / DOF / Motion Blur / Tone Mapping 等
│
└─ UI / Final Composition
     └─ 最终画面
          ↓
       Back Buffer
          ↓
        Present
          ↓
        显示器
```

实际渲染顺序可能因渲染路径、功能开关和依赖关系而变化；某些阶段会分支、合并或在不同时间执行。

---

## 七、不同 Pass 的输出速查

| Pass / 渲染任务 | 输出重点 |
|---|---|
| Depth PrePass | 摄像机视角下的深度 |
| Shadow Depth Pass | 光源视角下的深度或阴影相关数据 |
| Base Pass | 当前渲染路径需要的基础场景 / 表面数据 |
| Decal Pass | 贴花修改后的表面相关数据 |
| Lighting Pass | 光照贡献或光照后的场景颜色 |
| Reflection / GI | 反射与间接光照贡献 |
| Fog / Atmosphere | 雾和大气相关结果 |
| Translucency Pass | 半透明混合结果 |
| Velocity 相关阶段 | 运动矢量 |
| Post Process Pass | 后处理后的图像 |
| UI / Final Composition | 合成后的最终画面 |

**重要：不是每个 Pass 都输出一张可以直接显示的颜色图片。** 深度、法线、运动矢量和其他辅助数据也可以是 Pass 的输出。

也不要把所有 Pass 的输出理解成最后简单相加。更准确地说，是后续阶段根据需要读取前面的结果，继续计算、混合或合成，逐步形成最终画面。

---

## 八、在 UE 中观察 Pass

本节可以做一次初步观察实验，目的是把抽象的 Pass 概念和 UE 中实际发生的 GPU 工作对应起来。

### 实验步骤

1. 打开一个 UE 场景。
2. 放入一个 Sphere、一个 Directional Light，以及一个透明物体。
3. 在编辑器视口或运行场景时，打开 GPU Profiling / GPU Visualizer。
4. 使用控制台命令：

```text
ProfileGPU
```

5. 查看当前帧的 GPU 渲染事件。
6. 寻找类似以下名称或分组（具体名称会随 UE 版本、平台和渲染配置变化）：

```text
PrePass
BasePass
ShadowDepths
Translucency
PostProcessing
```

7. 对照本节的任务地图，尝试判断每个事件大致负责什么。

### 观察时需要注意

- GPU Visualizer 中的条目可能是 Pass、事件分组或具体 GPU 工作，不一定与本节的概念分类一一对应。
- 某些 Pass 可能因为场景没有相关对象或功能未启用而不出现。
- 不同 UE 版本、平台和渲染路径下，事件名称和组织结构可能不同。
- 本节只要求认识和定位，不要求理解每个事件内部的源码和算法。

如果要进一步观察 Render Target / Texture 的具体内容，可以在后续学习对应技术时使用纹理可视化工具进行分析。

---

## 九、本节需要记住的内容

### 1. Pass 和 Graphics Pipeline 不同

- Graphics Pipeline：描述一次图形绘制内部的处理阶段。
- Pass：描述一次具有明确渲染目标的 GPU 工作。

### 2. 一帧通常由多个 Pass 协同完成

不同任务需要不同输入、处理方式和输出。

### 3. Pass 不一定完整执行 Graphics Pipeline

有些 Pass 处理几何，有些直接处理已有 Texture / Buffer，也可能使用 Compute 等 GPU 工作。

### 4. Pass 的输出不一定是最终颜色

深度、阴影数据、法线、运动矢量、遮罩等都可能是输出。

### 5. 不要把示意图当成固定执行顺序

UE 的实际渲染过程会受到渲染路径、平台、场景内容和功能配置影响。

---

## 十、后续课程安排

本节只建立全景，不深入每一种 Pass 的内部机制。

后续按照技术之间的依赖关系继续学习：

1. **Forward Rendering 与 Deferred Rendering**：不同渲染路径如何组织材质和光照计算。
2. **GBuffer**：延迟渲染如何保存屏幕空间表面信息。
3. **Shadow Mapping / Virtual Shadow Maps**：阴影数据如何生成和使用。
4. **Lighting、GI 与 Lumen**：直接光照、间接光照和反射如何工作。
5. **Translucency**：半透明物体为什么需要特殊处理。
6. **Post Processing**：如何对已经生成的场景图像继续加工。

以后每次深入一个技术，都可以回到这张 Pass 地图，明确自己正在研究哪一类渲染任务。

**本节真正的学习目标：建立 UE 渲染系统的地图，而不是背诵一张 Pass 名称清单。**
