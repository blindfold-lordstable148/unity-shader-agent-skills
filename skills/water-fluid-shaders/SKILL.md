---
name: water-fluid-shaders
description: Mobile-optimized water shaders with depth coloring, foam, Gerstner waves, refraction, and caustics for Unity URP
---

# Water and Fluid Shaders for Mobile Unity

## TRIGGER
Use this skill when the user wants to create, modify, or optimize water, ocean, river, lake, pond, fluid, lava, or liquid shaders in Unity — especially for mobile or WebGL. Triggers include: "water shader", "ocean", "waves", "foam", "shoreline", "refraction", "caustics", "Gerstner", "fluid", "depth coloring", "stylized water", "toon water", or references to any water-related visual effect. Also trigger when the user references NVJOB, bmjoy, or Alexander Ameye water shader repos.

---

## CORE RULES — MOBILE WATER CONSTRAINTS

1. **No tessellation on mobile.** Mobile GPUs do not support hardware tessellation efficiently. Use vertex displacement on the existing mesh instead.

2. **Limit water shader to ≤ 4 texture samples.** A typical mobile water shader budget: 1 normal map (tiled, scrolled), 1 depth texture (for shoreline/foam), 1 scene color (for refraction), 1 foam/detail texture. Drop scene color refraction first if over budget.

3. **Depth Texture and Opaque Texture must be enabled** in URP Asset settings for depth-based shoreline detection and refraction. These add a copy pass — budget ~1ms on mobile. If this is too expensive, use vertex-color-based depth instead.

4. **Use single-layer normals.** Desktop water often blends 2-3 normal map samples at different scales. On mobile, use 1 normal map sample with UV scrolling for animation.

5. **Gerstner waves in vertex shader only.** Never compute wave displacement per-fragment. Limit to 2–3 wave octaves on mobile (4+ on desktop).

6. **Half precision for all water calculations** except world position and UV coordinates.

---

## MOBILE WATER SHADER — MINIMAL TEMPLATE (URP HLSL)

This template provides a production-ready starting point for mobile water:

