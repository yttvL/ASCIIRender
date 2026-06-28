\# Task: Generate an HTML Technical Review Document for the God Intern Unity Project



You are reviewing a Unity URP technical art project named \*\*God Intern\*\*.



The goal is to generate a detailed, source-code-grounded HTML document that explains the project's rendering, shader, and Render Graph architecture.



This document is for the project owner to review and study their own implementation. It should not be a short summary. It should be a structured, expandable technical learning document.



\## Important Input Files



Please read and use the following reference documents:



\- `Docs/AIContext/`



Then inspect the actual project source code, especially:



\- `Assets/` scripts related to URP Renderer Features

\- `Assets/` scripts related to ScriptableRendererFeature

\- `Assets/` scripts related to ScriptableRenderPass

\- Render Graph helper utilities

\- ASCII rendering pipeline scripts

\- HLSL shader files

\- compute shader files

\- terrain shaders

\- stencil / portal / object-mask related shaders

\- any materials or renderer data files if useful for context



\## Output File



Generate one standalone HTML file here:



`Docs/Generated/GodIntern_TechnicalReview.html`



The HTML must be readable directly in a browser.



Do not generate a Markdown report unless needed temporarily. The final deliverable should be the HTML file.



\## Main Goal



Create an HTML technical review document that preserves the large-scale structure of the Render Graph and shader pass pipeline while allowing detailed explanations to expand/collapse inside cards.



Do \*\*not\*\* simplify or aggressively compress the content just to keep the outline short.



Instead:



\- Keep the top-level outline clean.

\- Put detailed explanations inside collapsible sections.

\- Use source code references heavily.

\- Quote relevant code snippets when helpful.

\- Explain Unity API calls, syntax, and data flow.

\- Help the reader review both rendering concepts and concrete implementation syntax.



The document should behave like a study guide for reviewing:



\- Unity URP custom renderer features

\- ScriptableRendererFeature

\- ScriptableRenderPass

\- Render Graph

\- TextureHandle usage

\- pass declaration vs execution

\- RecordRenderGraph

\- command buffer execution timing

\- HLSL shader pass structure

\- compute shader structure

\- downscale / luminance / edge detection / ASCII mapping

\- depth, stencil, render queue, and compositing logic



\## Required HTML Style



Use plain HTML, CSS, and minimal vanilla JavaScript only.



Do not use external libraries or CDN links.



The HTML should include:



1\. A fixed or sticky table of contents

2\. Search/filter box if possible

3\. Collapsible cards using `<details>` and `<summary>`

4\. Code blocks with file path labels

5\. Clear module separation

6\. Visual hierarchy

7\. Small callout boxes for important concepts

8\. Tables where useful

9\. Cross-links between related sections



The visual style should be clean and readable. Prefer a technical documentation style.



\## Required Top-Level Structure



The HTML document should include at least the following major sections:



\### 1. Project Map



Explain the important files and folders.



For each important file, include:



\- file path

\- role in the project

\- which module it belongs to

\- why it matters

\- related files



This section should help me understand the project structure before reading code.



\### 2. Rendering Pipeline Overview



Explain the full rendering flow at a high level.



Include:



\- where the custom URP feature enters the pipeline

\- which RenderPassEvent or injection point is used

\- how the source color texture is read

\- how intermediate render textures are created or reused

\- how the ASCII post-processing result is written back

\- how depth/stencil resources are involved if applicable



Use a pipeline diagram made with HTML/CSS text boxes if possible.



Example style:



Scene Color

→ Downscale

→ Luminance / DoG

→ Sobel / Edge Detection

→ ASCII Mapping

→ Composite / Final Blit



But adapt this to the actual source code.



\### 3. Render Graph Architecture



This is a very important section. Explain Render Graph in this project in detail.



Cover:



\- what code declares resources

\- what code declares passes

\- what code only records future work

\- what code actually runs later

\- why RecordRenderGraph is not immediate rendering

\- what TextureHandle represents

\- imported textures vs created textures

\- transient or intermediate render targets

\- builder usage

\- pass data objects

\- SetRenderFunc or equivalent execution function

\- command buffer usage

\- how read/write dependencies are expressed

\- how pass order is inferred



Use actual source code snippets and references.



Please create collapsible subsections for each important Render Graph helper function or pass method.



For every Render Graph helper or pass function, explain:



\- function purpose

\- inputs

\- outputs

\- side effects

\- Render Graph resources touched

\- command buffer operations

\- why the code is written this way

\- common misunderstanding for beginners



\### 4. ASCII Pipeline



Analyze the ASCII rendering module in detail.



Include:



\- downscale pipeline

\- luminance calculation

\- DoG prefiltering

\- Sobel edge detection

\- edge direction classification if present

\- compute shader role if present

\- glyph atlas sampling

\- cell size assumptions

\- fill color and edge color logic

\- use of source color vs downscaled color

\- debug modes

\- final composite behavior



For each pass, include:



\- pass name

\- source file

\- input texture(s)

\- output texture(s)

\- shader/material used

\- important shader keywords/properties

\- algorithm explanation

\- key code snippets



Do not remove details. Put long explanations inside collapsible cards.



\### 5. Shader Pass Breakdown



Inspect the HLSL shader files and explain their structure.



For each important shader:



\- file path

\- shader role

\- Properties block

\- SubShader / Pass structure

\- Tags

\- render queue / render type if relevant

\- vertex function

\- fragment function

\- included URP libraries

\- important macros

\- texture declarations

\- sampler usage

