---
id: "kb-design-ios-liquid-glass-cross-platform-v1"
title: "Cross-Platform iOS Liquid Glass Design System & Architecture (Android & Web)"
version: "1.0.0"
status: "production-standard"
domain: "design-systems"
stack:
  - "iOS HIG (26/27)"
  - "Android (API 29-35+)"
  - "Expo / React Native"
  - "Skia / AGSL / RenderEffect"
  - "React / Next.js"
  - "Tailwind CSS"
  - "WebGPU / WebGL2 / CSS"
triggers:
  - "ios liquid glass ui design system"
  - "liquid glass android react native expo"
  - "liquid glass navbar tab bar dropdown card"
  - "concentric border radius design tokens"
  - "cross-platform glassmorphism architecture"
  - "fluid morphing dropdown mobile web"
related:
  - "kb-graphics-webgpu-liquid-glass-v2"
sources:
  - title: "Apple Human Interface Guidelines - Materials & Liquid Glass (iOS 26/27)"
    url: "https://developer.apple.com/design/human-interface-guidelines/materials"
  - title: "Production-Ready Apple Liquid Glass Architecture with WebGPU (Internal KB)"
    url: "../graphics-shaders/webgpu-liquid-glass.md"
  - title: "Android Graphics Shading Language (AGSL) & RenderEffect Documentation"
    url: "https://developer.android.com/develop/ui/views/graphics/agsl"
  - title: "React Native Skia - Shaders & Backdrop Filters"
    url: "https://shopify.github.io/react-native-skia/"
last_updated: "2026-11-22"
---

# Cross-Platform iOS Liquid Glass Design System & Architecture (Android & Web)

**Scope:** Production design system, component governance, shared token contract, and cross-platform rendering architecture for bringing iOS 26/27 **Liquid Glass** aesthetics and physics to **Android Mobile** (Expo / React Native & Native Android) and **Web** (React / Next.js / Tailwind / WebGPU) without sacrificing legibility, battery life, or frame rates.

## 1. Core Philosophy: The 3C Principles & Decision Scope

Liquid Glass is **not** generic CSS glassmorphism. It is a layered optical system governed by three non-negotiable principles:

- **Content-First:** Glass belongs exclusively to the **navigation, control, and transient overlay layer**. Scrollable content remains opaque, flat, and high-contrast.
- **Concentric Geometry:** Every nested element shares the mathematical curvature of its parent container or device screen bezel ($R_{\text{inner}} = \max(R_{\text{outer}} - P, 4)$).
- **Cohesive Fluidity:** Triggers and menus do not abruptly appear or fade in isolation; they **morph** from the same optical surface using spring physics and liquid blending.

### Ideal Use Cases

- **Floating Navigation Shells:** Floating bottom tab bars, detached circular search orbs, top action toolbars, and scroll-collapsing navbars.
- **Anchored Transient Overlays:** Morphing dropdowns, inline context menus, popovers, and translucent modal bottom sheets that preserve parent spatial context.
- **Media & Map HUDs:** Floating playback pills, map status cards, and camera/editor controls over rich visual backdrops (maps, photos, video, 3D scenes).
- **On-Interaction Controls:** Primary tinted FABs, slider thumbs, and segmented control selection lenses that gain refractive depth when touched or dragged.

### Anti-Patterns (Do NOT Use When)

- **Standard Content Cards & List Items:** NEVER apply real-time blur or optical refraction to repeating feed cards, transaction rows, or dashboard tables. It destroys WCAG contrast and triggers severe GPU overdraw (especially on Android `FlatList` / `RecyclerView`).
- **Glass-on-Glass Stacking:** NEVER stack two active real-time blur/refraction layers directly over one another without dimming or collapsing the lower layer.
- **Saturated Tints on Every Control:** Reserve colored glass tinting strictly for the **single primary CTA** per viewport. Secondary controls must remain neutral (`regular-glass`).

## 2. iOS 26/27 Production Benchmark Matrix

Synthesized from leading first-party and third-party iOS apps to define target UX behavior across platforms:

