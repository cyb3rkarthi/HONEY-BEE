# Architecture Document: Melli — Master 3D Artistry

This document details the architectural design, rendering pipelines, coordinate transformations, and state synchronization systems powering the **Melli — Master 3D Artistry** landing page.

---

## 1. High-Level System Architecture

The application is structured into five cohesive subsystems operating within a single-page reactive lifecycle:

```
┌────────────────────────────────────────────────────────────────────────┐
│                              BROWSER DOM                               │
│  ┌─────────────────────────┐  ┌─────────────────────────────────────┐  │
│  │ Fixed UI Layer (Z: 30)  │  │ Scroll Content Layer (Z: 1-3)       │  │
│  │ - Logo & Nav Links      │  │ - Hero, Section 01, 02, 03, Footer  │  │
│  │ - Theme Observer (Dark) │  │ - Parallax Flowers & Headings       │  │
│  │ - Bottom-Right Badge    │  │ - IntersectionObserver Triggers     │  │
│  └────────────┬────────────┘  └──────────────────┬──────────────────┘  │
│               │                                  │                     │
│               ▼                                  ▼                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                  SCROLL & INTERPOLATION ENGINE                   │  │
│  │  - Scroll Position (p = scrollY / innerHeight)                   │  │
│  │  - Cubic Easing & Dive Curve Generator                           │  │
│  │  - Keyframe Coordinate Matrix (Desktop vs. Mobile)               │  │
│  │  - Inertial Follow Dampener (1 - exp(-dt * k))                   │  │
│  └──────────────────────────────────┬───────────────────────────────┘  │
│                                     │                                  │
│                                     ▼                                  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │              THREE.JS 0.160.0 WEBGL RENDER PIPELINE              │  │
│  │  ┌────────────────────────┐      ┌────────────────────────────┐  │  │
│  │  │ Scene Graph Hierarchy  │      │ Procedural Wing Generator  │  │  │
│  │  │ - flight Group         │      │ - Dynamic Canvas Alphas    │  │  │
│  │  │ - beeRoot Group        │◄─────┤ - Double-Sided Wing Meshes │  │  │
│  │  │ - holder Group & GLTF  │      │ - Triple-Flap Ghost Blur   │  │  │
│  │  └───────────┬────────────┘      └────────────────────────────┘  │  │
│  │              ▼                                                   │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │ Camera, PMREM Environment, Dual DirLights & Hemisphere     │  │  │
│  │  └────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Scene Graph Hierarchy & Coordinate Frames

The Three.js scene hierarchy separates world spatial translation, banking/pitch rotation, and model centering:

```
Scene
 └── flight (THREE.Group)
      ├── Properties:
      │    • position.set(cur.x * halfW + mouse.sx * 0.12, cur.y * halfH + hover, z)
      │    • scale.setScalar(halfH * 2 * 0.30 * cur.s)   // 30% viewport height
      │    • rotation.y = -yawRad (heading direction)
      │
      └── beeRoot (THREE.Group)
           ├── Properties:
           │    • rotation.set(rollRad, 0, pitchRad, "YZX")  // Bank & nose dive
           │
           ├── holder (THREE.Group)
           │    └── model (GLTF Scene: Normalized body, scaled to 1.0 unit maxDim)
           │         • rotation.set(0, +π/2, 0) // Reorient head (+Z to +X)
           │         • position.sub(center)      // Centered at local origin (0,0,0)
           │
           ├── rightSide (THREE.Group) @ (anchorX, anchorY, +anchorZ)
           │    ├── Forewing Sweep Group (sweep: -0.42, lift: +0.02)
           │    │    └── 3 Flap Groups (offsets: 0.0, +0.55, -0.55)
           │    └── Hindwing Sweep Group (sweep: -0.95, lift: -0.01)
           │         └── 3 Flap Groups (offsets: 0.0, +0.55, -0.55)
           │
           └── leftSide (THREE.Group) @ (anchorX, anchorY, -anchorZ) [scale.z = -1]
                ├── Forewing Sweep Group (sweep: -0.42, lift: +0.02)
                │    └── 3 Flap Groups (offsets: 0.0, +0.55, -0.55)
                └── Hindwing Sweep Group (sweep: -0.95, lift: -0.01)
                     └── 3 Flap Groups (offsets: 0.0, +0.55, -0.55)
