---
layout:     post
title:      "Planar Shadow实现及应用"
subtitle:   "阴影的不同实现及优化"
date:       2018-02-28
author:     "Luffy"
header-img: "./img/BG/topbackground.jpg"
catalog: true
tags:
- U3D
- Shader
- 环境
- 优化

---

## 前言
阴影的实现有许多方式，由自带的Shadow Map 方式，转到使用Projector 方式， 最后参考自镇魔曲， 使用Planar Shadow方式来实现。对比分析不同实现方式。

## 自带阴影

最开始使用的是自带阴影，当时考虑的原因是自带shadow方便控制光的方向和统一表现，例如：水、物体、人物等需要光的方向、亮度等，都会有统一的方向。

#### 实现
使用Unity自带阴影在实现上是很简单的，创建方向灯，然后设上阴影的各参数，也可以在Quality界面设置好Shadow的各个质量选项。


原理是 :

* 以光源位置为视点被渲染。每个渲染图像的像素的深度被记录在一个“深度纹理”中（ShadowMap）。

* 场景从眼睛的位置渲染，但是用标准的阴影纹理把阴影贴图从灯的位置投影到场景中。在每个像素，深度采样与片段到灯的距离进行比较。如果后者大，该像素就不是最靠近灯源的表面。这意味着这个片段是阴影，它在着色过程中不应该接受光照。

--- CG Tutorial