```hlsl
Shader "Mobile/Water_Simple"
{
    Properties
    {
        _ShallowColor ("Shallow Color", Color) = (0.3, 0.8, 0.9, 0.6)
        _DeepColor ("Deep Color", Color) = (0.1, 0.2, 0.5, 0.9)
        _DepthMaxDistance ("Depth Max Distance", Float) = 3.0
        _NormalMap ("Normal Map", 2D) = "bump" {}
        _NormalScale ("Normal Scale", Float) = 0.5
        _WaveSpeed ("Wave Speed", Float) = 0.05
        _FoamColor ("Foam Color", Color) = (1, 1, 1, 1)
        _FoamDistance ("Foam Distance", Float) = 0.4
        _FoamCutoff ("Foam Cutoff", Float) = 0.5
    }
    
    SubShader
    {
        Tags {
            "RenderType" = "Transparent"
            "Queue" = "Transparent"
            "RenderPipeline" = "UniversalPipeline"
        }
        
        Pass
        {
            Blend SrcAlpha OneMinusSrcAlpha
            ZWrite Off
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"
            
            CBUFFER_START(UnityPerMaterial)
                half4 _ShallowColor;
                half4 _DeepColor;
                half _DepthMaxDistance;
                float4 _NormalMap_ST;
                half _NormalScale;
                half _WaveSpeed;
                half4 _FoamColor;
                half _FoamDistance;
                half _FoamCutoff;
            CBUFFER_END
            
            TEXTURE2D(_NormalMap); SAMPLER(sampler_NormalMap);
            
            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
            };
            
            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float2 uv : TEXCOORD0;
                float4 screenPos : TEXCOORD1;
                float3 positionWS : TEXCOORD2;
            };
            
            Varyings vert(Attributes v)
            {
                Varyings o;
                VertexPositionInputs posInputs = GetVertexPositionInputs(v.positionOS.xyz);
                o.positionCS = posInputs.positionCS;
                o.positionWS = posInputs.positionWS;
                o.screenPos = ComputeScreenPos(o.positionCS);
                o.uv = TRANSFORM_TEX(v.uv, _NormalMap);
                return o;
            }
            
            half4 frag(Varyings i) : SV_Target
            {
                // --- Depth-based coloring ---
                float2 screenUV = i.screenPos.xy / i.screenPos.w;
                float rawDepth = SampleSceneDepth(screenUV);
                float sceneEyeDepth = LinearEyeDepth(rawDepth, _ZBufferParams);
                float surfaceEyeDepth = i.screenPos.w;
                half depthDiff = saturate((sceneEyeDepth - surfaceEyeDepth) / _DepthMaxDistance);
                
                half4 waterColor = lerp(_ShallowColor, _DeepColor, depthDiff);
                
                // --- Animated normal map (single sample) ---
                float2 scrollUV = i.uv + _Time.y * _WaveSpeed;
                half3 normal = UnpackNormalScale(
                    SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, scrollUV),
                    _NormalScale
                );
                
                // --- Shoreline foam ---
                half foamDepth = saturate((sceneEyeDepth - surfaceEyeDepth) / _FoamDistance);
                half foam = step(foamDepth, _FoamCutoff);
                waterColor = lerp(waterColor, _FoamColor, foam * _FoamColor.a);
                
                // --- Simple fresnel ---
                half3 viewDir = normalize(_WorldSpaceCameraPos - i.positionWS);
                half fresnel = pow(1.0h - saturate(dot(half3(0, 1, 0), viewDir)), 3.0h);
                waterColor.a = lerp(waterColor.a, 1.0h, fresnel);
                
                return waterColor;
            }
            ENDHLSL
        }
    }
    
    Fallback "Universal Render Pipeline/Unlit"
}
```

---

## GERSTNER WAVE IMPLEMENTATION

Gerstner waves produce realistic trochoidal wave motion. Compute in vertex shader only:

```hlsl
// Add to vertex shader — call for each wave octave
float3 GerstnerWave(float3 worldPos, float2 direction, float steepness, 
                     float wavelength, float speed)
{
    float k = 2.0 * PI / wavelength;
    float c = sqrt(9.8 / k); // Phase speed from dispersion relation
    float2 d = normalize(direction);
    float f = k * (dot(d, worldPos.xz) - c * speed * _Time.y);
    float a = steepness / k; // Amplitude
    
    return float3(
        d.x * a * cos(f),
        a * sin(f),
        d.y * a * cos(f)
    );
}

// In vertex shader:
Varyings vert(Attributes v)
{
    float3 worldPos = TransformObjectToWorld(v.positionOS.xyz);
    
    // 2-3 wave octaves for mobile (vary direction, wavelength, steepness)
    float3 wave1 = GerstnerWave(worldPos, float2(1, 0.5), 0.25, 8.0, 1.0);
    float3 wave2 = GerstnerWave(worldPos, float2(0.3, 1), 0.15, 4.0, 1.2);
    
    worldPos += wave1 + wave2;
    
    // Recalculate normals from wave tangent/binormal
    // (simplified — use cross product of partial derivatives)
    
    Varyings o;
    o.positionCS = TransformWorldToHClip(worldPos);
    // ... rest of vertex setup
    return o;
}
```

---

## WATER FEATURE COST TIERS

Use this to decide which features fit your mobile budget:

### Tier 1: Ultra-Low Cost (low-end mobile, ~0.5ms)
- Vertex color or UV-based depth coloring (no depth texture)
- Single scrolling normal map
- No foam, no refraction
- Opaque rendering (no alpha blend)

### Tier 2: Standard Mobile (~1-2ms)
- Depth texture shoreline coloring
- Shoreline foam via depth threshold
- Single scrolling normal map
- Fresnel-based alpha
- 1-2 Gerstner wave octaves

