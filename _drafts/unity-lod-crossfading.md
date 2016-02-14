---
title: "Super sexy LOD fading"
date: 2016-02-13 11:11 +0200
categories: programming shaders gamedev unity3d
---

When performance is tight its often a good idea to use a level of detail system that renders a detailed model up close but a simpler version far away. This is an easy way to save vertices, and the user doesnt notice the lack of detail when the object is too far. Something that can be noticable however is the popping between the LOD levels. 

> <img src='images/lod-before.gif'>
>
> The dinosaur in question loses some of its credibility after popping like this.

Recently several solutions have popped up, today i'd like to show you how to implement one inspired by [Assassins Creed][as-solution]{:target="_blank"} in Unity. We'll briefly render both of the objects and apply a fade between them by clipping out certain parts. Luckily Unity already has some handles to implement this in their LOD system, as described in [the manual][lodgroup-u3d]{:target="_blank"}. In short by [enabling fading on the LODGroup][enable-fading-u3d]{:target="_blank"} whenever its time to switch between the LOD the following happens:

1. The shader keyword `LOD_FADE_CROSSFADE` is enabled on both the old and the new object.
2. The `unity_LODFade.x` shader variable is animated from 1 to 0 on the old object, and from 0 to 1 on the new. The duration is defined by the [crossfade duration][crossfade-duration-u3d].
3. After the duration has expired the old object is removed and only the new is visible.

Lets implement all this in a simple surface shader, though its easily adapted in a fragment shader too.

{% highlight C# linenos  %}
Shader "Custom/LODFade" {
	Properties {
		_MainTex ("Albedo (RGB)", 2D) = "white" {}
		_BumpTex ("Bump map", 2D) = "white" {}
		_EmissiveTex ("Emissive Map", 2D) = "white" {}
	}
	SubShader {		
		CGPROGRAM
		#pragma surface surf Standard fullforwardshadows
		#pragma multi_compile _ LOD_FADE_CROSSFADE

		sampler2D _MainTex, _BumpMap, _EmissiveTex, 
					_NoiseTex;
{% endhighlight %}

Most of all this should be familliar, we define the different diffuse/bump/emmisive textures. The additions we should note are the [multi_compile][multi-compile-u3d] statement, this will make sure Unity will compile this shader into two different variants; one with the fade enabled and the other without. This means there is 0 overhead when not fading.

{% highlight C# linenos linenostart='13' %}
struct Input {
	float2 uv_MainTex;
	
	#ifdef LOD_FADE_CROSSFADE
		float4 screenPos;
	#endif
};

{% endhighlight %}
	

{% highlight C# linenos %}

		void surf (Input IN, inout SurfaceOutputStandard o) {
			o.Albedo = tex2D(_MainTex, IN.uv_MainTex);
			o.Normal = UnpackNormal(tex2D(_BumpMap, IN.uv_MainTex));
			o.Emission = tex2D(_EmissiveTex, IN.uv_MainTex);
			

			#ifdef LOD_FADE_CROSSFADE
				half2 screenUV = IN.screenPos.xy / IN.screenPos.w;
				clip(unity_LODFade.x - tex2D(_NoiseTex,  screenUV * 3).a + 0.5);
			#endif
		}
		ENDCG
	}
	FallBack "Diffuse"
}
{% endhighlight %}

On line 10 we use the  feature 

> <img src='images/lod-after.gif'>
>
> Simple and smooth lod fading!


[as-solution]: http://simonschreibt.de/gat/assassins-creed-3-lod-blending/ "Assasins Creed example"
[lodgroup-u3d]: http://docs.unity3d.com/Manual/class-LODGroup.html "Unity documentation"
[enable-fading-u3d]: /images/lod-enable-on-group.png 
[crossfade-duration-u3d]: http://docs.unity3d.com/ScriptReference/LODGroup-crossFadeAnimationDuration.html "Crossfade duration"
[multi-compile-u3d]: http://docs.unity3d.com/Manual/SL-MultipleProgramVariants.html "Making multiple program variants"