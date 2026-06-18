---
layout: post
title: "Unity Post-Processing: Exponential Height Fog"
date: 2025-12-08
categories: [Shader, Unity]
image: "/post-img/exponential-height-fog-post-processing/Cover.png"
tags: [Unity, Fog Rendering, Post-Processing]
published: true
---

## Overview

In Unity's Universal Render Pipeline (URP), RenderFeature enables custom post-processing effects through a three-stage workflow:

1. RenderFeature → Initialize Pass parameters and enqueue the Pass
2. RenderPass → Receive and configure Shader parameters, then execute the Pass using CommandBuffer (Blit) for rendering. 
3. Shader → Handle post-processing logic with access to scene color, depth, and other relevant data

---

## Fog Preview
![Fog Preview](/post-img/exponential-height-fog-post-processing/Preview.gif)

---

## 1. Initialization Phase (Create)

In the `Create()` method of `ScriptableRendererFeature`:
- Initialize Pass parameters
- Call `EnqueuePass` to add the custom Pass to the render queue

### Implementation
```csharp
public override void Create()
{
    fogMaterial = CoreUtils.CreateEngineMaterial("Hidden/ExponentialHeightFog");
    fogPass = new ExponentialHeightFogPass(fogMaterial);
}
```

**Key Point**: Use `CoreUtils.CreateEngineMaterial()` to create materials, ensuring they load correctly in both editor and runtime.

---

## 2. Parameter Setup Phase (AddRenderPasses)

In the `AddRenderPasses()` method:
- Receive and set Shader parameters
- Execute `ExecutePass`, using CommandBuffer (Blit) for rendering
- Differentiate RT usage scenarios

### Implementation
```csharp
public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
{
    if (renderingData.cameraData.cameraType == CameraType.Game || 
        renderingData.cameraData.cameraType == CameraType.SceneView)
    {
        fogPass.UpdateParams(
            settings.fogColor,
            settings.fogDensity,
            settings.fogHeightFalloff,
            settings.fogStartDistance,
            settings.fogHeight,
            settings.fogMaxDistance
        );
        renderer.EnqueuePass(fogPass);
    }
}
```

**Best Practice**: Limit camera types to avoid executing post-processing on unnecessary cameras (such as preview cameras).

---

## 3. Shader Processing Phase (Execute)

This is the actual rendering logic execution phase, primarily handling:
- Post-processing related logic
- RT (RenderTexture) allocation and usage strategy

### Implementation
```csharp
public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
{
    if (fogMaterial == null) return;

    CommandBuffer cmd = CommandBufferPool.Get("Exponential Height Fog");
    source = renderingData.cameraData.renderer.cameraColorTargetHandle;
    
    // Set material parameters
    fogMaterial.SetColor("_FogColor", fogColor);
    fogMaterial.SetFloat("_FogDensity", fogDensity);
    // ... other parameters
    
    // Apply fog effect
    Blit(cmd, source, tempTexture.Identifier(), fogMaterial);
    Blit(cmd, tempTexture.Identifier(), source);
    
    context.ExecuteCommandBuffer(cmd);
    CommandBufferPool.Release(cmd);
}
```

---

## RT (RenderTexture) Allocation Strategy

Regarding RT allocation, you need to choose an appropriate strategy based on actual requirements:

### Allocate vs Shared RT

**When to use Allocate (dedicated RT)**:
- Need to pass data between different Passes
- Need to preserve intermediate results for subsequent use
- Effect requires multiple iterative calculations

**When to use Shared RT**:
- Only used within the current Pass
- Can save memory overhead
- No need to pass data between multiple Passes

### Implementation
In our fog implementation, we use temporary RT:

```csharp
public override void Configure(CommandBuffer cmd, RenderTextureDescriptor cameraTextureDescriptor)
{
    cmd.GetTemporaryRT(tempTexture.id, cameraTextureDescriptor);
}

public override void FrameCleanup(CommandBuffer cmd)
{
    cmd.ReleaseTemporaryRT(tempTexture.id);
}
```

This approach releases RT at the end of each frame, suitable for simple post-processing effects.

---

### Shader Core Logic

```hlsl
// Reconstruct world position from depth
float3 worldPos = ReconstructWorldPosition(i.uv, depth);

// Calculate fog distance
float fogDistance = max(0.0, linearDepth - _FogStartDistance);
fogDistance = min(fogDistance, _FogMaxDistance);

// Calculate height factor (key: exponential decay)
float heightFactor = max(0.0, (_FogHeight - worldPos.y) * _FogHeightFalloff);
float fogDensity = _FogDensity * exp(-heightFactor);

// Final fog intensity
float fogFactor = 1.0 - exp(-fogDensity * fogDistance);
fogFactor = saturate(fogFactor);
```

### Key Technical Points

1. **Depth Reconstruction**: Use `SampleSceneDepth()` and `LinearEyeDepth()` to obtain scene depth
2. **World Position Reconstruction**: Recover world space position from depth through inverse projection matrix
3. **Exponential Decay**: Use `exp(-heightFactor)` to implement height-related density attenuation
4. **Distance Control**: Control fog range through `_FogStartDistance` and `_FogMaxDistance`

### Property Definitions
![Fog Settings](/post-img/exponential-height-fog-post-processing/SettingCapture.png)

```csharp
[System.Serializable]
public class FogSettings
{
    public Color fogColor = new Color(0.5f, 0.6f, 0.7f, 1.0f);
    [Range(0, 0.1f)] public float fogDensity = 0.01f;
    [Range(0, 0.1f)] public float fogHeightFalloff = 0.01f;
    [Range(0, 100)] public float fogStartDistance = 10f;
    public float fogHeight = 0f;
    public float fogMaxDistance = 500f;
}
```

| Parameter | Purpose | Recommended Range |
|-----------|---------|-------------------|
| `fogColor` | Fog color | - |
| `fogDensity` | Base fog density | 0.001 - 0.1 |
| `fogHeightFalloff` | Height decay rate | 0.001 - 0.1 |
| `fogStartDistance` | Fog start distance | 0 - 100 |
| `fogHeight` | Fog base height | Set according to scene |
| `fogMaxDistance` | Maximum fog distance | 100 - 1000 |

---

## Key Implementation Notes

### Performance Optimization

- Perform camera type checks in `AddRenderPasses` to avoid unnecessary rendering
- Use temporary RT instead of persistent RT to reduce memory usage
- Properly set `renderPassEvent` to avoid conflicts with other effects

### Depth Sampling Configuration

Ensure Depth Texture is enabled in URP Asset:
```
URP Asset → General → Depth Texture: Enabled
```

### Coordinate System Considerations

When reconstructing world coordinates, note:
- UV start position may differ across platforms (use `UNITY_UV_STARTS_AT_TOP` macro)
- NDC coordinate range is [-1, 1]
- Need to consider projection matrix differences
