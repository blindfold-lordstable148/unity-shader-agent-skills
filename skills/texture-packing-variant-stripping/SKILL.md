---
name: texture-packing-variant-stripping
description: Channel packing, variant reduction, shader_feature vs multi_compile, and build size optimization for Unity
---

# Texture Packing and Shader Variant Stripping for Unity Mobile

## TRIGGER
Use this skill when the user is dealing with texture optimization, channel packing, shader variant bloat, build size, shader compilation time, or shader keyword management in Unity. Triggers include: "channel packing", "texture packing", "mask map", "MODS map", "RMA", "shader variants", "variant stripping", "multi_compile", "shader_feature", "build size", "shader keywords", "Always Included Shaders", "IPreprocessShaders", or complaints about slow shader compilation or large build sizes.

---

## TEXTURE CHANNEL PACKING

Channel packing stores multiple grayscale maps in a single RGBA texture. This reduces both texture sample count and memory bandwidth — the primary bottleneck on mobile.

### Standard URP Packing Convention
```
Mask Map (single RGBA texture):
  R = Metallic
  G = Occlusion (AO)
  B = Detail mask (or custom)
  A = Smoothness
```

### Best Practices

1. **Put the most visually important mask in the Green channel.** Human vision is most sensitive to green, and most texture compression formats (ETC2, ASTC, DXT) allocate more bits to green.

2. **Pack related data together.** If metallic and smoothness are always sampled together, put them in the same texture.

3. **Use linear color space** (not sRGB) for packed mask textures. In Unity, uncheck "sRGB" in the texture import settings for any non-color data (roughness, AO, metallic, height).

4. **One texture sample replaces four.** Instead of:
   ```hlsl
   // BAD — 4 texture samples
   half metallic = SAMPLE_TEXTURE2D(_MetallicMap, sampler_MetallicMap, uv).r;
   half ao = SAMPLE_TEXTURE2D(_AOMap, sampler_AOMap, uv).r;
   half detail = SAMPLE_TEXTURE2D(_DetailMask, sampler_DetailMask, uv).r;
   half smooth = SAMPLE_TEXTURE2D(_SmoothnessMap, sampler_SmoothnessMap, uv).r;
   ```
   Use:
   ```hlsl
   // GOOD — 1 texture sample
   half4 masks = SAMPLE_TEXTURE2D(_MaskMap, sampler_MaskMap, uv);
   half metallic = masks.r;
   half ao = masks.g;
   half detail = masks.b;
   half smooth = masks.a;
   ```

### Advanced: Terrain Packing
Unity's Shader Graph 17.4 terrain samples demonstrate advanced packing that reduces **12 texture samples to 5** for a 4-layer terrain:

- **CNS packing**: Color (RGB) + Normal (RG → reconstruct B) + Smoothness in a single set
- **CHNOSArray**: Texture arrays packing Color, Height, Normal, Occlusion, Smoothness

### HLSL Channel Packing Helper
```hlsl
// Pack 4 grayscale values into RGBA
half4 PackChannels(half metallic, half ao, half detail, half smoothness)
{
    return half4(metallic, ao, detail, smoothness);
}

// Reconstruct normal Z from XY (for normal map packing)
half3 UnpackNormalRG(half2 normalRG)
{
    half3 n;
    n.xy = normalRG * 2.0h - 1.0h;
    n.z = sqrt(1.0h - saturate(dot(n.xy, n.xy)));
    return n;
}
```

---

## SHADER VARIANT MANAGEMENT

### The Variant Explosion Problem
Each `multi_compile` keyword **doubles** variant count multiplicatively:
```
#pragma multi_compile _ _A        → 2 variants
#pragma multi_compile _ _A _B     → 3 variants
#pragma multi_compile _ _A
#pragma multi_compile _ _B        → 2 × 2 = 4 variants
#pragma multi_compile _ _A
#pragma multi_compile _ _B
#pragma multi_compile _ _C        → 2 × 2 × 2 = 8 variants
```

With URP's built-in keywords (shadows, fog, additional lights, etc.), a single shader can generate **thousands of variants**.

### Impact
- **Build time**: 20+ minutes for complex shaders
- **Build size**: 100MB+ shader data (critical for WebGL)
- **Runtime memory**: 20-40% extra draw calls from variant switching
- **WebGL**: Shader compilation blocks the main thread — more variants = longer freeze

### shader_feature vs multi_compile

| | `shader_feature` | `multi_compile` |
|---|---|---|
| Compiles | Only variants used by materials in build | ALL combinations |
| Use for | Material-specific toggles | Global state (fog, shadows) |
| Unused variants | Stripped automatically | Included regardless |
| Local version | `shader_feature_local` (64 max) | `multi_compile_local` (64 max) |
| Global version | `shader_feature` (256 max) | `multi_compile` (256 max) |

**Rule**: Use `shader_feature` for everything you control (material toggles, feature flags). Only use `multi_compile` for things Unity controls globally (fog, lightmaps, instancing).

### Recommended URP Keyword Stripping

Strip these keywords if you're not using the features:

