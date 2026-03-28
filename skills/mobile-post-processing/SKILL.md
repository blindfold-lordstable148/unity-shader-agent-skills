---
name: mobile-post-processing
description: Single-pass post-processing, URP Renderer Features, and mobile-safe screen effects for Unity
---

# Mobile Post-Processing for Unity URP

## TRIGGER
Use this skill when the user wants to add, create, or optimize post-processing effects in Unity for mobile or WebGL targets. Triggers include: "post-processing", "bloom", "vignette", "color grading", "blur", "outline", "edge detection", "chromatic aberration", "Renderer Feature", "Full Screen Pass", "Blit", "post-process mobile", or any mention of screen-space effects on constrained hardware. Also trigger when the user reports post-processing is causing frame drops on mobile.

---

## CORE RULES

1. **Every post-processing pass costs a full-screen blit** — reading the entire framebuffer from main memory and writing it back. On tiled mobile GPUs, this flushes tile memory. Budget 0.5–2ms per pass on mid-range mobile.

2. **Combine all effects into a single pass whenever possible.** Unity's default post-processing stack runs each effect as a separate pass. On mobile, this alone can add 6ms+ (measured: vignette alone = 6ms on some devices). Use a single-pass approach instead.

3. **Disable HDR unless you specifically need Bloom.** HDR doubles framebuffer bandwidth (16-bit float vs 8-bit per channel). If you only need color grading, apply it in LDR.

4. **Bloom is the most expensive default effect on mobile.** It requires downsampling, blurring, and upsampling — multiple passes. If you need it, reduce iterations (2–3 max), use a half-resolution blur, and skip the high-quality prefilter.

5. **Never use Motion Blur or Depth of Field on mobile.** They require multiple texture samples per fragment across multiple passes.

6. **Prefer Renderer Features over Volume-based post-processing** for custom effects in URP. Renderer Features give you direct control over when and how the blit happens.

---

## SINGLE-PASS POST-PROCESSING TEMPLATE

This approach combines vignette + color grading + simple bloom in one pass:

```hlsl
Shader "Mobile/PostProcess_SinglePass"
{
    Properties
    {
        _MainTex ("Source", 2D) = "white" {}
        _VignetteIntensity ("Vignette Intensity", Range(0, 1)) = 0.3
        _VignetteRadius ("Vignette Radius", Range(0, 1)) = 0.7
        _Contrast ("Contrast", Range(0.5, 1.5)) = 1.1
        _Saturation ("Saturation", Range(0, 2)) = 1.1
        _Brightness ("Brightness", Range(0.5, 1.5)) = 1.0
    }
    
    SubShader
    {
        Tags { "RenderPipeline" = "UniversalPipeline" }
        
        Pass
        {
            ZTest Always ZWrite Off Cull Off
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            
            CBUFFER_START(UnityPerMaterial)
                half _VignetteIntensity;
                half _VignetteRadius;
                half _Contrast;
                half _Saturation;
                half _Brightness;
            CBUFFER_END
            
            TEXTURE2D(_MainTex); SAMPLER(sampler_MainTex);
            
            struct Attributes { float4 positionOS : POSITION; float2 uv : TEXCOORD0; };
            struct Varyings  { float4 positionCS : SV_POSITION; float2 uv : TEXCOORD0; };
            
            Varyings vert(Attributes v)
            {
                Varyings o;
                o.positionCS = TransformObjectToHClip(v.positionOS.xyz);
                o.uv = v.uv;
                return o;
            }
            
            half4 frag(Varyings i) : SV_Target
            {
                half4 color = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.uv);
                
                // --- Vignette (no extra texture sample) ---
                half2 uvCentered = i.uv - 0.5h;
                half vignette = 1.0h - saturate(
                    length(uvCentered) / _VignetteRadius
                );
                vignette = smoothstep(0.0h, 1.0h, vignette);
                color.rgb *= lerp(1.0h, vignette, _VignetteIntensity);
                
                // --- Brightness / Contrast ---
                color.rgb *= _Brightness;
                color.rgb = (color.rgb - 0.5h) * _Contrast + 0.5h;
                
                // --- Saturation ---
                half luminance = dot(color.rgb, half3(0.2126h, 0.7152h, 0.0722h));
                color.rgb = lerp(luminance.xxx, color.rgb, _Saturation);
                
                return saturate(color);
            }
            ENDHLSL
        }
    }
}
```

