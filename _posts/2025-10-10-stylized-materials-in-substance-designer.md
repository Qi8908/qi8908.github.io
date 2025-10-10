---
title: "Substance Designer Case: Stylized Materials"
date: 2025-09-30 14:00:00 +0800
categories: [Materials]
image: "/post-img/stylized-materials-in-substance-designer/StylizedBricks.png"
tags: [SubstanceDesigner, Materials]
published: true
---

## Intro

Recently, I've been exploring stylized material creation in Substance Designer, focusing on building modular, parameter-driven materials that offer flexibility for different art styles.

A core principle in these materials is building each layer as an independent, reusable module. Each layer—whether moss, dirt, flowers, or base materials—is self-contained and can be dragged into new projects without reconnecting nodes. This approach creates a personal material library that dramatically speeds up future production.

Every layer features exposed parameters for:

PBR Properties: Independent control over roughness, metallic, and albedo
Scale: Adjustable tiling and size
Distribution: Customizable coverage, density, and placement

## Material 1: Overgrown Brick Wall

![Material 1: StylizedBricks](/post-img/stylized-materials-in-substance-designer/StylizedBricks.png)
![Material 1: StylizedBricksPBR](/post-img/stylized-materials-in-substance-designer/StylizedBricksPBR.jpg)

Layers:

Red brick base
Dirt overlay
Moss coverage
Clover patches
Flowering plants (petals + centers separated)

The flowers are built with petals and centers as separate components, allowing independent material properties for each. This means petals can be soft and matte while centers remain glossy and saturated—crucial for achieving the right stylized look.

## Material 2: Mossy Stone Ground

![Material 2: StylizedStones](/post-img/stylized-materials-in-substance-designer/StylizedStones.png)
![Material 2: StylizedStonesPBR](/post-img/stylized-materials-in-substance-designer/StylizedStonesPBR.jpg)

Layers:

Irregular stone base
Dirt accumulation
Moss growth (same reusable module from Material 1)

This material demonstrates the practical benefit of the modular approach—the moss layer is directly reused from the brick material, maintaining consistency while saving development time.

## Workflow Impact

Building reusable layers transforms material creation from repetitive node work into a mix-and-match workflow, enabling faster iteration and consistent visual quality across different assets.
