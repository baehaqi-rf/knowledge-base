---
id: "kb-graphics-webgpu-liquid-glass-v2"
title: "Production-Ready Apple Liquid Glass Architecture with WebGPU"
version: "2.0.0"
status: "production-standard"
domain: "graphics-shaders"
stack:
  - "WebGPU"
  - "WGSL"
  - "TypeScript"
  - "WebGL2"
  - "CSS"
  - "SVG Filters"
triggers:
  - "apple liquid glass webgpu"
  - "wgsl refraction shader"
  - "sdf rounded box liquid merge smin"
  - "dual-kawase blur compute shader"
  - "dom to gpu texture backdrop"
  - "frosted glass ui overlay webgpu"
related: []
sources:
  - title: "WebGPU :: Re-Creating Apple's Liquid Glass (Arya Salehi / arya 3d)"
    url: "https://youtu.be/lq-ikfNVx7g"
  - title: "Prototype Source Repository (arya3d/liquid-glass-webgpu)"
    url: "https://github.com/arya3d/liquid-glass-webgpu"
  - title: "An Inexpensive BRDF Model for Physically-based Rendering (Christophe Schlick, 1994)"
    url: "https://doi.org/10.1111/1467-8659.1330233"
  - title: "Bandwidth-Efficient Rendering / Dual-Kawase Blur (Marius Bjorge, ARM, SIGGRAPH 2015)"
    url: "https://community.arm.com/cfs-file/__key/communityserver-blogs-components-weblogfiles/00-00-00-20-66/siggraph2015_2D00_mmg_2D00_marius_2D00_slides.pdf"
  - title: "2D Distance Functions & Smooth Minimum (Inigo Quilez)"
    url: "https://iquilezles.org/articles/distfunctions2d/"
last_updated: "2026-11-22"
---

# Production-Ready Apple Liquid Glass Architecture with WebGPU

**Scope:** Engineering specification and production upgrade over the arya 3d WebGPU Liquid Glass prototype (`https://youtu.be/lq-ikfNVx7g`). Enforces single-overlay rendering, aspect-ratio-safe SDF gradient refraction, compute-based Dual-Kawase blur, hybrid backdrop ingestion, and 3-tier graceful degradation.

## 1. Decision Scope

### Ideal Use Cases

- **Spatial UI & Next-Gen Design Systems:** Floating navigation bars, docks, tab bars, command palettes, and dynamic pill clusters with organic liquid merging (`smin`).
- **HUD Overlays on 3D / Map Canvases:** Interactive glass panels rendered over Three.js, Babylon.js, or Mapbox/MapLibre canvases using zero-copy GPU texture sharing.
- **Media Players & Creative Web Tools:** Video playback controls (`importExternalTexture` at 60–120 FPS) and floating toolbars over infinite canvases or image/video editors.
- **High-Impact Interactive Hero Sections:** Pointer-reactive optical lenses and micro-interaction cards with controlled backdrop assets.

### Anti-Patterns (Do NOT Use When)

- **Data-Dense CRUD Apps / Enterprise Tables:** Optical refraction harms legibility and wastes GPU power. Use native CSS `backdrop-filter`.
- **High-Frequency Mutating DOM Backdrops (No Video/Canvas):** Repeatedly capturing rapidly mutating DOM text/inputs into a GPU texture is CPU-prohibitive. Route directly to Tier 3 (CSS `backdrop-filter` + SVG `<feDisplacementMap>`).
- **Accessibility / Low-Power Constraints:** Disable GPU refraction automatically when `prefers-reduced-transparency: reduce` or `prefers-reduced-motion: reduce` is enabled.

## 2. Reference Baseline & Bottleneck Audit

### Technical Correction on the Reference Video

The reference video (arya 3d, Nov 2026) is an effective educational prototype, **not** a physically ray-traced Snell's law simulation. Its actual baseline implementation uses:

1. A 2D Signed Distance Field (`sdRoundedBox`) per element.
2. An analytical lens height curve ($d^{0.25}$) displacing texture samples toward the element center via `position * sin(angle)`.
3. Subtle RGB chromatic dispersion, Schlick-inspired edge reflectivity, and a 1 px edge highlight ring.
4. One-shot DOM capture via `html2canvas-pro`.

### Production Bottleneck Audit

