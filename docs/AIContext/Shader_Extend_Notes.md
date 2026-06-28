# Procedural Shader 疑点整理：Fade Curve / Value Noise + FBM / Animated Noise / Grid Anti-Aliasing

## 0. 先确认 fwidth 的理解

你的理解基本是对的：

> `fwidth` 的作用可以理解成：根据当前图案在屏幕上的密度，自动给程序线条的边缘增加一个屏幕空间尺度的过渡宽度。

所以在 grid shader 里，它确实会造成这种结果：

- **近处 pattern / grid 比较稀，屏幕像素足够多**：`fwidth(uv)` 比较小，smoothstep 的过渡区域很窄，所以视觉上接近硬边，像 step 一样清晰。
- **远处 pattern / grid 很密，一个 pixel 覆盖很大 UV 范围**：`fwidth(uv)` 变大，smoothstep 的过渡区域变宽，所以线条边缘会更柔和，甚至远处会变成灰化/淡化的平均结果。

但更准确地说，它不是简单“把线变粗”，而是：

> 它把程序线条边界从硬切变成一个大约覆盖当前 pixel footprint 的软过渡。

如果只看公式：

```hlsl
float aa = fwidth(distanceToLine);
float line = 1.0 - smoothstep(lineWidth, lineWidth + aa, distanceToLine);
```

那么当 `aa` 变大时，从 `lineWidth` 到 `lineWidth + aa` 的过渡范围确实变宽了。结果就是边缘更柔和，远处不会因为硬采样而一帧有线、一帧没线。

---

## 1. 为什么程序 grid 会锯齿 / 闪烁

你的理解也基本对：

> 一个 pixel 只采样一次，但远处 grid 密度很高的时候，一个 pixel 覆盖的区域内部可能同时包含线条区域和非线条区域，也就是 pattern 内部既有 1 也有 0。单次采样只能拿到其中一个点的结果，于是会出现不稳定的 0/1 跳变。

### 1.1 从采样角度理解

程序 grid 通常是这样的二值图案：

```text
线条区域：1
非线条区域：0
```

例如：

```hlsl
float line = step(distanceToLine, lineWidth);
```

这意味着：

```text
distanceToLine < lineWidth  → 1
distanceToLine >= lineWidth → 0
```

这是一个硬切边界。

近处时，一个 grid cell 在屏幕上可能有很多像素宽：

```text
近处：
一条 grid 线可能占 3~5 个 pixel
一个格子可能占 50 个 pixel
```

这时每个 pixel 采样一次也还可以，因为图案变化相对慢。

但远处时，一个 pixel 可能覆盖很多 grid 线：

```text
远处：
1 个 pixel 内部可能跨过多条 grid line
```

也就是说，这个 pixel 覆盖的真实区域里可能同时有：

```text
线条：1
背景：0
线条：1
背景：0
```

但是 fragment shader 对这个 pixel 只算一次。如果采样点刚好落在线上，就输出 1；如果落在线外，就输出 0。

于是相机稍微一动，采样点在 pattern 里的位置发生微小变化，就可能从：

```text
这一帧采到线 → 1
下一帧采到线外 → 0
```

视觉上就变成闪烁、噪点、摩尔纹、锯齿。

这就是 aliasing：

> 图案频率高于屏幕采样频率时，单次采样无法代表 pixel 覆盖区域内的平均结果，于是高频图案被错误采样成低频噪声或闪烁。

---

## 2. fwidth 到底在估计什么

`fwidth(x)` 是屏幕空间导数函数，大致等于：

```hlsl
fwidth(x) = abs(ddx(x)) + abs(ddy(x));
```

其中：

- `ddx(x)`：当前 pixel 到水平方向相邻 pixel，`x` 变化了多少；
- `ddy(x)`：当前 pixel 到垂直方向相邻 pixel，`x` 变化了多少；
- `fwidth(x)`：`x` 在大约一个屏幕 pixel 范围内变化了多少。

所以它可以理解成：

> 当前这个 pixel 在程序图案坐标里覆盖了多大的范围。

对于 grid shader：

```hlsl
float2 aa = fwidth(gridUV);
```

这句话的意思不是“gridUV 自己要变模糊”，而是：

> 当前一个屏幕 pixel 覆盖了多少 gridUV 范围。这个范围越大，说明 grid 在屏幕上越密，需要越宽的抗锯齿过渡。

---

## 3. 为什么 fwidth + smoothstep 能抗锯齿