| Reference App | Category | Primary Glass Components | Key UX & Interaction Pattern to Replicate |
| :--- | :--- | :--- | :--- |
| **Apple Music, Safari, Photos** | First-Party Baseline | Floating Tab Bar, Bottom Accessory, Detached Search | Tab bar collapses (minimize-on-scroll) into a compact pill when scrolling down; mini-player floats as an optically paired bottom accessory; search splits into a right-thumb circular orb. |
| **Things 3** | Productivity | Tinted Primary FAB, Morphing Dropdown Menu | Primary `+` button uses tinted glass that subtly deforms on drag; toolbar triggers glow on press and fluidly **morph** directly into anchored dropdown panels. |
| **Todoist** | Task Manager | Floating Tab Bar, Sliding Selection Lens, Expanding Search | A refractive glass selection capsule slides smoothly between active tabs; search orb expands horizontally with spring physics; task list rows remain **100% opaque** while scrolling edge-to-edge underneath the bar. |
| **Flighty & Slopes** | Travel / Maps | Clear Glass HUD Pills, Map Controls | Uses high-transparency **Clear Glass** strictly over rich visual map/satellite canvas layers, never over dense body text. |
| **Gentler Streak & Mercury Weather** | Fitness / Weather | Adaptive Toolbars, Concentric Widgets, Bouncy Controls | Dynamically shifts tint and foreground ink between light and dark based on live backdrop luminance; applies playful bounce selectively to fitness interactions. |
| **OmniFocus 4, Overcast, Photoroom** | Pro Utility / Media | Inline Popovers, Glass Sheets, Tactile Sliders | Context menus anchor tightly to toolbar icons with uniform glyph alignment; slider thumbs transform into refractive droplets only while actively dragged. |

## 3. Four-Layer Visual Hierarchy & Component Governance

### System Layer Stack

```text
+=============================================================================+
| LAYER 4: DYNAMIC CONTEXT & ACCESSIBILITY GOVERNANCE                         |
|   Live Luminance Adaptation | Contrast Enforcement (>= 4.5:1)               |
|   Reduce Transparency / Low-RAM / Reduced Motion Overrides                  |
+=============================================================================+
| LAYER 3: OPTICAL PHYSICS & FLUID MOTION ENGINE                              |
|   Edge Refraction | Specular Rim Highlight | Chromatic Dispersion (Tier 1)  |
|   Spring Morphing (Trigger -> Menu) | Touch Deformation | Sliding Lens      |
+=============================================================================+
| LAYER 2: FLOATING NAVIGATION, CONTROLS & TRANSIENT OVERLAYS [GLASS ALLOWED] |
|   Floating Tab Bar | Top Toolbar Clusters | Detached Search Orb             |
|   Morphing Dropdowns | Modal Glass Sheets | Map/Media HUDs | Primary FAB    |
+=============================================================================+
| LAYER 1: CONTENT BASE LAYER                               [GLASS FORBIDDEN] |
|   Scrollable Lists | Content Cards | Data Tables | Forms | Articles | Media |
|   (Opaque elevated surfaces with concentric radii scrolling edge-to-edge)   |
+=============================================================================+
```

### Component-by-Component Decision Matrix