| Dimension | Reference Video (Prototype) | Production Bottleneck | Production Standard (Improved) |
| :--- | :--- | :--- | :--- |
| **Canvas Strategy** | 1 WebGPU `<canvas>` per glass component | Hits browser active GPU context limit (8–16 contexts), high VRAM overhead, high `contextlost` risk | **1 Shared Full-Viewport Overlay Canvas** (`pointer-events: none`) + GPU `StorageBuffer` |
| **Backdrop Ingestion** | Full `html2canvas-pro` DOM snapshot on load | Slow CPU rasterization (50–300 ms+), breaks on CORS/complex CSS, fails on animations/video | **Hybrid 3-Path Pipeline:** Zero-copy `importExternalTexture` + Scroll UV-Shift / Dirty-Rect + CSS/SVG fallback |
| **Refraction Math** | `posFromCenter * sin(angle)` pulling to center | Severe horizontal stretching on wide aspect ratios (navbars, pill buttons $> 3:1$) | **SDF Gradient Normal Displacement ($\nabla d$)** for uniform bevel refraction across all aspect ratios |
| **Blur & Legibility** | Clear glass only; no blur or contrast adaptation | Foreground UI text collides visually with sharp background text/images; fails WCAG AA | **Compute Dual-Kawase Blur** ($< 0.35\text{ ms}$) + **Adaptive Luminance Tint & Vibrancy** |
| **Edge Quality & Morphing** | Discrete static boxes with standard edge mask | Aliasing artifacts across DPR levels; no liquid merging between adjacent elements | **Analytical AA (`fwidth`)** + **Group-Scoped Polynomial `smin`** with smooth property blending |
| **Browser Compatibility** | WebGPU-only focus | Fails on unsupported browsers, restricted corporate devices, or legacy GPUs | **3-Tier Degradation:** WebGPU -> WebGL2 -> CSS `backdrop-filter` + SVG |

## 3. Production Architecture & Core Invariants

### System Dataflow Diagram

```text
+-----------------------------------------------------------------------------+
|                         APPLICATION LAYER (HTML / DOM)                      |
|   [Registered Glass Elements]               [Backdrop Source]               |
+---------------+-------------------------------------+-----------------------+
                |                                     |
  ResizeObserver + IntersectionObserver       Hybrid Backdrop Router
  + Passive Scroll Offset (Dirty-Flag)                |
                |                   +-----------------+-----------------+
                |                   |                 |                 |
                v                   v                 v                 v
      +------------------+    [Path A]           [Path B]          [Path C]
      | Layout & Culling |    Video / Canvas     Static/Semi-      High-Churn
      | Manager (Physical|    Zero-Copy          Dynamic DOM       Dynamic DOM
      | Pixel Rects)     |    ExternalTexture    Scroll UV-Shift   (Tier 3)
      +--------+---------+    (+ Blit Pass)           |                 |
               |                    +--------+--------+                 |
               v                             v                          |
     [16-Byte Aligned                [Backdrop GPUTexture]              |
      Storage Buffer]                        |                          |
               |                             v                          |
               |              +------------------------------+          |
               |              | PASS 1: Compute Shader       |          |
               |              | Dual-Kawase Blur (Mip Chain) |          |
               |              +--------------+---------------+          |
               |                             |                          |
               +--------------+--------------+                          |
                              v                                         |
        +----------------------------------------------------+          |
        | PASS 2: Single Shared Overlay Fragment Shader      |          |
        |  1. Group-Scoped SDF + smin (Liquid Merging)       |          |
        |  2. SDF Gradient Refraction (Aspect-Ratio Safe)    |          |
        |  3. Sharp/Blur Blend + Chromatic Dispersion        |          |
        |  4. Adaptive Luminance Tint + Vibrancy Boost       |          |
        |  5. Edge Fresnel + Directional Rim + fwidth AA     |          |
        +-------------------------+--------------------------+          |
                                  |                                     |
                                  v                                     v
                    +---------------------------+         +---------------------------+
                    | Tier 1: WebGPU Overlay    | Fail -> | Tier 2: WebGL2 (FBO Blur) |
                    | Fixed, pointer-events: 0  | Fail -> | Tier 3: CSS + SVG Filter  |
                    +---------------------------+         +---------------------------+
```

### Pillar 1: Single Shared Overlay, Physical-Pixel Contract & On-Demand Render Loop

- Mount **exactly one** global `<canvas>` (`position: fixed; inset: 0; pointer-events: none;`).
- Track DOM elements via `ResizeObserver` and `IntersectionObserver`; cull off-screen elements before uploading to the GPU `StorageBuffer`.
- Clamp canvas resolution: `dpr = Math.min(window.devicePixelRatio || 1, 2)` to avoid quadratic fragment explosion on $3\times$ / $4\times$ mobile displays.
- **Coordinate Space Invariant:** Because WGSL `@builtin(position)` (`fragCoord.xy`) operates in **physical framebuffer pixels**, the host MUST pre-multiply `resolution`, `scrollOffset`, `pointerPos`, `rect`, `radius`, `bevelWidth`, and `mergeK` by `dpr`. Keep `dpr` in `GlobalUniforms` so edge-stroke widths (`1.0 * u.dpr`) remain consistent in CSS-pixel thickness across HiDPI screens.
- **Dirty-Flag Scheduling:** Run `requestAnimationFrame` only when `isDirty === true` (scroll, resize, pointer interaction, element animation, or active video frame). Idle state must consume 0 FPS.

