---
name: shader-graph-best-practices
description: Node budgets, custom lighting, sub-graphs, precision control, and mobile workflows for Unity Shader Graph
---

# Shader Graph Best Practices for Mobile Unity

## TRIGGER
Use this skill when the user is working with Unity Shader Graph (the node-based visual shader editor) for URP or HDRP. Triggers include: "Shader Graph", "node-based shader", "sub-graph", "custom function node", "Shader Graph mobile", "Shader Graph optimization", "custom lighting Shader Graph", "Shader Graph precision", or any visual shader authoring workflow in Unity. Also trigger when the user references Ben Cloward, Cyanilux, or Daniel Ilett Shader Graph tutorials.

---

## CORE RULES

1. **Keep node count under 100 for mobile targets.** Each node compiles to shader instructions. Above ~100 nodes, mobile GPUs hit instruction cache limits and performance drops sharply. Use Sub Graphs to organize, but they don't reduce instruction count.

2. **Set graph-level precision to Half** for mobile. In the Graph Inspector, change Precision to "Half". Override individual nodes to "Float" only for positions and UVs.

3. **Use Sub Graphs to organize, not to optimize.** Sub Graphs improve readability but compile inline — they don't reduce instruction count. However, they enable reuse across graphs, which reduces total project maintenance.

4. **Custom Function nodes are the escape hatch.** When Shader Graph can't express something efficiently (or uses too many nodes for a simple operation), write a Custom Function node in HLSL. This is especially useful for lighting access, complex math, and platform-specific branching.

5. **Shader Graph generates all required passes automatically** (ForwardLit, ShadowCaster, DepthOnly, DepthNormals, Meta). You don't need to worry about multi-pass like in HLSL — but you also can't optimize individual passes.

---

## CUSTOM LIGHTING IN SHADER GRAPH

Shader Graph's default Lit target uses PBR (Physically Based Rendering). For mobile, you often want simpler lighting. Use Cyanilux's custom lighting sub-graphs:

### Install Custom Lighting Nodes
The **[Cyanilux/URP_ShaderGraphCustomLighting](https://github.com/Cyanilux/URP_ShaderGraphCustomLighting)** package adds these sub-graph nodes to Shader Graph:

- **MainLight** — direction, color, shadow attenuation
- **AdditionalLights** — loop over additional lights (use sparingly on mobile)
- **AmbientSH** — spherical harmonics ambient
- **MainLightShadows** — shadow sampling
- **Cel/Toon Shading** — step-based lighting

### Usage Pattern
1. Set the graph target to **Unlit** (not Lit) — this disables Unity's PBR lighting
2. Add Cyanilux's MainLight sub-graph to get light direction + color
3. Compute your own diffuse: `saturate(dot(Normal, LightDirection)) * LightColor`
4. Add ambient from the AmbientSH sub-graph
5. Multiply by your albedo texture
6. Output to the Unlit graph's Color output

This gives you full control over lighting complexity — critical for mobile where PBR's multi-texture GGX/Smith BRDF is often too expensive.

---

## NODE EFFICIENCY TIPS

### Combine operations
```
BAD:  Multiply → Add → Multiply → Saturate  (4 nodes)
GOOD: Custom Function node with: saturate(a * b + c) * d  (1 node)
```

### Use Shader Graph Variables (Cyanilux)
The **[ShaderGraphVariables](https://github.com/Cyanilux/ShaderGraphVariables)** package adds Register Variable and Get Variable nodes. This prevents recalculating the same value multiple times in complex graphs — the equivalent of a local variable in code.

### Texture sampling efficiency
- Use a single Sample Texture 2D node and split channels, rather than sampling the same texture multiple times
- For normal maps, use the Sample Texture 2D node with Type set to Normal — it handles unpacking automatically
- Channel-pack masks into a single texture's RGBA channels (see `texture-packing-variant-stripping` skill)

### Avoid expensive nodes on mobile
| Node | Cost | Alternative |
|------|------|-------------|
| Fresnel Effect | Medium | Manual: `1 - saturate(dot(ViewDir, Normal))` then Power |
| Voronoi | **High** | Pre-bake to texture |
| Gradient Noise | **High** | Use Simple Noise or pre-bake to texture |
| Triplanar | **High** (3 texture samples) | Use on specific meshes only, not terrain |
| Normal From Height | **High** | Use actual normal maps instead |
| Custom Render Texture | **High** | Pre-bake when possible |

---

## FULL SCREEN SHADER GRAPH (UNITY 6+)

Unity 6 introduced Full Screen Shader Graph for post-processing effects. Use with URP's Full Screen Pass Renderer Feature:

1. Create a new Shader Graph with Full Screen target
2. The graph receives URP's camera color and depth as inputs
3. Apply your post-processing effect (vignette, color correction, etc.)
4. Add a Full Screen Pass Renderer Feature to your URP Renderer
5. Assign the Full Screen Shader Graph material

**Mobile consideration**: Each Full Screen pass is a full-screen blit. Combine effects into a single Full Screen graph whenever possible.

---

## SHADER GRAPH + MOBILE CHECKLIST

- [ ] Graph-level precision set to Half
- [ ] Position and UV nodes overridden to Float where needed
- [ ] Total node count < 100
- [ ] No Voronoi or Gradient Noise nodes (pre-bake instead)
- [ ] Using Unlit target + custom lighting instead of Lit target (if lighting is simple)
- [ ] Texture samples ≤ 4
- [ ] No Triplanar on large surfaces
- [ ] Sub Graphs used for organization and reuse
- [ ] Keywords use shader_feature (not multi_compile) for toggleable features
- [ ] Tested in Shader Graph preview at target resolution

---

## REFERENCE RESOURCES

### Repos
- Cyanilux: Custom Lighting Sub-Graphs — https://github.com/Cyanilux/URP_ShaderGraphCustomLighting
- Cyanilux: Shader Graph Variables — https://github.com/Cyanilux/ShaderGraphVariables
- Unity: Shader Graph Example Library — https://github.com/UnityTechnologies/ShaderGraph_ExampleLibrary
- Unity: Production Ready Shader Graph Samples — https://docs.unity3d.com/Packages/com.unity.shadergraph@17.4/manual/Shader-Graph-Sample-Production-Ready-Detail.html

### Tutorials
- Cyanilux: Intro to Shader Graph (March 2025) — https://www.cyanilux.com/tutorials/intro-to-shader-graph/
- Daniel Ilett: 10+ Part Shader Graph Basics — https://danielilett.com/
- Ben Cloward: Weekly Shader Graph Tutorials — YouTube + https://www.bencloward.com
- MinionsArt: ~200 Shader Graph Tutorials — https://minionsart.github.io/tutorials/

### Key People
- **Ben Cloward** — Former Unity Shader Graph team, now at ILM. Weekly tutorials on both Unreal and Unity Shader Graph.
- **Cyanilux** — Most actively maintained URP Shader Graph repos. Custom lighting, code templates, variables.
- **Daniel Ilett** — Shader Graph Basics series, author of *Building Quality Shaders for Unity*.
- **MinionsArt (Joyce)** — "Shaderwitch", GIF-based quick shader tips, 200 tutorials.
- **Gabriel Aguiar** — VFX Graph + Shader Graph tutorials (portals, water, tornado).

## SOURCES

- Cyanilux: Shader Graph Tutorials — https://www.cyanilux.com/
- Daniel Ilett: Shader Graph Basics — https://danielilett.com/
- Shader Graph: Precision Modes — https://docs.unity3d.com/Packages/com.unity.shadergraph@6.9/manual/Precision-Modes.html
- Quick Unity Tips: Under 100 Nodes for Mobile — https://quickunitytips.blogspot.com/2025/11/unity-shader-optimization-guide-2025.html
- 80.lv: URP Render Graph in Unity 6 — https://80.lv/articles/introduction-to-urp-s-render-graph-in-unity-6
