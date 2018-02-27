---
layout:     post
title:      "Custom Water Shader"
subtitle:   "项目中水面Shader编写及演变过程"
date:       2017-12-12
author:     "Luffy"
header-img: "./img/BG/topbackground.jpg"
catalog: true
tags:
- U3D
- Shader
- 场景
- 优化

---

## 前言

不同游戏实现渲染水的方式也不同，卡通的、真实的、低模的等等。但基础原理是差不多的，不同的效果：UV移动、高光、真实反射、边缘模糊等，如组合一样，需要什么加什么。

## 实现及思路

童话风格游戏不求水的表现十分真实，求的是一些比较好的效果。

* 实现的功能有：透明、描边(随水深变化)、高光反射（PBR，传统高光不聚，不好看）等。

* 参考基础：Stylized Water 插件， U3D自带(Standard)高光实现

* 其中，为减少Draw Call，关闭Depth Test，描边不使用Depth Texture ， 使用事先计算好的Depth图

## 代码

包含头文件：_UnityPBSLighting.cginc_ 与 _UnityStandardBRDF.cginc_

计算公式需要的变量：

* UV偏移： `fixed mWaveSpeed1 = (_Time.g * (_WaveSpeed1 * 0.1));`
* 夹角等： `float NdotH = max(0.0, dot(normalDirection, halfDirection));` 

	`float NdotV = max(0.0, dot(normalDirection, viewDirection));`
	

计算高光：

```cs
float visTerm = SmithJointGGXVisibilityTerm(NdotL, NdotV, 1.0 - gloss);
float normTerm = max(0.0, GGXTerm(NdotH, 1.0 - gloss);
float specularPBL = (NdotL * visTerm * normTerm) * (UNITY_PI / 4);
specularPBL = max(0, specularPBL * NdotL);
float3 directSpecular = specularPBL * FresnelTerm(specularColor, LdotH);
```


原有使用DepthTexture实现透明度变化，以实现水深描边

```cs
float sceneZ = max(0, LinearEyeDepth(UNITY_SAMPLE_DEPTH(tex2Dproj(_CameraDepthTexture, UNITY_PROJ_COORD(i.projPos)))) - _ProjectionParams.g);
float partZ = max(0, i.projPos.z - _ProjectionParams.g);
float transparent = saturate((sceneZ - partZ) / _Depth) * _Transparency;
```

生成深度图片存储下来，并运用到shader中计算深度，去除DepthTexture依赖，减少Draw Call

计算：

	不使用顶点往下取深度，精确度不够
	
```cs
public void GetHeightMap()
{
	vertexBuffer.Clear();
	WaterMesh.GetComponent<MeshFilter>().sharedMesh.getVertices(vertexBuffer);
	
	float heightY = vertexBuffer.Count > 0 ? WaterMesh.TransformPoint(vertexBuffer[0]).y + yOffset : 0;
	
	float minX = float.MaxValue;
	float minZ,maxX,maxZ;
	
	for(int i = 0; i < vertexBuffer.Count; i++)
	{
		Vector3 wordPosW = WaterMesh.transformPoint(vertexBuffer[i]);
		if(wordPosW.x < minX) minX = worldPosW.x;
		...
	}
	
	Texture2D heightMap = new Texture2D(TextureSize, TextureSize, TextureFormat.RGBA32, false);
	
	Vector3 minPos = new Vector3(minX, heightY, minZ);
	float tempAddX = (maxX - minX) / (float)TextureSize;
	float tempAddZ = (maxZ - minZ) / (float)TextureSize;
	
	//Raycast from every point in 512*512 position, and get DepthMap
	for(float i = 0; i < TexturSize; i++)
		for(float i = 0; i < TexturSize; i++)
		{
			Vector3 tempPos = new Vector3(minPos.x + i * tempAddX, heightY, minPos.z + j * tempAddZ);
			
			Vector3 downDir = WaterMesh.TransformDirection(Vector3.down);
			RaycastHit hit;
			if(Physics.Raycast(tempPos, downDir, out hit, 10, 1 << LayerMask.NameToLayer("Terrain")))
				{
					terrainPos.r = Mathf.Clamp01(heightY - hit.point.y);
				}
			heightMap.SetPixel((int)i, (int)j, terrainPos);
		}
}
	
```

之后在Shader中，就可以运用生成的这张DepthMap：


```cs
fixed4 waterMapValue = tex2D(_DepthTex, i.uv_WaterMap);
float transparent = saturate(waterMapValue.r / _Depth) * _Transparency;
```

## 注意
由于地形的*Pixel Error*问题，而计算深度时总是会计算Pixel Error为0时的地形高度，而Pixel Error少时，Draw Call过高，可考虑Terrain to Mesh插件，转为Mesh。