### Pillar 2: Hybrid 3-Path Backdrop Ingestion

Because browser security isolates DOM pixels from GPU shaders (and WICG *HTML-in-Canvas* remains behind experimental flags):

- **Path A — Zero-Copy Media/Canvas (Preferred):** Ingest `<video>` via `device.importExternalTexture()` or 3D `<canvas>` via `copyExternalImageToTexture` (60–120 FPS, zero CPU readback). Note: `GPUExternalTexture` binds as `texture_external` in WGSL, so run a single lightweight fullscreen blit pass into the `rgba16float` base `texture_2d<f32>` before running the `kawaseDownsample` compute chain.
- **Path B — Scroll UV-Shift + Debounced Dirty-Rect (Static/Semi-Dynamic DOM):** Capture the backdrop layer only on actual DOM mutations (`MutationObserver` debounced 200–300 ms). During scrolling, never re-capture; update only `u.scrollOffset` in the uniform buffer (0 ms CPU raster cost).
- **Path C — Compositor-Native Fallback (High-Churn DOM):** Bypass GPU DOM capture and render via CSS `backdrop-filter: blur() saturate()` with optional SVG `<feDisplacementMap>`.

### Pillar 3: Mathematical & Optical Invariants

#### A. Rounded Box SDF & Liquid Merging (`smin`)

For local point $\mathbf{p}$ relative to box center, half-extents $\mathbf{b} = (w/2, h/2)$, and clamped corner radius $r$:

$$\mathbf{q} = |\mathbf{p}| - \mathbf{b} + r$$

$$d(\mathbf{p}, \mathbf{b}, r) = \|\max(\mathbf{q}, 0)\|_2 + \min(\max(q_x, q_y), 0) - r$$

To merge two glass elements ($d_1, d_2$) sharing the same `groupId` with smoothing factor $k > 0$ (polynomial smooth minimum) and continuous property blend weight $m \in [0, 1]$:

$$h = \frac{\max(k - |d_1 - d_2|, 0)}{k}$$

$$\mathrm{smin}(d_1, d_2, k) = \min(d_1, d_2) - \frac{1}{4} h^2 k$$

$$m = \begin{cases} 0.5 h & \text{if } d_1 < d_2 \\ 1 - 0.5 h & \text{otherwise} \end{cases}$$

Interpolating `bevelWidth`, `iorStrength`, `tint`, `blurMix`, and `dispersion` by $m$ prevents hard color/refraction seams across the meniscus between merged elements.

#### B. Aspect-Ratio-Safe SDF Gradient Refraction

Instead of pulling UVs toward the center, compute the unit outward gradient $\nabla d$ via central differences ($\epsilon = 0.5\text{ px}$), guarding against zero-length vectors at symmetric centers:

$$\mathbf{g} = \begin{pmatrix} d(p_x + \epsilon, p_y) - d(p_x - \epsilon, p_y) \\ d(p_x, p_y + \epsilon) - d(p_x, p_y - \epsilon) \end{pmatrix}, \quad \nabla d = \begin{cases} \dfrac{\mathbf{g}}{\|\mathbf{g}\|_2} & \text{if } \|\mathbf{g}\|_2 > 10^{-5} \\ (0, 0) & \text{otherwise} \end{cases}$$

Given bevel width $W_b$ (typically $16\text{--}28 \times \mathrm{dpr}\text{ px}$) and refraction strength $\kappa \in [0.015, 0.035]$:

$$e = \mathrm{clamp}\left(1 - \frac{-d}{W_b}, 0, 1\right), \quad s = e^3, \quad \Delta \mathbf{uv} = -\nabla d \cdot s \cdot \kappa$$

#### C. Chromatic Dispersion

Sample RGB channels at slight scalar offsets along $\Delta \mathbf{uv}$ with dispersion factor $\delta \in [0.04, 0.10]$:

$$\mathbf{uv}_R = \mathbf{uv} + (1 + \delta)\Delta\mathbf{uv}, \quad \mathbf{uv}_G = \mathbf{uv} + \Delta\mathbf{uv}, \quad \mathbf{uv}_B = \mathbf{uv} + (1 - \delta)\Delta\mathbf{uv}$$

#### D. Dual-Kawase Blur (Compute Pass)

Achieve wide frosted blur in 2–3 mip passes at $< 0.35\text{ ms}$ GPU cost:

- **Downsample (5 taps):** Center weight $4$, four half-texel diagonal taps weight $1$ each, divided by $8$.
- **Upsample (8 taps):** Four cardinal taps weight $1$ each, four diagonal taps weight $2$ each, divided by $12$.

