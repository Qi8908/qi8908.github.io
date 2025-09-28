---
title: Plant Rendering with Wind Shader
date: 2025-09-28 15:00:00 +0800
categories: [Graphics, Shader]
image: "/post-img/plant-rendering-with-wind-shader/Plant_compressed.gif"
tags: [Unity, Wind Shader, Vegetation]
published: true
---

## Intro

This project presents a comprehensive vegetation rendering shader built for Unity URP, 
featuring advanced wind simulation and realistic plant material representation. 

Key capabilities include:
- Multi-layered wind system with spatial turbulence and height-based intensity
- Dual-sided rendering with automatic normal correction
- Alpha clipping for complex plant silhouettes  
- PBR lighting with subsurface scattering simulation

```hlsl
float3 CalculateWind(float3 positionOS, float heightFactor, float time)
{
    float3 windDir = normalize(_WindDirection.xyz);
    float baseWind = sin(time * _WindFrequency) * _WindStrength;
    float turbulence = sin(time * _WindFrequency * 2.7 + positionOS.x) *
                      cos(time * _WindFrequency * 1.3 + positionOS.z) *
                      _WindTurbulence;
    float leafFlutter = sin(time * _WindFrequency * 4.0 + positionOS.y * 5.0) * _LeafFlutter;
    float windEffect = (baseWind + turbulence + leafFlutter) * heightFactor;
    return windDir * windEffect;
}
```

The system excels in open-world game vegetation systems, architectural visualization, and VR/AR natural environments, providing artist-friendly parameters while maintaining optimized performance through vertex-based wind calculations.

## Visual Result

![Plant Wind Animation](/post-img/plant-rendering-with-wind-shader/Plant_compressed.gif)

## Core Technical Features

### 1. Complex Wind System

#### Multi-Layered Wind Calculation
The wind system employs a composite algorithm based on multiple sine functions to achieve rich, layered animation effects:

- **Base Wind**: Primary swaying motion using `sin(time * frequency)`
- **Spatial Turbulence**: Position-based phase offsets prevent synchronized movement across all vegetation
- **High-Frequency Flutter**: Rapid leaf trembling effects for added realism

#### Height-Based Influence
Wind strength increases exponentially with vertex height, simulating real-world physics where higher parts of plants experience stronger wind effects:

```hlsl
float heightFactor = pow(input.positionOS.y, 3) * _HeightInfluence;
```

#### Vertex Color Control
![Plant Vertex Color](/post-img/plant-rendering-with-wind-shader/PlantVertexColor.png)
Artists can paint vertex colors on models to precisely control wind responsiveness in different areas, allowing for detailed animation control without additional geometry.

### 2. Dual-Sided Rendering

#### Cull Off Implementation
Uses dual-sided rendering to handle thin plant geometry like leaves and grass blades, ensuring visibility from all viewing angles.

#### Dynamic Normal Reversal
Automatically detects face orientation and adjusts normals accordingly:

```hlsl
normalWS *= vface > 0 ? 1.0 : -1.0;
```

This ensures proper lighting on both front and back faces of plant geometry.

### 3. Alpha Clipping System

#### Smart Pixel Culling
Implements alpha-based pixel clipping for complex plant silhouettes:

```hlsl
clip(surfaceInput.alpha - 0.33);
```

The 0.33 threshold provides optimal balance between edge detail preservation and transparent artifact elimination.

### 4. Physically-Based Rendering

#### Direct Lighting
Utilizes standard PBR lighting models for consistent material appearance across varying lighting conditions.

#### Subsurface Scattering Simulation
Mimics light transmission through leaves using scatter maps and color parameters:

```hlsl
surfaceInput.scaterringColor = scatterColor * _Scatter * _ScatterColor;
```

#### Indirect Lighting Support
Integrates spherical harmonics (SH) for accurate global illumination and environmental lighting.

The shader architecture balances visual fidelity with performance optimization, making it suitable for real-time applications while maintaining the natural complexity expected in modern vegetation rendering. Its modular design allows developers to adjust parameters based on specific project requirements, ensuring optimal visual-to-performance ratios across different hardware configurations.