硬 step 的问题是：边界从 0 到 1 是瞬间跳变。

```hlsl
float line = step(distanceToLine, lineWidth);
```

这会导致只要采样点跨过边界，结果就从 0 直接跳到 1。

抗锯齿的做法是：不要硬切，而是在大约一个 pixel footprint 的范围里做平滑过渡。

```hlsl
float aa = fwidth(distanceToLine);
float line = 1.0 - smoothstep(lineWidth, lineWidth + aa, distanceToLine);
```

这表示：

```text
distanceToLine < lineWidth
→ 基本在线里，line ≈ 1

distanceToLine > lineWidth + aa
→ 基本在线外，line ≈ 0

lineWidth 到 lineWidth + aa 之间
→ 从 1 平滑过渡到 0
```

这里的关键是 `aa` 不是固定值，而是来自 `fwidth`：

- 近处：`aa` 小，过渡窄，线条清晰；
- 远处：`aa` 大，过渡宽，线条柔和；
- 斜角/透视强的地方：`aa` 也会变大，减少斜线锯齿。

所以 fwidth 的本质是：

> 用屏幕空间导数估计当前 pixel 的采样 footprint，再用这个 footprint 作为 procedural edge 的平滑范围。

---

## 4. 这是不是等于“远处线条变宽”？

视觉上可以这么理解，但要更精确一点。

它不是把几何意义上的线宽直接变粗，而是把线条边界变成更宽的 coverage transition。

如果没有抗锯齿：

```text
线内：1
线外：0
边界：瞬间跳变
```

如果有 fwidth 抗锯齿：

```text
线内：1
边界附近：0.8 / 0.5 / 0.2
线外：0
```

所以它更像是在估计这个 pixel 被线条覆盖了多少。

例如一个 pixel 内部 30% 是线，70% 是背景，那么理想输出不应该是硬 0 或硬 1，而应该接近 0.3。`fwidth + smoothstep` 是一种便宜的近似 coverage 方法。

这就是为什么远处看起来更柔和：

> 它不再让高频 grid 在 0/1 之间乱跳，而是让它逐渐变成一个更稳定的中间灰度。

---

## 5. 为什么还需要 distance fade / horizon fade

`fwidth` 能解决线条边缘的锯齿，但不能解决所有远处问题。

当 grid 远到一个 pixel 内部覆盖大量线条时，就算做了 smoothstep，视觉上也可能变成一片灰噪声或闪烁区域。

所以程序 grid 通常还会额外加：

```hlsl
distFade = exp(-distance * fadeStrength);
```

让远处 grid 逐渐淡掉。

如果 grid 是通过 ray-plane intersection 算出来的，还会遇到另一个问题：当视线几乎平行于地面时，ray 和 plane 的交点会非常远。

交点公式是：

```text
t = (planeY - cameraY) / rayDir.y
```

如果 `rayDir.y` 非常小，`t` 会非常大。也就是说，当前 pixel 对应的地面交点可能在极远处，gridUV 变化会非常剧烈。

所以需要 horizon fade：

```hlsl
float horizonFade = smoothstep(0.0, horizonFadeWidth, abs(rayDir.y));
```

当 `abs(rayDir.y)` 很小时，说明视线接近平行地面，grid 就淡掉，避免远处交点不稳定导致的闪烁。

---

## 6. Fade Curve：为什么 noise 里要用它

很多 noise 都有一个共同结构：

```text
先找到当前点所在的 grid cell
再拿 cell 四个角上的随机值 / 随机梯度
最后根据当前点在 cell 内的位置做插值
```

假设当前点在 cell 内的局部坐标是：

```hlsl
float2 f = frac(p);
```

`f.x` 和 `f.y` 都是 0 到 1。最简单的插值就是直接用 `f`：

```hlsl
lerp(a, b, f.x)
```

但问题是：linear interpolation 在 cell 边界处斜率不连续。也就是说，值本身可能是连续的，但变化速度突然变了。视觉上就容易出现格子感、接缝感、不自然的转折；用在 displacement 时会出现不平滑的坡度。

fade curve 就是把原本的 `f` 先改造成一个更平滑的权重：

```hlsl
float fade(float t)
{
    return t * t * t * (t * (t * 6 - 15) + 10);
}
```

也就是经典 Perlin fade：

```text
6t^5 - 15t^4 + 10t^3
```

它的特点是：

```text
t = 0 时，fade = 0
t = 1 时，fade = 1

并且：
t = 0 和 t = 1 处的一阶导数是 0
t = 0 和 t = 1 处的二阶导数也是 0
```

