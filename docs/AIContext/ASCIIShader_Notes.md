God Intern 复习结构
A. 高层数据流：这个项目到底怎么跑
B. 算法层：Gaussian / DoG / Sobel / glyph classification
C. RenderGraph / API 层：它怎么被 Unity 管起来
D. Debug case：从“做出来”变成“能测试/能排查”



1.Unity URP RenderGraph 后处理管线
先从当前 camera color 中提取亮度和边缘信息，再通过多阶段 HLSL pass 生成中间结果，最后用 Compute Shader 按 8×8 cell 选择普通 ASCII 字符或边缘字符，并把结果合成回 camera color。项目中还加入了 debug view、stencil-aware composite 和 ScriptableObject preset workflow，用来支持视觉调试和参数复现。

RendererFeature 负责暴露资源和参数、创建 material 和 pass；RenderPass 在每帧的 RecordRenderGraph 里读取 camera color，创建中间 TextureHandle，注册 blit pass 和 compute pass，最后根据 render mode 把结果全屏或 stencil-aware 地合成回 camera color。

1.1 ASCIIRendererFeature.cs：配置入口 / Inspector 面板 / pass 注册
Feature 层主要做 Unity/URP 的接入和配置管理。它通过 serialized settings 暴露 compute shader、ASCII atlas、edge atlas、DoG/Sobel 参数、debug toggles、render mode 和 injection point。Create 里创建 material 和 render pass；AddRenderPasses 里判断资源是否完整、当前 camera 是否需要应用效果，然后更新 pass settings 并 enqueue pass。

1.2 ASCIIRenderPass.cs：真正的 RenderGraph 管线组织
这个文件是核心。它的职责是：
-从 URP frame data 里拿当前 camera color；
-根据 camera descriptor 创建所有中间 texture；
-import 外部 atlas texture；
-把 C# settings 写入 material；
-注册一连串 raster blit pass；
-注册 compute pass；
-根据 debug view / render mode 决定最终输出；
-如果是 stencil 模式，就绑定 depth/stencil attachment 做局部合成。

1.3 ASCIIURP.shader：HLSL fullscreen pass 集合
它相当于 RenderGraph blit chain 的实际图像处理执行者。RenderPass 只是注册 “哪个 source → 哪个 destination → 用 shader 的第几个 pass”；shader 里才是真正对像素做处理。

1.4 ASCIICompute.compute：cell 级字符分类与最终 ASCII 输出
Compute Shader 是最后的分类阶段：
-输入 Sobel texture；
-输入 downscale 后的颜色/亮度 texture；
-输入普通 ASCII atlas；
-输入 edge ASCII atlas；
-输出 _ASCII_ResultTex；
-每个 8×8 thread group 对应一个 ASCII cell；
-最终决定这个 cell 用 fill glyph 还是 edge glyph。
前面的 HLSL pass 负责准备亮度和边缘信息，Compute Shader 负责把这些信息转换成最终字符化画面。


2. Feature 层工作流：URP 接入范式
2.1 Serialized settings 暴露参数
ASCIIRendererFeature 里有一个 [Serializable] ASCIISettings，里面按 Header 分了几组
面试说法：
我把效果需要的资源和调试参数放在 RendererFeature 的 settings 里暴露给 Inspector，这样测试不同视觉配置时不用改代码。比如 atlas、compute shader、edge threshold、fill/edge color、debug view、render mode、injection point 都是可配置的。
这里可以顺便接美术测试：
这也方便做视觉验证，因为同一个效果可以快速切换不同参数和 debug mode。

2.2 Create：创建 material 和 render pass
Create() 里做三件事：
先释放旧 material；
-如果 asciiShader 为空，pass 直接置空；
-用 CoreUtils.CreateEngineMaterial(asciiShader) 创建材质；
-new 一个 ASCIIRenderPass(asciiMaterial, settings)；
-设置 asciiPass.renderPassEvent = settings.renderPassInjectionPoint。
面试说法：
Create 是 RendererFeature 初始化或 Inspector 参数变化后重新创建资源的地方。我在这里创建 full-screen material 和 ASCIIRenderPass，并把 injection point 传给 pass。
你不用说 “Create 每帧执行”，因为不是。要强调它是 feature 初始化 / 重建阶段。

