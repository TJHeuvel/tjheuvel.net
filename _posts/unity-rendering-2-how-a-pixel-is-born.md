---
title:  "Unity Rendering Part 2: How a pixel is born"
date:   2023-04-03 10:34:17 +0100
categories: programming gamedev graphics
---

# Rendering in Unity Part 2: How a pixel is born

In Part One we gained an understanding how Unity renders a scene, in this part we'll explore how we can render ourself, and exactly what is going on.

We left off with this rather complicated looking line of code: 

<%
mul(UNITY_MATRIX_VP, mul(unity_ObjectToWorld, float4(pos, 1.0)))
%>

In order for us to fully appreciate this, we will use exclusively our own inputs. Lets remove the UnityCG.ginc, and go fully custom. This has no practical use, other than being educational. Our shader now looks [like this](https://github.com/TJHeuvel/UnityRenderingTutorial/blob/fbc613344ba4877928501123862c89a250c50ddc/Assets/UnlitShader.shader#L35), naked without UnityCG.cginc, and a todo to fill out the vertex output.  

The capitalized bits behind our parameters are called Shader Semantics, we can read in [the manual](https://docs.unity3d.com/Manual/SL-ShaderSemantics.html) what they mean:

- POSITION is the 'vertex position input'
- SV_POSITION is described as: 'A vertex shader needs to output the final clip space position of a vertex, so that the GPU knows where on the screen to rasterize it, and at what depth. This output needs to have the SV_POSITION semantic, and be of a float4 type.'

This explains why the method we used before is called ObjectToClip, but sadly not what clip space actually *is*. Even the [HLSL manual on semantics](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics) doesnt help us. We'll dive into that topic here. 

## From Object to World

Our vertex function gets an input of the object space position of each vertex, in our identity cube ranging from -0.5 to 0.5. Ideally though, we could render objects not only where they are modelled, but drag them around to new areas in the world. 

This is why the first step in our new vert method will be, determine the position of this vertex, in world space. 

<%
float4x4 custom_ObjectToWorld;

//in vert
float4 worldPos = mul(custom_ObjectToWorld, v.vertex);
%>

Thats all it takes, except we stubbornly refuse to use Unity's provided inputs, so we'll have to fill in the custom object to world ourself. Lets make a new script called RenderSystem and attach it to our cube. Lets add ExecuteAlways too, so our views update when we are outside of play mode too.
<%

void LateUpdate()
{
    MaterialPropertyBlock matProps = new MaterialPropertyBlock();
    matProps.SetMatrix("custom_ObjectToWorld", transform.localToWorldMatrix);
    GetComponent<Renderer>().SetPropertyBlock(matProps);
}

%>

We're using a MaterialPropertyBlock, this is something that we can provide *per renderer*. If we'd set this property on the material, or in a shader global variable, we wouldnt be able to have multiple cubes!

### Matrices

In RenderDoc we encountered matrices, and now here again our `custom_ObjectToWorld` is a matrix, set to the `transform.localToWorldMatrix`. Graphics APIs love those things, so we better get used to them. A complete introduction to matrices is far outside the scope of this tutorial, i'll just show the absolute basics. For a deeper appreciation of all things game math, i can highly recommend getting a copy of [3D Math Primer for Graphics and Game Development](https://gamemath.com/). Its contents are now also completely readable online!

A matrix is a grid of numbers, in our case 4 rows and 4 columns. Just like how Vectors can represent both position and direction, there are several different types of matrices. 

What we are looking at here is a transformation matrix, that contains translation (position), rotation and scale. We also saw the contents of one in Renderdoc. Its worth logging them in Unity and experimenting how they change when you move, rotate, or reparent objects.

Their general layout within Unity is like this:

<blockquote style="white-space:pre">
R * Sx		R			R		Px
R			R * Sy		R		Py
R			R			R * Sz	Pz
0			0			0		1
</blockquote>
Where R is rotation, S is scale and the P is position. The last column always contains our position, the first 3x3 block rotation and scale, and the bottom row is always 0,0,0,1. Back in RenderDoc we can verify if that this is true, by checking the matrix passed to our cube.

Different graphics APIs have different conventions, but Unity internally uses OpenGL conventions, and where applicable has [utilities](https://docs.unity3d.com/ScriptReference/GL.GetGPUProjectionMatrix.html) to convert it to something appropiate for the current platform. This also means other tutorials will define matrices differently. We can make our own Transformation, Rotation and Scale matrices with [float4x4.TRS](https://docs.unity3d.com/Packages/com.unity.mathematics@1.2/api/Unity.Mathematics.float4x4.TRS.html).

## From World to Camera

Our cube's position is 3 units to the right, in world space. However, in our game view, we see the cube to our left. This is because the camera itself is moved 6 units to the right. Our camera moves around in the world so we can see objects from new angles, the position of those objects does not change in itself.

We can say the we observe objects in the world, relative to our camera. In our shader we can accomplish this by adding:

<%
float4 viewPos = mul(custom_ViewMatrix, worldPos);
%>

Of course we have to pass this new parameter as well, because we're too stubborn to use the builtin `unity_MatrixV`.

<%
Shader.SetGlobalMatrix("custom_ViewMatrix", Camera.main.worldToCameraMatrix);
%>

Observe we deliberately use a global variable here. Each individual object we will render has to have their own position, but we *all* render them with the same camera position. The `worldToCameraMatrix` differs slightly from `worldToLocalMatrix`, namely:

- The scale of the camera GameObject is not present. When we make our camera gameobject twice as big, we do not want to have our entire world grow with us in size. The size of the camera is not a factor, so we remove it. We could re-add this if we'd like to do something funky by multiplying our view with `Matrix.Scale(Camera.main.transform.localScale)`. This would be incorrect, but quite interesting.
- The Z is flipped, negative Z is forward. This is a convention Unity/OpenGL has, we'd be looking backwards if we dont do this.

## From Camera to Clip

Ok, so we're doing all this apparently to output a clip space position. Lets try to understand what this is. [Wikipedia tells us])(https://en.wikipedia.org/wiki/Clip_coordinates) the following:

1. The clip coordinate system is a homogeneous coordinate system in the graphics pipeline that is used for clipping.
2. Clip coordinates are for clipping algorithms. A point is within the viewing volume if it satisfied the inequality -Wc < Xc < Wc
3. All coordinates may then be divided by the W component in what is called the perspective division.

Whew! Lets break this down into more managable chunks. Homogeneous is ancient greek for<sub>1</sub> float4. Our point will be clipped, e.g. not visible, when the X coordinate is outside of the -W or +W. Lastly, magically, by dividing by W we achieve perspective. 

This still doesnt make a whole lot of sense, what *exactly* does our GPU want from us? Lets take a peek at [the final step](https://github.com/TJHeuvel/UnityRenderingTutorial/blob/e27ce02cf8e75d28b204d0601f964cf55ff16d70/Assets/UnlitShader.shader#L41) in our shader:


<%
	float4 worldPos = mul(custom_ObjectToWorld, v.vertex);
    float4 viewPos = mul(custom_ViewMatrix, worldPos);
    float4 clipPos = mul(custom_ProjectionMatrix, viewPos);

    o.vertex = clipPos;
%>

Alright apparently we've entered clip space by multiplying by [the projection matrix](https://docs.unity3d.com/ScriptReference/Camera-projectionMatrix.html). Lets visualise what this actually is with one of the strongest points of Unity, a custom gizmo. Observe in [our gizmo](https://github.com/TJHeuvel/UnityRenderingTutorial/commit/8e87a61b4bb497c2a6a358c2cc912adf67aca37f) we're doing exactly the same as our shader.

<video autoplay="true" controls="true" height="400px">
	<source src="https://dl.dropboxusercontent.com/s/yehvwiiee64h76v/Unity_hkTnAHnGXk.mp4">
</video>

Here, we draw a wire cube at the center of the projection, at the distance of our cube. The **size** of our gizmo is the **W** component. I like to think of this as, the W component represents a slice of the view frustrum, at that particular depth.

Lets look back at our earlier rules. We can clearly see, our cube is is outside the view its outside the green lines, i.e. W component. We'd rather not render those, and clip them away.

Now for perspective division. By dividing the size of our object, with the size of the slice of the frustum, we ask ourself; how much of our view is taken up by our object? When the object is up close, our frustum is small, so the answer is a lot. The more we move back, the bigger the frustum gets, the ratio of object to frustum-slice-size decreases. Perspective!
If our frustum would ever be of 0 size we'd be dividing by zero, an obviously bad thing. This is why the near clipping plane cannot be set to 0.

<%
//C#:
Shader.SetGlobalMatrix("custom_ProjectionMatrix", GL.GetGPUProjectionMatrix(Camera.main.projectionMatrix, true));
%>

The projection matrix contains our view frustum, e.g. the 6 planes that make up the conical shape we can see in. This is another 'type' of matrix, it doesnt represent a translation, rotation, scale, but rather the 6 planes of our view frustum. For the sake of this tutorial; our projection matrix is a magical box that fills in the W value to the size of the view frustum, at our objects depth. 
If this wasnt difficult enough, graphics APIs want matrices in different formats, hence the GL call.

Now when we enter play mode we should be seeing our cube as it has always been done by Unity. Interestingly, our scene view is all kinds of weird. We're sending the parameters of the main camera, but not our scene view camera. We could solve this by sending the camera parameters in OnPreCull, or the new renderpipelines equivalent. 
When we step through the vertex shader we can see our custom logic is applied, please try to do so and try to see if the outputs make sense. What view space position do you expect, what is actually calculated?

One optimalisation done by Unity is not sending the View and Projection matrices separately, rather as one pre multiplied matrix. This is simply done as `SetGlobalMatrix("Unity_MatrixVP", Camera.main.viewMatrix * GL.GetGPUProjectionMatrix(Camera.main.projectionMatrix, true));` Please try applying this yourself! 

## What can we do with our newfound knowledge?

Alright, thats a lot of dense information, what can we do with this? Lets imagine we are making a first person shooter, and our artists ask: When the players adjust their FOV, could you not make our hands and weapon stretch as well?

We can now identify that we wish to have the same ObjectToWorld, the same ViewMatrix, but rather a different Projection matrix. We can create our own using [float4x4.PerspectiveFov](https://docs.unity3d.com/Packages/com.unity.mathematics@0.0/api/Unity.Mathematics.float4x4.html#Unity_Mathematics_float4x4_PerspectiveFov_System_Single_System_Single_System_Single_System_Single_) or [Matrix4x4.Perspective](https://docs.unity3d.com/ScriptReference/Matrix4x4.Perspective.html). 

<%
void LateUpdate()
{
	MaterialPropertyBlock matProps = new MaterialPropertyBlock();
	matProps.SetMatrix("Unity_MatrixVP", Camera.main.worldToCameraMatrix * GL.GetGPUProjectionMatrix(float4x4.Perspective(45, Camera.main.aspect, Camera.main.near, Camera.main.far), true));
	GetComponent<Renderer>().SetMaterialPropertyBlock(matProps);
}
%> 

This will work with any builtin shader, because more specific properties overwrite less specific ones. We deliberately only want to set the projection matrix on this current object. We can now piece together that the UnityObjectToClip method will use the input we have snuck in, which fixes the field of view at 45.

We could achieve other effects, for example we could draw some objects with the regular perspective matrix of our camera, and another with the ortographic matrix of our UI camera. This would essentially give us on screen tags, without doing too much work on the CPU.

<sub>1:</sub> Sincerest apologies to any ancient greeks and mathematicians reading this. 