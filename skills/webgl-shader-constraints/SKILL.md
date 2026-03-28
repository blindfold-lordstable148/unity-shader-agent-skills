---
name: webgl-shader-constraints
description: WebGL 1.0/2.0 shader restrictions, variant stripping, loop/indexing rules, and browser deployment workarounds for Unity
---

# WebGL Shader Constraints and Workarounds for Unity

## TRIGGER
Use this skill when the user is developing Unity shaders for WebGL browser deployment. Triggers include: "WebGL", "browser game", "web build", "WebGL 1.0", "WebGL 2.0", "WebGPU", "GLSL", "OpenGL ES 2.0/3.0", "shader compilation web", "web build size", "shader variants web", "Always Included Shaders", or any mention of deploying Unity content to browsers. Also trigger when the user reports shader errors only on web builds, or when dealing with build size issues related to shaders.

---

## CORE RULES

1. **WebGL 1.0 (OpenGL ES 2.0) forbids dynamic array indexing.** You can only index arrays with constants or loop counter variables. This is the #1 cause of shader failures on WebGL 1.0.

2. **WebGL 1.0 only allows counting `for` loops.** No `while` loops, no `do-while` loops. The loop counter must be initialized, compared, and incremented in the `for` statement. The loop bound must be a constant expression.

3. **WebGL 2.0 (OpenGL ES 3.0) removes loop and indexing restrictions** but adds mandatory precision qualifiers. All variables must have explicit precision (`highp`, `mediump`, `lowp`) or use the default precision statement.

4. **WebGL only supports baked GI and non-directional lightmaps.** No real-time GI, no directional lightmaps, no light probes with L2 spherical harmonics.

5. **Never use the Standard Shader for WebGL builds.** It compiles hundreds of variants and can add 50MB+ to build size. Use Simple Lit, Unlit, or custom shaders with minimal keywords.

6. **Aggressively strip shader variants for web.** Every variant increases download size and initial shader compilation time (which blocks the main thread in the browser).

---

## WEBGL 1.0 vs 2.0 FEATURE COMPARISON

| Feature | WebGL 1.0 (ES 2.0) | WebGL 2.0 (ES 3.0) |
|---------|--------------------|--------------------|
| Array indexing | Constants/loop index only | Dynamic indexing allowed |
| Loops | Counting `for` only | All loop types |
| GPU instancing | **No** | Yes |
| 3D textures | No | Yes |
| Texture arrays | No | Yes |
| Integer textures | No | Yes |
| `texelFetch` | No | Yes |
| Uniform Buffer Objects | No | Yes |
| Vertex texture units | 0 guaranteed | ≥16 guaranteed |
| MRT (Multiple Render Targets) | No | Yes |
| Deferred rendering | **Not possible** | Possible (but expensive) |
| Directional lightmaps | No | No (Unity limitation) |
| Precision qualifiers | Optional | **Mandatory** |

---

## LOOP AND INDEXING WORKAROUNDS (WebGL 1.0)

```hlsl
// FAILS on WebGL 1.0 — dynamic array indexing
half4 colors[4];
int index = someCalculation();
half4 result = colors[index]; // ERROR: non-constant index

// WORKAROUND — unroll with if/else
half4 result;
if (index == 0) result = colors[0];
else if (index == 1) result = colors[1];
else if (index == 2) result = colors[2];
else result = colors[3];

// FAILS on WebGL 1.0 — while loop
int i = 0;
while (i < count) { /* ... */ i++; } // ERROR

// WORKAROUND — counting for loop with constant bound
for (int i = 0; i < 16; i++) // 16 is a constant
{
    if (i >= count) break; // dynamic exit
    // ... loop body
}

// FAILS on WebGL 1.0 — non-constant loop bound
for (int i = 0; i < _DynamicCount; i++) { } // ERROR

// WORKAROUND — constant max with early break
#define MAX_ITERATIONS 32
for (int i = 0; i < MAX_ITERATIONS; i++)
{
    if (i >= _DynamicCount) break;
    // ... loop body
}
```

---

## SHADER VARIANT STRIPPING FOR WEB

Shader variants are the single biggest contributor to WebGL build size. Each variant must be downloaded and compiled by the browser.

### Step 1: Remove Always Included Shaders
In Project Settings > Graphics > Always Included Shaders, remove everything you don't need. The Standard Shader alone adds hundreds of variants.

### Step 2: Use `shader_feature` instead of `multi_compile`
```hlsl
// BAD for web — compiles ALL keyword combinations
#pragma multi_compile _ _FEATURE_A
#pragma multi_compile _ _FEATURE_B
// = 4 variants (2 × 2)

// GOOD for web — only compiles variants used by materials in build
#pragma shader_feature _ _FEATURE_A
#pragma shader_feature _ _FEATURE_B
// = only variants referenced by materials in scenes
```

