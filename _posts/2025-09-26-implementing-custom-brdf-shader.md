---
title: Custom PBR Shader Implementation vs Unity URP
date: 2025-09-26 15:00:00 +0800
categories: [Graphics, Shader]
tags: [Unity, PBR, BRDF, Shader, URP]
---

## Background

During my graphics programming journey, I implemented a custom Physically Based Rendering (PBR) shader from scratch and compared it with Unity's built-in URP Lit Shader.

## Technical Implementation

### BRDF Model
- Implemented Cook-Torrance BRDF model
- Includes Fresnel term, Normal Distribution Function (NDF), and Geometry term
- Supports metallic-roughness workflow

### Key Code Snippets
```hlsl
// Core BRDF implementation
float3 F_Schlick(float3 F0, float cosTheta) {
    return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
}

float D_GGX(float NoH, float roughness) {
    float a = roughness * roughness;
    float a2 = a * a;
    float denom = (NoH * NoH) * (a2 - 1.0) + 1.0;
    return a2 / (PI * denom * denom);
}
