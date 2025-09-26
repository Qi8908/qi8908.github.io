---
title: Layered Material Implementation
date: 2025-09-26 15:00:00 +0800
categories: [Graphics, Shader]
image: "/post-img/procedural-layered-material/Cover.png"
tags: [Unity, Layered Material]
published: true
---

## Technical Implementation

### Material Blending Methods
Before implementing the layered wall material, I explored four primary blending techniques available in modern 3D software:
![Four Blend Methods](/post-img/procedural-layered-material/Blend.png){: width="100%"} <br />

1. Height Blend
Uses height maps to determine layer priority
Best for organic, natural blending effects

2. Vertex Blend
Manual control via vertex painting
Performance-friendly, suitable for real-time applications

3. Mask Blend
Uses grayscale masks to control blend weights
This is the method I chose for the wall material

4. Texture Array Blend
Blends multiple textures via array indexing
Memory efficient, suitable for terrain systems

## Visual Result
![Wall](/post-img/procedural-layered-material/layered-material.gif){: width="100%"} <br />

### Layer Structure
The material system consists of four main layers:

Base Layer: Basic stone wall surface
Moss Layer: Organic growth in crevices and damp areas
Snow Layer: Environmental accumulation on top surfaces, using world-space Y-axis gradient (upward) + noise to create natural buildup on horizontal surfaces
Crack Layer: Surface damage and fissures

This mask-driven approach provides precise control while maintaining natural-looking transitions between layers.
```