而造轮子的工程有人也做过，如[A希亿博客](http://blog.csdn.net/aceyan0718/article/details/52067264)，使用Unity从头开始生成ShadowMap并应用到阴影中。

主要难点是取视锥体矩阵和应用ShadowMap，原博客都已经提及。


但这种方法的阴影是有__缺陷__的，在做优化时发现，DrawCall有很大一部分时由于阴影造成的。如下图所示：

![frame debugger信息](/img/U3D/Performance/DepthFrameDebugger.png)

可以看到，DepthFrameDebugger显示出，使用这种方式生成阴影，会有大量的DrawCall浪费在绘制ShadowMap，Depth测试。

## Projector Shadow方式

首先想到改善的方式是使用Projector方式生成Shadow，其中有插件，如[Dynamic Shadow Projector](https://assetstore.unity.com/packages/tools/particles-effects/dynamic-shadow-projector-35558)。

方法如贴花系统一样，每个shadow由每个projector产生，大约占3DrawCall。

同样参考了[A希亿关于projector方式产生的阴影](http://blog.csdn.net/aceyan0718/article/details/52279594)。

也看了些[其他人实现的方式](https://www.cnblogs.com/czaoth/p/5785625.html)。

由一个projector参生阴影，但计算Projector的矩阵公式有所不同（可能是博主搞错了)。

公式采用[这个Blog ： http://blog.csdn.net/ronintao/article/details/51649664](http://blog.csdn.net/ronintao/article/details/51649664),使用的公式，计算包围盒。

是要做这样一个prefab ： 上面有一个projector要进行所有阴影投影，还有一个脚本__Shadow Projector.cs__来动态取得projector的包围盒大小、方向，其中还包含有一个replace shader，来提高渲染效率。

在每帧计算包围盒大小公式：

```cs
public static void SetLightCameraBound(Bounds b, Camera lightCamera)
{
	Matrix4x4 lightw2v = lightCamera.transform.worldToLocalMatrix;
	
	//six point, min ~ max	
	Vector4 vnLeftUp = lightw2v * new Vector3(b.max.x, b.max.y, b.max.z);
	
	//AABB
	float maxX = -float.MaxValue;
	float maxY,maxZ,minX,minY,minZ;
	
	float xsize = (maxX - minX) / 2;
	float ysize= (maxY - minY) / 2;
	float zsize = (maxZ - minZ) / 2;
	
	Vector3 positionVec = new Vector3((minX + maxX) / 2, (minY + maxY) / 2, 0);
	Matrix4x4 v2wMatrix = lightCamera.transform.localToWorldMatrix;
	Vector4 result = v2wMatrix * positionVec;
	lightCamera.transform.position = new Vector3(result.x, result.y, result.z);
	
	lightCamera.orthographicSize = Mathf.Max(xsize, ysize);
	lightCamera.nearClipPlane = minZ;
	lightCamera.farClipPlane = maxZ;
}	

```

其实就是把原工程的计算包围盒部分，换为这个blog的公式。

## Planar Shadow

Projector Shadow也会有些问题，比如： 台阶处会出现bug，DrawCall在阴影接收面多的时候变多等等。

所以，参照镇魔曲的阴影实现方式，同时在网上查了查，确定最后项目使用Planar Shadow这种实现方式。

主要参照以下几篇文章：

1. [https://forum.unity.com/threads/shader-sharing-the-most-simple-shadow-shader-planar-projection-shadowing.319830/](https://forum.unity.com/threads/shader-sharing-the-most-simple-shadow-shader-planar-projection-shadowing.319830/)
2. [https://github.com/ozlael/PlannarShadowForUnity](https://github.com/ozlael/PlannarShadowForUnity)

主要参考这两篇文章实现Planar Shadow，原理是基于下面这个公式：

```cs
float4 vPosWorld = mul( _Object2World, v.vertex);
float4 lightDirection = -normalize(_WorldSpaceLightPos0); 
float opposite = vPosWorld.y - _PlaneHeight;
float cosTheta = -lightDirection.y;	// = lightDirection dot (0,-1,0)
float hypotenuse = opposite / cosTheta;
float3 vPos = vPosWorld.xyz + ( lightDirection * hypotenuse );
o.pos = mul (UNITY_MATRIX_VP, float4(vPos.x, _PlaneHeight, vPos.z ,1)); 
```

根据光照方向和给定的shadow平面高度，则可以通过计算得出相应平面上的Shadow位置。

第一篇文章是将shadow计算加到渲染通道中，第二篇则是单独新添的skinned mesh渲染阴影。


实际项目中，将为单独材质，shader为： __Char_PlanarShadow.shader__ ， 贴上源码：

```cs
Shader "Game/Characters/PlanarShadow"
{
	Properties
	{
		[Space(10)]
		_ShadowColor("Shadow Color", Color) = (0.1,0.1,0.1,0.5)
		_PlaneHeight("Plane Height", Float) = 0
	}
	
	SubShader{
		Tags
		{
			"RenderType"="Transparent"
			"Queue"="Transparent"
		}
		
		Pass
		{
			ZTest LEqual
			ZWrite On
			Blend OneMinusSrcAlpha SrcAlpha
			
			Stencil
			{
				Ref 1
				Comp NotEuqal
				Pass Replace
				Fail Keep
				ZFail Keep
				ReadMask 1
				WriteMask 1
			}
			
			CGPROGRAM
			#pragma vertex shadow_vert
			#pragma fragment shadow_frag
			#include "UnityCG.cginc"
			
			float4 _ShadowColor;
			float _PlaneHeight;
			
			struct shadow_v2f
			{
				half4 Pos : SV_POSITION;
				float4 Color : COLOR0;
			};
			
			shadow_v2f shadow_vert(appdata_base v)
			{
				shadow_v2f o;
				//Calculation
				float4 positionInWorldSpace = mul(unity_ObjectToWorld, v.vertex);
				float4 lightDirectionInWorldSpace = 0normalize(_WorldSpaceLightPos0);
				float opposite = positionInWorldSpace.y - _PlaneHeight;
				float cosTheta = -lightDirectionInWorldSpace.y;
				float hypotenuse = opposite / cosTheta;
				float3 vPos = positionInWorldSpace.xyz + lightDirectionInWorldSpace * hypotenuse;
				float4 result = mul(UNITY_MATRIX_VP, float4(vPos.x, _PlaneHeight, vPos.z, 1);
				result.z -= 0.0001;
				
				o.Pos = result;
				o.Color = _ShadowColor;
				
				return o;
			}
			
			fixed4 shadow_frag(shadow_v2f i) : COLOR
			{
				return i.Color;
			}
			ENDCG
		}
	}
		
	Fallback "Diffuse"
}
			
```

运行时，根据人物所处高度，动态设置`_PlaneHeight`

## 小结

本篇讨论了项目中shadow使用的演化过程，几种方法并没有一定要用哪种。 其实，ShadowMap-->ProjectorShadow-->PlanarShadow 方法是越来越`low` 。 意思是，效果大体来讲是越来越受局限的。   


但是，项目目标是手机平台上，且效果要求并非是要完美处理遮挡关系、”桥上桥下“等问题，则使用Planar Shadow方式，在能接受的范围内最可取。