\- material properties

\- depth/stencil/blend/cull/zwrite/ztest settings

\- how Unity binds this shader from C# if applicable



Please explain syntax as well as purpose.



The intended reader is learning shader syntax and Unity rendering API, so syntax-level explanations are valuable.



\### 6. Compute Shader Breakdown



If compute shaders exist, explain:



\- file path

\- kernel names

\- numthreads

\- dispatch dimensions

\- input textures / buffers

\- output textures / buffers

\- group shared memory if used

\- barriers such as GroupMemoryBarrierWithGroupSync if used

\- RWTexture / StructuredBuffer / RWStructuredBuffer usage

\- how C# dispatches the compute shader

\- how this compute pass fits into the Render Graph / rendering pipeline



Include source code snippets and explain the algorithm step by step.



\### 7. Terrain Shader / PCG Module



Analyze terrain-related shader and script files.



Include:



\- vertex displacement

\- noise sampling

\- top / side color separation

\- fake lighting direction

\- toon-style highlight/shadow logic if present

\- how normals or world position are used

\- important exposed material properties

\- what should be tested visually



\### 8. Stencil / Depth / Portal / Object Mode Experiments



Analyze stencil/depth/render queue related files.



Include:



\- stencil writer logic

\- stencil reader logic

\- Comp / Ref / Pass settings

\- depth testing behavior

\- render queue behavior

\- interaction with URP depth texture

\- why stencil requires shared depth-stencil attachment

\- boundary/mask issues if relevant

\- object-only ASCII or stencil-masked ASCII concepts if represented in code



This section should connect code implementation to rendering concepts.



\### 9. API and Syntax Review Index



Create an index of important Unity / URP / Render Graph / HLSL APIs used in this project.



For each API or syntax item, include:



\- name

\- where it appears

\- what it does

\- why this project uses it

\- common mistake

\- related source file



Examples may include but are not limited to:



\- ScriptableRendererFeature

\- ScriptableRenderPass

\- RecordRenderGraph

\- RenderGraph

\- TextureHandle

\- UniversalResourceData

\- ContextContainer

\- RenderPassEvent

\- AddBlitPass

\- SetRenderAttachment

\- SetRenderAttachmentDepth

\- SetRenderFunc

\- CommandBuffer

\- Blitter

\- Material.SetTexture / SetFloat / SetVector

\- HLSL Properties

\- TEXTURE2D / SAMPLER

\- SAMPLE\_TEXTURE2D

\- TransformObjectToHClip

\- SV\_Target

\- SV\_Position

\- RWTexture2D

\- numthreads

\- Dispatch



Use only the APIs actually present in the project when possible. If an API is not present but is needed to explain the code, mark it clearly as contextual.



\### 10. Debugging and Testing Notes



Extract useful debugging and testing notes from the code and reference document.



Include:



\- Frame Debugger checkpoints

\- Render Graph viewer / pass order checks if applicable

\- intermediate texture inspection

\- common visual bugs

\- color format issues

\- Scene View vs Game View differences if mentioned

\- overexposure / luminance issues if mentioned

\- stencil boundary issues if mentioned

\- things to test in Unity Inspector



\### 11. Suggested Future Improvements



Based on the code and notes, list possible future improvements.



Keep this grounded in the existing project. Do not invent unrelated features.



Possible categories:



\- reducing temporary RT allocation

\- making cell size configurable

\- validating glyph atlas dimensions

\- better Inspector warnings

\- separating edge color and fill color

\- object-mode ASCII

\- stencil toggles

\- terrain fake lighting improvements

\- debug mode organization



If a suggestion comes from the reference document rather than current code, clearly label it as such.



\## Source Code Citation Requirements



The document must cite source locations frequently.



For every major claim about implementation, include one or more of:



\- file path

\- class name

\- method name

\- shader pass name

\- property name

\- approximate line number if available



Example:



```html

<div class="source-ref">

Source: Assets/Scripts/Rendering/AsciiRenderFeature.cs → AsciiRenderPass.RecordRenderGraph()

</div>



Important Writing Instructions
The document should be written for a technical art / shader learning context.
Assume the reader understands Unity basics but is still learning:
URP Render Graph HLSL compute shaders GPU rendering pipeline stencil/depth behavior
Therefore, explain both:
What the code does. Why the rendering pipeline needs it. What the syntax means. What would go wrong if it were changed incorrectly.
Avoid generic textbook explanations unless they directly help explain this project.
Prefer this style:
“In this project, this function acts as...” “This pass reads X and writes Y...” “The important Render Graph idea here is...” “This code does not execute the rendering immediately; it records a pass that Render Graph schedules later...” “This shader property is exposed because...” “A common mistake here would be...” Do Not Do These
Do not produce a short executive summary.
Do not remove implementation details to make the document shorter.
Do not only paraphrase the reference markdown.
Do not ignore the source code.
Do not generate a generic Unity rendering tutorial.
Do not place all details directly in the main page body. Use collapsible sections.
Do not use external JS/CSS frameworks.
Do not modify project source code unless absolutely necessary.
Expected Output Quality
The final HTML should be long, detailed, and useful as a review document.
A good result should let me open Docs/Generated/GodIntern_TechnicalReview.html and study:
how my Render Graph code is organized how pass declaration and execution are separated how my ASCII shader pipeline flows how each shader pass works what each important API call means where each piece of logic lives in the source code what parts I should test or improve later
Please read Docs/AIContext/Copilot_Task.md, then inspect the project source code and generate the requested HTML document.