2.3 AddRenderPasses：每帧判断是否 enqueue
AddRenderPasses() 做的是 gatekeeping：
-如果 pass/material 缺失，return；
-如果 settings 或 compute/atlas 资源缺失，return；
-判断 camera type；
-Game camera 一定跑；
-SceneView camera 根据 applyToSceneView 决定是否跑；
-更新 renderPassEvent；
-asciiPass.SetSettings(settings)；
-renderer.EnqueuePass(asciiPass)。
面试说法：
AddRenderPasses 是每帧 URP 问这个 feature 要不要加 pass 的地方。我会先检查资源完整性和 camera type，避免在没有 compute shader 或 atlas 的情况下注册 pass。然后把最新 settings 传给 pass，并 enqueue 到 renderer。

2.4 Preset workflow：保存/恢复参数
我加了 ScriptableObject preset workflow，用来保存和恢复 RenderFeature 的参数组合。因为这个项目调试视觉效果时参数很多，如果只靠 Inspector 手动记，很难复现某个版本。preset 可以帮助我在最终效果、debug view 和不同 render mode 之间快速切换。


4. RecordRenderGraph 里实际做了什么
4.1 前置检查
RecordRenderGraph() 开头会检查：
material / settings / compute 是否存在；
ascii atlas / edge atlas 是否存在；
是否支持 compute shader；
是否 active target 是 backbuffer；
cameraColor 是否 valid。
面试说法：
这些检查是为了避免在资源不完整或目标不可写的时候注册 RenderGraph pass，尤其是 active target 是 backbuffer 时不能直接按这个方式处理。

Back Buffer 是最终要显示到屏幕的缓冲区，不是普通中间 RT。我的后处理需要读写 activeColorTexture，所以如果 active target 已经是 Back Buffer，就跳过，避免 RenderGraph 对最终屏幕目标做不安全的读写。

4.2 从 URP frameData 取资源
在 RenderGraph 版本里，不是自己去找 camera target RT，而是从 URP 的 frameData 里拿当前 active color/depth texture。color 用来做后处理输入和最终输出，depth 主要在 stencil-aware composite 时作为 depth/stencil attachment 绑定给 raster pass。

4.3 创建中间 TextureHandle
用 CreateRenderGraphTexture() 统一创建临时贴图，它基于 camera descriptor 修改

4.4 Import 外部 atlas texture
asciiTex 和 edgeTex 是外部 Texture，不是 RenderGraph 自己创建的临时 RT。
ASCII atlas 和 edge atlas 是 Inspector 里指定的外部资源，不是每帧临时生成的 RenderGraph texture，所以我先用 RTHandle 包一层，再 import 到 RenderGraph，之后 compute pass 就可以声明读取这两个 TextureHandle。

Texture
= Unity 普通纹理资源
= 例如 Inspector 里拖进去的 ASCII atlas PNG/Texture2D

RTHandle
= SRP 里的 render target handle / texture wrapper
= 可以包装已有 texture，也可以管理可缩放 RT

TextureHandle
= RenderGraph 内部使用的资源句柄
= 用来声明某个 pass 读/写这张 texture

ASCII atlas 和 edge atlas 是项目资源，不是 RenderGraph 每帧创建的临时 texture。RenderGraph 内部使用 TextureHandle 管理资源依赖，所以我先用 RTHandle 包装外部 Texture，再通过 renderGraph.ImportTexture 把它引入当前帧的 RenderGraph。这样 compute pass 就可以显式声明它读取 atlas。



5. RenderGraph pass 注册范式
5.1 Raster blit pass 的范式
UseTexture 声明读取 source，SetRenderAttachment 声明写入 destination，SetRenderFunc 在 RenderGraph 后续执行时调用 Blitter.BlitTexture。
RenderGraph pass 的重点是先声明资源依赖，再提供执行函数。比如一个 blit pass 里，我会把 source 存进 passData，用 builder.UseTexture 声明这个 pass 读它；用 SetRenderAttachment 声明 destination 是这个 pass 的 color target；最后 SetRenderFunc 里才真正把 Blitter.BlitTexture 命令写进 command buffer。
所以它不是直接 Blit，而是先注册一个 pass 定义：passData 保存执行所需的数据，builder 声明依赖，SetRenderFunc 记录真正的 GPU command。