| Component | Layer | Glass Verdict | Visual & Behavioral Contract | Android & Web Implementation Note |
| :--- | :--- | :--- | :--- | :--- |
| **Floating Tab Bar** | Layer 2 | **REQUIRED** (`regular-glass`) | Detached floating pill, concentric margin, sliding selection lens, optional collapse-on-scroll, separate search orb. | Anchor above Android system navigation bar (`safe-area-inset-bottom`); avoid overlapping system gesture pill. |
| **Navbar / Top Toolbar** | Layer 2 | **REQUIRED** (`regular-glass`) | Grouped icon clusters (do not mix raw text and icons inside one pill); apply soft **scroll-edge fade** behind the bar. | Use masked gradient fade (`mask-image` on Web, `LinearGradient` scrim on Android) on scrolled content beneath the bar. |
| **Dropdown / Context Menu** | Layer 2 + 3 | **REQUIRED + MORPH** | Anchored popover that morphs smoothly out of its trigger button; uniform icon + label rows; dismiss on outside tap. | Web: Framer Motion `layoutId` or WebGPU `smin`. Android: Reanimated shared layout + Skia/`RenderEffect` backdrop. |
| **Standard Content Card** | Layer 1 | **FORBIDDEN** | Opaque elevated surface (`solid-elevated-surface`) with concentric corner radius and crisp hairline border. | Keeps scroll at 60–120 FPS on mid-range Android devices and guarantees WCAG AA readability. |
| **HUD / Overlay Card** | Layer 2 | **ALLOWED** (`clear-glass`) | Permitted **only** when floating over full-bleed maps, photos, camera feeds, or video canvases. | Pair with adaptive text shadow or dark luminance scrim (when $L > 0.65$). |
| **Modal / Bottom Sheet** | Layer 2 | **RECOMMENDED** | Translucent blurred sheet keeping parent context visible; dims underlying Layer 2 glass bars to avoid glass-on-glass. | Clamp sheet backdrop blur radius on Android to avoid thermal throttling during drag gestures. |
| **Interactive Controls / FAB** | Layer 2 + 3 | **ON-INTERACTION** | Primary FAB uses `tinted-primary-glass`; toggles and sliders stay crisp at rest and activate refractive glass on press/drag. | Drive touch deformation via UI-thread animations (Reanimated worklets on Android, GPU transforms/uniforms on Web). |

## 4. Shared Cross-Platform Design Tokens

Both Android and Web clients MUST consume the same semantic token contract so visual hierarchy and geometry remain identical across platforms.

### 4.1. Concentric Corner Radius Formula

Nested elements must compute their radius mathematically from their parent container radius $R_{\text{outer}}$ and uniform padding $P$:

$$R_{\text{inner}} = \max(R_{\text{outer}} - P, 4)$$

Conversely, when wrapping an inner element of radius $R_{\text{inner}}$ with padding $P$:

$$R_{\text{outer}} = R_{\text{inner}} + P$$

*Example:* A floating pill container with $R_{\text{outer}} = 28\text{ dp}$ and internal padding $P = 6\text{ dp}$ MUST give its internal active tab capsule $R_{\text{inner}} = 22\text{ dp}$.

### 4.2. Adaptive Luminance & Contrast Invariant

Compute background relative luminance $L$ (Rec. 709):

$$L = 0.2126 R + 0.7152 G + 0.0722 B$$

- **Light Backdrop ($L > 0.65$):** Apply cool light glass fill (`rgba(255, 255, 255, 0.58)`), dark ink foreground (`#0F172A`), and subtle outer contrast stroke (`rgba(15, 23, 42, 0.08)`).
- **Dark Backdrop ($L < 0.35$):** Apply translucent dark glass fill (`rgba(24, 24, 27, 0.52)`), bright ink foreground (`#F8FAFC`), and luminous top-left specular rim (`rgba(255, 255, 255, 0.28)`).
- **Mid-Tone Backdrop ($0.35 \le L \le 0.65$):** Apply neutral adaptive scrim (`regularGlass.lightFill` / `darkFill` matching system theme) and boost contrast stroke before rendering foreground glyphs.
- **Hard Contrast Floor:** Foreground text and icons on glass MUST maintain a contrast ratio $\ge 4.5:1$ (normal text) and $\ge 3.0:1$ (large text / UI icons).

### 4.3. Canonical Token Specification (TypeScript Contract)