### Step 3: Use local keywords (max 64 per shader)
```hlsl
// Global keywords (shared pool of 256, wasteful)
#pragma multi_compile _ _MY_KEYWORD

// Local keywords (per-shader pool of 64, efficient)
#pragma shader_feature_local _ _MY_KEYWORD
```

### Step 4: Conditional compilation for web
```hlsl
// Strip expensive features on WebGL
#if !defined(SHADER_API_GLES) && !defined(SHADER_API_GLES3)
    // Desktop-only features here (SSR, complex lighting, etc.)
#endif

// Or target mobile/web specifically
#if defined(SHADER_API_MOBILE) || defined(SHADER_API_GLES) || defined(SHADER_API_GLES3)
    // Simplified mobile/web path
#else
    // Full desktop path
#endif
```

### Step 5: Programmatic stripping with IPreprocessShaders
```csharp
using UnityEditor.Build;
using UnityEditor.Rendering;
using System.Collections.Generic;

class WebGLShaderStripper : IPreprocessShaders
{
    public int callbackOrder => 0;
    
    public void OnProcessShader(Shader shader, ShaderSnippetData snippet, 
                                 IList<ShaderCompilerData> data)
    {
        for (int i = data.Count - 1; i >= 0; i--)
        {
            // Strip deferred variants (useless on WebGL)
            if (snippet.passType == PassType.Deferred)
            {
                data.RemoveAt(i);
                continue;
            }
            
            // Strip specific keywords
            var keywords = data[i].shaderKeywordSet;
            if (keywords.IsEnabled(new ShaderKeyword("_SHADOWS_SOFT")) ||
                keywords.IsEnabled(new ShaderKeyword("_ADDITIONAL_LIGHTS")))
            {
                data.RemoveAt(i);
            }
        }
    }
}
```

---

## WEBGL PERFORMANCE TIPS

1. **Shader compilation blocks the main thread.** Unlike native builds, WebGL compiles shaders synchronously. Many shaders = long freeze on first load. Minimize variant count.

2. **Use URP instead of Built-in for web.** URP produces smaller builds and fewer shader variants when properly configured.

3. **Texture compression matters.** Use ASTC (preferred) or ETC2 for WebGL 2.0. DXT/BC formats only work on desktop browsers.

4. **Memory is severely limited.** WebGL runs inside the browser's memory sandbox. Large shader variant collections can trigger OOM crashes.

5. **Test on mobile browsers.** iOS Safari WebGL performance is significantly worse than desktop Chrome. Many effects that work on desktop Chrome will fail or stutter on mobile Safari.

---

## WEBGPU (FUTURE)

Unity 6 has experimental WebGPU support. When available, WebGPU removes many WebGL restrictions: compute shaders, storage buffers, better threading. However, as of 2025, WebGL 2.0 remains the production target for browser games. Write shaders targeting WebGL 2.0 and plan WebGPU as a progressive enhancement.

---

## SOURCES

- Unity WebGL Graphics — https://docs.unity3d.com/530/Documentation/Manual/webgl-graphics.html
- Unity 6 WebGL2 Docs — https://docs.unity3d.com/6000.3/Documentation/Manual/WebGL2.html
- Unity Web Graphics API Intro — https://docs.unity3d.com/6000.3/Documentation/Manual/web-graphics-apis-intro.html
- Unity Web Optimization: Graphics — https://docs.unity3d.com/2022.3/Documentation/Manual/web-optimization-graphics.html
- Unity: Profile and Optimize Web Builds — https://unity.com/how-to/profile-optimize-web-build
- Unity Web: Remove Unused Resources — https://docs.unity3d.com/2022.3/Documentation/Manual/web-optimization-remove-resources.html
- WebGL2 Fundamentals: What's New — https://webgl2fundamentals.org/webgl/lessons/webgl2-whats-new.html
- Radiator Blog: WebGL Tips 2023 — https://www.blog.radiator.debacle.us/2023/01/unity-webgl-tips-advice-in-2023.html
- Backtrace: WebGL Memory Issues — https://backtrace.io/blog/memory-and-performance-issues-in-unity-webgl-builds
- Playgama: Unity WebGL 2025 — https://playgama.com/blog/general/unity-for-browser-games-boost-performance-and-engagement-effortlessly/
- JohannesDeml: WebGL Loading Test (700+ builds) — https://github.com/JohannesDeml/UnityWebGL-LoadingTest
- CrazyGames: Unity Optimizations Package — https://github.com/CrazyGamesCom/unity-optimizations-package
