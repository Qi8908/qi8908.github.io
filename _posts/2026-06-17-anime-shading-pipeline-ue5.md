---
layout: post
title: "Making 3D Look 2D: An Anime Shading Pipeline in UE5"
date: 2026-06-17
categories: [Unreal, Shading]
image: "post-img/anime-shading-pipeline-ue5/Cover2.png"
tags: [cartoon-shading, material-graph, sdf]
published: true
---

## Overview

This post walks through the toon shading pipeline I built in UE5 for an anime-style character render. The goal was to take a fully 3D character model and make it read as a flat, 2D illustration — which meant working against several things the engine does by default: physically based lighting, perspective distortion, and highlight behavior that assumes a "normal" surface.

![Final Render](post-img/anime-shading-pipeline-ue5/Overview.gif)

Below are the five parts of the pipeline that involved the most problem-solving. A full project write-up with the complete material graph and a demo video is linked at the end.

---

##  Lighting System: Custom Lambert Lighting

UE built-in lighting is PBR-based, with parameters that are fixed and hard to adjust — not well suited for the hard-edged shadows that cartoon rendering usually wants. So instead of using the engine's native lighting, this project implements a custom Lambert lighting setup using material nodes.

The approach is to take the dot product of the vertex normal (VertexNormalWS) and the sky light direction (SkyAtmosphereLightDirection), which gives a value representing light intensity. This value is continuous, so used directly it produces soft shadow edges — not what cartoon rendering is going for. To fix that, the value is fed into a custom curve (CurveLinearColor), and the shape of the curve controls how hard or soft the shadow edge is: a steeper slope gives a sharper light/shadow boundary, a gentler slope gives a softer transition. Two additional parameters, ShadowSmooth and ShadowOffset, control shadow sharpness and shadow offset respectively.

![Lambert Lighting](post-img/anime-shading-pipeline-ue5/Lambert.png)

The advantage of this setup is that shadow behavior is entirely controlled by the curve — adjusting the curve shape changes the shadow distribution without touching the underlying material logic.

---

## Hair Highlights: Using View Direction Instead of Normal

Calculating highlights on a normal surface relies on the surface normal, but hair is made up of a large number of thin strands and doesn't have a stable surface normal. Computing highlights the usual way produces unnatural results, and the highlight position wouldn't shift with the camera angle.

The approach here is to substitute the view direction V (the vector from the model toward the camera) for the normal N, and manually build a tangent space coordinate system from there (the N axis is replaced by V, then crossed to get the tangent T and bitangent B — the cross product order matters, since getting it wrong flips the direction of the resulting coordinate system). Once this coordinate system is computed, the result is remapped to the [0, 1] range and used as the UV coordinate to sample a MatCap texture.

![Matcap](post-img/anime-shading-pipeline-ue5/Matcap.png)
![Matcap](post-img/anime-shading-pipeline-ue5/HairPanel.png)

This makes the highlight position shift with the camera angle, producing the kind of viewer-dependent highlight sliding seen in typical anime hair rendering, rather than a fixed highlight that doesn't react to viewing angle.

![Hair Rendering](post-img/anime-shading-pipeline-ue5/HairHighlights.gif)

---

## WPO Perspective Correction: Reducing the Near-Far Size Difference

A 3D model is subject to standard perspective projection, which produces a near-large/far-small effect. Anime-style illustration typically doesn't show this much perspective distortion on a character, so rendering the 3D model directly makes limbs look slightly off in proportion when they're close to the camera — the result doesn't feel "2D" enough.

The fix is to apply a vertex offset in the material using WPO (World Position Offset). First, the depth distance of each vertex relative to the camera is calculated (vertex position dotted with camera direction). A larger depth value means the vertex is farther from the camera and should be compressed more by perspective. A parameter called FOV_Fix (default value 1, meaning no correction) controls how much this depth value affects the result — vertices are pulled back toward the model's center, with vertices farther from the center pulled back more.

![FOV_Fix](post-img/anime-shading-pipeline-ue5/WPO1.png)
![FOV_Fix](post-img/anime-shading-pipeline-ue5/WPO2.png)

