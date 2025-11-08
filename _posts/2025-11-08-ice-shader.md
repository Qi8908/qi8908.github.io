---
layout: post
title: "Ice Shader: Opaque Ice"
date: 2025-11-08
categories: [Graphics, Shader]
image: "/post-img/ice-shader/Cover.png"
tags: [Unity, Ice Shader, SSS]
published: true
---

## Preview
![Ice Rendering Preview](/post-img/ice-shader/Cover.png)

## Main Points
- [1. Parallax Mapping](#1-parallax-mapping)
- [2. Subsurface Scattering](#2-subsurface-scattering)
- [3. MatCap Reflection Enhancement](#3-matcap-reflection-enhancement)

---

This article introduces two approaches to creating ice materials. The first approach is opaque ice, utilizing **Parallax Mapping** to simulate internal depth, **Subsurface Scattering** to represent light penetration effects, and **MatCap Reflection** to enhance surface highlights.

---

## 1. Parallax Mapping

Parallax mapping offsets UV coordinates based on a height map to simulate the three-dimensional structure inside the ice.

### Core Function
```hlsl
half2 BumpOffset(float2 uv, half3 viewDir, half height)
{
    half2 offset = (viewDir.xy / viewDir.z) * height;
    uv = uv + offset;
    return uv;
}
```

### Implementation
```hlsl
// Sample height map
half height = SAMPLE_TEXTURE2D(_HeightMap, sampler_HeightMap, input.uv * _HeightTiling) * _HeightRange;

// Construct TBN matrix to transform view direction to tangent space
half3 bitTangentWS = cross(input.normalWS, input.tangentWS.xyz) * input.tangentWS.w;
half3x3 TBN = half3x3(input.tangentWS.xyz, bitTangentWS, input.normalWS.xyz);
half3 viewTS = mul(TBN, normalize(input.viewDirWS));

// Calculate offset UV and sample inner texture
half2 uvOffest = BumpOffset(input.uv * _InnerMapTiling, viewTS, height);
half4 innerColor = SAMPLE_TEXTURE2D(_InnerMap, sampler_InnerMap, uvOffest);
half3 albedo = (baseMap.rgb * _BaseColor.rgb + innerColor * _InnerColor) * 0.3;
```

The UV offset is calculated based on view angle and height information to create a depth illusion for inner textures. The TBN matrix is used for coordinate space transformation, and the final multiplication by 0.3 ensures energy conservation.

---

## 2. Subsurface Scattering

Uses thickness maps combined with wrap lighting to simulate light scattering effects inside the ice.

### Property Definitions
```hlsl
_ThicknessMap("Thickness Map", 2D) = "white" {}
_ThicknessScale("Thickness Scale", Range(0, 50)) = 10.0
_ScatterColor("Scatter Color", Color) = (1,1,1,1)
```

### Thickness Sampling
```hlsl
half thickness = SAMPLE_TEXTURE2D(_ThicknessMap, sampler_ThicknessMap, input.uv).r * _ThicknessScale;

surfaceInput.scatteringColor = _ScatterColor;
surfaceInput.thickness = thickness;
```

The thickness map simulates volume variations across the material. Thinner areas allow light to penetrate more easily with stronger scattering effects, while thicker areas show the opposite behavior.

### SSS Lighting Calculation
```hlsl
#if defined(_ICE)
    // Wrap lighting
    half Wrap = 1.2;
    half WrapNoL = saturate((-dot(N,L) + Wrap) / pow(1+Wrap, 2));
    
    // Scattering term
    half VoL = dot(V,L);
    float Scatter = D_GGX(saturate(-VoL), 0.6 * 0.6);
    
    // Final transmitted light
    half3 transmition = light.color * light.shadowAttenuation * WrapNoL * Scatter * 
                        surface_data.scatteringColor * (1.0 + surface_data.thickness * 3.0);
    
    lightingresult.directDiffuse += transmition;
#endif
```

Wrap lighting captures backlight penetration effects, while GGX distribution controls the angular distribution of scattering. The thickness parameter directly affects transmitted light intensity: larger thickness values indicate thicker regions where light travels longer distances through internal scattering, resulting in more pronounced scattering effects.

---

## 3. MatCap Reflection Enhancement

Uses pre-rendered spherical maps to simulate environment reflections with minimal performance overhead.

### Implementation
```hlsl
// Transform world space normal to view space
half3 viewNormal = normalize(TransformWorldToViewDir(normalWS));

// Map normal XY components to UV coordinates: [-1,1] → [0,1]
float2 matcapUV = viewNormal.xy * 0.5 + 0.5;

// Sample and add to albedo
half3 reflectColor = SAMPLE_TEXTURE2D(_MatCap, sampler_MatCap, matcapUV) * _MatCapIntensity;
albedo.rgb += reflectColor;
```

Maps normal directions to UV coordinates to sample a spherical map, simulating environment reflections and highlight effects.
