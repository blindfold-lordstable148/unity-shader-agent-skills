---
name: mobile-shader-optimization
description: GPU architecture, precision types, fillrate, overdraw, baked lighting, and LOD optimization for Unity mobile/WebGL shaders
---

# Mobile Shader Optimization for Unity

## TRIGGER
Use this skill when the user is writing, reviewing, or optimizing Unity shaders for mobile platforms (Android, iOS) or any GPU-constrained environment (Meta Quest VR, low-end devices). Triggers include: mentions of "mobile", "Android", "iOS", "Mali", "Adreno", "PowerVR", "performance", "optimization", "fillrate", "overdraw", "bandwidth", "frame budget", "half precision", "LOD", "baked lighting", or any shader work targeting sub-60fps scenarios. Also trigger when the user is profiling shaders, asking about GPU architecture, or comparing Simple Lit vs Complex Lit in URP.

---

## CORE RULES — HARD CONSTRAINTS

These rules must always be followed when writing mobile shaders:

1. **Use `half` precision by default.** Only use `float` for world-space positions, UV coordinates requiring high precision, and depth calculations. On Mali and Adreno GPUs, `half` triggers 16-bit fast paths that are 2x faster.

2. **Never use `discard` (clip/alpha-test) on PowerVR or Apple GPUs.** It breaks Early-Z/Hidden Surface Removal, forcing the GPU to process fragments it would otherwise skip. Use alpha blending instead, or move discard to a separate pass.

3. **Keep Shader Graph node count under 100 for mobile.** Each node compiles to shader instructions. Exceeding ~100 nodes risks exceeding mobile GPU instruction limits and causes performance cliffs.

4. **Never use `while` or `do-while` loops in shaders targeting OpenGL ES 2.0 / WebGL 1.0.** Only counting `for` loops are allowed. Even on ES 3.0+, avoid dynamic loop bounds — they prevent the compiler from unrolling.

5. **Avoid dynamic branching on mobile.** Mobile GPUs often execute both branches regardless. Use `step()`, `lerp()`, `saturate()` for branchless alternatives.

6. **Multiply scalars before vectors.** `float3 result = scalar1 * scalar2 * vector` is faster than `float3 result = vector * scalar1 * scalar2` because the first scalar multiply is 1 ALU op instead of 3.

7. **Limit texture samples.** Each sample costs bandwidth. Target ≤4 texture samples per fragment on low-end mobile. Use channel packing to combine maps (see `texture-packing-variant-stripping` skill).

8. **Use `noforwardadd` surface shader directive** to skip additional per-pixel light passes. Each additional light pass doubles overdraw.

---

## MOBILE GPU ARCHITECTURE

Understanding tile-based rendering is essential for mobile shader optimization.

### Tile-Based Deferred Rendering (TBDR)
Mobile GPUs split the screen into small tiles (16×16 for Mali, variable for Adreno) and process each tile entirely on-chip before writing to main memory. This means:

- **Bandwidth is the primary bottleneck**, not compute. Every texture read, framebuffer write, and render target switch costs main memory bandwidth.
- **Overdraw is extremely expensive** because each overlapping fragment consumes on-chip tile memory and potentially triggers main memory writes.
- **MSAA is nearly free** on tiled GPUs (resolved on-chip), unlike desktop where it multiplies framebuffer bandwidth.
- **Render target switches are costly** — each switch flushes tile memory to main memory. Minimize post-processing passes.
- **Power budget is 3–6W** (vs 300W desktop). Thermal throttling will reduce clock speeds within minutes if shaders are too complex.

### Per-GPU Family Notes
- **Mali (ARM)**: 16×16 tiles. Use Mali Offline Compiler to check cycle counts. Forward rendering preferred over deferred.
- **Adreno (Qualcomm)**: Flexible tile size (~256K). Supports LRZ (Low-Resolution Z) for early rejection. Slightly more tolerant of complex fragment shaders.
- **PowerVR (Apple/Imagination)**: True TBDR with HSR (Hidden Surface Removal). Alpha testing/discard breaks HSR. Apple GPUs have dedicated `half` hardware.
- **Apple A-series**: Tightly integrated with Metal. `half` is genuinely 16-bit and 2x throughput. `float` is 32-bit. No `fixed` type.