5.2 为什么需要 PassData
因为 SetRenderFunc 是之后由 RenderGraph 调用的，不能依赖当前函数里的临时局部变量，所以需要把执行阶段用到的数据打包进 PassData。Raster pass 和 compute pass 需要的数据不同，所以我分别定义了 BlitPassData 和 ComputePassData。

5.3 有第二张输入 texture 怎么办
普通 BlitTexture 默认只传一个 source。如果某个 shader pass 还需要额外采样一张 texture，我会在 RenderGraph 里显式声明它是 read dependency，然后在 SetRenderFunc 里用 SetGlobalTexture 绑定给 shader。因为这是全局 shader state 修改，所以需要 AllowGlobalStateModification。

5.4 Compute pass 的范式
Compute pass 和 raster pass 的区别是它不通过 SetRenderAttachment 写 color target，而是用 UseTexture 声明 result 是 write texture，并在 SetRenderFunc 里通过 SetComputeTextureParam 绑定 UAV 输出。最后 DispatchCompute，让 compute shader 写入 _ASCII_ResultTex。
这里可以补一句：
因为 result 是 compute shader 输出，所以创建 texture 时要 enableRandomWrite。

5.5 Stencil-aware composite 怎么接到 RenderGraph
stencil 不是主算法，而是最后 composite 阶段的 masking。关键是 raster pass 如果要做 stencil test，需要把当前 camera depth/stencil attachment 绑定进来。所以我在这个 helper 里除了 color destination，也通过 SetRenderAttachmentDepth 绑定 activeDepthTexture，然后 shader pass 里的 stencil state 才能生效。

6. Settings 从 Inspector 到 Shader / Compute 的数据路径
6.1 C# settings → Material → HLSL shader pass
HLSL fullscreen pass 的参数是通过 material uniform 传进去的。Feature 持有 settings，Pass 每帧拿最新 settings，然后在 RecordRenderGraph 开始时 UpdateSettingsToMaterial，把 sigma、tau、threshold、stencilRef 这些写进 material。

6.2 C# settings → Compute shader
Compute Shader 不经过 material，所以相关参数是在 compute pass 的执行函数里通过 command buffer 的 SetComputeXXXParam 绑定。这样 raster shader 和 compute shader 的参数路径是分开的：HLSL blit pass 走 material，compute pass 走 SetComputeTextureParam / SetComputeIntParam。


7. Debug view / render mode 在 A 层怎么说
debug view 不是另写一套管线，而是在 final composite 前把 finalSource 切到某张中间 texture，比如 DoG 或 Sobel。这样我可以直接把中间结果输出到屏幕，判断问题发生在最终合成之前还是之后。
render mode 只影响最后 composite，不改变前面的 ASCII 生成流程。这样同一套中间结果可以用于全屏效果，也可以用于局部 masking 或 debug 展示。





B 层总览：两条信息流在 cell 级别融合
ASCII 生成分两条信息流：亮度流和边缘流。亮度流用来决定普通字符密度，边缘流用 Gaussian/DoG 和 Sobel 得到边缘强度与方向。最后 Compute Shader 按 8×8 cell 统计这个 cell 里是否有稳定的主边缘方向。如果有，就从 edge atlas 里选对应方向的字符；如果没有，就根据 downscaled luminance 从普通 ASCII atlas 里选字符。

2. Gaussian Blur：先做平滑，避免边缘检测太碎
Gaussian blur 是一种按距离加权的平滑滤波，离中心越近的像素权重越高，越远权重越低。在这个项目里它不是最终视觉效果，而是边缘检测前的预处理。因为直接对原图 Sobel 会把很多材质纹理、噪点和小细节都当成边缘，先 blur 可以让后续 DoG/Sobel 更偏向画面结构。
2D Gaussian 是可分离的，所以可以拆成 horizontal blur 和 vertical blur 两个 1D pass。这样采样次数比完整二维卷积少很多，性能更好，也更符合后处理里常见的 separable blur 写法。
gaussianKernelSize：采样范围，越大考虑周围像素越多，成本也更高。
stdev / sigma：模糊扩散范围，越大越平滑，细节越少。
太小：边缘更细，但噪声更多。
太大：边缘更干净，但可能丢失小结构。