```csharp
// In an IPreprocessShaders implementation:
static readonly ShaderKeyword[] STRIP_KEYWORDS = new[]
{
    new ShaderKeyword("_ADDITIONAL_LIGHTS"),          // If using main light only
    new ShaderKeyword("_ADDITIONAL_LIGHT_SHADOWS"),   // If no additional light shadows
    new ShaderKeyword("_SHADOWS_SOFT"),               // If using hard shadows only
    new ShaderKeyword("_MIXED_LIGHTING_SUBTRACTIVE"), // If not using mixed lighting
    new ShaderKeyword("_SCREEN_SPACE_OCCLUSION"),     // If not using SSAO
    new ShaderKeyword("LIGHTMAP_SHADOW_MIXING"),      // If not mixing lightmaps + realtime
    new ShaderKeyword("_LIGHT_LAYERS"),               // If not using light layers
    new ShaderKeyword("_REFLECTION_PROBE_BLENDING"),  // If not blending reflection probes
    new ShaderKeyword("_REFLECTION_PROBE_BOX_PROJECTION"), // If not using box projection
};
```

### URP Asset Feature Stripping
In URP Asset settings, disable unused features to prevent their variants from being compiled:

- Additional Lights → Off (if only using directional)
- Shadows → Reduce cascade count, disable soft shadows
- Post-Processing → Disable if not used
- HDR → Disable if not needed
- Environment Reflections → Disable if using baked reflections

### Platform-Conditional Compilation
```hlsl
// Include features only on platforms that can handle them
#if defined(SHADER_API_MOBILE) || defined(SHADER_API_GLES) || defined(SHADER_API_GLES3)
    // Mobile/WebGL: simplified path
    #define MOBILE_SHADER 1
#else
    // Desktop: full features
    #define MOBILE_SHADER 0
#endif

half4 frag(Varyings i) : SV_Target
{
    half4 color = SampleAlbedo(i.uv);
    
    #if !MOBILE_SHADER
        // Desktop-only: normal mapping, specular, etc.
        color.rgb += SampleNormalAndLight(i);
    #endif
    
    return color;
}
```

---

## TEXTURE COMPRESSION FOR MOBILE/WEB

| Format | Platform | Bits/Pixel | Quality | Notes |
|--------|----------|------------|---------|-------|
| ASTC 4×4 | Android (ES 3.0+), iOS | 8 | Best | Preferred for all mobile |
| ASTC 6×6 | Android (ES 3.0+), iOS | 3.56 | Good | Use for less important textures |
| ASTC 8×8 | Android (ES 3.0+), iOS | 2 | Acceptable | Background/distant textures |
| ETC2 | Android (ES 3.0+), WebGL 2.0 | 4-8 | Good | WebGL 2.0 default |
| ETC1 | Android (ES 2.0) | 4 | OK | No alpha channel support |
| PVRTC | iOS (older) | 2-4 | Variable | Legacy, prefer ASTC |

**For channel-packed textures**: Use ASTC 4×4 or ETC2 with linear color space. Avoid PVRTC for mask maps — its block compression causes visible artifacts on sharp mask boundaries.

---

## OPTIMIZATION CHECKLIST

- [ ] All grayscale maps packed into RGBA channels of shared textures
- [ ] Packed textures imported as Linear (not sRGB)
- [ ] Green channel used for most important mask data
- [ ] `shader_feature` used instead of `multi_compile` for material toggles
- [ ] Local keywords used where possible (`shader_feature_local`)
- [ ] Unused URP features disabled in URP Asset
- [ ] Always Included Shaders list cleaned (Project Settings > Graphics)
- [ ] IPreprocessShaders stripping script added for unused keywords
- [ ] Build log checked for variant count (look for "Compiled N shader variants")
- [ ] ASTC or ETC2 compression used for all mobile textures
- [ ] Standard Shader removed from project (use Simple Lit or custom)

---

## SOURCES

- Unity 6 URP Channel-Packed Textures — https://docs.unity3d.com/6000.3/Documentation/Manual/urp/shaders-in-universalrp-channel-packed-texture.html
- Unity Learn: Texture Channel Packing for Mobile — https://learn.unity.com/course/3d-art-optimization-for-mobile-gaming-5474/
- Shader Graph 17.4: Terrain Packing — https://docs.unity3d.com/Packages/com.unity.shadergraph@17.4/manual/Shader-Graph-Sample-Terrain-Packing.html
- Polycount Wiki: Channel Packing — http://wiki.polycount.com/wiki/ChannelPacking
- Unity 6: Shader Variant Stripping — https://docs.unity3d.com/6000.3/Documentation/Manual/shader-variant-stripping.html
- Unity Blog: Stripping Scriptable Shader Variants — https://unity.com/blog/engine-platform/stripping-scriptable-shader-variants
- Unity Blog: Shader Variants Optimization Tips — https://unity.com/blog/engine-platform/shader-variants-optimization-troubleshooting-tips
- Unity Manual: shader_feature vs multi_compile — https://docs.unity3d.com/2020.2/Documentation/Manual/SL-MultipleProgramVariants.html
- TheGamedev.Guru: Shader Variant Stripping — https://thegamedev.guru/unity-cpu-performance/shader-variants-stripping/
- Unity Web: Remove Unused Resources — https://docs.unity3d.com/2022.3/Documentation/Manual/web-optimization-remove-resources.html
- Unity Learn: Optimizing Graphics — https://learn.unity.com/tutorial/optimizing-graphics-in-unity