```

### Bee Coordinate Frame Definition:
- **`+X`**: Head facing forward.
- **`+Y`**: Dorsal direction (upward towards the spine).
- **`+Z`**: Right lateral direction (toward the right wing).

---

## 3. Procedural Wing Processing Pipeline

Because the raw 3D bumblebee mesh (`BEE_GLB`) contains no wings, the engine procedurally builds photorealistic iridescent wings at runtime:

```
WING_TEX (CloudFront 16:9 PNG on Pure Black Background)
   │
   ▼
Offscreen HTML5 Canvas (1024 × 576)
   │
   ├─► Pixel Analysis: Calculate luminance: L = (0.30*R + 0.59*G + 0.11*B) / 255
   ├─► Alpha Extraction: Alpha = clamp((L - 0.035) * 4.2, 0, 1) * 255
   ├─► Color Grading:
   │     • R = min(255, R * 1.5 + 30)
   │     • G = min(255, G * 1.5 + 30)
   │     • B = min(255, B * 1.5 + 40)
   ▼
THREE.CanvasTexture (SRGBColorSpace, needsUpdate = true)
   │
   ▼
THREE.MeshBasicMaterial (transparent: true, side: DoubleSide, depthWrite: false)
   │
   ├─► Main Flap (opacity: 0.95, phase offset: 0.0)
   ├─► Ghost Blur Flap 1 (opacity: 0.28, phase offset: +0.55)
   └─► Ghost Blur Flap 2 (opacity: 0.28, phase offset: -0.55)
```

---

## 4. Scroll Interpolation & Physics Flight Dynamics

1. **Section Progress Calculation**:
   $$\text{clampedP} = \text{clamp}\left(\frac{\text{scrollY}}{\text{innerHeight}}, 0, K.\text{length} - 1\right)$$
   $$i = \min(K.\text{length} - 2, \lfloor \text{clampedP} \rfloor), \quad f = \text{clampedP} - i$$
   $$t = \text{ease}\left(\text{clamp}\left(\frac{f - 0.12}{0.76}, 0, 1\right)\right)$$

2. **Cubic Easing & Swoop Dive**:
   $$\text{ease}(t) = \begin{cases} 4t^3 & \text{if } t < 0.5 \\ 1 - \frac{(-2t + 2)^3}{2} & \text{otherwise} \end{cases}$$
   $$\text{dive} = \sin(t \cdot \pi) \cdot (-0.22)$$

3. **Inertial Lag Follow**:
   Smooth follow uses exponential decay:
   $$k = 1 - e^{-\text{dt} \cdot 3.2}$$
   $$k_{\text{yaw}} = 1 - e^{-\text{dt} \cdot 2.6}$$
   $$\text{cur}.\text{prop} \leftarrow \text{cur}.\text{prop} + (\text{target}.\text{prop} - \text{cur}.\text{prop}) \cdot k$$

4. **Dynamic Banking & Pitching**:
   - **Velocity**: $v_x = \frac{\Delta x}{\text{dt}}$, $v_y = \frac{\Delta y}{\text{dt}}$
   - **Pitch**: $\text{pitch} = \text{cur}.\text{pitch} - \text{clamp}(v_y \cdot 22, -28, 28) + \sin(t \cdot 2.1 + 1) \cdot 3^\circ + \text{mouse.sy} \cdot 6^\circ$
   - **Roll (Bank)**: $\text{roll} = \text{cur}.\text{roll} + \text{clamp}(v_x \cdot 18, -30, 30) + \sin(t \cdot 1.7) \cdot 4^\circ$

---

## 5. Keyframe Coordinate Matrix

| Section | Identifier | X (NDC) | Y (NDC) | Scale | Yaw (deg) | Pitch (deg) | Roll (deg) |
|---|---|---|---|---|---|---|---|
| **Hero** | `#top` | -0.03 | -0.12 | 1.00 | 150° | +6° | -4° |
| **01** | `#about` | -0.30 | -0.04 | 1.00 | 125° | +10° | +6° |
| **02** | `#access` | +0.22 | -0.30 | 1.00 | 40° | -6° | -8° |
| **03** | (S3) | -0.22 | +0.02 | 0.96 | 160° | +8° | +4° |
| **Footer** | `#contact` | +0.02 | -0.08 | 0.92 | 105° | +4° | 0° |

*(Mobile screens $<760\text{px}$ utilize dedicated mobile-tuned coordinates).*

---

## 6. Hosting & Deployment Specifications

- **Hosting Platform**: Vercel
- **Build Command**: `npm run build`
- **Output Directory**: `.` (Root static output)
- **Routing**: Static clean URLs configured via `vercel.json`
- **Asset Strategy**: High-resolution textures, models, and background imagery streamed on-demand via CloudFront CDN with HTTP/2 and CORS origin headers.