```ts
export const LIQUID_GLASS_TOKENS = {
  materials: {
    regularGlass: {
      blurRadiusPx: 20,
      saturation: 1.8,
      lightFill: 'rgba(255, 255, 255, 0.58)',
      darkFill: 'rgba(28, 28, 30, 0.54)',
      specularRimLight: 'rgba(255, 255, 255, 0.72)',
      specularRimDark: 'rgba(255, 255, 255, 0.24)',
      refractionStrength: 0.022,
      bevelWidthPx: 20,
    },
    clearGlass: {
      // Strictly for Map / Video / Full-bleed Photo HUDs
      blurRadiusPx: 10,
      saturation: 1.4,
      lightFill: 'rgba(255, 255, 255, 0.26)',
      darkFill: 'rgba(18, 18, 20, 0.30)',
      specularRimLight: 'rgba(255, 255, 255, 0.80)',
      specularRimDark: 'rgba(255, 255, 255, 0.32)',
      refractionStrength: 0.028,
      bevelWidthPx: 24,
    },
    solidElevatedCard: {
      // Layer 1 Content Cards (NO GLASS)
      lightFill: '#FFFFFF',
      darkFill: '#1C1C1E',
      lightBorder: 'rgba(15, 23, 42, 0.08)',
      darkBorder: 'rgba(255, 255, 255, 0.10)',
    },
    reducedTransparencyFallback: {
      lightFill: 'rgba(248, 250, 252, 0.94)',
      darkFill: 'rgba(24, 24, 27, 0.94)',
    },
  },
  radii: {
    floatingPillOuter: 28,
    floatingPillPadding: 6,
    floatingPillInner: 22, // 28 - 6
    dropdownOuter: 24,
    dropdownPadding: 8,
    dropdownItemInner: 16, // 24 - 8
    contentCard: 20,
    sheetTop: 32,
  },
  springs: {
    morph: { stiffness: 380, damping: 32, mass: 0.9 },        // Button -> Dropdown / Sheet
    lensSlide: { stiffness: 420, damping: 34, mass: 0.8 },    // Tab Bar selection pill
    tactilePress: { stiffness: 520, damping: 28, mass: 0.7 }, // FAB / Slider touch
  },
} as const;
```

## 5. Cross-Platform Rendering Architecture (Android & Web)

### Unified Multi-Platform Pipeline

```text
                     +------------------------------------------+
                     |   SHARED DESIGN TOKENS & STATE CONTRACT  |
                     | (Concentric Radii, Layer Rules, Springs) |
                     +--------------------+---------------------+
                                          |
                 +------------------------+------------------------+
                 |                                                 |
                 v                                                 v
+----------------------------------------+    +----------------------------------------+
|        ANDROID MOBILE PIPELINE         |    |              WEB PIPELINE              |
|   (Expo / React Native & Native API)   |    |    (React / Next.js / Tailwind CSS)    |
+----------------------------------------+    +----------------------------------------+
| [Tier 1] Android 13+ (API 33+)         |    | [Tier 1] WebGPU Shared Overlay         |
|  - Skia RuntimeEffect (SkSL) or AGSL   |    |  - Single fixed overlay + StorageBuf   |
|  - BackdropFilter blur + SDF rim       |    |  - Compute Dual-Kawase + SDF Gradient  |
|  - Reanimated UI-thread spring morph   |    |  - Liquid merging via smin()           |
+----------------------------------------+    +----------------------------------------+
| [Tier 2] Android 12 (API 31-32)        |    | [Tier 2] WebGL2 Fallback               |
|  - Hardware RenderEffect blur          |    |  - Ping-pong FBO Dual-Kawase blur      |
|  - Expo Blur (Dimezis) + Gradient rim  |    |  - GLSL ES 3.00 SDF refraction         |
+----------------------------------------+    +----------------------------------------+
| [Tier 3] Android <= 11 / Low-RAM       |    | [Tier 3] CSS + SVG Compositor Fallback |
|  - High-opacity translucent scrim      |    |  - backdrop-filter: blur() saturate()  |
|  - Specular hairline border (0ms blur) |    |  - Inset specular shadows + SVG filter |
+----------------------------------------+    +----------------------------------------+
```

### 5.1. Android Mobile Architecture (Expo / React Native & Native Android)

Android has wider hardware fragmentation than iOS. Real-time blur and custom shaders MUST use strict API-level tiering and UI-thread execution:

#### Capability Tiering on Android

