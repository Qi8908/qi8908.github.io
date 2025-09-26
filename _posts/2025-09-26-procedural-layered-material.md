---
title: Procedural Layered Material
date: 2025-09-26 20:00:00 +0800
categories: [Graphics, Material]
image: /post-img/procedural-layered-material/Cover.jpg
tags: [Unity, Shader, Procedural, Layered Material]
published: true
---
## Visual Result

![Layered Material Animation](/post-img/procedural-layered-material/layered-material.gif)

The animation demonstrates the sequential layering process using mask-based blending in Unity. Each frame reveals how masks control the visibility and intensity of different material layers:

- **Base Wall**: Clean stone wall foundation
- **+ Moss**: Green organic growth concentrated in crevices and damp areas
- **+ Snow**: White accumulation on upward-facing surfaces
- **+ Cracks**: Surface damage and fissures

This mask-driven approach provides precise control while maintaining natural-looking transitions between layers.

## Technical Implementation

### Material Blending Methods

Before implementing the layered wall material, I explored four primary blending techniques available in modern 3D software:

![Blending Methods Comparison](/post-img/procedural-layered-material/Blend.png)
*Four blending approaches: Height Blend, Vertex Blend, Mask Blend, and Texture Array Blend*

**1. Height Blend**
- Uses height maps to determine layer priority
- Best for organic, natural blending effects

**2. Vertex Blend**
- Manual control via vertex painting
- Performance-friendly, suitable for real-time applications

**3. Mask Blend**
- Uses grayscale masks to control blend weights
- **This is the method I chose for the wall material**

**4. Texture Array Blend**
- Blends multiple textures via array indexing
- Memory efficient, suitable for terrain systems

### Layer Structure

The material system consists of four main layers:

1. **Base Layer**: Basic stone wall surface
2. **Moss Layer**: Organic growth in crevices and damp areas
3. **Snow Layer**: Environmental accumulation on top surfaces, using world-space Y-axis gradient (upward) + noise to create natural buildup on horizontal surfaces
4. **Crack Layer**: Surface damage and fissures

### My Implementation: Mask-Based Blending

I chose the **Mask Blend** approach for its flexibility and controllability.