### Tier 3: High-Quality Mobile (~2-4ms)
- Depth + opaque texture for refraction
- Dual normal map blending
- Gerstner waves (3 octaves)
- Animated foam texture
- Specular highlights from main light
- Caustics (projected texture or shader-generated)

### Tier 4: Desktop Only
- Planar reflections or SSR
- Tessellation
- FFT-based ocean simulation
- Subsurface scattering
- 4+ Gerstner octaves

---

## SHADER GRAPH WATER — NODE SETUP GUIDE

For Shader Graph users, here's the node-efficient approach to mobile water:

1. **Depth Fade**: Scene Depth node → subtract Fragment Eye Depth → divide by max distance → saturate → lerp between shallow/deep colors

2. **Normal Animation**: UV node → Add (Time × speed vector) → Sample Texture 2D (normal map, set to Normal) → Normal Strength node

3. **Shoreline Foam**: Same depth calculation but with smaller distance → Step node → multiply by foam color → Add to water color

4. **Fresnel**: Fresnel Effect node (power ~3) → lerp alpha between base and 1.0

5. **Vertex Displacement** (waves): Position node (Object Space) → custom sub-graph with sine/cosine → add to Position output

**Total node budget**: Keep under 60 nodes for the water shader on mobile. Use Sub Graphs to organize — they don't add overhead but keep the graph readable.

---

## REFERENCE REPOS — COPY AND STUDY

These are the best open-source water shaders to reference:

| Repo | Mobile? | Features | License |
|------|---------|----------|---------|
| [nvjob/nvjob-water-shader-simple-and-fast](https://github.com/nvjob/nvjob-water-shader-simple-and-fast) | **YES** | Normal mapping, parallax, no tessellation | MIT |
| [nvjob/nvjob-water-shaders-v2](https://github.com/nvjob/nvjob-water-shaders-v2) | **YES** | V1 + reflections + shoreline foam | MIT |
| [bmjoy/Stylized-Water-Shader-Unity-URP](https://github.com/bmjoy/Stylized-Water-Shader-Unity-URP) | **YES** | URP, depth color, foam, refraction | — |
| [N7RX/Simple-Water-Shader](https://github.com/N7RX/Simple-Water-Shader) | **YES** | Lightweight mobile-first | — |
| [danielshervheim/unity-stylized-water](https://github.com/danielshervheim/unity-stylized-water) | Partial | Stylized with material presets (~953★) | — |
| [MatrixRex/Uber-Stylized-Water](https://github.com/MatrixRex/Uber-Stylized-Water) | Partial | Unity 6, highly customizable | — |
| [tuxalin/water-shader](https://github.com/tuxalin/water-shader) | No | Multi-platform, Gerstner, caustics | — |

---

## SOURCES

- Alexander Ameye: Stylized Water Shader — https://ameye.dev/notes/stylized-water-shader/
- Roystan: Toon Water — https://roystan.net/articles/toon-water/
- Cyanilux: 2D Water Shader Breakdown — https://www.cyanilux.com/tutorials/2d-water-shader-breakdown/
- Harry Alisavakis: Water Shaders (CC0) — https://halisavakis.com/
- Catlike Coding: Flow Series — https://catlikecoding.com/unity/tutorials/
- 80.lv: Nature Shaders (water + sand) — https://80.lv/articles/tutorial-how-to-make-nature-shaders-in-unity
- 80.lv: Lava Lamp (fluid sim) — https://80.lv/articles/tutorial-creating-a-lava-lamp-shader-with-unity
- Brabl: Mobile Water Shader — https://brabl.com/unity-5-mobile-water-shader/
- Linden Reid: Simple Water Shader — https://lindenreidblog.com/2017/12/15/simple-water-shader-in-unity/
- Gabriel Aguiar: 2D Water / Waterfall VFX — https://www.gabrielaguiarprod.com
