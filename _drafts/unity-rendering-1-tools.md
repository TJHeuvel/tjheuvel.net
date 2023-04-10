---
title:  "Unity Rendering Part 1: Tools of the trade"
date:   2023-04-03 10:34:17 +0100
categories: programming gamedev graphics
---

1. Rendering in Unity - Tools
How to use Renderdoc, how does a frame appear? Concepts of d3d11

2. Rendering in Unity - How a pixel is born?
Vertex shader, clip space. Custom projection matrix.

3. Rendering in Unity - Submitting draws

4. Rendering in Unity - Culling
Frustrum culling. TODO: learn calculateplanes

5. Rendering in Unity - Instancing

GPU Instancing, SRP Batcher

6. Rendering in Unity - Indirect  

# Rendering in Unity Part 1: Tools of the trade

## What and why
There are [many](https://catlikecoding.com/unity/tutorials/) [great](https://github.com/Xibanya/ShaderTutorials/) [tutorials](https://learn.unity.com/mission/creative-core-shaders-and-materials) around explaining shaders in Unity, focussing mainly on the shading aspect. What pretty colors do we give pixels?
This series will delve deeper into rendering itself; how do we get the GPU to do anything, and how do we convince it to draw pixels at the right location.

## Setup

The project we will be using can be found here, and uses Unity 2022.2, with the Universal Render Pipeline 14. A lot of features have been disabled for clarity. This includes SRP Batcher, which we will revisit in a later part. We'll use the newer Unity.Mathematics API's, float3 rather than Vector3, because they look more similar to whats going on in our shaders. This tutorial focusses on DirectX 11.

Observe a rather empty scene, with a cube offset to the right, and a camera placed back and to the right.

## Frame Debugger
Our first friend is the [Unity Frame Debugger](https://docs.unity3d.com/2022.2/Documentation/Manual/FrameDebugger.html). This is a great tool to get a very high level view of what and how your scene is rendered. Its great in showing us data, within the context of Unity. It knows a lot of Unity abstractions.

It falls short in detail, if we want to know more we'll need better tools.

## Renderdoc

[RenderDoc](https://renderdoc.org/) is a free and opensource graphics debugger. It is not Unity specific, it captures and replays the interaction between the graphics api, in our case dx11.



However, the names of these buffers are not tremendously useful. In order to get more information we have to ask Unity to generate debug symbols, [the manual](https://docs.unity3d.com/Manual/SL-DebuggingD3D11ShadersWithVS.html) tells us we can do this by adding:

> #pragma enable_d3d11_debug_symbols

Please note this is for every graphics API, not only d3d11. After we add this, and recapture, suddenly our inputs have much better names. We have also unlocked the ability to [debug our shaders with breakpoints](https://renderdoc.org/docs/how/how_debug_shader.html), for example in the mesh Input Assembly stage, right click any vertex and select Debug. We can now step through our shader code, exactly like how we can do with our C#! 

## Piecing it all together

We now can see that a draw call is provided with different buffers, PerFrame, PerMaterial and PerDraw. However, our original shader file seemingly is without any of this. Where did this come from?

The shaders we write are transformed in many different ways. In order to get a closer view of what is actually compiled, we can enable the `preprocess only` checkmark on our Shader importer, and press Compile. We observe all includes have been resolved, and pasted into our output file. Scrolling further, we see that our variables have been put in 'cbuffers', that match up with what we've seen in RenderDoc. Constant buffers, [the DirectX manual tells us](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-constants), are reserved memory. 
We can make them ourself in Unity using the GraphicsBuffer API, and bind them by using Material.SetBuffer etc. 

This flat view also allows us to find out exactly what methods are included. Another way is by refering to the builtin shader source, available [here](https://unity.com/releases/editor/archive) under downloads. 

With this information, we can see exactly what our Vertex shader does. It outputs a float4 value called vertex by doing the following transform:

<%
// Tranforms position from object to homogenous space
inline float4 UnityObjectToClipPos(in float3 pos)
{
    return mul(UNITY_MATRIX_VP, mul(unity_ObjectToWorld, float4(pos, 1.0)));
}

o.vertex = UnityObjectToClipPos(in.vertex);
%>

In the next part of our tutorial we will dive into this confusing looking piece of code!

FIND OUT HOW TO SQUISH THIS IN?
## What else?
Nvidia NSight, AMD GpuOpen, Microsoft PIX are also great tools. Besides debugging, they offer more profiling options.