---
layout:     post
title:      "ShadowCaster的应用"
subtitle:   "阴影穿透BUG及使用ShadowCaster解决方法"
date:       2017-06-01
author:     "Luffy"
header-img: "./img/BG/topbackground.jpg"
catalog: true
tags:
- Shader
- U3D
- BUG

---

## 前言

在写完场景的自发光Shader之后，在实际应用中会发现一个问题： 当开启阴影时会发生阴影的显示不正确、穿透等问题，如： 人物的影子出现在树上等。  

出现的原因在查了一些资料后发现，是由于Unity在生成深度图时，需要Shader中含有ShadowCaster的Pass，由于自己写的Shader没有此通道导致相应部分没有写入深度图，从而导致开启阴影时所造成的各种显示问题。

## 阴影生成原理及ShadowCaster

在Unity中，阴影采用ShadowMap生成，原理参考[这篇BLOG](http://blog.csdn.net/aceyan0718/article/details/52067264) ： 通过光源位置为视点渲染一张阴影贴图；然后从眼睛位置渲染，将每一像素与光源的距离与对应位置的阴影贴图（深度图）进行比较，若前者较大则为阴影(相关实现留在以后再写)。 之前的Unity采用ReplaceShader的方式渲染深度图，现在则使用ShadowCaster的方式。

参考[官方文档](https://docs.unity3d.com/Manual/SL-VertexFragmentShaderExamples.html)中关于ShadowCaster的描述。

要使物体可以生成阴影，则需要在Shader中的任一SubShader或fallback中拥有ShadowCaster通道。所以可以使用以下方式来最简单的使用ShadowCaster:

```c
UsePass "Legacy Shaders/VertexLit/SHADOWCASTER"
```

等同于如下Pass：

```c
Pass
        {
            Tags {"LightMode"="ShadowCaster"}

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_shadowcaster
            #include "UnityCG.cginc"

            struct v2f { 
                V2F_SHADOW_CASTER;
            };

            v2f vert(appdata_base v)
            {
                v2f o;
                TRANSFER_SHADOW_CASTER_NORMALOFFSET(o)
                return o;
            }

            float4 frag(v2f i) : SV_Target
            {
                SHADOW_CASTER_FRAGMENT(i)
            }
            ENDCG
        }
```

但是有个缺陷： 因为树木的Shader中有`clip`操作，从而使得生成的ShadowMap错误，解决方式是在手写的ShadowCaster通道中也加入相应的clip操作，如下所示：

```c
float4 frag(v2f i) : SV_Target
{
	fixed4 col = tex2D(_MainTex, i.uv) * _Color;
	//discard
	clip(col.a - _Cutoff);
	SHADOW_CASTER_FRAGMENT(i)
}
```

这样一来，就可以生成正确的ShadowMap，从而显示正确的阴影。

## 小结

使用ShadowCaster来解决相应的阴影显示问题，ShadowMap生成原理可以作为下一篇BLOG内容。