3. DoG：用两个模糊结果相减，突出结构边缘
两个不同模糊尺度的图相减，会留下那些在尺度变化中差异明显的位置，也就是亮度变化比较强的结构。

4. Threshold / Invert：把连续 DoG 结果变成边缘 mask

5. Sobel：从 edge mask / luminance structure 估计方向
Sobel 算的是梯度方向，梯度方向是亮度变化最快的方向，它和视觉边缘线本身是垂直关系。所以 Gx 强通常说明有竖向边，Gy 强通常说明有横向边。

6. Downscale：为什么还要 1/8 的颜色/亮度输入
ASCII 字符不是逐像素输出，而是按 glyph cell 输出。我的 glyph 是 8×8，所以我把 camera color 降采样到 1/8，让 compute shader 能用 cell 级别的颜色/亮度去决定普通 fill glyph 的密度和颜色。

7. Compute Shader：8×8 cell classification
我把 glyph 的像素尺寸和 compute thread group 对齐。每个 8×8 thread group 处理屏幕上的一个 8×8 cell，最后这个 cell 选择一个 ASCII glyph。这样 cell 尺寸、glyph 尺寸和 compute group 是一致的。
Compute shader 每个 thread group 对应一个 cell，group 内 64 个 thread 分别看这个 cell 内对应像素的边缘信息。它会统计这个 cell 里四种边缘方向的票数，找 dominant direction。如果某个方向票数超过 edgeThreshold，就认为这个 cell 有稳定边缘，用 edge atlas；否则说明这个 cell 没有一致结构，就用 downscaled luminance 去选普通 ASCII glyph。

10. Debug toggles 在 B 层怎么讲
debug view。比如 viewDog / viewSobel 可以直接看中间 edge preprocessing 和 Sobel 输出；debugEdges / viewQuantizedSobel 可以看方向量化是否正确；noEdges / noFill 可以单独关闭边缘或填充，验证最终问题来自 tone flow 还是 edge flow。




C 层：RenderGraph / API 层深挖
我对 RenderGraph 的理解是，它把以前手写 RT 生命周期和一串 Blit 的流程，改成了“声明资源依赖图”。在我的 ASCII pass 里，每个 pass 都会明确声明读哪些 TextureHandle、写哪个 TextureHandle，真正的 Blit 或 DispatchCompute 是写在 SetRenderFunc 里，由 RenderGraph 后续执行。RenderGraph 是把 pass 和 resource dependency 显式声明出来的渲染组织方式。对我这个项目来说，它的价值是让多阶段后处理里每张中间 texture 的读写关系更清楚，也方便我在 Frame Debugger / RenderGraph 里检查每个阶段输出是否正确。

C1. RendererFeature 和 RenderPass 的分工
1. RendererFeature：接入 URP + 暴露配置 + 决定是否 enqueue
Feature 负责“要不要加 pass”，RenderPass 负责“加了以后每帧具体声明哪些 RenderGraph pass 和 texture 依赖”。所以 RenderGraph 相关的主要逻辑都在 ASCIIRenderPass.RecordRenderGraph()。

C2. RecordRenderGraph 的标准结构
Step 1：资源合法性检查
Step 2：取 URP 当前帧资源
在 RenderGraph 版本里，我不是自己拿 camera target RT，而是从 URP frameData 里取当前 active color/depth texture。color 是后处理输入，depth/stencil 在局部合成时绑定给 raster pass
Step 3：准备 descriptor
Step 4：settings 写入 material
HLSL fullscreen pass 的参数走 material uniform，所以在 RecordRenderGraph 开头把 settings 上传到 material。Compute shader 的参数不走 material，而是在 compute pass 的 SetRenderFunc 里用 SetComputeXXXParam 绑定。
Step 5：import 外部 atlas
atlas 是 Inspector 拖进去的外部 Texture，不是 RenderGraph 创建的临时 RT。为了让 RenderGraph 能追踪 compute pass 对 atlas 的读取，我先用 RTHandle 包装，再 import 成 TextureHandle。
Step 6：创建中间 TextureHandle
Step 7：注册 pass chain