---

## PRECISION TYPES

```hlsl
// HLSL/ShaderLab precision mapping to GLSL:
// float  → highp   (32-bit) — positions, UVs, depth
// half   → mediump (16-bit) — colors, normals, most calculations
// fixed  → lowp    (11-bit) — simple color ops only (deprecated on many GPUs)

// USE half FOR:
half4 color = tex2D(_MainTex, uv);         // Texture samples
half3 normal = normalize(i.worldNormal);    // Normals
half NdotL = saturate(dot(normal, lightDir)); // Lighting
half fresnel = pow(1.0h - NdotL, 4.0h);    // Fresnel

// USE float FOR:
float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz; // World position
float2 uv = TRANSFORM_TEX(v.texcoord, _MainTex);         // UV transforms
float depth = Linear01Depth(rawDepth);                     // Depth
```

---

## VERTEX VS FRAGMENT OPTIMIZATION

On fillrate-bound mobile GPUs, vertex count is orders of magnitude lower than fragment count. Move calculations to the vertex shader whenever visually acceptable:

```hlsl
// BAD — per-pixel calculation (runs millions of times)
half4 frag(v2f i) : SV_Target {
    half rim = 1.0h - saturate(dot(normalize(i.viewDir), normalize(i.worldNormal)));
    half rimPower = pow(rim, _RimPower);
    return _RimColor * rimPower;
}

// GOOD — per-vertex calculation (runs thousands of times)
v2f vert(appdata v) {
    v2f o;
    o.vertex = TransformObjectToHClip(v.vertex.xyz);
    float3 viewDir = normalize(GetWorldSpaceViewDir(TransformObjectToWorld(v.vertex.xyz)));
    float3 worldNormal = TransformObjectToWorldNormal(v.normal);
    o.rimFactor = pow(1.0 - saturate(dot(viewDir, worldNormal)), _RimPower);
    return o;
}

half4 frag(v2f i) : SV_Target {
    return _RimColor * i.rimFactor; // Just interpolated value
}
```

### What to move to vertex shader:
- Rim/fresnel calculations
- View direction normalization
- Fog factor computation
- Simple UV animation offsets
- Light direction dot products (for flat/low-poly styles)

### What must stay in fragment shader:
- Texture sampling
- Normal map lookups
- Precise specular highlights
- Screen-space effects

---

## SHADER LOD AND FALLBACK STRATEGY

Use LOD tiers to serve different shader complexity per device capability:

```hlsl
Shader "Game/Character" {
    // LOD 300: Full quality (desktop, high-end mobile)
    SubShader {
        LOD 300
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass {
            // Normal map + specular + rim + emission
        }
    }
    
    // LOD 150: Medium quality (mid-range mobile)
    SubShader {
        LOD 150
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass {
            // Diffuse + simple lighting, no normal map
        }
    }
    
    // LOD 100: Minimum quality (low-end mobile)
    SubShader {
        LOD 100
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }
        Pass {
            // Vertex-lit only, single texture
        }
    }
    
    Fallback "Universal Render Pipeline/Simple Lit"
}
```

Set at runtime based on device:
```csharp
// In a device detection script:
if (SystemInfo.graphicsMemorySize < 1024)
    Shader.globalMaximumLOD = 100;
else if (SystemInfo.graphicsMemorySize < 2048)
    Shader.globalMaximumLOD = 150;
else
    Shader.globalMaximumLOD = 300;
```

Built-in LOD values: Mobile/VertexLit = 100, Mobile/Diffuse = 150, Mobile/Bumped = 200.

---

## BAKED LIGHTING FOR MOBILE

Fully baked lighting eliminates real-time shadow maps and reduces fragment shaders to a single lightmap fetch. This is often the difference between 30 and 60 FPS on low-end devices.

### Rules:
- **Use baked lighting for all static geometry on mobile.** The lightmap sample is far cheaper than real-time shadow mapping.
- **Limit real-time lights to 1 directional light** (the main light). Additional lights add per-pixel passes.
- **Directional lights are cheapest**, point lights most expensive (require 6-direction shadow maps if real-time).
- **Use Light Probes for dynamic objects** instead of real-time lights.
- **Deferred rendering is poor on mobile** — it requires multiple render targets, which means multiple tile memory flushes. Always use Forward rendering.

