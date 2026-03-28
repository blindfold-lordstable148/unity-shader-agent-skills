---
name: urp-hlsl-templates
description: Production-ready URP HLSL/ShaderLab code templates with SRP Batcher compatibility for Unity
---

# URP HLSL Shader Code Templates

## TRIGGER
Use this skill when the user wants to write a Unity shader in HLSL/ShaderLab code (not Shader Graph) for Universal Render Pipeline. Triggers include: "URP shader", "HLSL", "ShaderLab", "write a shader", "custom shader", "vertex fragment shader", "SRP Batcher", "CBUFFER", "custom lit shader", "unlit shader", or any code-based shader authoring in URP. Also trigger when converting Built-in pipeline shaders to URP, or when the user references Cyanilux templates, phi-lira gists, or NedMakesGames tutorials.

---

## CORE RULES — URP HLSL

1. **Always use `HLSLPROGRAM` / `ENDHLSL`, never `CGPROGRAM` / `ENDCG`.** URP uses HLSL, not CG. CG blocks will compile but miss URP-specific includes and macros.

2. **All material properties must be in a `CBUFFER_START(UnityPerMaterial)` / `CBUFFER_END` block** for SRP Batcher compatibility. Without this, each material breaks batching and gets its own draw call.

3. **Include URP Core, not legacy Unity includes.**
   ```hlsl
   // CORRECT (URP)
   #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
   #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
   
   // WRONG (Built-in / legacy)
   #include "UnityCG.cginc"
   #include "Lighting.cginc"
   ```

4. **Use URP transform macros**, not legacy ones:
   ```hlsl
   // URP                              // Legacy equivalent
   TransformObjectToHClip(posOS)       // UnityObjectToClipPos(v.vertex)
   TransformObjectToWorld(posOS)       // mul(unity_ObjectToWorld, v.vertex)
   TransformObjectToWorldNormal(nOS)   // UnityObjectToWorldNormal(v.normal)
   TransformWorldToHClip(posWS)        // mul(UNITY_MATRIX_VP, float4(posWS, 1))
   GetVertexPositionInputs(posOS)      // (convenience struct)
   GetVertexNormalInputs(nOS, tanOS)   // (convenience struct)
   ```

5. **Tags must include `"RenderPipeline" = "UniversalPipeline"`** or the shader won't be recognized by URP.

6. **Use `TEXTURE2D` and `SAMPLER` macros** instead of `sampler2D`:
   ```hlsl
   TEXTURE2D(_MainTex);
   SAMPLER(sampler_MainTex);
   // Sample with:
   half4 col = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv);
   ```

---

## TEMPLATE: URP UNLIT (MOBILE-READY)

Minimal unlit shader with SRP Batcher compatibility. Use as starting point for mobile shaders:

```hlsl
Shader "Custom/URP_Unlit_Mobile"
{
    Properties
    {
        _BaseMap ("Base Map", 2D) = "white" {}
        _BaseColor ("Base Color", Color) = (1, 1, 1, 1)
    }
    
    SubShader
    {
        Tags
        {
            "RenderType" = "Opaque"
            "RenderPipeline" = "UniversalPipeline"
            "Queue" = "Geometry"
        }
        
        Pass
        {
            Name "ForwardLit"
            Tags { "LightMode" = "UniversalForward" }
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_fog
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            
            // SRP Batcher compatible: all properties in CBUFFER
            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                half4 _BaseColor;
            CBUFFER_END
            
            TEXTURE2D(_BaseMap);
            SAMPLER(sampler_BaseMap);
            
            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
            };
            
            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float2 uv : TEXCOORD0;
                half fogFactor : TEXCOORD1;
            };
            
            Varyings vert(Attributes input)
            {
                Varyings output;
                VertexPositionInputs posInputs = GetVertexPositionInputs(input.positionOS.xyz);
                output.positionCS = posInputs.positionCS;
                output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
                output.fogFactor = ComputeFogFactor(posInputs.positionCS.z);
                return output;
            }
            
            half4 frag(Varyings input) : SV_Target
            {
                half4 color = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv) * _BaseColor;
                color.rgb = MixFog(color.rgb, input.fogFactor);
                return color;
            }
            ENDHLSL
        }
        
        // Shadow caster pass (required for objects to cast shadows)
        Pass
        {
            Name "ShadowCaster"
            Tags { "LightMode" = "ShadowCaster" }
            
            ZWrite On ZTest LEqual ColorMask 0
            
            HLSLPROGRAM
            #pragma vertex ShadowPassVertex
            #pragma fragment ShadowPassFragment
            #include "Packages/com.unity.render-pipelines.universal/Shaders/ShadowCasterPass.hlsl"
            ENDHLSL
        }
        
        // Depth pass (required for depth-based effects)
        Pass
        {
            Name "DepthOnly"
            Tags { "LightMode" = "DepthOnly" }
            
            ZWrite On ColorMask R
            
            HLSLPROGRAM
            #pragma vertex DepthOnlyVertex
            #pragma fragment DepthOnlyFragment
            #include "Packages/com.unity.render-pipelines.universal/Shaders/DepthOnlyPass.hlsl"
            ENDHLSL
        }
    }
    
    Fallback "Universal Render Pipeline/Unlit"
}
```

