1. Terrain.shader：主系统
Terrain shader 是 God Intern 的场景主体。它用 object-space xz 采样 FBM noise，在 vertex stage 做 y 方向位移；fragment stage 里根据法线区分 top/side，用噪声控制顶面颜色，用 object-space 高度做侧面渐变和顶部 crust band，再结合 fake sun 做顶面 toon lighting 和侧面 ndotl lighting。

Terrain 里先实现了一个 2D gradient Perlin noise：你代码里用的是 gradient noise 思路：每个格点通过 hash 生成一个 2D 梯度向量，然后当前点和四个角的 offset 做 dot，最后用 smootherstep-like fade curve 插值。

2.2 为什么用 fade curve
不是直接用 linear frac 插值，而是用 fade curve 处理 cell 内坐标。它会让插值在格子边界处导数更平滑，所以噪声视觉更连续。

2.3 FBM：多层 Perlin 叠加
FBM 是把多层不同频率和振幅的 noise 叠起来。低频层决定大形，高频层增加细节。Lacunarity 控制频率增长，Persistence 控制振幅衰减，Octaves 控制层数。这个 shader 里 FBM 同时用于 vertex displacement 和顶面颜色变化。

2.4 动态采样：scrolling noise
我用 object-space xz 作为 noise sample position，这样噪声跟地形模型绑定；再加上 time-based scroll offset，让地形像在持续流动或生成。NoiseScale 控制地形纹理大小，ScrollSpeed 和 ScrollDirection 控制运动方向和速度。

2.5 Vertex displacement：只影响顶面区域
我不是直接把所有顶点都按 noise 位移，而是用 object-space y 做一个 displacement mask。接近上表面的顶点影响更强，侧面和底部影响较弱，这样地形像顶部在流动，而整体体积还保持稳定。HeightThreshold 和 BlendRange 是为了控制位移从哪一段开始，以及过渡是否柔和。

3. Terrain normal / lighting
因为顶面位移是 shader 里程序生成的，原模型 normal 不能反映位移后的坡度。所以我在 vertex stage 附近采样四个高度，用左右/上下高度差近似高度场梯度，再构造一个 terrain normal，用来做顶面的 fake lighting。normal 可以理解成高度场切线叉乘后的近似结果。我的写法是把高度差塞进 x/z，y 用 2eps 保持向上分量。

3.2 原 normalWS 和 terrainNormalWS 的区别
你的 fragment 里同时用了：
我保留了两套 normal：原始 mesh normal 用来区分 top/side，因为它更稳定；procedural terrain normal 用来给顶面 fake lighting，让顶面光照跟噪声坡度变化产生关系。

4. Top / Side material separation
4.1 topMask：用 normalWS.y 判断顶面/侧面
我用 world normal 的 y 分量判断 surface 是顶面还是侧面。朝上的面 topMask 高，垂直侧面 topMask 低。用 smoothstep 而不是 step，是为了让 top/side 材质有可控过渡。

4.2 顶面颜色：noise 驱动 low/high color lerp
顶面不是贴图，而是用同一套 noise 值在 low/high color 之间插值。TopNoisePower 用来调整颜色分布，比如让低色区域占比更大或高色区域更明显。

5. Top toon lighting
顶面光照不是接 Unity 真实 light，而是用 fake sun 的 world position 自己算方向。每个 fragment 用 normalize(lightPosition - positionWS) 得到 light direction，再和程序 normal 做 dot。之后用 smoothstep 把 ndotl 变成一段 toon band，在 shadow color 和 light color 之间插值，形成更风格化的受光层次。

6. Side shading / crust band
侧面不使用 noise 主导，而是用 object-space y 做 vertical gradient。这样侧面更稳定，有体积感，不会和顶面一样太碎。
Crust band 是侧面顶部的一圈边缘色。我不是用固定高度硬画一条线，而是把当前 xz 对应的动态顶部高度传到 fragment，用 fragment 的 object-space y 和 dynamicTopY 的距离算 mask。这样 crust 会跟随顶部 displacement 弯曲，而不是一条死板的水平线。