---

## URP ASSET SETTINGS FOR MOBILE

Configure the URP Asset for mobile:

- **Rendering**: Forward rendering, no Depth Priming (adds a prepass), MSAA 2x or 4x (cheap on tiled GPUs)
- **Quality**: Disable HDR unless needed for Bloom (HDR doubles framebuffer bandwidth). Disable Additional Lights if only using main directional.
- **Shadows**: Reduce shadow resolution (1024 or 512 for mobile). Reduce shadow distance. Use 1 shadow cascade for mobile (2 max).
- **Post-Processing**: Disable if not needed. If needed, use a single-pass approach (see `mobile-post-processing` skill).

---

## OPTIMIZATION CHECKLIST

When reviewing a mobile shader, verify:

- [ ] All color/normal calculations use `half` precision
- [ ] Only world positions and UVs use `float`
- [ ] No `discard`/clip on iOS/PowerVR targets (or isolated in separate pass)
- [ ] No dynamic branching (replaced with step/lerp/saturate)
- [ ] Texture samples ≤ 4 per fragment
- [ ] No `while`/`do-while` loops
- [ ] Scalar multiplications happen before vector multiplications
- [ ] View-independent calculations moved to vertex shader
- [ ] SRP Batcher compatible (properties in CBUFFER)
- [ ] LOD SubShaders defined for at least 2 tiers
- [ ] `noforwardadd` used on surface shaders (or additional lights disabled in URP)
- [ ] No Standard Shader — use Simple Lit or custom Unlit

---

## PROFILING TOOLS

- **Unity Frame Debugger** (Window > Analysis > Frame Debugger) — inspect draw calls, shader passes, overdraw
- **Mali Offline Compiler** — check instruction cycle counts for ARM Mali GPUs
- **Snapdragon Profiler** — profile Adreno GPU counters
- **Arm Performance Studio (Streamline)** — detailed Mali GPU performance analysis
- **RenderDoc** — frame capture and per-draw analysis
- **Xcode GPU Profiler** — Apple GPU profiling with shader cost heatmaps

---

## SOURCES

- Unity Manual: Shader Performance — https://docs.unity3d.com/Manual/SL-ShaderPerformance.html
- Unity Manual: Mobile Optimization — https://docs.unity3d.com/2019.4/Documentation/Manual/MobileOptimisation.html
- Unity Manual: Data Types and Precision — https://docs.unity3d.com/2019.4/Documentation/Manual/SL-DataTypesAndPrecision.html
- Unity: Shader Optimization Tips — https://create.unity.com/shader-optimization
- Unity: Mobile Art Optimization Part 2 — https://unity.com/how-to/mobile-game-optimization-tips-part-2
- Meta Developer: GPU Tiled Rendering — https://developers.meta.com/horizon/documentation/unity/gpu-tiled/
- ARM Community: Post-Processing on Mobile — https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/post-processing-effects-on-mobile-optimization-and-alternatives
- Quick Unity Tips: Shader Optimization 2025 — https://quickunitytips.blogspot.com/2025/11/unity-shader-optimization-guide-2025.html
- Toxigon: Mobile Shader Optimization — https://toxigon.com/optimizing-unity-shaders-for-mobile-performance
- UWA Blog: GPU Pressure — https://blog.en.uwa4d.com/2022/12/28/unity-mobile-game-performance-optimization-the-balance-between-graphics-performance-and-gpu-pressure/
- Unity Forum: Unity 6 Mobile Tips — https://discussions.unity.com/t/unity-6-mobile-graphics-optimization-tips-and-tricks/1593293
- Android Developer: URP Asset Settings — https://developer.android.com/develop/xr/unity/performance/urp-asset-settings
- TheGamedev.Guru: Overdraw — https://thegamedev.guru/unity-gpu-performance/overdraw-optimization/
- ColinLeung-NiloCat: URP Toon Shader — https://github.com/ColinLeung-NiloCat/UnityURPToonLitShaderExample
- awesome-mobile-graphics — https://github.com/shihchinw/awesome-mobile-graphics
