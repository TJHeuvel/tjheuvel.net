---
title:  "Unity Rendering Part 4: Instancing"
date:   2023-04-03 10:34:17 +0100
categories: programming gamedev graphics
---

# Rendering in Unity Part 4: Instancing

We've made a cool little 'renderer', but despite our culling efforts its stil not very fast. Currently, we render 10 thousand cubes, but send a new draw call to the GPU for each one of them. This is a lot of communication, which we'd like to avoid. 

Rather than individual draws, it would be much quicker if we could tell the GPU to render 10k instances of the same mesh, but of course with a different transform for each of them. Thats the premise of instanced rendering.

Our rough process will be this:

1. Upload an array of transformation matrices
2. Convince the GPU to render our mesh a bunch of times
3. For every draw, find the correct objectToWorld to use and apply that


## Uploading matrices

We could use [SetMatrixArray](https://docs.unity3d.com/ScriptReference/MaterialPropertyBlock.SetMatrixArray.html), however this has some interesting behaviour:

> The array length can't be changed once it has been added to the block. If you subsequently try to set a longer array into the same property, the length will be capped to the original length and the extra items you tried to assign will be ignored. If you set a shorter array than the original length, your values will be assigned but the original values will remain for the array elements beyond the length of your new shorter array.

All Shader/Material/PropertyBlock Set Array behave like this. Asking our players to restart our game every time they see more objects is rather bad for retention, so lets manage this data ourself.

We've seen constant buffers before, lets manage our own so we can supply the GPU with the right information. The GraphicsBuffer* class is how Unity exposes this control.

<%

GraphicsBuffer perObjectData = new GraphicsBuffer(GraphicsBuffer.Target.ConstantBuffer, 10, UnsafeUtility.SizeOf<float4x4>());

%> 

This is how we make a Constant buffer that would fit up to 10 matrices. With [SetData](https://docs.unity3d.com/ScriptReference/GraphicsBuffer.SetData.html) we can supply it with information. 

## Adding it to our renderer

We'll modify our draw call to now give us a List of all ObjectToWorlds for our cubes. Then, when drawing, we'll figure out which are within our frustum, and upload these to the GPU buffer. Finally we ask our GPU to render up to *that* amount of objects. 

<%

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

private List<DrawCall> DrawCalls;

void OnEnable()
{

}

%>

Within our late update we will now fill a list with our visible transforms, and then update the Constant buffer with that.

<%
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

%>

Notice we are now filling in the instance count, we're asking our GPU to render this mesh once for each visible instance of it. All we now have to do is read out the index in our vertex shader, and make s

### The Vertex shader

Now our shader first has to accept the new constant buffer with positions. That looks like this:

<%
CBUFFER_START(_ObjectToWorldBuffer)

float4x4 objectToWorlds;

CBUFFER_END

%>
Unity has difference in platforms handled with macros, see the precompiled source for your specific output. Next; our GPU passes over the index of the current draw (which goes from 0.. instanceCount supplied in the RenderMeshPrimitives) with the shader semantic `SV_InstanceID`.


<%
struct Input
{
	uint instanceId : SV_InstanceID;
}

Custom_ObjectToWorld = _ObjectToWorldBuffer[i.instanceID];
%>

Press play, and observe great success!

One important thing we have to do is Release our graphics buffer when we no longer want to use it. If we dont, we leak GPU memory! For our case, in OnDisable seems to be a good place to do this.

## Moving beyond cubes


* There is also a ComputeBuffer, which is simply an older API that [we shouldnt use](https://forum.unity.com/threads/graphicsbuffer-and-mesh.636631/#post-4268896). 
