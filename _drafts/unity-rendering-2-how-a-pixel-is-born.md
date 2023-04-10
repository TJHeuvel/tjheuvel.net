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

In order for us to fully appreciate this, we will use exclusively our own inputs. Lets remove the UnityCG.ginc, and go fully custom. This has no practical use, other than for educational.


<%

float4 vert(float4 vertex : POSITION) : SV_POSITION 
{
	??
}
%> 

The capitalized : bits are called Shader Semantics, we can read in [the manual](https://docs.unity3d.com/Manual/SL-ShaderSemantics.html) what both of them mean:

- POSITION is the 'vertex position input'
- SV_POSITION is described as: 'A vertex shader needs to output the final clip space position of a vertex, so that the GPU knows where on the screen to rasterize it, and at what depth. This output needs to have the SV_POSITION semantic, and be of a float4 type.'

This explains why the method we used before is called ObjectToClip, but sadly not what clip space actually *is*. We'll dive into that topic here.

## From Object to World

Our vertex function gets an input of the object space position of each vertex, in our identity cube ranging from -0.5 to 0.5. Ideally though, we could render objects not only where they are modelled, but drag them around to new areas in the world. 

This is why the first step in our new vert method will be, determine the position of this vertex, in world space. 

<%
float4 worldPos = mul(Custom_ObjectToWorld, vertex);
%>

Thats all it takes, except we stubbornly refuse to use Unity's own provided inputs. Lets supply our own, as learning.

<%

void LateUpdate()
{
	MaterialPropertyBlock matProps = new MaterialPropertyBlock();
	matProps.SetMatrix("Custom_ObjectToWorld", transform.localToWorldMatrix);
	GetComponent<Renderer>().SetMaterialPropertyBlock(matProps);
}

%>

We use material properties because these are *per renderer*, per object. 1


### Matrices

The RenderMesh API required us to pass in an ObjectToWorld as a Matrix4x4, or float4x4 in the new math library. A complete introduction to matrices is far outside the scope of this tutorial, and the knowledge of your dear author. However, we cannot get around some basic idea. If this topic interests you, this book is the most accessible resource i know of. https://gamemath.com/

A matrix is a grid of numbers, in our case 4 rows and 4 columns. Just like how Vectors can represent both position and direction, there are several different types of matrices. 

What we are looking at here is a transformation matrix, that contains translation (position), rotation and scale. Within Unity* its layout looks like this: 

```
Rx * Sx		0			0		Posx
0			Ry * Sy		0		Posy
0			0			Rz * Sz	Posz
0			0			0		1
```

Different graphics API's have different conventions, but Unity internally uses OpenGL conventions, and has [utilities](https://docs.unity3d.com/ScriptReference/GL.GetGPUProjectionMatrix.html) to convert it to something appropiate. This also means other tutorials will define matrices differently. We can make our own Transformation, Rotation and Scale matrices with [float4x4.TRS](https://docs.unity3d.com/Packages/com.unity.mathematics@1.2/api/Unity.Mathematics.float4x4.TRS.html).

Again, the whys and hows are outside of our scope. Try moving the cube around, and seeing the objectToWorld change. Visualising it by logging out the `Transform.localToWorld` can also be very useful to get a better idea of them. 

## From World to Camera

Our cube's position is 3 units to the right, in world space. However, in our game view, we see the cube to our left. This is because the camera itself is moved 6 units to the right. When we move around in the world, and see objects from different angles. This doesnt change the position of those objects, it simply changes the position of the camera. 

We can say the we observe objects in the world, relative to our camera. 

<%
float4 viewPos = mul(Custom_ViewMatrix, worldPos);
%>

Unity has the default, `Unity_MatrixV`, but lets explore how we can provide this ourself. After providing the object to world, lets also make sure to send this.

<%
Shader.SetGlobalMatrix("Custom_ViewMatrix", Camera.main.worldToCameraMatrix);
%>

Observe we deliberately use a global variable here. Each individual object we will render has to have their own position, but we *all* render them with the same camera position. This maps to `UnityPerFrame` and `UnityPerObject`, what we have seen before. 

The `worldToCameraMatrix` differs slightly from `worldToLocalMatrix`, namely:

- The scale of the camera GameObject is not present. When we make our camera gameobject twice as big, we do not want to have our entire world grow with us in size. The size of the camera is not a factor, so we remove it. We could re-add this if we'd like to do something funky by multiplying our view with `Matrix.Scale(Camera.main.transform.localScale)`. This would be incorrect, but quite interesting.
- The Z is flipped, negative Z is forward. This is a convention Unity/OpenGL has, we'd be looking backwards if we dont do this.

## From Camera to Clip

Ok, so we're doing all this apparently to output a clip space position. Lets try to understand what this is. [Wikipedia tells us])(https://en.wikipedia.org/wiki/Clip_coordinates) the following:

> The clip coordinate system is a homogeneous coordinate system in the graphics pipeline that is used for clipping.
> Clip coordinates are convenient for clipping algorithms as points can be checked if their coordinates are outside of the viewing volume. For example, a coordinate Xc for a point is within the viewing volume if it satisfied the inequality -Wc < Xc < Wc
> All coordinates may then be divided by the W component in what is called the perspective division.

Whew! Lets break this down into more managable chunks. Homogeneous is ancient greek for*, float4. Our point will be clipped, e.g. not visible, when the X coordinate is outside of the -W or +W. Lastly, magically, by dividing by W we achieve perspective. 

This still doesnt make a whole lot of sense, what *exactly* does our GPU want from us? A great part of Unity is the ease with which we can visualise any concept, lets do exactly that.

Please read through the script, and follow along.

Here, we draw a wire cube at the center of the projection, at the distance of our cube. The **size** of our gizmo is simply the **W** component. I like to think of this, as, the W component represents a slice of the view frustrum, at that particular depth.

Lets look back at our earlier rules. We can clearly see, the object is outside of our view when its outside of the cube, i.e. W component. We'd rather not render those, and clip them away.

Now for perspective division. By dividing the size of our object, with the size of the slice of the frustum, we ask ourself; how much of our view is taken up by our object? When the object is up close, our frustum is also small, so the answer is a lot. The more we move back, the bigger the frustum gets, the ratio of object to frustum-slice-size decreases. Perspective!

Let us also integrate this into our vertex shader, and C# code, as such:

<%
//Shader:
float4 clipPos = mul(Custom_ProjectionMatrix, viewPos); //NIET DE ANDERE KANT OP?

//C#:
Shader.SetGlobalMatrix("Custom_ProjectionMatrix", GL.GetGPUProj(Camera.main.projectionMatrix));
%>

The projection matrix contains our view frustum, e.g. the 6 planes that make up the conical shape we can see in. This is another 'type' of matrix, it doesnt represent a translation, rotation, scale, but rather the 6 planes of our view frustum. For the sake of this tutorial; our projection matrix is a magical box that fills in the W value to the size of the view frustum, at our objects depth. 
If this wasnt difficult enough, graphics APIs want matrices in different formats, hence the GL call.

Now when we enter play mode we should be seeing our cube as it has always been done by Unity. When we step through the vertex shader we can see our custom logic is applied however. Hopefully the vertex outputs start to make sense as well. 

One optimalisation done by Unity is not sending the View and Projection matrices separately, rather as one pre multiplied matrix. This is simply done as `SetGlobalMatrix("Unity_MatrixVP", Camera.main.viewMatrix * GL.thing(Camera.main.projectionMatrix));` Please try applying this yourself! TODO DOUBLE CHECK.

## Lets verify!

We can see that it works, lets just run though it one more time using RenderDoc. 


## What can we do with our newfound knowledge?

Alright, thats a lot of dense information, what can we do with this? Lets imagine we are making a first person shooter, and our artists ask: When the players adjust their FOV, could you not make our hands and weapon stretch as well?

We can now identify that we wish to have the same ObjectToWorld, the same ViewMatrix, but rather a different Projection matrix. We can create our own using [float4x4.PerspectiveFov](https://docs.unity3d.com/Packages/com.unity.mathematics@0.0/api/Unity.Mathematics.float4x4.html#Unity_Mathematics_float4x4_PerspectiveFov_System_Single_System_Single_System_Single_System_Single_) or [Matrix4x4.Perspective](https://docs.unity3d.com/ScriptReference/Matrix4x4.Perspective.html). 

<%
void LateUpdate()
{
	MaterialPropertyBlock matProps = new MaterialPropertyBlock();
	matProps.SetMatrix("Unity_MatrixVP", Camera.main.worldToCameraMatrix * GL.GetProj(float4x4.Perspective(45, Camera.main.aspect, Camera.main.near, Camera.main.far)));
	GetComponent<Renderer>().SetMaterialPropertyBlock(matProps);
}
%> 

This will work with any builtin shader, because more specific properties overwrite less specific ones. We deliberately only want to set the projection matrix on this current object. We can now piece together that the UnityObjectToClip method will use the input we have snuck in, which fixes the field of view at 45.

We could achieve other effects, for example we could draw some objects with the regular perspective matrix of our camera, and another with the ortographic matrix of our UI camera. This would essentially give us on screen tags, without doing too much work on the CPU.

1: Sincerest apologies to any ancient greeks and mathemathicians WRONG reading this. 

2: Scheduling also comes into play here. If we'd have three objects, and all would set the *same* localtoworld global variable, they would overwrite eachother. 