- **Tier 1 — High-End Modern Android (`API >= 33` / Android 13+, Non-Low-RAM):**
  - **Expo / React Native:** Use `@shopify/react-native-skia` (`<Canvas>`, `<BackdropFilter>` / `<Blur>`, and custom **SkSL `RuntimeEffect`**) for true edge refraction, specular rim highlights, and SDF smooth-min morphing on floating navbars, tab bars, and HUDs.
  - **Native Android (Jetpack Compose / Views):** Use **AGSL `RuntimeShader`** paired with `RenderEffect.createRuntimeShaderEffect()` and `RenderEffect.createBlurEffect()`.
  - **Motion:** Drive all morphing, touch deformation, and selection-lens sliding on the native UI thread via `react-native-reanimated` shared values (or Compose `animateDpAsState` / `spring`).
- **Tier 2 — Standard Android (`API 31–32` / Android 12):**
  - Native hardware blur is supported via `RenderEffect`, but `RuntimeShader` (AGSL) is unavailable.
  - Use hardware-accelerated blur (`expo-blur` with `experimentalBlurMethod="dimezisBlurView"` on Android or native `RenderEffect` blur) layered with a directional `LinearGradient` border (top-left bright highlight -> bottom-right subtle shadow) to emulate Fresnel rim reflection cleanly.
- **Tier 3 — Legacy / Low-RAM Android (`API <= 30` or `ActivityManager.isLowRamDevice`):**
  - **Disable real-time backdrop blur entirely** (software RenderScript blur causes severe frame drops and memory spikes).
  - Render a translucent high-opacity surface (`0.88–0.94` alpha) with crisp directional specular hairline borders and subtle ambient elevation.

#### Android-Specific Engineering Invariants

- **Single-Region Blur Budget:** Never exceed **$1\text{--}2$ simultaneous real-time blurred regions** on screen at once (e.g., 1 floating tab bar + 1 active dropdown/top toolbar).
- **Zero Blur in Virtualized Lists:** NEVER mount `BlurView` or Skia `BackdropFilter` inside `FlatList`, `FlashList`, or `RecyclerView` rows.
- **System Insets & Gesture Safe Zones:** Always pad floating tab bars with `useSafeAreaInsets().bottom` (typically $+8\text{--}12\text{ dp}$ above the Android gesture bar) and ensure edge-swipe back gestures do not collide with floating horizontal controls.
- **Software Keyboard Behavior:** Hide or smoothly collapse floating bottom tab bars when the Android IME keyboard opens (`KeyboardAvoidingView` / `windowSoftInputMode="adjustResize"`).

### 5.2. Web Architecture (React / Next.js / Tailwind CSS)

For Web implementations, defer low-level WebGPU/WGSL shader specifics to the dedicated specification **kb-graphics-webgpu-liquid-glass-v2** (`../graphics-shaders/webgpu-liquid-glass.md`), and integrate it with the Design System layer as follows:

- **Tier 1 (WebGPU — Chrome/Edge/Modern Safari Desktop & Mobile):**
  - Mount **1 shared full-viewport overlay `<canvas>`** (`pointer-events: none`) synced to DOM components tagged with `data-liquid-glass`.
  - Use **Compute Dual-Kawase Blur**, **SDF Gradient Refraction ($\nabla d$)**, and `smin()` for liquid morphing between adjacent pills or trigger-to-dropdown transitions.
  - Use **Zero-Copy `importExternalTexture`** over `<video>` or 3D `<canvas>` backdrops; use **Scroll UV-Shift** for static/semi-dynamic DOM backdrops.
- **Tier 2 (WebGL2 Fallback):**
  - Ping-pong FBO Dual-Kawase blur + equivalent GLSL ES 3.00 fragment shader when `navigator.gpu` is unavailable.
- **Tier 3 (CSS + SVG Compositor-Native Default for Dynamic DOM):**
  - Ideal default for dynamic SaaS/web apps where DOM behind the bar mutates frequently.
  - Combine `backdrop-filter: blur(20px) saturate(180%)`, multi-layered inset `box-shadow` specular rims, CSS `mask-image` scroll-edge fade on the scrollable viewport beneath top/bottom chrome, and `framer-motion` / `motion` `layoutId` for spring-driven morphing dropdowns and sliding tab lenses.