---

## TEMPLATE: URP LIT WITH MAIN LIGHT

Adds Lambert diffuse lighting from the main directional light:

```hlsl
Shader "Custom/URP_SimpleLit_Mobile"
{
    Properties
    {
        _BaseMap ("Base Map", 2D) = "white" {}
        _BaseColor ("Base Color", Color) = (1, 1, 1, 1)
    }
    
    SubShader
    {
        Tags
        {
            "RenderType" = "Opaque"
            "RenderPipeline" = "UniversalPipeline"
            "Queue" = "Geometry"
        }
        
        Pass
        {
            Name "ForwardLit"
            Tags { "LightMode" = "UniversalForward" }
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile _ _MAIN_LIGHT_SHADOWS _MAIN_LIGHT_SHADOWS_CASCADE
            #pragma multi_compile _ _SHADOWS_SOFT
            #pragma multi_compile_fog
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
            
            CBUFFER_START(UnityPerMaterial)
                float4 _BaseMap_ST;
                half4 _BaseColor;
            CBUFFER_END
            
            TEXTURE2D(_BaseMap);
            SAMPLER(sampler_BaseMap);
            
            struct Attributes
            {
                float4 positionOS : POSITION;
                float3 normalOS : NORMAL;
                float2 uv : TEXCOORD0;
            };
            
            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float2 uv : TEXCOORD0;
                float3 normalWS : TEXCOORD1;
                float3 positionWS : TEXCOORD2;
                half fogFactor : TEXCOORD3;
            };
            
            Varyings vert(Attributes input)
            {
                Varyings output;
                VertexPositionInputs posInputs = GetVertexPositionInputs(input.positionOS.xyz);
                VertexNormalInputs normInputs = GetVertexNormalInputs(input.normalOS);
                
                output.positionCS = posInputs.positionCS;
                output.positionWS = posInputs.positionWS;
                output.normalWS = normInputs.normalWS;
                output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
                output.fogFactor = ComputeFogFactor(posInputs.positionCS.z);
                return output;
            }
            
            half4 frag(Varyings input) : SV_Target
            {
                half4 baseColor = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv) * _BaseColor;
                
                // Main light (directional)
                float4 shadowCoord = TransformWorldToShadowCoord(input.positionWS);
                Light mainLight = GetMainLight(shadowCoord);
                
                half3 normal = normalize(input.normalWS);
                half NdotL = saturate(dot(normal, mainLight.direction));
                half3 diffuse = baseColor.rgb * mainLight.color * NdotL * mainLight.shadowAttenuation;
                
                // Ambient
                half3 ambient = SampleSH(normal) * baseColor.rgb;
                
                half3 finalColor = diffuse + ambient;
                finalColor = MixFog(finalColor, input.fogFactor);
                
                return half4(finalColor, baseColor.a);
            }
            ENDHLSL
        }
        
        // Include ShadowCaster and DepthOnly passes (same as unlit template above)
        UsePass "Custom/URP_Unlit_Mobile/ShadowCaster"
        UsePass "Custom/URP_Unlit_Mobile/DepthOnly"
    }
    
    Fallback "Universal Render Pipeline/Simple Lit"
}
```

---

## BUILT-IN → URP CONVERSION CHEAT SHEET

