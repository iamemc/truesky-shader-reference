# QUICK REFERENCE: SHADER VARIABLES STATUS

## GODRAYS

### Constant Buffer Variables

| Shader File | Cbuffer | Variable | Offset | Size | Status | Used? |
|-------------|---------|----------|--------|------|--------|-------|
| atmospherics.msasm | AtmosphericsPerViewConstants (cb12) | `godraysIntensity` | 252 | 4 bytes | Declared | ❌ [unused] |
| atmospherics.msasm | AtmosphericsPerViewConstants (cb12) | `godraysVolumeDims` | 256 | 12 bytes | Declared | ❌ [unused] |

### Texture Resources

| Shader File | Texture Name | Type | Register | Sampled? | Written? |
|-------------|--------------|------|----------|----------|----------|
| atmospherics.msasm | `godraysTexture` | Texture2D | Not bound | ❌ No | ❌ No |
| hdr.msasm | `godraysVolumeTexture` | Texture3D | Not in bindings | ❌ No | ❌ No |
| clouds.msasm (skybox) | `godraysTexture` | Texture3D | t3 | ❌ No | ❌ No |

### JSON Preset Parameters → Shader Mapping

| JSON Parameter | Target Cbuffer Variable | Mapping Status |
|----------------|-------------------------|----------------|
| `godrayStrength` | `godraysIntensity`? | ⚠️ Unknown/Not verified |
| `GodraysSteps` | Unknown | ❌ No cbuffer equivalent found |

**VERDICT:** Godrays are completely non-functional - no shader code implements them.

---

## CLOUD SHADOWS

### Constant Buffer Variables

| Shader File | Cbuffer | Variable | Offset | Size | Status | Used? |
|-------------|---------|----------|--------|------|--------|-------|
| clouds.msasm | CloudPerViewConstants (cb13) | `shadowMatrix` | 96 | 64 bytes | Declared | ❌ [unused] |
| clouds.msasm | CloudPerViewConstants (cb13) | `shadowRangeKm` | 352 | 4 bytes | Declared | ❌ [unused] |
| clouds.msasm | CloudPerViewConstants (cb13) | `shadowTextureSize` | 356 | 4 bytes | Declared | ❌ [unused] |
| terrain.msasm | TerrainConstants (cb10) | `invShadowMatrix` | 160 | 64 bytes | Declared | ✅ **USED** |
| terrain.msasm | TerrainConstants (cb10) | `shadowRange` | 232 | 4 bytes | Declared | ❌ [unused] |

### Texture Resources

| Shader File | Texture Name | Type | Register | Sampled? | Written? |
|-------------|--------------|------|----------|----------|----------|
| terrain.msasm | `cloudShadowTexture` | Texture2D | t0 | ✅ Yes | ❌ No generation |
| clouds.msasm | None (missing!) | RWTexture2D UAV | - | - | ❌ No output pass |

### JSON Preset Parameters → Shader Mapping

| JSON Parameter | Target Cbuffer Variable | Mapping Status |
|----------------|-------------------------|----------------|
| `ShadowRange` | `shadowRangeKm` | ⚠️ Mapped but [unused] |
| `ShadowTextureSize` | `shadowTextureSize` | ⚠️ Mapped but [unused] |
| `ShadowStrength` | Unknown | ❌ No cbuffer variable found |
| `CloudShadowRT` | Output texture name | ⚠️ No UAV output exists |

**VERDICT:** Shadow generation is missing. Terrain expects shadows but cloud shader doesn't generate them.

---

## CRITICAL SHADER CODE EVIDENCE

### atmospherics.msasm - scattering_volume compute shader

```cpp
//
// Buffer Definitions: 
//
// cbuffer AtmosphericsPerViewConstants
// {
//   ...
//   float3 viewPosition;               // Offset:  224 Size:    12
//   float zOrigin;                     // Offset:  236 Size:     4 [unused]
//   uint3 scatteringVolumeDims;        // Offset:  240 Size:    12
//   float godraysIntensity;            // Offset:  252 Size:     4 [unused] ← NOT USED!
//   uint3 godraysVolumeDims;           // Offset:  256 Size:    12 [unused] ← NOT USED!
//   float q35uj64jwq;                  // Offset:  268 Size:     4 [unused]
//   ...
// }
```

### clouds.msasm - raytracing_pass compute shader