## 6. Reference Implementation Patterns

### 6.1. Shared Concentric Radius & Tier Detection Helper (TypeScript)

```ts
export function getConcentricRadius(outerRadius: number, padding: number): number {
  return Math.max(outerRadius - padding, 4);
}

export type AndroidGlassTier = 'skia-agsl' | 'hardware-blur' | 'opaque-scrim';

export function resolveAndroidGlassTier(params: {
  apiLevel: number;
  isLowRamDevice: boolean;
  reduceTransparency: boolean;
}): AndroidGlassTier {
  if (
    params.reduceTransparency ||
    params.isLowRamDevice ||
    !Number.isFinite(params.apiLevel) ||
    params.apiLevel < 31
  ) {
    return 'opaque-scrim';
  }
  if (params.apiLevel >= 33) {
    return 'skia-agsl';
  }
  return 'hardware-blur';
}
```

### 6.2. Android / Expo React Native: Tiered Floating Glass Pill Shell

```tsx
import React from 'react';
import { Platform, StyleSheet, View, type ViewStyle } from 'react-native';
import { BlurView } from 'expo-blur';
import { LIQUID_GLASS_TOKENS, getConcentricRadius, resolveAndroidGlassTier } from './glass-tokens';

interface GlassPillProps {
  children: React.ReactNode;
  isDark: boolean;
  reduceTransparency?: boolean;
  isLowRamDevice?: boolean;
  renderSkiaShell?: (props: { style: ViewStyle; children: React.ReactNode }) => React.ReactNode;
  style?: ViewStyle;
}

export function FloatingGlassPill({
  children,
  isDark,
  reduceTransparency = false,
  isLowRamDevice = false,
  renderSkiaShell,
  style,
}: GlassPillProps) {
  const apiLevel = Platform.OS === 'android' ? Number(Platform.Version) : 99;
  const tier =
    Platform.OS === 'android'
      ? resolveAndroidGlassTier({ apiLevel, isLowRamDevice, reduceTransparency })
      : reduceTransparency
        ? 'opaque-scrim'
        : 'hardware-blur';

  const outerRadius = LIQUID_GLASS_TOKENS.radii.floatingPillOuter;
  const padding = LIQUID_GLASS_TOKENS.radii.floatingPillPadding;
  const spec = LIQUID_GLASS_TOKENS.materials.regularGlass;

  const containerStyle: ViewStyle = {
    borderRadius: outerRadius,
    padding,
    overflow: 'hidden',
    borderWidth: StyleSheet.hairlineWidth * 1.5,
    borderColor: isDark ? spec.specularRimDark : spec.specularRimLight,
    backgroundColor:
      tier === 'opaque-scrim'
        ? isDark
          ? LIQUID_GLASS_TOKENS.materials.reducedTransparencyFallback.darkFill
          : LIQUID_GLASS_TOKENS.materials.reducedTransparencyFallback.lightFill
        : isDark
          ? spec.darkFill
          : spec.lightFill,
  };

  if (tier === 'opaque-scrim') {
    return <View style={[styles.shadow, containerStyle, style]}>{children}</View>;
  }

  if (tier === 'skia-agsl' && renderSkiaShell) {
    return (
      <View style={[styles.shadow, { borderRadius: outerRadius }, style]}>
        {renderSkiaShell({ style: containerStyle, children })}
      </View>
    );
  }

  return (
    <View style={[styles.shadow, { borderRadius: outerRadius }, style]}>
      <BlurView
        intensity={isDark ? 45 : 60}
        tint={isDark ? 'dark' : 'light'}
        experimentalBlurMethod={Platform.OS === 'android' ? 'dimezisBlurView' : undefined}
        style={containerStyle}
      >
        {children}
      </BlurView>
    </View>
  );
}

const styles = StyleSheet.create({
  shadow: {
    elevation: 10,
    shadowColor: '#000000',
    shadowOffset: { width: 0, height: 10 },
    shadowOpacity: 0.16,
    shadowRadius: 20,
  },
});

// Active tab item inside FloatingGlassPill MUST use:
export const ACTIVE_TAB_RADIUS = getConcentricRadius(
  LIQUID_GLASS_TOKENS.radii.floatingPillOuter,
  LIQUID_GLASS_TOKENS.radii.floatingPillPadding
); // 22dp
```