| Built-in | URP Equivalent |
|----------|----------------|
| `#include "UnityCG.cginc"` | `#include ".../Core.hlsl"` |
| `#include "Lighting.cginc"` | `#include ".../Lighting.hlsl"` |
| `UnityObjectToClipPos(v)` | `TransformObjectToHClip(v.xyz)` |
| `UnityObjectToWorldNormal(n)` | `TransformObjectToWorldNormal(n)` |
| `_WorldSpaceLightPos0` | `GetMainLight().direction` |
| `_LightColor0` | `GetMainLight().color` |
| `UNITY_LIGHTMODEL_AMBIENT` | `SampleSH(normal)` |
| `SHADOW_COORDS` / `TRANSFER_SHADOW` | `TransformWorldToShadowCoord(posWS)` |
| `SHADOW_ATTENUATION(i)` | `GetMainLight(shadowCoord).shadowAttenuation` |
| `tex2D(_Tex, uv)` | `SAMPLE_TEXTURE2D(_Tex, sampler_Tex, uv)` |
| `sampler2D _Tex` | `TEXTURE2D(_Tex); SAMPLER(sampler_Tex);` |
| `UNITY_FOG_COORDS` / `APPLY_FOG` | `ComputeFogFactor()` / `MixFog()` |
| `v2f` struct convention | `Varyings` struct convention |
| `appdata` struct convention | `Attributes` struct convention |
| `Tags { "LightMode" = "ForwardBase" }` | `Tags { "LightMode" = "UniversalForward" }` |
| Surface shader `#pragma surface` | No equivalent — write vertex/fragment manually |

---

## MULTI-PASS AND ADDITIONAL PASSES

URP requires explicit pass names for the renderer to recognize them:

| LightMode Tag | Purpose |
|---------------|---------|
| `UniversalForward` | Main rendering pass |
| `ShadowCaster` | Shadow map generation |
| `DepthOnly` | Depth prepass |
| `DepthNormals` | Depth + normals prepass (for SSAO, depth-normals effects) |
| `Meta` | Lightmap baking |
| `Universal2D` | 2D renderer pass |
| `SRPDefaultUnlit` | Fallback when no LightMode matches |

---

## COMMON PITFALLS

1. **Forgetting CBUFFER** → Breaks SRP Batcher. Every draw call becomes unique. Check with Frame Debugger > SRP Batcher column.

2. **Using `sampler2D` instead of `TEXTURE2D` macro** → Works but loses cross-platform sampler control and may break on some platforms.

3. **Missing ShadowCaster pass** → Object won't cast shadows. Always include it or use `UsePass`/`Fallback`.

4. **Missing DepthOnly pass** → Depth-based effects (fog, post-processing, water depth) won't see this object.

5. **Using `multi_compile` instead of `shader_feature`** → `multi_compile` compiles ALL keyword variants. `shader_feature` only compiles variants used by materials in the build. For material-specific toggles, always use `shader_feature`.

---

## REFERENCE REPOS AND GISTS

- Cyanilux URP Code Templates (CC0) — https://github.com/Cyanilux/URP_ShaderCodeTemplates
- Cyanilux URP Shader Code Tutorial — https://www.cyanilux.com/tutorials/urp-shader-code/
- phi-lira URP Lit Template — https://gist.github.com/phi-lira/225cd7c5e8545be602dca4eb5ed111ba
- NedMakesGames 9-Part URP HLSL Series — https://nedmakesgames.medium.com/
- Halftone Toon URP Gist — https://gist.github.com/wonkee-kim/3b5c1821975b8414f420f93ef6cdf5a0
- Wide Outlines URP Gist — https://gist.github.com/ScottJDaley/6cddf0c8995ed61cac7088e22c983de1
- Toon Shader URP 11 Gist — https://gist.github.com/FaultyPine/08306e52971d77081ea159128cb3e5b5
- URP Custom Lit BlinnPhong — https://gist.github.com/xCyborg/2923ce9e57e65af9d417706dda4db4be
- ColinLeung-NiloCat URP Toon Lit — https://github.com/ColinLeung-NiloCat/UnityURPToonLitShaderExample

## SOURCES

- Cyanilux: URP Shader Code Templates — https://github.com/Cyanilux/URP_ShaderCodeTemplates
- NedMakesGames: Writing URP Shaders with Code — https://nedmakesgames.medium.com/
- Catlike Coding: Custom SRP — https://catlikecoding.com/unity/tutorials/
- Daniel Ilett: Shader Graph + HLSL — https://danielilett.com/
- ColinLeung-NiloCat: Mobile URP Toon — https://github.com/ColinLeung-NiloCat/UnityURPToonLitShaderExample
- Unity: Shader Performance — https://docs.unity3d.com/Manual/SL-ShaderPerformance.html