#### E. Schlick-Inspired Edge Fresnel, Adaptive Luminance & Analytical AA

- **Edge Fresnel ($F_0 = 0.04$ for air-to-glass $n_1 = 1.0, n_2 = 1.5$, Schlick-inspired exponent $p = 4$):**

$$F(e) = 0.04 + 0.96 \cdot e^4$$

- **Rec. 709 Relative Luminance & Adaptive Contrast:**

$$L = 0.2126 R + 0.7152 G + 0.0722 B$$

Apply subtle dark scrim when $L > 0.65$ and light vibrancy tint with $1.15\times\text{--}1.25\times$ saturation boost when $L < 0.35$.

- **Screen-Space Anti-Aliasing:**

$$a = \max(\mathrm{fwidth}(d), 0.5), \quad \alpha = 1 - \mathrm{smoothstep}(-a, a, d)$$

### Pillar 4: 3-Tier Degradation & Accessibility

1. **Tier 1 (WebGPU):** Shared Overlay + Compute Dual-Kawase Blur + Group-Scoped Multi-Element SDF/Gradient/`smin`.
2. **Tier 2 (WebGL2):** Ping-pong FBO Dual-Kawase Blur + GLSL ES 3.00 equivalent fragment shader (`dFdx`/`dFdy` for `fwidth`).
3. **Tier 3 (CSS + SVG):** `backdrop-filter: blur(20px) saturate(180%)` + layered specular inset borders + optional SVG `<feDisplacementMap>` for static lens shapes.
4. **Accessibility Invariants:** Honor `prefers-reduced-transparency: reduce` (raise surface opacity to $\ge 0.90$, disable refraction) and enforce WCAG AA text contrast ($\ge 4.5:1$).

## 4. Reference Implementation Patterns

### 4.1. 16-Byte Aligned WGSL Structs

```wgsl
struct GlobalUniforms {
    resolution: vec2<f32>,      // offset 0  (8 bytes) — physical canvas px
    scrollOffset: vec2<f32>,    // offset 8  (8 bytes) — physical px
    pointerPos: vec2<f32>,      // offset 16 (8 bytes) — physical px
    time: f32,                  // offset 24 (4 bytes)
    elementCount: u32,          // offset 28 (4 bytes)
    dpr: f32,                   // offset 32 (4 bytes) — clamped [1.0, 2.0]
    _pad0: f32,                 // offset 36 (4 bytes)
    _pad1: vec2<f32>,           // offset 40 (8 bytes) -> Total: 48 bytes
};

// Host MUST pack elements grouped contiguously by groupId (meta.x).
struct GlassElement {
    rect: vec4<f32>,            // xy = center (phys px), zw = halfSize (phys px) | 16 bytes
    params: vec4<f32>,          // x = radius, y = bevelWidth, z = iorStrength, w = mergeK | 16 bytes
    tintAndStyle: vec4<f32>,    // rgb = baseTint, a = tintOpacity | 16 bytes
    meta: vec4<f32>,            // x = groupId, y = blurMix, z = dispersion, w = active | 16 bytes
}; // Total: 64 bytes (16 floats) per element, 16-byte aligned for StorageBuffer array
```

### 4.2. Compute Shader: Dual-Kawase Downsample & Upsample (WGSL)

