---
title:  "Unity Rendering Part 4: Instancing"
date:   2023-04-03 10:34:17 +0100
categories: programming gamedev graphics
---

# Rendering in Unity Part 4: Instancing

We've made a cool little 'renderer', but despite our culling efforts its stil not very fast. Currently, we render 500 cubes, but send a new draw call to the GPU for each one of them. This is a lot of communication, which we'd like to avoid. 

Rather than individual draws, it would be much quicker if we could tell the GPU to render 500 instances of the same mesh, but of course with a different transform for each of them. Thats the premise of instanced rendering.

Our rough process will be this:

1. Upload an array of transformation matrices
2. Convince the GPU to render our mesh a bunch of times
3. For every draw, find the correct objectToWorld to use and apply that


## Uploading matrices

We could use [SetMatrixArray](https://docs.unity3d.com/ScriptReference/MaterialPropertyBlock.SetMatrixArray.html), however this has some interesting behaviour:

> The array length can't be changed once it has been added to the block. If you subsequently try to set a longer array into the same property, the length will be capped to the original length and the extra items you tried to assign will be ignored. If you set a shorter array than the original length, your values will be assigned but the original values will remain for the array elements beyond the length of your new shorter array.

All Shader/Material/PropertyBlock Set Array behave like this. Asking our players to restart our game every time they see more objects is rather bad for retention, so lets manage this data ourself.

We've seen constant buffers before, lets manage our own so we can supply the GPU with the right information. The GraphicsBuffer<sub>1</sub> class is how Unity exposes this control.

{% highlight C# %}

GraphicsBuffer perObjectData = new GraphicsBuffer(GraphicsBuffer.Target.ConstantBuffer, 10, UnsafeUtility.SizeOf<float4x4>());

{% endhighlight %} 

This is how we make a Constant buffer that would fit up to 10 matrices. With [SetData](https://docs.unity3d.com/ScriptReference/GraphicsBuffer.SetData.html) we can supply it with information. Lets also not forget to [Release](https://docs.unity3d.com/ScriptReference/GraphicsBuffer.Release.html) the buffer once we're done with it in OnDisable, otherwise we'll leak GPU memory!

## Adding it to our renderer

We'll modify our draw call to now give us a List of all ObjectToWorlds for our cubes. Then, when drawing, we'll figure out which are within our frustum, and upload these to the GPU buffer. Finally we ask our GPU to render up to *that* amount of objects. 

{% highlight C# %}

struct PerObjectData
{
	public float4x4 ObjectToWorld;
	public Bounds WorldBounds;
}
struct DrawCall
{
	public Mesh Mesh;
	public Material Material;
	public GraphicsBuffer ObjectToWorlds;

	public List<PerObjectData> PerObjectData;
}
, 
private List<DrawCall> DrawCalls;

void OnEnable()
{
	DrawCalls = new List<DrawCall>();

	foreach(var renderer in FindObjectsOfType<MeshRenderer>())
	{

	}
}

{% endhighlight %}

Within our late update we will now fill a list with our visible transforms, and then update the Constant buffer with that.

{% highlight C# %}
void LateUpdate()
{
	List<float4x4> visibleObjectToWorlds = new List<float4x4>();

	foreach(var draw in Drawcalls)
	{
		visibleObjectToWorlds.Clear();

		foreach(var objData in draw.PerObjectData)
		{
			if(IsObjectWithinFrustum(planes, draw[i].Bounds))
				visibleObjectToWorlds.Add(objData.ObjectToWorld);
		}
		draw.ObjectToWorlds.SetData(visibleObjectToWorlds);

		propBlock.SetBuffer("_ObjectToWorldBuffer", draw.ObjectToWorlds);

		Graphics.RenderMeshPrimitives(new RenderParams(draw.Material) 
		{
			matProps = propBlock; 
		}, draw.Mesh, subMeshIndex: 0, instanceCount: visibleObjectToWorlds.Count);

	}
}

{% endhighlight %}

Notice we are now filling in the instance count, we're asking our GPU to render this mesh once for each visible instance of it. All we now have to do is read out the index in our vertex shader, and use that object to world instead.

### The Vertex shader

Now our shader first has to accept the new constant buffer with positions. That looks like this:

{% highlight C# %}
CBUFFER_START(_ObjectToWorldBuffer)
float4x4 objectToWorlds;
CBUFFER_END

{% endhighlight %}
**TODO VERIFY THAT SYNTAX!**
Unity has difference in platforms handled with macros, see the precompiled source for your specific output. Next; our GPU passes over the index of the current draw (which goes from 0.. instanceCount supplied in the RenderMeshPrimitives) with the shader semantic `SV_InstanceID`.

{% highlight C# %}
struct Input
{
	uint instanceId : SV_InstanceID;
}

//in Vert
Custom_ObjectToWorld = _ObjectToWorldBuffer[i.instanceID];
{% endhighlight %}

Press play, and observe great success! With a single draw we can now do all 500 cubes. 

## Pushing to the limit

This will work for for up to 1023 cubes. We can read a Constant Buffer [can store up to 4096 float4s](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-constants), and since a matrix is 4 float4's that means our buffer is full at 1023. Coincidentally, Unitys instancing also supports up to 1023 unique draws.

If your game for some reason only consists of cubes we can get around this by using [StructuredBuffers](https://learn.microsoft.com/en-us/windows/win32/direct3d11/direct3d-11-advanced-stages-cs-resources#structured-buffer). They are a newer(introduced in d3d11) way to store GPU data, that is only limited by the size of VRAM. The downside is they are slightly slower to read from, but if this is the difference between breaking batches you'll more than make up for it. You make them in the same way, but instead of Contant pass in the type Structured.

* There is also a ComputeBuffer, which is simply an older API that [we shouldnt use](https://forum.unity.com/threads/graphicsbuffer-and-mesh.636631/#post-4268896) anymore. 