C3. Raster blit pass 的 RenderGraph 范式
对 raster blit pass，我用 AddRasterRenderPass 创建 pass 和 builder，然后把执行阶段需要的数据放进 PassData。builder 负责声明资源依赖，比如 source 是 read、destination 是 render attachment。最后 SetRenderFunc 里才记录实际的 fullscreen blit command。
如果某个 shader pass 需要第二张输入贴图，比如 SobelHorizontal 需要额外采样 luminance，我会额外 UseTexture(extra, Read)，然后在执行函数里 SetGlobalTexture 绑定给 shader。因为修改了 global shader state，所以要允许 global state modification。

C4. Compute pass 的 RenderGraph 范式
Compute pass 和 raster pass 最大区别是它不通过 SetRenderAttachment 写输出，而是通过 UAV/random write texture 写 result。所以 result 创建时要 enableRandomWrite，然后 compute pass 里 UseTexture(result, Write)，执行时用 SetComputeTextureParam 绑定 _Result，最后 DispatchCompute。



Gaussian / Sobel 数学提炼版
1.1 高斯函数本质
它的形状是中心最高、越远越低的 bell curve。
在图像处理中，Gaussian blur 就是对邻域像素做加权平均：

sigma 控制理论上的模糊范围，kernel size 控制实际采样窗口。sigma 大但 kernel 太小会截断高斯尾部；kernel 大则更准确但采样成本更高。

Gaussian blur 可以拆成两个一维 pass，因为二维 Gaussian 是 separable 的。这样从 N² 次采样降到 2N 次采样，适合 fullscreen post-process。

Gaussian 在这里是 edge detection 的预处理，用平滑降低高频噪声，让后面的 DoG/Sobel 更偏向结构轮廓，而不是每个材质纹理都出边。

2. DoG 数学提炼
两个不同尺度的 blur 可以理解成：
小尺度 blur：保留局部结构；
大尺度 blur：只保留更大范围的低频变化；
相减后：共同的低频部分被抵消，局部变化强的地方留下来。
所以 DoG 是一种 band-pass / edge-enhancing filter。

3. Sobel
Sobel 的 Gx 可以看作 x 方向差分和 y 方向平滑的组合，Gy 则是 y 方向差分和 x 方向平滑的组合。这样做的意义是让梯度估计基于局部 3×3 neighborhood，而不是只依赖单行/单列的两个像素差。它在估计某个方向梯度的同时，会在垂直方向上做加权平均，从而降低单像素噪声和高频纹理对方向判断的影响。

Gaussian blur 在这里是边缘检测前的平滑预处理。数学上它是用高斯权重对邻域像素做加权平均，sigma 控制模糊尺度。因为二维 Gaussian 可分离，所以我把它拆成 horizontal 和 vertical 两个 pass，降低采样成本。
DoG 是两个不同 sigma 的 Gaussian 结果相减，用小尺度和大尺度的差异突出结构边缘。因为它是相减，结果可能有负值，所以中间 texture 要用 signed float。
Sobel 则是在 DoG/edge mask 后估计离散亮度梯度，用 Gx/Gy 两个 3×3 kernel 得到 x/y 方向变化量，再用 magnitude 表示边缘强度，用 atan2 得到方向。需要注意 Sobel 的方向是梯度方向，和视觉边缘线方向垂直。最后我把方向量化成四类，供 compute shader 在 8×8 cell 内做主方向投票。



D. Debug case：从“做出来”变成“能测试/能排查”
Debug Case 1：GameView 某个 diagonal edge direction 丢失
最开始的现象是：SceneView 里 diagonal edge 是正常的，但 GameView 里某一个斜向方向的边缘明显丢失。本来应该是一条 diagonal edge 的地方，在最终 ASCII 结果里经常被拆成 horizontal 和 vertical 的组合。

