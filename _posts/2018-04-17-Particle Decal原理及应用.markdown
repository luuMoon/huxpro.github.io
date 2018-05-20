---
layout:     post
title:      "HSV与RGB颜色转换"
subtitle:   "通过hsv调整饱和度"
date:       2018-04-17
author:     "Luffy"
header-img: "./img/BG/topbackground.jpg"
catalog: true
tags:
- U3D
- 效果
- shader

---

## 前言
通过shader，将color的rgb转换为hsv空间，调整图片。

## 原理及实现

[wiki关于HSV及RGB的解释](https://zh.wikipedia.org/wiki/HSL和HSV色彩空间)

HSV,即色相（H，颜色），饱和度（S纯度），亮度（V/L）。 

使用圆锥或圆筒表示，转换公式wiki中附带。

## 优化及代码

往上有一些针对GPU命令优化转换的版本：

1.[HSV shader算法优化](http://lolengine.net/blog/2013/07/27/rgb-to-hsv-in-glsl)

2.[Uniy 开源HSV转换](https://github.com/greggman/hsva-unity)

针对特定算法，基于GPU指令优化，其中2里的转换公式与1相同。


```cs

float3 rgb2hsv(float3 c) {
      float4 K = float4(0.0, -1.0 / 3.0, 2.0 / 3.0, -1.0);
      float4 p = lerp(float4(c.bg, K.wz), float4(c.gb, K.xy), step(c.b, c.g));
      float4 q = lerp(float4(p.xyw, c.r), float4(c.r, p.yzx), step(p.x, c.r));

      float d = q.x - min(q.w, q.y);
      float e = 1.0e-10;
      return float3(abs(q.z + (q.w - q.y) / (6.0 * d + e)), d / (q.x + e), q.x);
}

float3 hsv2rgb(float3 c) {
      c = float3(c.x, clamp(c.yz, 0.0, 1.0));
      float4 K = float4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
      float3 p = abs(frac(c.xxx + K.xyz) * 6.0 - K.www);
      return c.z * lerp(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
}
          
```

