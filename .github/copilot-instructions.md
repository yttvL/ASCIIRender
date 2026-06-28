You are helping me review and document a Unity URP / Render Graph technical art project called God Intern.



Project characteristics:

\- Unity URP project

\- Heavy use of ScriptableRendererFeature / ScriptableRenderPass

\- Render Graph API usage

\- Fullscreen post-processing pipeline

\- ASCII rendering / downscale / edge detection

\- Terrain shader

\- Stencil / depth / portal-like shader experiments

\- Hand-written HLSL shaders and compute shaders



Important:

This is not a general gameplay systems project.

This is a rendering / shader / technical art project.



Documentation goals:

The main goal is to help me understand and explain the code architecture, especially:

1\. Render Graph API usage

2\. pass declaration vs pass execution

3\. resource creation / importing / reading / writing

4\. TextureHandle lifetime and dependency declaration

5\. PassData usage

6\. builder syntax

7\. SetRenderFunc / command buffer recording

8\. why command buffer calls are recorded rather than immediately executed

9\. pipeline order and pass dependency

10\. how HLSL / compute shader algorithm logic is embedded inside Render Graph passes



Major modules:

1\. ASCII post-process system

2\. Terrain shader system

3\. Other shaders:

&#x20;  - stencil/depth/portal/mirror-like shaders

&#x20;  - object mask / reader-writer style shaders

&#x20;  - any helper or debug shaders



Rules:

\- Do not modify source code unless explicitly asked.

\- You may create or edit documentation files only under docs/review/.

\- Use Chinese for final documentation.

\- Keep API names, class names, method names, shader names, and Unity terms in English.

\- Cite filenames/classes/methods when explaining code.

\- Separate code facts from assumptions.

\- Mark uncertain statements as “需要人工确认”.

\- Do not over-simplify Render Graph.

\- Explain both the high-level pipeline and the exact syntax/API pattern.

\- Explain "recording" carefully: command buffer calls inside Render Graph passes are recorded and scheduled, not immediately executed in the C# call order.



Preferred documentation style:

\- Use HTML when possible.

\- Create self-contained HTML files with embedded CSS.

\- Use a left sidebar table of contents.

\- Use anchor links.

\- Use cards, callouts, diagrams, and pipeline boxes.

\- Use code blocks for key syntax patterns.

\- Use warning boxes for confusing Render Graph concepts.