Tuning FOV_Fix takes some back and forth: too weak and there's no visible effect, too strong and the model starts to deform. It needs to be adjusted while watching the result as the camera moves.

![WPO](post-img/anime-shading-pipeline-ue5/WPO.gif)

---

## Face Shadow Fix: Switching from Vertex Lighting to an SDF Texture

The face initially used the same Lambert lighting + curve setup described above, but testing revealed two problems: the shadow position was often wrong when the light angle changed or the face rotated, and the shadow edges looked broken up rather than forming one coherent shape. The cause is that the face has a lot of complex curvature, and per-vertex normal dot lighting alone can't produce a coherent shadow shape on a surface this detailed.

The fix was to switch to a signed distance field (SDF) shadow texture — a pre-baked texture that defines how shadows should be distributed across different regions of the face. This keeps the shadow shape fixed and prevents it from breaking apart due to small variations in vertex normals. 

![Face Panel](post-img/anime-shading-pipeline-ue5/Face_SDF.png)

A custom HLSL snippet (in a custom node) was also added to do a 3x3 box blur, averaging the 9 surrounding pixels at each sample point to reduce jagged edges.

```hlsl
float shadow = 0;
float texelsize = 1./BlurIntensity*Dither;

for (int x = -1; x <= 1; ++x)
{
    for (int y = -1; y <= 1; ++y)
    {
        float TexR = Texture2DSample(TexObject, TexObjectSampler,
            TexCoord + float2(x,y)*texelsize).r;
        shadow += TexR;
    }
}
shadow /= 9.;
return shadow;
```

One issue that came up along the way: after switching to SDF, there was a patch of unwanted shadow appearing near the mouth. At first this looked like a blur or curve parameter issue, and a lot of time was spent adjusting those without success. It turned out the UV channel referenced by the SDF node's TexCoord didn't match the UV channel actually used when the SDF texture was baked for the model. To track this down, the skeletal mesh was temporarily converted to a static mesh, opened in the static mesh editor, and each UV channel was checked visually to find the one used for the face's SDF bake (UV channel 3). Once that index was set correctly in the material's TexCoord node, the problem went away.

The last piece was making the shadow respond to light direction in real time: a separate actor blueprint tracks the light direction, an arrow component attached to the head bone marks the face's forward direction, and the angle between the two is calculated and passed to the material through a material parameter collection to drive the SDF shadow sampling position. 

![BP SDF](post-img/anime-shading-pipeline-ue5/BP_SDF.png)

Along the way, the shadow direction turned out to be mirrored (fixed by removing an extra 1-x node), and the timing of the shadow shift didn't match the timing of the light rotation (fixed by adding a remapping curve, CB_SDF_Mapped) — both were adjusted after comparing results visually.

![Face Shadow](post-img/anime-shading-pipeline-ue5/FaceShadow.gif)

---

## Dynamic Focus Perspective: Camera Push-In on Action

The goal here was a specific effect: when the character performs an action (like a punch), the frame should briefly draw visual focus toward a key point (the fist), creating a sense of impact — a common technique in anime cuts.

The implementation has two parts. First, setting up the spatial relationship: the camera direction vector is crossed with the world up vector to get the horizontal axis of the camera plane, and that axis is crossed with the camera direction again to get the vertical axis (cross product order matters here too — getting it wrong flips the axis direction). 

![Build Camera-Plane Projection](post-img/anime-shading-pipeline-ue5/DynamicFocus1.png)

Each vertex's vector to the camera is then dotted with these two axes, giving the vertex's projected coordinates on the camera plane. Second, setting up the falloff: the character's chest bone is chosen as the focus point, and the distance from each vertex to that point is calculated — the farther a vertex is from the focus, the less it should be displaced. This distance is mapped through a curve to get a falloff coefficient, which then scales the amount of vertex displacement.

![Set Up Falloff Coefficient](post-img/anime-shading-pipeline-ue5/DynamicFocus2.png)

Finally, the focus bone's world position (the fist or chest) is passed to the material parameter collection every frame through the character blueprint's Tick event, so the focus point updates in real time as the animation plays, instead of staying fixed.

![Dynamic Focus Perspective](post-img/anime-shading-pipeline-ue5/Punch.png)