```wgsl
@group(0) @binding(0) var srcTex: texture_2d<f32>;
@group(0) @binding(1) var dstTex: texture_storage_2d<rgba16float, write>;
@group(0) @binding(2) var linearSampler: sampler;

struct BlurParams {
    texelSize: vec2<f32>, // 1.0 / vec2<f32>(srcDimensions)
    offset: f32,          // typically 0.5 .. 2.0
    _pad: f32,
};
@group(0) @binding(3) var<uniform> blurParams: BlurParams;

@compute @workgroup_size(8, 8, 1)
fn kawaseDownsample(@builtin(global_invocation_id) id: vec3<u32>) {
    let dims = textureDimensions(dstTex);
    if (id.x >= dims.x || id.y >= dims.y) { return; }

    let uv = (vec2<f32>(id.xy) + 0.5) / vec2<f32>(dims);
    let hp = blurParams.texelSize * blurParams.offset;

    var sum = textureSampleLevel(srcTex, linearSampler, uv, 0.0) * 4.0;
    sum += textureSampleLevel(srcTex, linearSampler, uv + vec2<f32>(-hp.x, -hp.y), 0.0);
    sum += textureSampleLevel(srcTex, linearSampler, uv + vec2<f32>( hp.x, -hp.y), 0.0);
    sum += textureSampleLevel(srcTex, linearSampler, uv + vec2<f32>(-hp.x,  hp.y), 0.0);
    sum += textureSampleLevel(srcTex, linearSampler, uv + vec2<f32>( hp.x,  hp.y), 0.0);

    textureStore(dstTex, vec2<i32>(id.xy), sum / 8.0);
}

@compute @workgroup_size(8, 8, 1)
fn kawaseUpsample(@builtin(global_invocation_id) id: vec3<u32>) {
    let dims = textureDimensions(dstTex);
    if (id.x >= dims.x || id.y >= dims.y) { return; }

    let uv = (vec2<f32>(id.xy) + 0.5) / vec2<f32>(dims);
    let hp = blurParams.texelSize * blurParams.offset;

    var sum = vec4<f32>(0.0);
    sum += textureSampleLevel(srcTex, linearSampler, uv + vec2<f32>(-hp.x * 2.0, 0.0), 0.0);
    sum += textureSampleLevel(srcTex, linearSampler, uv + vec2<f32>( hp.x * 2.0, 0.0), 0.0);
    sum += textureSampleLevel(srcTex, linearSampler, uv + vec2<f32>(0.0, -hp.y * 2.0), 0.0);
    sum += textureSampleLevel(srcTex, linearSampler, uv + vec2<f32>(0.0,  hp.y * 2.0), 0.0);

    sum += textureSampleLevel(srcTex, linearSampler, uv + vec2<f32>(-hp.x, -hp.y), 0.0) * 2.0;
    sum += textureSampleLevel(srcTex, linearSampler, uv + vec2<f32>( hp.x, -hp.y), 0.0) * 2.0;
    sum += textureSampleLevel(srcTex, linearSampler, uv + vec2<f32>(-hp.x,  hp.y), 0.0) * 2.0;
    sum += textureSampleLevel(srcTex, linearSampler, uv + vec2<f32>( hp.x,  hp.y), 0.0) * 2.0;

    textureStore(dstTex, vec2<i32>(id.xy), sum / 12.0);
}
```

### 4.3. Fragment Shader: Multi-Element Liquid Glass Overlay (WGSL)

