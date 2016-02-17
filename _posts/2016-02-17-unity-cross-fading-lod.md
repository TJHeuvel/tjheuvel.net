---
title: "Cross fading your LOD levels"
date: 2016-02-13 11:11 +0200
categories: programming shaders gamedev unity3d
---

If you have a tight performance budget one of the first things developers utilize is a Level of Detail system. This means that when the camera is close to an object a high detailed model is used, while when it moves away a simpler one appears. This is an easy way to save GPU time, and if applied properly, the user doesn't notice the lack of detail when the object is in the distance. Something that can be noticeable however is the switching between the LOD levels, often described as popping. 

> <img src='images/lod-before.gif'>
>
> The dinosaur in question loses some of its credibility after popping like this. [Made by Muelsa](https://www.assetstore.unity3d.com/en/#!/content/45319)

Recently several solutions have popped up, today i'd like to show you how to implement one inspired by [Assassins Creed][as-solution]{:target="_blank"} in Unity. We'll briefly render both the high and low detail objects, and apply a dither animation to mask the transition. Luckily Unity already has some handles to integrate this in their LOD system, as described in [the manual][lodgroup-u3d]{:target="_blank"}. In short, by [enabling fading on the LODGroup][enable-fading-u3d]{:target="_blank"} a switch between the LOD levels will cause the following:

1. The [shader keyword][shader-keyword-u3d] `LOD_FADE_CROSSFADE` is enabled on both the old and the new object's shader.
2. The `unity_LODFade.x` shader variable is animated from 1 to 0 on the old object, and from 0 to 1 on the new. The timespan is defined by the [crossfade duration][crossfade-duration-u3d].
3. After the duration has expired the old object is removed and only the new one is visible. Finally the `LOD_FADE_CROSSFADE` keyword is disabled.

Let's take a look at the shader that implements this:

{% highlight C# %}
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

Most of all this should be familliar: we define the different diffuse/bump/emissive textures. The additions to note are the [multi_compile][multi-compile-u3d] statement, which will make sure Unity compiles this shader into two different variants: one with the fade enabled and the other without. This means there is no overhead when there is no fade active. 

We also add a sampler for the `_NoiseTex`, this is a texture that we'll use to define the transition. A gradient for example would always fade from left to right. We'll use a [noise texture][noise-texture]{:target="_blank"} that has a random value for the Alpha color. Because it's not in the properties block above, it should be defined as a shader global together with the crossfade duration.

{% highlight C# %}
struct Input {
	float2 uv_MainTex;
	
	#ifdef LOD_FADE_CROSSFADE
		float4 screenPos;
	#endif
};

{% endhighlight %}

The `_NoiseTex` will be sampled in screenspace instead of using the UV of the object. [This gif][screenspace-example]{:target="_blank"} shows what sampling a texture in screen space actually means, as you can see the texture doesn't move with our ferocious reptile but instead is dependent on where it is on the screen. We do this so it works on every model, and is consistent when the camera moves around. In order to get the position on the screen we define the `float4 screenPos` variable as input, but only when we actually are fading. 

{% highlight C# %}

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

The actual surface method first assigns the albedo and other channels, as any normal shader would. What is more interesting is what happens when the `LOD_FADE_CROSSFADE` keyword is defined. Firstly we transform the screen position to an UV that we can use to sample the texture with, because the incoming position is a [homogeneous coordinate][homo-coord] and we want a viewport space that runs from 0,0 to 1,1. The [clip][hlsl-clip-manual] function discards the pixel when the argument is below zero, so we'll need to come up with a formula that animates from something negative to something positive using the `unity_LODFade.x` variable as the percentage. By offsetting this with the alpha from the noiseTexture we apply the grain effect. Additionally, by multiplying the UV by 3 it's repeated over the screen three times. This increases the detail, thus requiring a smaller texture. 

> <img src='images/lod-after.gif'>
>
> Simple and smooth lod fading!

All that is left to do is to create a script that [sets the global noise texture][set-global-texture-u3d], and assigns the [crossfade duration][crossfade-duration-u3d]. You can find this and the complete shader in [this unity package][package-download]. If you have any questions send me a tweet, or email.

[as-solution]: http://simonschreibt.de/gat/assassins-creed-3-lod-blending/ "Assasins Creed example"
[lodgroup-u3d]: http://docs.unity3d.com/Manual/class-LODGroup.html "Unity documentation"
[enable-fading-u3d]: /images/lod-enable-on-group.png 
[crossfade-duration-u3d]: http://docs.unity3d.com/ScriptReference/LODGroup-crossFadeAnimationDuration.html "Crossfade duration"
[multi-compile-u3d]: http://docs.unity3d.com/Manual/SL-MultipleProgramVariants.html "Making multiple program variants"
[noise-texture]: /images/lod-noise-texture.tga
[screenspace-example]: /images/lod-screenspace-example.gif
[hlsl-clip-manual]: https://msdn.microsoft.com/en-us/library/windows/desktop/bb204826(v=vs.85).aspx "HLSL manual on Clip"
[package-download]: /images/LODFade.unitypackage
[homo-coord]: http://glprogramming.com/red/appendixf.html
[set-global-texture-u3d]: http://docs.unity3d.com/ScriptReference/Shader.SetGlobalTexture.html
[shader-keyword-u3d]: http://docs.unity3d.com/ScriptReference/Shader.EnableKeyword.html