---
layout: post
title: "URP Shell-based Fur Rendering"
date: 2026-01-16
categories: [Shader]
image: "/post-img/shell-based-fur-rendering/Cover.png"
tags: [Unity, Fur Rendering]
published: true
---

## Overview

This post introduces a real-time **Shell-based Fur Rendering** technique implemented in Unity URP. 
The core idea is to render multiple layers (shells) of the same mesh, each offset along the normal direction, combined with noise-based alpha clipping to simulate individual fur strands. GPU Instancing is used to efficiently render all shell layers in a single draw call batch.

---

## Fur Preview
![Fur Preview](/post-img/shell-based-fur-rendering/Preview.gif)

---

## 1. Implementation Approach

The shell-based method works by stacking multiple copies of the mesh on top of each other. Each layer is pushed outward along the surface normal by a small amount. A noise texture controls which pixels are visible on each layer — pixels near the tips of the fur are progressively clipped away, creating the illusion of tapered fur strands.

The key parameters driving this are:
- FurStep: a value from 0 to 1 representing which shell layer is being rendered (0 = base, 1 = tip)
- FurLength: the total length of the fur
- Noise textures: used to define the shape and density of each strand

---

## 2. Script Implementation

The `FurGenerator` C# script is responsible for generating all shell layers at runtime using `Graphics.DrawMeshInstanced`. Each instance represents one shell layer, with its `_FurStep` value passed via a `MaterialPropertyBlock`.

### Implementation
Mesh Acquisition — supports both regular `MeshFilter` and `SkinnedMeshRenderer`:
```csharp
MeshFilter meshFilter = GetComponent();
if (meshFilter != null)
{
    furMesh = meshFilter.sharedMesh;
}
else
{
    SkinnedMeshRenderer skinnedMesh = GetComponent();
    if (skinnedMesh != null) furMesh = skinnedMesh.sharedMesh;
}
```

Dynamic Instance Count — arrays are re-initialized when `instanceCount` changes:
```csharp
if (instanceCount != _lastInstanceCount)
{
    instanceCount = Mathf.Max(2, instanceCount);
    matrices = new Matrix4x4[instanceCount];
    _FurStepArray = new float[instanceCount];
    _lastInstanceCount = instanceCount;
}
```

Per-Instance FurStep — each layer gets a unique `_FurStep` value and shares the same world matrix:
```csharp
for (int i = 0; i < instanceCount; i++)
{
    float furStep = (float)i / (instanceCount - 1);
    _FurStepArray[i] = furStep;
    matrices[i] = baseWorldMatrix;
}
materialPropertyBlock.SetFloatArray("_FurStep", _FurStepArray);
Graphics.DrawMeshInstanced(furMesh, 0, furMaterial, matrices, instanceCount, materialPropertyBlock, ShadowCastingMode.Off, true, gameObject.layer)
```

---

## 3. Shader Implementation

This shader handles shell expansion in the vertex stage and noise-based alpha clipping in the fragment stage, with full PBR lighting support.

### Vertex Shader — Shell Expansion & Curly
Each shell is offset along the object-space normal based on `_FurStep`:
```hlsl
half3 positionOffset = furStep * _FurLength * input.normalOS.xyz;
input.positionOS.xyz += positionOffset;
```

A curl effect is applied using a rotation matrix driven by a noise-based random seed:
```hlsl
half noise = SAMPLE_TEXTURE2D_LOD(_FurControlMap1, sampler_FurControlMap1, 
    input.texcoord * _FurThinness1, 0).r;
half3 curlVector = FurCurl(input.normalOS, input.tangentOS, _CurlFactor, noise, furStep, _CurlRadius);
input.positionOS.xyz += curlVector * _FurLength * furStep;
```

The `FurCurl` function rotates the tangent vector around the normal using a per-strand random angle:
```hlsl
float3 FurCurl(float3 normal, float3 tangent, float frequence, float id, float curlAmount, float curlRadius)
{
    float random = RND(id);
    return mul(rotationMatrix(normal, 2 * 3.14159 * frequence * curlAmount * random), 
        float4(tangent, 0) * curlRadius * 0.1 * (random - frequence));
}
```

### Fragment Shader — Alpha Clipping
Two noise textures are sampled and combined to define strand shapes. Alpha is computed by subtracting the squared `furStep` from the noise, creating a natural taper toward the tips:
```hlsl
half noise1 = SAMPLE_TEXTURE2D(_FurControlMap1, sampler_FurControlMap1, input.uv * _FurThinness1);
half noise2 = SAMPLE_TEXTURE2D(_FurControlMap2, sampler_FurControlMap2, input.uv * _FurThinness2);
half noise = max(noise1, noise2);
half alpha = saturate(noise - (furStep * furStep));
```
PBR lighting is then calculated using the sampled BaseColor, MetallicSmoothness, NormalMap, and OcclusionMap textures.
---

## 4. Adjustable Parameters

FurControlMap, FurLength, FurThinness, CurlFactor, CurlRadius, Color

![Parameters](/post-img/shell-based-fur-rendering/Parameters.gif)

---

## 5. Advantages & Disadvantages

### Advantages

- Simple to implement — no geometry shader or complex mesh generation required
- Easy to control — all parameters are exposed and adjustable in real time
- GPU Instancing — all layers share one material and are batched efficiently
- Compatible with any mesh — works on both static and skinned meshes

### Disadvantages

- Short fur only — the shell method breaks down visually for long fur, as gaps between layers become visible from grazing angles
- Transparency overdraw — every shell layer uses alpha blending, which causes significant overdraw and can be expensive on complex meshes
- Limited mobile support — 20-30 layers is acceptable on PC, but should be reduced to 5-10 layers or fewer for mobile platforms

---

## 6. Other Fur Rendering Approaches

- Geometry-based Fur (Real Modeling) 
Artists manually model individual fur strands or clumps directly into the mesh. This produces the highest quality results but is expensive in terms of polygon count and artist time. Best suited for hero characters in cinematics.

- Fin / Card-based Fur 
Thin polygonal cards (fins) are inserted perpendicular to the surface, with a fur texture applied with alpha clipping. This approach handles long fur and silhouettes much better than shell-based methods, and is more performance-friendly. Widely used in real-time games for animals and characters with longer fur.