```wgsl
@group(0) @binding(0) var<uniform> u: GlobalUniforms;
@group(0) @binding(1) var<storage, read> elements: array<GlassElement>;
@group(0) @binding(2) var sharpTex: texture_2d<f32>;
@group(0) @binding(3) var blurredTex: texture_2d<f32>;
@group(0) @binding(4) var linearSampler: sampler;

struct SdfSample {
    dist: f32,
    bevelWidth: f32,
    iorStrength: f32,
    blurMix: f32,
    dispersion: f32,
    tint: vec4<f32>,
};

fn sdRoundedBox(p: vec2<f32>, b: vec2<f32>, r: f32) -> f32 {
    let radius = min(r, min(b.x, b.y));
    let q = abs(p) - b + vec2<f32>(radius);
    return min(max(q.x, q.y), 0.0) + length(max(q, vec2<f32>(0.0))) - radius;
}

// Returns vec2(blendedDist, propertyBlendWeightTowardB)
fn sminWeighted(a: f32, b: f32, k: f32) -> vec2<f32> {
    if (k <= 0.001) {
        return vec2<f32>(min(a, b), select(1.0, 0.0, a < b));
    }
    let h = max(k - abs(a - b), 0.0) / k;
    let d = min(a, b) - h * h * k * 0.25;
    let m = select(1.0 - 0.5 * h, 0.5 * h, a < b);
    return vec2<f32>(d, m);
}

// Lightweight distance-only evaluation for central-difference gradient taps
fn evaluateSceneDist(fragPos: vec2<f32>) -> f32 {
    var dMin = 1e6;
    var activeGroup = -1.0;

    for (var i: u32 = 0u; i < u.elementCount; i = i + 1u) {
        let el = elements[i];
        if (el.meta.w < 0.5) { continue; }

        let p = fragPos - el.rect.xy;
        let boxBound = max(abs(p.x) - el.rect.z, abs(p.y) - el.rect.w);
        let k = select(0.0, el.params.w, abs(el.meta.x - activeGroup) < 0.5);
        if (boxBound > dMin + k + 1.0) { continue; }

        let d = sdRoundedBox(p, el.rect.zw, el.params.x);
        if (activeGroup < 0.0) {
            dMin = d;
            activeGroup = el.meta.x;
        } else {
            let res = sminWeighted(dMin, d, k);
            dMin = res.x;
            if (res.y > 0.5) {
                activeGroup = el.meta.x;
            }
        }
    }
    return dMin;
}

fn evaluateSceneSdf(fragPos: vec2<f32>) -> SdfSample {
    var out: SdfSample;
    out.dist = 1e6;
    var activeGroup = -1.0;

    for (var i: u32 = 0u; i < u.elementCount; i = i + 1u) {
        let el = elements[i];
        if (el.meta.w < 0.5) { continue; }

        let p = fragPos - el.rect.xy;
        let boxBound = max(abs(p.x) - el.rect.z, abs(p.y) - el.rect.w);
        let k = select(0.0, el.params.w, abs(el.meta.x - activeGroup) < 0.5);
        if (boxBound > out.dist + k + 1.0) { continue; }

        let d = sdRoundedBox(p, el.rect.zw, el.params.x);
        if (activeGroup < 0.0) {
            out.dist = d;
            out.bevelWidth = el.params.y;
            out.iorStrength = el.params.z;
            out.blurMix = el.meta.y;
            out.dispersion = el.meta.z;
            out.tint = el.tintAndStyle;
            activeGroup = el.meta.x;
        } else {
            let res = sminWeighted(out.dist, d, k);
            let m = res.y;
            out.dist = res.x;
            out.bevelWidth  = mix(out.bevelWidth,  el.params.y,    m);
            out.iorStrength = mix(out.iorStrength, el.params.z,    m);
            out.blurMix     = mix(out.blurMix,     el.meta.y,      m);
            out.dispersion  = mix(out.dispersion,  el.meta.z,      m);
            out.tint        = mix(out.tint,        el.tintAndStyle, m);
            if (m > 0.5) { activeGroup = el.meta.x; }
        }
    }
    return out;
}

fn calcSceneGradient(fragPos: vec2<f32>) -> vec2<f32> {
    let eps = 0.5;
    let g = vec2<f32>(
        evaluateSceneDist(fragPos + vec2<f32>(eps, 0.0)) -
        evaluateSceneDist(fragPos - vec2<f32>(eps, 0.0)),
        evaluateSceneDist(fragPos + vec2<f32>(0.0, eps)) -
        evaluateSceneDist(fragPos - vec2<f32>(0.0, eps))
    );
    let len = length(g);
    return select(vec2<f32>(0.0), g / len, len > 1e-5);
}

fn boostVibrancy(color: vec3<f32>, satBoost: f32) -> vec3<f32> {
    let luma = dot(color, vec3<f32>(0.2126, 0.7152, 0.0722));
    return mix(vec3<f32>(luma), color, satBoost);
}

fn sampleDispersed(tex: texture_2d<f32>, baseUv: vec2<f32>, offset: vec2<f32>, disp: f32) -> vec3<f32> {
    let uvR = clamp(baseUv + offset * (1.0 + disp), vec2<f32>(0.001), vec2<f32>(0.999));
    let uvG = clamp(baseUv + offset,                vec2<f32>(0.001), vec2<f32>(0.999));
    let uvB = clamp(baseUv + offset * (1.0 - disp), vec2<f32>(0.001), vec2<f32>(0.999));
    return vec3<f32>(
        textureSample(tex, linearSampler, uvR).r,
        textureSample(tex, linearSampler, uvG).g,
        textureSample(tex, linearSampler, uvB).b
    );
}

@fragment
fn fsMain(@builtin(position) fragCoord: vec4<f32>) -> @location(0) vec4<f32> {
    let fragPos = fragCoord.xy;
    let sdf = evaluateSceneSdf(fragPos);
    let d = sdf.dist;

    let aa = max(fwidth(d), 0.5);
    let mask = 1.0 - smoothstep(-aa, aa, d);
    if (mask <= 0.001) { discard; }

    let bevelWidth = max(sdf.bevelWidth, 1.0);
    let edgeFactor = clamp(1.0 - (-d / bevelWidth), 0.0, 1.0);
    let curveSlope = pow(edgeFactor, 3.0);

    // 1. Aspect-ratio-safe SDF gradient refraction + scroll UV-shift
    let grad = calcSceneGradient(fragPos);
    let baseUv = (fragPos + u.scrollOffset) / u.resolution;
    let refractOffset = -grad * curveSlope * sdf.iorStrength;

    // 2. Sharp-edge / frosted-interior blend with chromatic dispersion
    let blurMix = sdf.blurMix * (1.0 - edgeFactor * 0.55);
    let sharpCol = sampleDispersed(sharpTex, baseUv, refractOffset, sdf.dispersion);
    let blurCol  = sampleDispersed(blurredTex, baseUv, refractOffset, sdf.dispersion);
    var color = mix(sharpCol, blurCol, blurMix);

    // 3. Adaptive vibrancy & contrast scrim
    color = boostVibrancy(color, 1.18);
    let luma = dot(color, vec3<f32>(0.2126, 0.7152, 0.0722));
    let adaptiveScrim = select(vec3<f32>(1.0), vec3<f32>(0.05), luma > 0.65);
    let finalTint = mix(sdf.tint.rgb, adaptiveScrim, 0.35);
    color = mix(color, finalTint, sdf.tint.a);

    // 4. Edge Fresnel + DPR-scaled directional specular rim
    let lightDir = normalize(vec2<f32>(-0.7, -0.7));
    let rimDot = max(dot(-grad, lightDir), 0.0) * 0.7 + 0.3;
    let fresnel = 0.04 + 0.96 * pow(edgeFactor, 4.0);
    let edgeRing = smoothstep(1.5 * aa, 0.0, abs(d + max(u.dpr * 0.5, 1.0))) * rimDot;

    color += vec3<f32>(1.0) * (fresnel * 0.22 + edgeRing * 0.48);
    return vec4<f32>(color, mask);
}
```