因为 SceneView 看起来正常，所以一开始没有优先怀疑 render target format 或 URP camera target 差异。最直观的怀疑是compute 阶段 voting 算法不理想被scene view和game view分辨率差异放大。尤其是 diagonal edge 在最终结果里像是被拆成 horizontal 和 vertical，我当时以为可能是 edge mask 太宽，或者 8×8 cell 内方向投票不稳定，导致 dominant edge 被投错。

我原来的 debugEdges 只能显示每个 8×8 cell 投票后的 common edge color，所以它能看到最终 cell 分类错了，但不能证明错误发生在 voting 之前还是 voting 过程中。为了排除 voting，我补了一个更底层的 debug view。原来的 debugEdges 是显示每个 8×8 cell 投票后的 common edge color；新的 debug view 则显示 cell 内 64 个 pixel 在 voting 前各自的方向颜色。这样我可以直接看 Sobel/sorting 输出给 voting 的原始方向数据。对比之后我发现，不是 voting 把 diagonal 投错了，而是在 voting 之前，那个方向的 pixel-level direction 就已经缺失了。所以问题更早，应该在 Sobel 方向计算或者方向 sorting / quantization 阶段。

继续看以后我发现它不是随机少一些边，而是某个带正负符号关系的方向几乎整体丢失。因为方向分类依赖 Sobel 的正负梯度和 angle，所以我开始怀疑是不是中间 texture format 没有保留负值。最后定位到GameView 下中间 render texture 继承到了 unsigned HDR format，不支持负值；但 DoG 是相减，Sobel gradient / angle sorting 都依赖正负信息。负值被截断后，某些方向分类自然就全丢了。修复后我把 DoG、Sobel 相关中间 RT 显式改成 signed float format，保证负值不会被截断。之后 GameView 里的 diagonal direction 恢复正常，pixel-level debug 和 cell-level voting debug 也能对上。


D2. Debug Case 2：Compute 阶段亮度对比变灰
旧版 ASCII pipeline 参考 built-in render pipeline 的实现，里面有一个 Pack Luminance pass。它会先 extract luminance，然后把原图 RGB 保留在 RGB 通道，把 luminance pack 到 alpha 通道。之后这张 packed texture 会被连续 downscale 三次，compute shader 最后直接读取 1/8 texture 的 A 通道作为 luminance。
我后来发现最终 ASCII 的明暗层级不够稳定，但 Frame Debugger 里看 downscale 后的 RGB 结果又是正常的，所以一开始不容易看出问题。
排查时我意识到，Frame Debugger 里主要看到的是 RGB 显示结果，但旧流程里 compute 真正用的是 alpha 通道里的 packed luminance。RGB 看起来正常，并不能证明 alpha 里的 luminance 在三次 downscale 后仍然保持正确的数值范围。
这个设计的问题是，一张 texture 同时承担颜色和 luminance 数据存储，luminance 被藏在 alpha 里，经过多次 downscale 后不太直观，也不方便验证。为了让数据流更清楚，我删掉了 Pack Luminance pass，不再提前把 luminance 存进 alpha。
修改后，我让 downscale 只处理正常 RGB，到 1/8 分辨率后，在 compute shader 里直接根据 downscaled RGB 重新 dot 计算 luminance。这样 compute 使用的亮度和实际 downscaled color 对应起来，Frame Debugger 里看到的 RGB 结果也更能直接解释最终 ASCII 的明暗表现。这个改动让 pipeline 更简单，也减少了隐藏通道带来的调试成本。

我现在处理这种视觉 bug 会先避免直接猜最终 shader 哪里错，而是把 pipeline 拆开看。先用 debug view 输出中间 texture，找到第一个异常出现的 pass，再检查这个 pass 的输入、输出 format、通道含义和参数。修完以后用同一个 camera、同一套 preset 和 debug view 做前后对比。多 pass 后处理里，中间 RT 不只是“看起来对不对”，还要明确每个通道存的是什么。比如 RGB 显示正常，不代表 alpha 里 pack 的数据也正常；format 支持颜色显示，也不代表它适合存梯度、差分或方向这种需要正负值的数据。