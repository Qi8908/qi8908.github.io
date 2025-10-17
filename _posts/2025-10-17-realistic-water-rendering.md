---
layout: post
title: "Realistic Water Rendering"
date: 2025-10-07
categories: [Graphics, Shader]
image: "/post-img/realistic-water-rendering/WaterRender.jpg"
tags: [Unity, Water Shader]
published: true
---

## Preview
![Water Rendering Preview](/post-img/realistic-water-rendering/WaterRender.gif)

## Main Points
- [1. Flow Map](#1-flow-map)
- [2. BRDF Lighting](#2-brdf-lighting)
- [3. Planar Reflection](#3-planar-reflection)
- [4. Refraction](#4-refraction)
- [5. Depth Fade](#5-depth-fade)
- [6. Caustics](#6-caustics)
- [7. Scatter](#7-scatter)
- [8. Foam](#8-foam)
- [9. Vertex Displacement (Wave)](#9-vertex-displacement-wave)
- [Key Optimizations](#key-optimizations)

---

## 1. Flow Map
![Flow Map](/post-img/realistic-water-rendering/ShaderToy.png)

Implements seamless looping flow using dual-layer texture blending to avoid repetition.
```hlsl
// Generate two phases with 0.5 offset for seamless looping
float speed = _Time.x * _FlowSpeed;
half phase0 = frac(speed + 0.5);
half phase1 = frac(speed + 1.0);
half flowlerp = saturate(abs(0.5 - phase0) / 0.5); 

// Dual-layer normal map interpolation
half4 normalMap1 = SAMPLE_TEXTURE2D(_BumpMap, sampler_BumpMap, input.worldUV.xy + phase0 * flowDir);
half4 normalMap2 = SAMPLE_TEXTURE2D(_BumpMap, sampler_BumpMap, input.worldUV.xy + phase1 * flowDir);
half4 normalBlend1 = lerp(normalMap1, normalMap2, flowlerp);
```

---

## 2. BRDF Lighting

Uses standard PBR pipeline to simulate water's high glossiness and non-metallic properties.
```hlsl
surfaceData.metallic = 0;
surfaceData.smoothness = 0.9;
half4 color = StandardLighting(lightingInput, surfaceData);
```

---

## 3. Planar Reflection
![Reflection](/post-img/realistic-water-rendering/Reflection.gif)

### C# Script Implementation Steps

#### Step 1: Create Reflection Camera
```csharp
private Camera CreateMirrorObjects()
{
    var go = new GameObject("Planar Reflections", typeof(Camera));
    var reflectionCamera = go.GetComponent<Camera>();
    reflectionCamera.depth = -10;
    reflectionCamera.enabled = false;
    return reflectionCamera;
}
```

#### Step 2: Calculate Reflection Plane
```csharp
// Define reflection plane (water surface position + normal)
Vector3 pos = target.transform.position + Vector3.up * m_planeOffset;
Vector3 normal = target.transform.up;
var d = -Vector3.Dot(normal, pos) - m_settings.m_ClipPlaneOffset;
var reflectionPlane = new Vector4(normal.x, normal.y, normal.z, d);
```

#### Step 3: Build Reflection Matrix
```csharp
// Calculate reflection matrix based on reflection plane
private static void CalculateReflectionMatrix(ref Matrix4x4 reflectionMat, Vector4 plane)
{
    reflectionMat.m00 = (1F - 2F * plane[0] * plane[0]);
    reflectionMat.m01 = (-2F * plane[0] * plane[1]);
    // ... Complete reflection matrix calculation
    reflectionMat.m11 = (1F - 2F * plane[1] * plane[1]);
}
```

#### Step 4: Update Camera Position and Transform
```csharp
// Reflection camera position
var oldPosition = realCamera.transform.position - new Vector3(0, pos.y * 2, 0);
var newPosition = ReflectPosition(oldPosition);
_reflectionCamera.transform.position = newPosition;

// Apply reflection matrix to worldToCameraMatrix
_reflectionCamera.worldToCameraMatrix = realCamera.worldToCameraMatrix * reflection;
```

#### Step 5: Setup Oblique Projection
```csharp
// Calculate clip plane to render only objects above water surface
var clipPlane = CameraSpacePlane(_reflectionCamera, pos - Vector3.up * 0.1f, normal, 1.0f);
var projection = realCamera.CalculateObliqueMatrix(clipPlane);
_reflectionCamera.projectionMatrix = projection;
```

#### Step 6: Render and Pass Texture
```csharp
private void ExecutePlanarReflections(ScriptableRenderContext context, Camera camera)
{
    UpdateReflectionCamera(camera);
    PlanarReflectionTexture(camera); // Create RenderTexture
    
    UniversalRenderPipeline.RenderSingleCamera(context, _reflectionCamera);
    
    // Pass to Shader
    Shader.SetGlobalTexture(_planarReflectionTextureId, _reflectionTexture);
}
```

#### Step 7: Script Settings
![Water Rendering Preview](/post-img/realistic-water-rendering/Planar Reflection.png)

Do not set Reflect Layers to "Everything". 
Exclude the water layer (e.g., create a "Water" layer, assign it to the water object, and uncheck it in Reflect Layers) to prevent the water from reflecting itself.

### Using Reflection in Shader
```hlsl
// Sample reflection texture with normal distortion
half2 reflectionDistortion = normalWS.xz * _ReflectionDistortion;
half4 reflectionColor = SAMPLE_TEXTURE2D(_PlanarReflection, sampler_PlanarReflection, 
                         screenUV + reflectionDistortion) * _ReflectionIntensity;

surfaceData.reflectionColor = reflectionColor;
```

---

## 4. Refraction

Sample opaque texture with normal-distorted UV.
```hlsl
// Distort screen UV based on normal
half2 distortedUV = screenUV + normalWS.xz * _RefractionStrength;

// Mask to prevent distortion above water surface
half refractionMask = step(waterDepth, 0);
distortedUV.xy = refractionMask * screenUV + (1 - refractionMask) * distortedUV;

float3 underWaterColor = SampleSceneColor(distortedUV);
```

---

## 5. Depth Fade

Calculate water depth for color transition.
```hlsl
// Get scene depth and water depth
float sceneDepth = LinearEyeDepth(SampleSceneDepth(screenUV), _ZBufferParams);
float waterDepth = sceneDepth - input.screenPos.w;
waterDepth = max(0, waterDepth);

// Depth attenuation
float depthFactor = saturate(exp2(-waterDepth * _DepthGradient));
float alphaFactor = 1 - saturate(exp2(-waterDepth * _WaterClean));

// Color blending
half4 albedo = lerp(_DeepColor, _ShallowColor, depthFactor);
```

---

## 6. Caustics
![Caustics](/post-img/realistic-water-rendering/Caustics.gif)

RGB channel separation sampling to simulate chromatic dispersion.
```hlsl
float causticFactor = saturate(exp2(-waterDepth * _CausticDepth));
float3 channel_offset = _Channel_Offset * 0.01;

// Sample per channel
float r1 = SAMPLE_TEXTURE2D(_CausticMap, sampler_CausticMap, 
           input.worldUV.xy * _CausticTiling + speed * flowDir + channel_offset.x).r;
float g1 = SAMPLE_TEXTURE2D(_CausticMap, sampler_CausticMap, 
           input.worldUV.xy * _CausticTiling + speed * flowDir + channel_offset.y).g;
float b1 = SAMPLE_TEXTURE2D(_CausticMap, sampler_CausticMap, 
           input.worldUV.xy * _CausticTiling + speed * flowDir + channel_offset.z).b;

// Dual-layer reverse sampling with minimum value
half3 caustic = min(caustics1, caustics2) * causticFactor;
albedo.rgb += caustic;
```

---

## 7. Scatter

**Approach 1: Simplified Reflection-Based Scattering** (Mobile-friendly)
```hlsl
// Suitable for shallow water / mobile platforms / distant views
float3 R = reflect(V, N);
half3 scatter = SchlickPhase(0.2, dot(-light.direction, R)) * surface_data.scatteringColor;
```
- Uses Schlick phase function for forward scattering
- Performance-friendly for mobile devices
- Good for lakes and shallow water bodies

**Approach 2: Dedicated SSS Function** (High-quality)
```hlsl
// Suitable for deep ocean / PC platforms / close-up views
half3 scatter = CalculateSSSColor(L, N, V) * surface_data.scatteringColor;
lightingresult.directDiffuse += light.color * light.shadowAttenuation * scatter;
```
- More accurate subsurface scattering calculation
- Better for deep water and close observation
- Higher quality with increased performance cost

### Integration in Lighting Model

The scattering is integrated into the custom shading model (`ShadingModels.hlsl`):
```hlsl
#if defined(_WATER)
    // Calculate SSS based on light, normal, and view directions
    half3 scatter = CalculateSSSColor(L, N, V) * surface_data.scatteringColor;
    
    // Add to direct diffuse lighting with shadow consideration
    lightingresult.directDiffuse += light.color * light.shadowAttenuation * scatter;
#endif
```

---

## 8. Foam
![Foam](/post-img/realistic-water-rendering/Foam.gif)

Generate foam at shoreline based on depth.
```hlsl
float foamFactor = saturate(exp2(-waterDepth * _FoamRange));

// Dual-layer foam blending
half4 foam1 = SAMPLE_TEXTURE2D(_FoamMap, sampler_FoamMap, 
              input.worldUV.xy * _FoamTiling1 + flowDir * speed * 2);
half4 foam2 = SAMPLE_TEXTURE2D(_FoamMap, sampler_FoamMap, 
              input.worldUV.zw * _FoamTiling2 + flowDir * speed * 2);
half4 foamBlend = lerp(foam1, foam2, 0.5);

albedo += foamBlend.r * _FoamIntensity * foamFactor;
```

---

## 9. Vertex Displacement (Wave)
![Water Wave](/post-img/realistic-water-rendering/Wave.png)

Dual-layer noise drives vertex offset to create waves.
```hlsl
// In vertex shader
float2 flowDir = _Time.x * float2(_FlowVertexDirX, _FlowVertexDirY);
float2 offsetUV1 = uv * half2(_HeightTilingX, _HeightTilingY) + flowDir;
float2 offsetUV2 = (uv + float2(0.4, 0.35)) * 1.6 + flowDir;

// Sample height map
half height1 = SAMPLE_TEXTURE2D_LOD(_NoiseMap, sampler_NoiseMap, offsetUV1, 0);
half height2 = SAMPLE_TEXTURE2D_LOD(_NoiseMap, sampler_NoiseMap, offsetUV2, 0);
half offset = (height1 + height2) * _HeightFactor;

input.positionOS.xyz += float3(0, offset, 0);
```

---

## Key Optimizations

### World Space Sampling
Avoids UV stretching on low-poly meshes.
```hlsl
output.worldUV.xy = vertexInput.positionWS.xz * 0.001 * _Tiling1;
```

### Manual Alpha Blending
Precise transparency control.
```hlsl
color.rgb = color * surfaceData.alpha + (1 - surfaceData.alpha) * underWaterColor;
```

### Double-Sided Rendering
Supports underwater observation.
```hlsl
normalWS *= vface > 0 ? 1 : -1;
```

---

## References
- Unity URP Documentation
- Water rendering techniques