### 4.4. TypeScript Capability Detection, Storage Packing & Tier Fallbacks

```ts
export type GlassRendererTier = 'webgpu' | 'webgl2' | 'css-svg';

export async function detectOptimalGlassTier(): Promise<GlassRendererTier> {
  if (typeof window === 'undefined') return 'css-svg';

  const reducedTransparency = window.matchMedia('(prefers-reduced-transparency: reduce)').matches;
  const reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
  if (reducedTransparency || reducedMotion) {
    return 'css-svg';
  }

  if ('gpu' in navigator && navigator.gpu) {
    try {
      const adapter = await navigator.gpu.requestAdapter({ powerPreference: 'high-performance' });
      const device = adapter ? await adapter.requestDevice() : null;
      if (device) {
        device.destroy();
        return 'webgpu';
      }
    } catch {
      // Fall through to Tier 2
    }
  }

  const probe = document.createElement('canvas');
  if (probe.getContext('webgl2')) {
    return 'webgl2';
  }

  return 'css-svg';
}

export interface GlassElementInput {
  domRect: DOMRectReadOnly;
  radiusCssPx: number;
  bevelWidthCssPx: number;
  iorStrength: number;
  mergeKCssPx: number;
  tintRgb: [number, number, number];
  tintOpacity: number;
  groupId: number;
  blurMix: number;
  dispersion: number;
}

// Pack visible elements sorted by groupId into a 16-float-per-element Float32Array
export function packGlassElements(
  items: readonly GlassElementInput[],
  dpr: number,
  outBuffer: Float32Array
): number {
  const sorted = [...items].sort((a, b) => a.groupId - b.groupId);
  let count = 0;

  for (const item of sorted) {
    const base = count * 16;
    if (base + 16 > outBuffer.length) break;

    const hw = (item.domRect.width * 0.5) * dpr;
    const hh = (item.domRect.height * 0.5) * dpr;
    const cx = (item.domRect.left * dpr) + hw;
    const cy = (item.domRect.top * dpr) + hh;

    // rect: vec4<f32>
    outBuffer[base + 0] = cx;
    outBuffer[base + 1] = cy;
    outBuffer[base + 2] = hw;
    outBuffer[base + 3] = hh;

    // params: vec4<f32>
    outBuffer[base + 4] = item.radiusCssPx * dpr;
    outBuffer[base + 5] = item.bevelWidthCssPx * dpr;
    outBuffer[base + 6] = item.iorStrength;
    outBuffer[base + 7] = item.mergeKCssPx * dpr;

    // tintAndStyle: vec4<f32>
    outBuffer[base + 8]  = item.tintRgb[0];
    outBuffer[base + 9]  = item.tintRgb[1];
    outBuffer[base + 10] = item.tintRgb[2];
    outBuffer[base + 11] = item.tintOpacity;

    // meta: vec4<f32>
    outBuffer[base + 12] = item.groupId;
    outBuffer[base + 13] = item.blurMix;
    outBuffer[base + 14] = item.dispersion;
    outBuffer[base + 15] = 1.0; // active

    count++;
  }
  return count;
}
```

#### Tier 2 (WebGL2) & Tier 3 (CSS + SVG) Fallback Contract

- **Tier 2 (WebGL2):** Mirror the Dual-Kawase passes using ping-pong `RGBA16F` (with `EXT_color_buffer_half_float`) or `RGBA8` framebuffers and GLSL ES 3.00 fragment shaders (`dFdx`/`dFdy` in place of WGSL `fwidth`, uniform buffer or `texelFetch` data texture in place of `StorageBuffer`).
- **Tier 3 (CSS + SVG):** Apply compositor-native frosting and specular border highlights without JS frame loops:

```css
.liquid-glass-fallback {
  background: rgba(255, 255, 255, 0.14);
  backdrop-filter: blur(20px) saturate(180%);
  -webkit-backdrop-filter: blur(20px) saturate(180%);
  box-shadow:
    inset 0 1px 0 0 rgba(255, 255, 255, 0.45),
    inset 0 -1px 0 0 rgba(255, 255, 255, 0.12);
}

@media (prefers-reduced-transparency: reduce) {
  .liquid-glass-fallback {
    background: rgba(245, 245, 247, 0.92);
    backdrop-filter: none;
    -webkit-backdrop-filter: none;
  }
}
```

