# UE 渲染系统 09：Forward Rendering 与 Deferred Rendering

## 一、本节学习目标

前面已经学习了：

```text
Vertex
↓
Rasterization
↓
Fragment
↓
Pixel Shader
↓
Render Target
```

也知道了一帧画面通常会由多个 Pass 协同完成。

这一节继续解决一个很自然的问题：

> Fragment 已经有了表面的各种信息以后，最终的光照颜色应该什么时候计算？

围绕这个问题，会出现两种常见的渲染组织方式：

- Forward Rendering
- Deferred Rendering

这一节不深入性能优化，也不讨论复杂实现，只先建立最基本的理解：

1. 什么是 Forward Rendering？
2. 什么是 Deferred Rendering？
3. 两者最核心的区别是什么？
4. 为什么会出现这两种不同的渲染方式？

---

# 二、先从一个 Fragment 开始

假设某个 Triangle 已经经过 Rasterization，产生了一个 Fragment。

这个 Fragment 可以得到很多表面信息，例如：

```text
BaseColor
Normal
Roughness
Metallic
...
```

这些数据描述的是：

> 这个表面本身是什么样。

但玩家最终看到的并不只是“表面本身”。

还需要考虑：

```text
灯从哪里照过来？
灯有多亮？
表面的 Normal 朝哪里？
表面有多粗糙？
```

最后才能得到这个位置真正显示出来的颜色。

于是就出现了一个问题：

> 表面信息算出来以后，是立刻计算光照，还是先把这些表面信息保存下来，之后再计算光照？

Forward 和 Deferred，就是对这个问题的两种不同处理方式。

---

# 三、Forward Rendering

Forward Rendering 可以先理解成：

> Fragment 在被处理的时候，直接把材质和光照一起算完。

流程可以简化成：

```text
Geometry
↓
Rasterization
↓
Fragment
↓
计算表面信息
BaseColor
Normal
Roughness
Metallic
...
↓
读取 Light
↓
计算光照
↓
得到最终颜色
↓
Scene Color
```

例如一个 Fragment：

```text
Fragment F

BaseColor = 红色
Normal = ...
Roughness = 0.4
Metallic = 0
```

场景里有一盏灯。

Forward 会直接：

```text
Fragment F
+
Light
↓
Pixel Shader 中计算
↓
得到受光后的颜色
```

所以 Forward 最直观的一句话就是：

> **表面被绘制的时候，顺便把光照结果一起算出来。**

---

# 四、为什么这种方式叫 Forward？

因为整个过程非常直接：

```text
几何
↓
Fragment
↓
材质
↓
光照
↓
颜色
```

数据一路向前处理，最后直接得到场景颜色。

因此可以先把 Forward 理解为：

> **直接渲染出受光后的结果。**

---

# 五、Deferred Rendering

Deferred Rendering 做的事情不同。

它处理 Fragment 的时候，不急着完成最终光照。

而是先计算：

```text
BaseColor
Normal
Roughness
Metallic
...
```

然后把这些表面信息保存下来。

流程可以先理解成：

```text
Geometry
↓
Rasterization
↓
Fragment
↓
计算表面信息
↓
保存
```

之后，再进行：

```text
保存下来的表面信息
+
Light
↓
计算光照
↓
Scene Color
```

也就是说：

```text
第一步：
这个表面是什么样？

第二步：
灯照到这个表面以后是什么样？
```

这两个问题被拆开了。

---

# 六、为什么叫 Deferred？

Deferred 的意思可以理解为：

> 延后。

这里延后的主要就是：

> 光照计算。

Forward：

```text
Fragment
↓
表面信息
↓
马上计算光照
↓
颜色
```

Deferred：

```text
Fragment
↓
表面信息
↓
先保存
```

之后：

```text
保存的表面信息
↓
再计算光照
↓
颜色
```

所以：

> **Deferred Rendering = 先保存表面信息，把光照延后到后面的阶段处理。**

---

# 七、两者最核心的区别

到这里其实只需要记住一件事。

## Forward

```text
Fragment
↓
Material
+
Lighting
↓
Scene Color
```

也就是：