直觉上就是：

> 它让插值在 cell 边界处“轻轻进入、轻轻离开”，而不是用一条直线硬接过去。

`smoothstep` 也是 fade curve 的一种：

```hlsl
smoothstep(0, 1, t)
// 等价于 t * t * (3 - 2t)
```

它保证边界处一阶导数为 0，已经比 linear 平滑。Perlin 的五次 fade curve比 smoothstep 更平滑，因为二阶导数也在边界处归零。

简单理解：

```text
linear：值连续，但斜率突然变
smoothstep：值和斜率更平滑
Perlin fade：值、斜率、弯曲变化都更平滑
```

在你的 Terrain shader 里，fade curve 用于 Perlin noise 的 cell 内插值，让高度场更连续。面试可以说：

> noise 里我用 fade curve 处理 cell 内坐标，而不是直接 linear interpolation。这样在格子边界处变化更平滑，尤其用在 vertex displacement 时，可以减少地形表面的格子接缝和不自然的斜率跳变。

---

## 7. Value Noise：它到底是什么，为什么还要叠 FBM

### 7.1 Value noise 和 Perlin noise 的区别

Value noise 是：

```text
每个 grid corner 存一个随机 scalar value
然后在 corner value 之间平滑插值
```

比如四个角：

```text
v00, v10, v01, v11
```

它们都是 0–1 的随机数。当前点的 noise 值就是四个随机数插值出来的。

所以它叫 value noise：因为格点上存的是“随机值”。

Perlin / gradient noise 是：

```text
每个 grid corner 存一个随机 gradient vector
当前点到 corner 的 offset vector 和 gradient 做 dot
再把 dot 结果插值
```

区别：

```text
Value noise：随机值 → 插值
Gradient / Perlin noise：随机方向 dot offset → 插值
```

直觉区别：

- value noise 更简单、更便宜；
- Perlin noise 更像自然连续的坡度；
- value noise 单层看起来有时更 blob / 更糊；
- Perlin / gradient noise 结构性更强。

### 7.2 为什么 value noise 还要叠 FBM？

你以前做简单 dissolve 用单层 Perlin，这完全合理。单层 noise 可以做 dissolve，因为 dissolve 很多时候只需要一张 0–1 mask：

```hlsl
clip(noise - threshold);
```

只要 noise 有连续随机区域，它就能溶解。

但单层 noise 的问题是：

> 它只有一个尺度。

也就是说，整张图只有一种大小的 blob / pattern：

```text
scale 小：大块、大团、很平
scale 大：细碎、密集、但没有大结构
```

如果你想要更自然、更丰富的纹理，一般需要同时有：

- 大形；
- 中等变化；
- 小细节。

FBM 就是干这个的。

FBM = Fractal Brownian Motion。在 shader 里它通常就是：

```hlsl
float fbm(float2 p)
{
    float value = 0;
    float amplitude = 0.5;
    float frequency = 1.0;

    for (int i = 0; i < octaveCount; i++)
    {
        value += noise(p * frequency) * amplitude;
        frequency *= lacunarity;
        amplitude *= persistence;
    }

    return value;
}
```

它的意思是：

```text
第 1 层：低频，大形
第 2 层：更高频，中细节
第 3 层：更高频，小细节
第 4 层：更碎的细节
```

最后叠起来：

```text
大形 + 中形 + 小细节 = 更自然的 procedural texture
```

所以不是“单层不对”，而是用途不同。

单层 Perlin dissolve 适合：

- 简洁风格；
- 低频溶解；
- 卡通块状消失；
- 不需要太多内部细节。

FBM dissolve / energy noise 适合：

- 能量体；
- 火焰；
- 魔法边缘；
- 腐蚀；
- 云雾；
- 复杂发光核心。

项目里可以这样说：

> FakeSunHDR 用的是更轻量的 value noise + FBM。Value noise 本身只是平滑随机值，单层会比较平，所以用 FBM 叠多个尺度，让发光核心和 dissolve band 有更复杂的破碎边缘。Terrain 则用 FBM 叠加多层 Perlin，让地形同时有大形起伏和局部细节。

---

## 8. Animated Noise：为什么不只是 `uv += time`

最简单的 animated noise 是：

```hlsl
float2 uv = input.uv;
uv += _Time.y * float2(0.1, 0.0);
float n = noise(uv);
```

这会让 noise 整体向一个方向平移。它叫 scrolling noise。

优点：