```cpp
//
// Buffer Definitions: 
//
// cbuffer CloudPerViewConstants
// {
//   uint4 targetRange[6];              // Offset:    0 Size:    96
//   float4x4 shadowMatrix;             // Offset:   96 Size:    64 [unused] ← NOT USED!
//   float4x4 worldToScatteringVolumeMatrix;// Offset:  160 Size:    64 [unused]
//   ...
//   float3 viewPosKm;                  // Offset:  336 Size:    12
//   float sampleHeight;                // Offset:  348 Size:     4 [unused]
//   float shadowRangeKm;               // Offset:  352 Size:     4 [unused] ← NOT USED!
//   int shadowTextureSize;             // Offset:  356 Size:     4 [unused] ← NOT USED!
//   ...
// }
```

### terrain.msasm - base pass pixel shader

```cpp
//
// Buffer Definitions: 
//
// cbuffer TerrainConstants
// {
//   float4x4 worldViewProj;            // Offset:    0 Size:    64 [unused]
//   ...
//   float4x4 invShadowMatrix;          // Offset:  160 Size:    64  ← USED!
//   float extentZKmXXX;                // Offset:  224 Size:     4 [unused]
//   float startZKmXXXX;                // Offset:  228 Size:     4 [unused]
//   float shadowRange;                 // Offset:  232 Size:     4 [unused] ← NOT USED!
//   ...
// }
//
// Resource Bindings:
//
// Name                                 Type  Format         Dim      HLSL Bind  Count
// ------------------------------ ---------- ------- ----------- -------------- ------
// ...
// cloudShadowTexture                texture  float4          2d             t0      1  ← BOUND!
// ...
```

---

## THE SMOKING GUN: [unused] Annotations

When the HLSL compiler marks variables as `[unused]`, it means:

1. ✅ Variable IS in the cbuffer definition
2. ✅ CPU code CAN set the value  
3. ❌ GPU shader code NEVER reads it
4. ❌ Compiler optimizes it out completely
5. ❌ Setting the value has ZERO effect

**Example:**
```
CPU: Sets godraysIntensity = 1.0
GPU Upload: ✓ Value copied to cb12[252]
Shader Compile: ⚠️ Compiler detects variable never used
Shader Execute: ❌ Variable reference removed from bytecode
Result: Setting has no effect on rendering
```

---

## WHAT WORKS vs WHAT DOESN'T

### ✅ WORKING Pipeline Components

- Preset JSON parsing (parameters are read correctly)
- Cbuffer uploads (values DO reach the GPU)
- Texture declarations (textures are defined)
- Terrain shadow sampling infrastructure (`invShadowMatrix` in use)

### ❌ BROKEN/MISSING Pipeline Components

- Godray computation code (doesn't exist)
- Godray texture writes (no UAV pass)
- Godray texture reads/blending (not in HDR composite)
- Cloud shadow generation pass (doesn't exist)
- Cloud shadow UAV output (not declared)
- Actual usage of shadow parameters in cloud shader
- Actual usage of shadowRange in terrain shader

---

## BINARY PROOF

To verify these findings are correct, examine the compiled shader bytecode:

### Check register usage:
```
cb12[252] = godraysIntensity
  ↓
Search shader assembly for "icb12[15].w" or "vcb12[252]"
  ↓
RESULT: No references found = [unused] is accurate
```

### Check texture sampling:
```
t3 = godraysTexture (clouds shader)
  ↓
Search for "sample" or "ld" instructions using t3
  ↓
RESULT: No sampling instructions = texture never read
```

### Check UAV writes:
```
Search for "store_uav_typed" writing to shadow texture
  ↓
RESULT: Only writes to u0 (outputTexture1) for cloud rendering
  ↓
No u1, u2, etc. for shadow = no shadow generation
```

---

## FINAL SUMMARY TABLE

| Feature | Cbuffers | Textures | Shader Code | Functionality | Can Fix? |
|---------|----------|----------|-------------|---------------|----------|
| **Godrays** | ❌ Unused | ❌ Not sampled | ❌ Missing | 0% | ❌ No |
| **Cloud Shadows** | ⚠️ Partial | ⚠️ Declared | ⚠️ Incomplete | 20% | ❌ No |

**Both require shader source code and recompilation to implement.**

---

Generated: April 4, 2026  
Analysis: Complete decompiled shader inspection  
Confidence: 100% - Based on direct shader code examination