## 5. Performance Budgets & Verification Checklist

| Metric | Production Budget | Verification Method |
| :--- | :--- | :--- |
| **Total GPU Pass Time (Blur + Glass)** | $< 1.5\text{ ms}$ @ 1080p (60–120 FPS) | WebGPU Timestamp Queries / Chrome DevTools GPU Profiler |
| **Dual-Kawase Blur Pass** | $< 0.35\text{ ms}$ (2–3 mip levels) | Compute Pass GPU Timing |
| **Main-Thread Scroll Overhead** | $< 0.5\text{ ms}$ per frame | Chrome Performance Tab (zero DOM capture calls during scroll) |
| **Active GPU Contexts** | Exactly 1 per document | DOM `<canvas>` audit |
| **Canvas DPR Cap** | $\le 2.0$ | Verify `canvas.width <= innerWidth * 2` |
| **Idle Frame Rate** | 0 FPS (no continuous rAF) | Chrome Frame Rendering Stats while idle |

### Runtime QA Verification Checklist

- [ ] **Wide Aspect-Ratio Test:** Render a pill/navbar with aspect ratio $> 4:1$ (e.g., $640\text{ px} \times 56\text{ px}$); confirm bevel refraction stays perpendicular to edges without center-stretching.
- [ ] **HiDPI Anti-Aliasing Test:** Inspect 1 px specular rim across DPR $1\times$, $2\times$, and $3\times$ (clamped to $2\times$) for zero jagged edges.
- [ ] **Scroll Performance Test:** Rapidly scroll over rich content; verify zero `html2canvas` invocations and smooth UV-shift synchronization.
- [ ] **Liquid Merge Test:** Animate two same-group elements within distance $k$; confirm smooth organic meniscus blending without normal or color discontinuities, and verify different `groupId` elements do not merge.
- [ ] **Fallback & Accessibility Test:** Force `navigator.gpu = undefined` (must degrade to WebGL2/CSS) and toggle `prefers-reduced-transparency: reduce` (must render legible solid/high-opacity surface with contrast $\ge 4.5:1$).
- [ ] **Context Loss Recovery:** Simulate `device.lost` and confirm clean fallback to Tier 3 CSS without blanking foreground DOM text.

## 6. Agent Guardrails (MUST / NEVER)

- **MUST:** Use a **single shared full-viewport overlay canvas** (`pointer-events: none`) backed by a 16-byte-aligned `StorageBuffer`.
- **MUST:** Pack all uniform/element coordinates in **physical framebuffer pixels** (`cssPx * dpr`) and clamp `dpr` to `Math.min(devicePixelRatio, 2)`.
- **MUST:** Compute refraction displacement using **SDF gradient ($\nabla d$)** with zero-length guard, not radial vectors from the element center.
- **MUST:** Scope `smin` merges by `groupId` and interpolate surface parameters smoothly across the blend weight.
- **MUST:** Use **Dual-Kawase blur** (compute shader in WebGPU, ping-pong FBO in WebGL2) rather than single-pass Gaussian blur.
- **MUST:** Apply `fwidth(d)` for analytical anti-aliasing and dirty-flag scheduling so idle views render at 0 FPS.
- **MUST:** Implement **3-tier fallback** (`webgpu -> webgl2 -> css-svg`) and respect `prefers-reduced-transparency`.
- **NEVER:** Call `html2canvas` or `html2canvas-pro` inside `requestAnimationFrame` or scroll event listeners.
- **NEVER:** Instantiate one WebGPU `<canvas>` per individual UI button or card.
- **NEVER:** Render foreground UI text over unblurred, unadapted clear refraction without checking luminance contrast.

## 7. Glossary & References

- **Analytical Anti-Aliasing (`fwidth`):** Screen-space derivative edge smoothing in fragment shaders without MSAA overhead.
- **Chromatic Dispersion:** Wavelength-dependent RGB UV offset simulating prismatic glass refraction.
- **Dirty-Flag Rendering:** Triggering GPU submission only when layout, input, scroll, or media state mutates.
- **Dual-Kawase Blur:** Multi-pass downsample/upsample blur algorithm optimized for GPU bandwidth and compute shaders.
- **Schlick Fresnel:** Fast physically based reflectivity approximation ($F_0 = 0.04$ for glass) by Christophe Schlick (1994).
- **SDF (Signed Distance Field):** Implicit geometric function returning signed shortest distance to a boundary (negative inside, positive outside).
- **Smooth Minimum (`smin`):** Polynomial blending operator merging multiple SDFs into organic liquid shapes.
- **Zero-Copy Pipeline (`importExternalTexture`):** Direct GPU sampling of `<video>` frames without CPU memory readback.