7. Side fake lighting
侧面光照用 directional-style fake light，直接 dot(normalWS, lightDirection)，再加 ambient。顶面则用 fake sun position 做 per-fragment direction。这样侧面保持稳定体积感，顶面有更明显的光照层次。

8. Terrain 最终合成
Terrain shader 的 vertex stage 用 object-space xz 采样 scrolling FBM noise，并通过 height threshold 控制 displacement influence，只让上方区域产生主要位移。同时我用周围四次高度采样近似 terrain normal，给顶面 fake lighting 使用。
fragment stage 里先用 normalWS.y 做 top/side mask。顶面颜色由 noise 在 low/high color 间插值，再用 terrainNormalWS 和 fake sun position 做 toon light band。侧面用 object-space y 做 vertical gradient，并根据 dynamicTopY 和当前 fragment y 的距离生成顶部 crust band。最后侧面再乘一个简单 fake ndotl lighting，然后用 topMask 在 topColor 和 sideColor 之间混合。
这个 shader 的重点不是物理真实，而是把 FBM 位移、程序 normal、top/side mask、toon lighting、side gradient 和 crust band 组合成一个可控的风格化地形。



9. FakeSunOrbitController：fake light 数据来源
9.1 轨道运动
也就是 fake sun 围绕 orbitOrigin 做椭圆轨道运动。代码里支持 Play Mode 自动旋转，也支持 Edit Mode preview。

9.2 MaterialPropertyBlock 传参
FakeSunOrbitController 不是 gameplay 系统，而是 terrain lighting 的控制脚本。它让一个 fake sun 物体沿 orbitOrigin 周围运动，然后通过 MaterialPropertyBlock 把 light direction 和 light position 传给 terrain shader。这样 shader 不需要依赖 Unity Light，也可以在 Edit Mode 和 Play Mode 下预览风格化光照变化。

9.3 为什么用 MaterialPropertyBlock
MaterialPropertyBlock 可以在不实例化/修改共享材质的情况下给某个 Renderer 设置 shader 参数，适合这种展示用的 per-renderer fake light 控制。




10. FakeSunHDR.shader：发光太阳 / dissolve VFX
FakeSunHDR 是 fake sun 物体的视觉 shader，用 animated noise 生成类似溶解/能量核心的发光效果。它不是项目主技术点，主要是配合 terrain fake lighting 的可视化对象。

这个 shader 用的是比较轻量的 value noise + FBM。Value noise 在 cell corner 生成随机值，再平滑插值；FBM 叠多层频率，形成更丰富的能量纹理。

它采样了两套不同 scale/speed/direction 的 noise，避免单层噪声太机械，让溶解区域有更复杂的流动。

它本质是用 noise threshold 做 procedural dissolve。不同于直接 clip，我没有丢弃像素，而是把 threshold 附近作为 band，低于 threshold 的区域作为 core，高于 threshold 的区域作为 body，再分别给 HDR color 和 emission。这样能形成一个有核心、有发光边带、有主体颜色的 fake sun。




11. InnerBox.shader：内部空间 grid / stencil 辅助
对每个 fragment，从 camera 到当前 box surface 形成一条 object-space ray，求这条 ray 打到内部地板平面的位置，再用这个 hit point 的 xz 坐标生成 grid。它不是直接用模型 UV 贴 grid，而是在 fragment 里做 object-space ray-plane intersection。这样无论 box 表面怎么投影，grid 都像是落在内部 floor plane 上。

11.3 Horizon fade
当视线接近平行地面，交点会非常远，grid 会变得不稳定，所以用 horizon fade 把它淡掉。

11.4 GridLine + fwidth 抗锯齿
grid line 用 frac 生成周期性线条，用 fwidth 做屏幕空间抗锯齿。fwidth 可以根据当前像素里 uv 变化速度调整 smoothstep 的过渡宽度，避免远处 grid 线严重闪烁或 aliasing。

11.5 Major / minor grid

11.6 Distance fade
远处 grid 用 exponential fade 衰减，这样内部空间不会被无限延伸的线条占满，视觉更干净。


