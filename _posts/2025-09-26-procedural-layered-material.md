---
title: Layered Material Implementation
date: 2025-09-26 15:00:00 +0800
categories: [Graphics, Shader]
image: "/post-img/procedural-layered-material/layered-material.gif"
tags: [Unity, Layered Material]
published: true
---

## Intro

Before implementing the layered wall material, I explored four primary blending techniques available in Unity:

![Four Blend Methods](/post-img/procedural-layered-material/Blend.png)

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

## Visual Result
![Wall](/post-img/procedural-layered-material/layered-material.gif){: width="100%"} <br />

### Layer Structure

The material system consists of four main layers:

1. **Base Layer**: Basic stone wall surface
2. **Moss Layer**: Organic growth in crevices and damp areas
3. **Snow Layer**: Environmental accumulation on top surfaces, using world-space Y-axis gradient (upward) + noise to create natural buildup on horizontal surfaces
4. **Crack Layer**: Surface damage and fissures

This mask-driven approach provides precise control while maintaining natural-looking transitions between layers. The layered material workflow offers several key advantages: non-destructive editing allows easy adjustments without starting over, procedural generation ensures variation across instances, and the modular system can be reused for different surface types beyond walls.


