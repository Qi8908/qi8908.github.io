---
layout: post
title: "Unity URP Volumetric Fog"
date: 2026-01-29
categories: [Graphics, Unity]
image: "/post-img/volumetric-fog-unity/Cover.png"
tags: [Unity, URP, VolumetricFog]
published: true
---

## Overview

This post documents the implementation of a local volumetric fog effect in Unity URP using a custom HLSL shader. Unlike global fog solutions or third-party plugins, this approach uses **raymarching** inside a bounding box to simulate realistic, controllable 
fog volumes directly in the scene.

The core technique involves:
- Ray-AABB intersection to define the fog volume boundary
- Raymarching through the volume to accumulate optical depth
- 3D Perlin Noise texture sampling for organic, animated fog turbulence
- Height gradient and edge fade for natural blending

---

## Fog Preview
![Fog Preview](/post-img/volumetric-fog-unity/Preview.gif)

---

## Adjustable Parameters
![Adjustable Parameters](/post-img/volumetric-fog-unity/Parameters.png)

---

## Shader Implementation Approach

The shader works by casting a ray from the camera through each fragment, computing 
where it enters and exits the fog bounding box, then marching along the ray and 
accumulating density at each step.

### 1. Ray-AABB Intersection

The first step is determining where the camera ray enters and exits the fog volume. 
The bounding box is defined in object space as a unit cube (`-0.5` to `0.5`), then 
transformed to world space.
```hlsl
bool rayAABBIntersection(float3 rayOrigin, float3 rayDir, 
                          float3 boundsMin, float3 boundsMax,
                          out float tMin, out float tMax)
{
    float3 invRayDir = 1.0 / rayDir;
    float3 t1 = (boundsMin - rayOrigin) * invRayDir;
    float3 t2 = (boundsMax - rayOrigin) * invRayDir;
    float3 tMin3 = min(t1, t2);
    float3 tMax3 = max(t1, t2);
    tMin = max(max(tMin3.x, tMin3.y), tMin3.z);
    tMax = min(min(tMax3.x, tMax3.y), tMax3.z);
    return tMax >= tMin && tMax > 0;
}
```

### 2. 3D Perlin Noise Sampling

This shader requires a **3D Perlin Noise texture** assigned to the `_Noise3D` slot. A standard 2D noise will not work — you need a true `Texture3D` asset. You can generate one via a Unity Editor script or import a pre-baked `.asset` file.

The noise is sampled in 3D space and animated over time using `_NoiseSpeed`, giving the fog a natural flowing turbulence:
```hlsl
float sampleNoise3D(float3 position)
{
    float3 samplePos = position * _NoiseScale + _NoiseSpeed.xyz * _Time.y;
    return SAMPLE_TEXTURE3D(_Noise3D, sampler_Noise3D, samplePos).r;
}
```

The noise is then blended into the density function:
```hlsl
float noiseValue = sampleNoise3D(noisePos) * _NoiseStrength;
return _FogDensity * fade * heightFactor * (1.0 + noiseValue);
```

### 3. Raymarching Loop

The ray is marched from the entry point to the exit point. At each step, density is sampled and accumulated into `opticalDepth`. An early-exit optimization breaks the loop when the fog becomes fully opaque:
```hlsl
for (int i = 0; i < numSteps; i++)
{
    float density = fogDensity(currentPos, boundsMinWS, boundsMaxWS) * stepSize;
    opticalDepth += density;
    currentPos += rayDir * stepSize;
    if (opticalDepth > 10.0) break;
}
```

### 4. Final Transparency Calculation

Beer-Lambert law is used to convert optical depth into fog alpha:
```hlsl
float transparency = exp(-opticalDepth);
float alpha = 1.0 - transparency;
```
