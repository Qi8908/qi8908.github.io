---
title: procedural layered material
date: 2025-09-26 15:00:00 +0800
categories: [Graphics, Shader]
image: "/post-img/implementing-custom-brdf-shader/Cover.png"
tags: [Unity, PBR, BRDF, Shader, URP]
published: true
---

## Visual Comparison
![Croissant](/post-img/implementing-custom-brdf-shader/BRDF1.png){: width="100%"} <br />
![Metal Plate](/post-img/implementing-custom-brdf-shader/BRDF2.png){: width="100%"} <br />

## Technical Implementation

### Core BRDF Model

My custom shader implements the Cook-Torrance BRDF model based on the microfacet theory, following the physically-based rendering principles outlined in the LearnOpenGL PBR tutorial. The implementation consists of three main components:

**Cook-Torrance Specular BRDF:**
```
f_specular = (D * F * G) / (4 * (N·V) * (N·L))
```

Where:
- **D** = Normal Distribution Function (GGX/Trowbridge-Reitz)
- **F** = Fresnel equation (Schlick approximation)
- **G** = Geometry function (Smith's method with Schlick-GGX)

### Implementation Details

#### 1. Normal Distribution Function (D)
The GGX distribution determines how microfacet normals are distributed across the surface, controlling specular highlight shape and size:

```hlsl
half D_GGX_TR(float3 N, float3 H, half roughness)
{
    half a = roughness * roughness;
    half a2 = a * a;
    half NdotH = saturate(dot(N, H));
    half NdotH2 = NdotH * NdotH;
    
    half nom = a2;
    half denom = (NdotH2 * (a2 - 1.0) + 1.0);
    denom = PI * denom * denom;
    
    return nom / max(denom, 0.0000001);
}
```

#### 2. Geometry Function (G)
Smith's method with Schlick-GGX approximation accounts for microfacet self-shadowing and occlusion:

```hlsl
half GeometrySmith(float3 N, float3 V, float3 L, half roughness)
{
    half NdotV = saturate(dot(N, V));
    half NdotL = saturate(dot(N, L));
    half k = (roughness + 1.0) * (roughness + 1.0) / 8.0; // Direct lighting
    
    half ggx1 = NdotV / (NdotV * (1.0 - k) + k);
    half ggx2 = NdotL / (NdotL * (1.0 - k) + k);
    
    return ggx1 * ggx2;
}
```

#### 3. Fresnel Equation (F)
Schlick's approximation efficiently calculates surface reflectance at different viewing angles:

```hlsl
half3 fresnelSchlick(half cosTheta, half3 F0)
{
    return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
}
```

The base reflectivity F0 is computed based on the metallic workflow:
```hlsl
half3 F0 = lerp(0.04, albedo, metallic);
```
- Non-metals: F0 ≈ 0.04 (dielectric materials)
- Metals: F0 = albedo (colored reflectance)

### Energy Conservation

The shader implements energy conservation by ensuring the sum of reflected and refracted light never exceeds incident light:

```hlsl
half3 kS = F; // Specular reflection ratio (from Fresnel)
half3 kD = (1.0 - kS) * (1.0 - metallic); // Diffuse ratio
```

Note that metallic surfaces have `kD = 0` since metals absorb all refracted light.

### Lighting Equation

The complete lighting equation integrates both direct and indirect illumination:

```hlsl
// Direct Lighting
half3 directDiffuse = diffuse / PI; // Lambertian diffuse
half3 directSpecular = (D * F * G) / (4 * NoV * NoL);
half3 DirectLight = (directDiffuse + directSpecular) * radiance;

// Indirect Lighting (IBL)
half3 indirectDiffuse = SampleSH(N) * diffuse; // Spherical harmonics
half3 indirectSpecular = GlossyEnvironmentReflection(R, roughness, 1) * 
                         EnvBRDFApprox(F0, roughness, NoV);

// Final Result
half3 result = DirectLight + (indirectDiffuse + indirectSpecular) * AO;
```

### Material Workflow

The shader uses the metallic-roughness workflow with PBR texture maps:

- **Albedo Map**: Base color (diffuse for non-metals, F0 tint for metals)
- **ORM Map**: Packed texture containing:
  - R channel: Ambient Occlusion
  - G channel: Roughness (controls microfacet distribution)
  - B channel: Metallic (0 = dielectric, 1 = conductor)
- **Normal Map**: Tangent-space surface details




