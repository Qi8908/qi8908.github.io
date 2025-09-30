---
title: "Magic Mirror Implementation: Depth Testing and Stencil Testing"
date: 2025-08-05 15:00:00 +0800
categories: [Graphics, Shader]
image: "/post-img/magic-mirror-depth-stencil-testing/magic-mirror.gif"
tags: [Unity, Stencil Buffer, Depth Test, Mirror Effect]
published: true
---

## Intro

A magic mirror effect using stencil buffer masking to create portal-like visibility. The mirror surface defines a region where an alternate scene becomes visible through precise render queue ordering.

## Visual Result

![Magic Mirror](/post-img/magic-mirror-depth-stencil-testing/magic-mirror.gif)

## Technical Implementation

### Render Queue Strategy

The magic mirror uses a four-layer rendering system with strict ordering:
Queue 2000 → Queue 2460 → Queue 2465 → Queue 2470
Front Objects  Mirror Mask  Mirror Skybox  Mirror Content

### How It Works

1. **Front objects** (mirror frame, front cards) render normally using standard shaders
2. **Mirror mask** writes stencil value 1 without depth, creating an invisible portal region
3. **Mirror skybox** fills the background, always rendering behind mirror content
4. **Mirror world** renders only where stencil equals 1, with normal depth testing

The mask's `ZWrite Off` is critical - it marks the region without blocking geometry behind it.

### Key Detail: Dual Material System for Cards

Cards in the scene use two separate material systems:

**Front Cards (in front of mirror):**
- Shader: Unlit
- Render Queue: 2000
- No stencil test, always visible

**Mirror Cards (behind mirror):**
- Shader: Same as mirror scene
- Stencil Settings: Ref=1, Comp=Equal
- Render Queue: 2470
- Only visible within mirror region

This design separates the real world from the mirror world, allowing the same type of objects to exist independently in both spaces.

## Setup Checklist

**Mask Material:**
- Stencil: Ref=1, Comp=Always, Pass=Replace
- ZWrite: Off
- Queue: 2460

**Mirror Content Material (scene and cards):**
- Stencil: Ref=1, Comp=Equal
- ZWrite/ZTest: Default
- Queue: 2470

**Mirror Skybox Material:**
- Stencil: Ref=1, Comp=Equal
- ZTest: Always
- Queue: 2465

**Front Objects Material:**
- Standard shader (e.g., Unlit)
- No stencil test
- Queue: 2000

## Summary

A three-layer system: mask marks the region (stencil), skybox provides depth (ZTest Always), content respects both (stencil + depth). Queue ordering ensures proper buffer states at each stage. The dual material system achieves complete separation between the real world and mirror world.