---

## URP RENDERER FEATURE (C# SETUP)

To inject the post-processing pass into URP:

```csharp
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class MobilePostProcessFeature : ScriptableRendererFeature
{
    [System.Serializable]
    public class Settings
    {
        public Material material;
        public RenderPassEvent renderPassEvent = RenderPassEvent.AfterRenderingPostProcessing;
    }
    
    public Settings settings = new Settings();
    MobilePostProcessPass pass;
    
    public override void Create()
    {
        pass = new MobilePostProcessPass(settings);
    }
    
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (settings.material == null) return;
        renderer.EnqueuePass(pass);
    }
    
    class MobilePostProcessPass : ScriptableRenderPass
    {
        Settings settings;
        RTHandle tempRT;
        
        public MobilePostProcessPass(Settings settings)
        {
            this.settings = settings;
            this.renderPassEvent = settings.renderPassEvent;
        }
        
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            var cmd = CommandBufferPool.Get("MobilePostProcess");
            var source = renderingData.cameraData.renderer.cameraColorTargetHandle;
            
            // Single blit with our combined shader
            Blitter.BlitCameraTexture(cmd, source, source, settings.material, 0);
            
            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
    }
}
```

---

## MOBILE-SAFE EFFECTS (COST RANKING)

| Effect | Cost | Mobile Verdict | Notes |
|--------|------|----------------|-------|
| Color grading (LDR) | ~0.1ms | **SAFE** | Pure math, no extra samples |
| Vignette | ~0.1ms | **SAFE** | UV distance calculation only |
| Film grain | ~0.2ms | **SAFE** | Noise from UV math, no texture |
| Simple outline (depth) | ~0.5ms | **CAUTION** | Requires depth texture + 4 neighbor samples |
| Bloom (2-iteration) | ~1-2ms | **CAUTION** | Use half-res, 2 iterations max |
| Chromatic aberration | ~0.3ms | **CAUTION** | 3 texture samples instead of 1 |
| Blur (single-pass box) | ~0.5ms | **CAUTION** | Use 4-tap box blur, never Gaussian |
| Motion blur | ~3-5ms | **AVOID** | Multiple samples along velocity |
| Depth of field | ~3-5ms | **AVOID** | CoC calculation + variable blur |
| Screen-space reflections | ~4-8ms | **AVOID** | Ray marching in screen space |
| Ambient occlusion (SSAO) | ~2-4ms | **AVOID** | Multiple depth samples per fragment |

---

## REFERENCE REPOS

| Repo | Purpose | Why It Matters |
|------|---------|----------------|
| [demonixis/FastPostProcessing](https://github.com/demonixis/FastPostProcessing) | Single-pass mobile post-FX | **Go-to for mobile.** Everything in one pass. |
| [TPiotr/SimpleMobilePostProcessing-Unity](https://github.com/TPiotr/SimpleMobilePostProcessing-Unity) | Blur + vignette | Created because Unity's vignette cost 6ms alone |
| [yahiaetman/URPCustomPostProcessingStack](https://github.com/yahiaetman/URPCustomPostProcessingStack) | URP Volume-compatible custom FX | 8 example effects with proper URP integration |
| [QianMo/X-PostProcessing-Library](https://github.com/QianMo/X-PostProcessing-Library) | Dozens of effects (MIT) | Desktop-quality reference; adapt for mobile |
| [GarrettGunnell/Post-Processing](https://github.com/GarrettGunnell/Post-Processing) | Modular bloom/grade/edge | Notes on single-pass condensation |

---

## SOURCES

- Febucci: Custom Post-Processing in URP — https://blog.febucci.com/2022/05/custom-post-processing-in-urp/
- ARM Community: Post-Processing on Mobile — https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/post-processing-effects-on-mobile-optimization-and-alternatives
- Unity Forum: Unity 6 Mobile Tips (Bloom/HDR benchmarks) — https://discussions.unity.com/t/unity-6-mobile-graphics-optimization-tips-and-tricks/1593293
- Android Developer: URP Asset Settings — https://developer.android.com/develop/xr/unity/performance/urp-asset-settings
- UWA Blog: Bloom Optimization — https://blog.en.uwa4d.com/2022/12/28/unity-mobile-game-performance-optimization-the-balance-between-graphics-performance-and-gpu-pressure/