> 表面和光照一起完成。

---

## Deferred

```text
Fragment
↓
Material
↓
保存表面信息
```

之后：

```text
保存的表面信息
↓
Lighting
↓
Scene Color
```

也就是：

> 先得到表面，再单独计算光照。

---

# 八、为什么会出现 Deferred？

Forward 本身非常直接，也很好理解。

但是当场景越来越复杂以后，开发者会发现：

```text
Material
和
Lighting
```

一直绑定在同一个 Fragment 着色过程中，并不总是最方便。

于是产生了另一种思路：

> 能不能先把当前屏幕上的表面信息统一保存下来，然后让后面的 Lighting 只处理这些已经整理好的表面数据？

于是就有了 Deferred。

它相当于在：

```text
Material
```

和：

```text
Lighting
```

之间增加了一层中间结果。

也就是：

```text
Material
↓
表面数据
↓
Lighting
```

这个保存表面数据的中间层，就是下一节要学习的：

# GBuffer

---

# 九、为什么 Deferred 必须保存数据？

假设 Fragment 已经计算出了：

```text
BaseColor
Normal
Roughness
Metallic
```

Forward 马上用这些数据计算光照，所以用完就可以继续往后走。

但 Deferred 说：

> 光照以后再算。

那就意味着：

> 现在得到的这些表面数据不能丢。

否则等到后面的 Lighting 阶段，就不知道这个屏幕位置原来的：

```text
BaseColor
Normal
Roughness
Metallic
```

是多少了。

因此 Deferred 必须：

```text
先计算表面数据
↓
保存起来
↓
后面重新读取
↓
Lighting
```

所以 GBuffer 并不是一个凭空出现的技术。

它是 Deferred Rendering 自然产生的需求：

> **既然光照要延后，那么表面信息就必须先保存。**

---

# 十、Forward 和 Deferred 的直观对比

## Forward

```text
Fragment
│
├─ BaseColor
├─ Normal
├─ Roughness
├─ Metallic
└─ Light
     ↓
  Lighting
     ↓
Scene Color
```

可以理解为：

> 一次把这个表面最终长什么样算出来。

---

## Deferred

```text
Fragment
│
├─ BaseColor
├─ Normal
├─ Roughness
└─ Metallic
     ↓
   保存
```

然后：

```text
保存的表面数据
+
Light
↓
Lighting
↓
Scene Color
```

可以理解为：

> 先记录表面，再决定灯光照上去以后是什么样。

---

# 十一、两种方式谁更好？

现在还不能简单回答：

```text
Forward 更好
```

或者：

```text
Deferred 更好
```

因为它们只是采用了不同的组织方式。

非常粗略地说：

### Forward

优点：

```text
流程直接
中间数据较少
```

代价：

```text
Material 和 Lighting 结合得更紧
```

### Deferred

优点：

```text
Material 和 Lighting 被拆开
后面的 Lighting 可以统一读取表面数据
```

代价：

```text
必须额外保存并读取表面数据
```

这一节只需要知道这些。

至于：

- 哪一种性能更高？
- 大量灯光时有什么区别？
- 为什么 Deferred 会占更多显存和带宽？
- 为什么某些项目选择 Forward？
- 为什么 UE 默认桌面路径常使用 Deferred？

这些都需要更多后续知识才能真正解释清楚，现在不展开。

---

# 十二、本节总结

Forward Rendering：

> **Fragment 被处理时，直接完成表面和光照计算，得到最终颜色。**

```text
Fragment
→ Material
→ Lighting
→ Scene Color
```

Deferred Rendering：

> **Fragment 先生成并保存表面数据，Lighting 在后面的阶段再计算。**

```text
Fragment
→ Material
→ Surface Data
→ Lighting
→ Scene Color
```

两者最核心的区别：

```text
Forward
= 现在算光照
```

```text
Deferred
= 先保存表面信息
  光照以后再算
```

而 Deferred 既然需要保存表面信息，就自然产生了下一节的核心概念：

# GBuffer

下一节研究：

> **GBuffer 到底是什么？为什么一组二维 Buffer 可以保存当前屏幕上的表面信息？**