- 简单；
- 成本低；
- 容易控制方向和速度。

缺点：

- 很容易看出来整张图在平移；
- pattern 本身没有变形；
- 循环感强；
- 像贴图在滑，不像能量在流。

更丰富的 animated noise 常见做法是叠两层或多层 noise，每层：

- scale 不同；
- speed 不同；
- direction 不同；
- 权重不同。

例如：

```hlsl
float n1 = fbm(uv * scale1 + time * dir1 * speed1);
float n2 = fbm(uv * scale2 + time * dir2 * speed2);

float n = n1 * 0.7 + n2 * 0.3;
```

这样得到的不是单纯整体平移，而是：

```text
大形在慢慢走
小细节用不同方向和速度流动
两层相互干涉
视觉上更像内部在变化
```

更高级一点还有 domain warping：

```hlsl
float2 warp = float2(
    noise(uv + time),
    noise(uv + 17.3 + time)
);

float n = noise(uv + warp * warpStrength);
```

这不是只移动 noise，而是用另一张 noise 去扭曲采样坐标。视觉上更像图案内部被扭曲、卷动、流体化。

项目里可以这样说：

> Terrain 的 animated noise 是 scrolling FBM，用 object-space xz 加上 time-based offset，让地形表面像在持续生成或流动。FakeSunHDR 则用两层不同 scale / speed / direction 的 animated FBM，让它不像一张贴图简单平移，而是有内部能量流动感。

---

## 9. 面试压缩版

> 这些 supporting shaders 里主要用了几类 procedural shader 技术。Noise 部分，fade curve 是为了让 cell 内插值在边界处更平滑，避免程序噪声出现格子接缝；FBM 是把多层不同频率和振幅的 noise 叠起来，让单层 noise 从简单 blob 变成有大形和细节的纹理。Terrain 用 FBM 做地形起伏，FakeSunHDR 用 value noise + FBM 做发光能量纹理。  
>   
> Animated noise 则是对 noise 采样坐标加 time offset，或者叠加多层不同方向和速度的 noise，让图案不是静态的。Grid shader 里，程序线条容易因为频率过高产生 aliasing：一个 pixel 内部可能同时覆盖线和背景，但 shader 只采样一次，于是结果会在 0/1 之间不稳定跳变。我的处理是用 fwidth 估计当前 pixel 覆盖的 UV 范围，再用 smoothstep 给线条边缘一个基于 pixel footprint 的过渡宽度；近处过渡窄、线条清晰，远处过渡宽、结果更接近稳定的 coverage 平均。再结合 distance fade 和 horizon fade，避免极远处或视线接近平行地面时 grid 闪烁。

---

## 10. 追问短答

### Q：fade curve 解决什么问题？

> 解决插值在 grid cell 边界处变化不平滑的问题。它把线性权重变成边界处导数更平滑的权重，让 noise 不容易出现格子接缝。

### Q：value noise 和 Perlin noise 区别？

> Value noise 是格点上存随机 scalar value，然后插值；Perlin / gradient noise 是格点上存随机 gradient vector，用 gradient 和 offset 的 dot product 再插值。Value noise 更简单，Perlin 更像连续坡度。

### Q：为什么 FakeSun 用 value noise 还要 FBM？

> 单层 value noise 只有一个尺度，会比较平。FBM 叠加多个频率后，可以同时有大形和小细节，让能量核心和 threshold band 更丰富。

### Q：animated noise 是不是就是 uv 加 time？

> 最简单是这样，但更丰富的做法是叠多层不同 scale、speed、direction 的 noise，甚至用 domain warping 扭曲采样坐标。这样比整张图单纯平移更自然。

### Q：fwidth 是什么？

> fwidth 是屏幕空间导数，近似表示一个 pixel 范围内某个变量变化多少。用在程序线条里，可以根据当前屏幕尺度自动调整 smoothstep 过渡宽度，减少锯齿和闪烁。

### Q：为什么 grid 会闪烁？

> 因为远处 grid 太密时，一个 pixel 内部可能同时覆盖线和背景，但 fragment shader 只采样一次。采样点稍微移动就可能从线内变成线外，输出在 1 和 0 之间跳，所以会闪烁。

### Q：为什么 grid 还要 distance fade / horizon fade？

> fwidth 解决边缘抗锯齿，但远处 grid 太密时仍然会变成噪声。distance fade 让远处淡掉；horizon fade 处理视线几乎平行地面时 ray-plane intersection 交点过远导致的不稳定。