### 6.3. Web (CSS / Tailwind): Tier 3 Compositor-Native Liquid Glass & Scroll Edge Fade

```css
/* Tier 3 production fallback & dynamic-DOM default for Layer 2 floating bars/dropdowns */
.liquid-glass-surface {
  --glass-outer-radius: 28px;
  --glass-padding: 6px;
  --glass-inner-radius: calc(var(--glass-outer-radius) - var(--glass-padding));

  border-radius: var(--glass-outer-radius);
  padding: var(--glass-padding);
  background: rgba(255, 255, 255, 0.58);
  backdrop-filter: blur(20px) saturate(180%);
  -webkit-backdrop-filter: blur(20px) saturate(180%);

  /* Simulated directional Fresnel rim + subtle depth elevation */
  box-shadow:
    inset 0 1px 0 0 rgba(255, 255, 255, 0.85),
    inset 1px 0 0 0 rgba(255, 255, 255, 0.45),
    inset 0 -1px 0 0 rgba(15, 23, 42, 0.08),
    0 12px 32px -8px rgba(15, 23, 42, 0.14);
  border: 1px solid rgba(255, 255, 255, 0.42);
}

.dark .liquid-glass-surface {
  background: rgba(28, 28, 30, 0.54);
  box-shadow:
    inset 0 1px 0 0 rgba(255, 255, 255, 0.28),
    inset 1px 0 0 0 rgba(255, 255, 255, 0.14),
    inset 0 -1px 0 0 rgba(0, 0, 0, 0.45),
    0 16px 36px -10px rgba(0, 0, 0, 0.55);
  border: 1px solid rgba(255, 255, 255, 0.14);
}

/* Apply to the scrollable content viewport beneath the fixed top navbar (NOT the navbar itself) */
.scroll-edge-mask-top {
  mask-image: linear-gradient(to bottom, transparent 0%, black 24px, black 100%);
  -webkit-mask-image: linear-gradient(to bottom, transparent 0%, black 24px, black 100%);
}

/* Mandatory accessibility override */
@media (prefers-reduced-transparency: reduce) {
  .liquid-glass-surface {
    backdrop-filter: none;
    -webkit-backdrop-filter: none;
    background: rgba(248, 250, 252, 0.96);
  }
  .dark .liquid-glass-surface {
    background: rgba(24, 24, 27, 0.96);
  }
}
```

## 7. Cross-Platform Performance Budgets & Verification Checklist

| Metric | Android Mobile Budget | Web Budget | Verification Method |
| :--- | :--- | :--- | :--- |
| **Frame Time** | $< 16.6\text{ ms}$ (60 Hz) / $< 8.3\text{ ms}$ (120 Hz) | $< 16.6\text{ ms}$ (GPU pass $< 1.5\text{ ms}$ @ 1080p) | Android GPU Rendering Profile / Chrome Performance Tab |
| **Simultaneous Live Blur Regions** | $\le 2$ regions on screen | $1$ shared overlay canvas (Tier 1/2) or $\le 3$ CSS layers | UI hierarchy & overdraw inspector |
| **Virtualized List Cell Blur** | $0$ (strictly forbidden) | $0$ (strictly forbidden) | Component audit on `FlatList` / `FlashList` / virtual scroll |
| **Foreground Text Contrast** | $\ge 4.5:1$ (WCAG AA) | $\ge 4.5:1$ (WCAG AA) | Contrast audit against worst-case bright & dark backdrops |
| **Idle CPU/GPU Activity** | $0\%$ animation worklets when idle | $0\text{ FPS}$ (dirty-flag loop) | Systrace / Chrome Frame Rendering Stats |

### Cross-Platform Runtime QA Verification Checklist

- [ ] **No Glass-on-Content-Card Audit:** Confirm scrollable list items, expense/data cards, and form panels are opaque Layer 1 surfaces.
- [ ] **Concentric Geometry Audit:** Measure floating tab bar, inner active capsule, and morphing dropdown padding/radius to verify $R_{\text{inner}} = \max(R_{\text{outer}} - P, 4)$.
- [ ] **Android Fragmentation Matrix:** Test on Android API 34+ (Tier 1), Android API 31/32 (Tier 2), and a Low-RAM or API 29 emulator (Tier 3 opaque fallback) with zero jank during fast scrolling.
- [ ] **Android Safe Area & IME Test:** Verify floating bottom tab bar clears gesture navigation insets and hides cleanly when the soft keyboard opens.
- [ ] **Morphing & Interaction Continuity:** Confirm dropdown triggers morph smoothly into menus without layout jumps and slider/FAB touch states run exclusively on the UI/GPU thread.
- [ ] **Accessibility Override Test:** Toggle Reduce Transparency / Reduced Motion on both Android and Web; confirm refraction disables cleanly and text contrast remains $\ge 4.5:1$.

## 8. Agent Guardrails (MUST / NEVER)

- **MUST:** Restrict Liquid Glass strictly to **Layer 2** (floating tab bars, navbars, detached search orbs, morphing dropdowns, modal sheets, map/media HUDs, and on-interaction controls).
- **MUST:** Keep **Layer 1** content cards, list rows, and data tables opaque with concentric corner radii.
- **MUST:** Enforce concentric corner radius math ($R_{\text{inner}} = \max(R_{\text{outer}} - P, 4)$) across all nested pills, buttons, and menu items.
- **MUST:** Implement **3-tier capability degradation** on both Android (`skia-agsl` -> `hardware-blur` -> `opaque-scrim`) and Web (`webgpu` -> `webgl2` -> `css-svg`).
- **MUST:** Honor `prefers-reduced-transparency` / `prefers-reduced-motion` and guarantee $\ge 4.5:1$ WCAG AA text contrast via adaptive luminance scrims.
- **MUST:** Run all morphing, sliding lens, and scroll-collapse animations on the native UI thread (Reanimated / Compose) on Android or GPU compositor/WebGPU on Web.
- **NEVER:** Apply real-time blur or optical glass shaders inside scrollable list cells (`FlatList`, `FlashList`, `RecyclerView`, or long DOM tables).
- **NEVER:** Stack active translucent glass panels directly over other active glass panels (**no glass-on-glass**).
- **NEVER:** Tint every secondary toolbar button with saturated glass colors; reserve tinted glass for the primary CTA only.
- **NEVER:** Allow floating bottom bars on Android to collide with system gesture bars or stay awkwardly pinned above the open software keyboard.

## 9. Glossary

- **AGSL (Android Graphics Shading Language):** Android 13+ (API 33+) shader language closely aligned with SkSL, enabling custom `RuntimeShader` optical refraction and rim effects on Android views/Compose.
- **Concentric Corner Radius:** Geometric rule ensuring nested rounded shapes share the same center of curvature ($R_{\text{inner}} = \max(R_{\text{outer}} - P, 4)$).
- **Detached Search Orb:** Circular floating glass search trigger separated from the main tab bar pill for ergonomic thumb reach.
- **Fluid Morphing:** Spring-driven continuous shape transition where a trigger button expands organically into its dropdown menu or popover.
- **Glass-on-Glass Overdraw:** Anti-pattern of layering multiple real-time translucent blur surfaces over each other, causing contrast loss and GPU bandwidth spikes.
- **RenderEffect:** Android 12+ (API 31+) hardware-accelerated visual effect API for native GPU blur and shader composition.
- **Scroll Edge Effect:** Subtle gradient fade or optical transition behind floating top/bottom bars so scrolled content dissolves cleanly beneath glass chrome.
