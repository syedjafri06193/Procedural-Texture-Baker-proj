# Procedural Texture Baker — Design & Build Guide

**Project:** Node-graph texture generator with GPU-accelerated live preview and one-click PBR map export for albedo, normal, roughness, and AO
**Language:** TypeScript + WGSL
**Status of this document:** planning + reference

---

## Table of contents

1. [Executive summary and scope](#1-executive-summary-and-scope)
2. [Reality check](#2-reality-check)
3. [Color space](#3-color-space)
4. [Normal maps](#4-normal-maps)
5. [The tiling invariant](#5-the-tiling-invariant)
6. [The memory arithmetic](#6-the-memory-arithmetic)
7. [Graph evaluation](#7-graph-evaluation)
8. [Derived maps](#8-derived-maps)
9. [The GPU backend](#9-the-gpu-backend)
10. [Preview](#10-preview)
11. [Export](#11-export)
12. [Tech stack and setup](#12-tech-stack-and-setup)
13. [Repository layout](#13-repository-layout)
14. [Milestone ladder](#14-milestone-ladder)
15. [Reference implementations](#15-reference-implementations)
16. [Testing](#16-testing)
17. [Stretch goals](#17-stretch-goals)
18. [References](#18-references)

---

## 1. Executive summary and scope

### The original statement

> Node-graph texture generator with GPU-accelerated live preview and one-click PBR map export for albedo, normal, roughness, and AO.

Five findings reshape this:

1. **Color space is the single most common correctness failure in PBR authoring, and it is not uniform across the four maps.** Albedo is sRGB-encoded; normal, roughness, and AO are linear. All graph math must happen in linear regardless. Getting this wrong produces materials that look subtly washed or crushed and nobody can explain why. See section 3.
2. **Normal maps have a green-channel convention split, and picking wrong inverts your lighting.** Unity expects OpenGL (+Y); Unreal expects DirectX (−Y). The same file makes bumps look like dents in one engine and correct in the other. It must be an export option, and the preview must declare which convention it's showing. See section 4.2.
3. **Tiling is a graph-wide invariant that one node can silently destroy.** Every kernel operation — blur, Sobel, warp — must wrap at the edges rather than clamp, and the noise generators must be periodic. A single clamped blur breaks seamlessness for everything downstream of it. See section 5.
4. **The memory arithmetic forces preview-at-low-resolution.** A 4096² RGBA16F texture is 128 MiB. A thirty-node graph caching intermediates at 4K is roughly **3.75 GiB** — past what a browser will give you. At 512² preview it's 60 MiB. That 64× difference is the whole reason for a two-resolution architecture. See section 6.
5. **WebGPU compute is the right backend here, which is the opposite conclusion from a shader playground.** WebGL2 has no compute pipeline at all; image processing on it means encoding data as textures and abusing draw calls as dispatches, with no shared memory and no atomics. There's no external GLSL corpus to stay compatible with — the nodes are yours. See section 9.1.

### Revised project statement

> A WebGPU-compute node-graph texture generator with a linear-light evaluation core, a tiling invariant enforced across every node, content-hashed dirty propagation, dual-resolution evaluation (fast preview, tiled full-resolution export with graph-derived apron regions), and export presets that handle color encoding, bit depth, channel packing, and normal-map convention per target engine.

### The competitive position

Substance 3D Designer is the industry standard and it is very good. Material Maker is a capable free alternative. Blender's shader nodes overlap, though they're render-time rather than baked.

Three honest positions:

**Zero install, shareable.** A browser tool with a URL that opens someone else's graph is a genuinely different product from a desktop application with a license server.

**Correctness as the feature.** Most homegrown texture tools get color space and normal conventions wrong. A tool that is demonstrably right about linear-light math, encoding, bit depth, and conventions — and says so — is useful even with a fraction of the node count.

**It's a learning project.** Then optimize for the hard parts: the evaluator, the tiling invariant, and the export pipeline.

### Explicit non-goals

- **Not a 3D painter.** No mesh painting, no UV projection. Substance Painter's territory.
- **Not a mesh baker.** No high-to-low poly baking of normals or AO from geometry. That's a different pipeline entirely, and §8.3 explains why the AO here is a different thing.
- **Not a material library.** You generate; you don't curate a marketplace.
- **Not a renderer.** The preview is a preview, not a production render.
- **No WebGL2 fallback in v1.** See §9.1.

---

## 2. Reality check

### 2.1 The failure modes

| Symptom | Cause |
|---|---|
| Material looks washed out or too dark in engine | Albedo exported linear, or sRGB applied twice (§3) |
| Roughness behaves wrong | Roughness exported with sRGB encoding (§3.3) |
| **Bumps look like dents** | Normal green channel convention mismatch (§4.2) |
| Normal map bands visibly on smooth surfaces | 8-bit linear quantization (§3.4) |
| Lighting looks flat where two normals blend | Normal maps lerped instead of properly combined (§4.3) |
| Seams visible when tiled | A kernel op clamped instead of wrapped (§5) |
| Seams appear only at high resolution | Non-periodic noise (§5.2) |
| Browser tab crashes on 4K export | Caching intermediates at full resolution (§6) |
| Export doesn't match preview | Two evaluation paths (§7.4) |
| Tiled export shows grid artifacts | Apron too small for the accumulated kernel radius (§11.3) |
| "Roughness" map makes things shiny | Target engine wants smoothness = 1 − roughness (§11.5) |

Most of these are silent. The material just looks slightly wrong, and the artist assumes they did something wrong in the engine.

### 2.2 What makes this genuinely hard

- **Two-resolution evaluation** that provably matches
- **The tiling invariant** holding across an arbitrary user-authored graph
- **Bounded memory** with a cache that can't hold everything
- **Apron computation** that depends on the graph's accumulated kernel support
- **Conventions** that differ per engine and must not be the user's problem

The node library is the visible part and the easy part. The evaluator is the project.

---

## 3. Color space ★

### 3.1 The rule

| Map | Stored encoding | Why |
|---|---|---|
| **Albedo / base color** | **sRGB** | It's a color; sRGB gives perceptually even 8-bit steps |
| **Normal** | **Linear** | It's a vector, not a color |
| **Roughness** | **Linear** | It's a scalar parameter |
| **Metallic** | **Linear** | Scalar |
| **AO** | **Linear** | Scalar |
| **Height / displacement** | **Linear** | Scalar |

Engines infer this from the texture slot, not from the file. Assign a texture to the base-color slot and the engine applies sRGB decode; assign it to roughness and it doesn't. **So the file must be encoded to match what the slot expects.**

### 3.2 Graph math is always linear

Every operation in the graph — blending, filtering, blurring, warping — is only correct in linear light. Blurring sRGB-encoded values produces results that are too dark, because sRGB is a non-linear encoding and averaging it doesn't average the light.

```
                 ┌──────────────────────────────────┐
  inputs ───────▶│  entire graph evaluates LINEAR   │──┬──▶ albedo  → sRGB encode
  (decoded to    │  RGBA16F throughout              │  ├──▶ normal  → linear, 16-bit
   linear on     └──────────────────────────────────┘  ├──▶ rough   → linear
   import)                                             └──▶ AO      → linear
```

Encoding happens **once, at export**, per output. Never inside the graph.

This also means imported images need decoding on the way in. A PNG dropped onto a Color input is sRGB and must be linearized; the same PNG dropped onto a Height input is linear data and must not be.

**Make the input's expected encoding part of the node's port type**, so the system knows which to apply rather than guessing.

### 3.3 The double-encode and the missed-encode

Two symmetric bugs, both silent:

**Double encode** — the graph already encoded to sRGB, then the exporter encodes again. Result: washed out, low contrast.

**Missed encode** — albedo exported as raw linear values into an 8-bit PNG that the engine will sRGB-decode. Result: dark, crushed shadows.

The defense is a type-tagged pipeline:

```ts
type Encoding = "linear" | "srgb";

interface Texture {
  data: GPUTexture;
  encoding: Encoding;      // tracked, not assumed
  channels: 1 | 2 | 3 | 4;
}

function toSrgb(t: Texture): Texture {
  if (t.encoding === "srgb") {
    throw new Error("already sRGB-encoded — double encode");
  }
  return { ...t, data: encodeSrgb(t.data), encoding: "srgb" };
}
```

Throwing rather than silently no-op'ing is the point. A double encode should be a bug report during development, not a subtly wrong material six months later.

### 3.4 8-bit linear is a quantization problem ★

sRGB's curve exists to distribute 8-bit codes perceptually, so 8-bit sRGB is fine for color. **Linear data in 8 bits is not.**

| Map | 8-bit | 16-bit |
|---|---|---|
| Albedo (sRGB) | Fine | Unnecessary |
| Roughness | Usually acceptable | Better for smooth gradients |
| AO | Usually acceptable | — |
| **Normal** | **Visible banding on smooth surfaces** | **Use this** |
| **Height** | **Terraced stepping** | **Use this** |

Normals are the worst case. A normal map encodes direction, and 256 steps per axis means the surface orientation quantizes into visible facets on anything gently curved — the classic "banded sphere" artifact. Height maps are similar: 256 elevation steps across a displacement range produces terracing.

**Default to 16-bit for normal and height**, 8-bit for albedo, and make roughness and AO configurable.

This has a practical consequence in the browser: **`canvas.toBlob()` only produces 8-bit PNG.** There's no browser API for 16-bit output, so you need your own encoder (§11.4).

---

## 4. Normal maps

### 4.1 Encoding

A tangent-space normal is a unit vector remapped from [−1, 1] into [0, 1]:

```wgsl
// pack
let encoded = n * 0.5 + 0.5;

// unpack
let n = normalize(encoded * 2.0 - 1.0);
```

The characteristic flat lavender-blue of an unperturbed normal map is `(0.5, 0.5, 1.0)` — the vector (0, 0, 1), straight out of the surface.

### 4.2 The green channel convention ★

There are two conventions, and they differ only in the sign of Y:

| Convention | Green channel | Used by |
|---|---|---|
| **OpenGL** (+Y, "Y-up") | Y points up | Unity, Blender, Godot, most DCC tools |
| **DirectX** (−Y, "Y-down") | Y points down | Unreal Engine, 3ds Max viewport |

Feed a DirectX map to an engine expecting OpenGL and the lighting inverts along one axis: **bumps read as dents and dents read as bumps.** It's immediately obvious once you know to look for it, and completely mystifying if you don't.

```ts
function flipGreen(normal: Texture): Texture {
  // G = 1 - G. That's the entire difference between the two conventions.
  return applyShader(normal, "flip_green");
}
```

Three requirements:

- **Export presets set it**, not the user (§11.5)
- **The preview declares which convention it's showing**, in the UI, always
- **A one-click flip** on the normal output, because people receive maps in the wrong convention constantly

### 4.3 Blending normals is not lerping ★

Naive interpolation between two normal maps is wrong. It produces a vector that isn't unit length and doesn't represent the composition of the two perturbations — the result looks flattened where the blend is strongest.

Three correct-ish approaches, in increasing order of quality and cost:

```wgsl
// Linear (WRONG — shown for contrast)
fn blend_linear(a: vec3f, b: vec3f, t: f32) -> vec3f {
    return normalize(mix(a, b, t));   // flattens detail
}

// Whiteout / UDN — cheap, widely used, good enough for most detail overlay
fn blend_whiteout(a: vec3f, b: vec3f) -> vec3f {
    return normalize(vec3f(a.xy + b.xy, a.z * b.z));
}

// Reoriented Normal Mapping — correct, slightly more work
fn blend_rnm(base: vec3f, detail: vec3f) -> vec3f {
    let t = base  * vec3f( 1.0,  1.0,  1.0) + vec3f(0.0, 0.0, 1.0);
    let u = detail * vec3f(-1.0, -1.0,  1.0);
    return normalize(t * dot(t, u) - u * t.z);
}
```

**RNM is the right default** for combining a base normal with a detail normal. Whiteout is fine for cheap overlays. Linear should not be offered at all — if a user wants to fade a normal's intensity, that's a different operation (scale the XY before renormalizing), not a blend.

### 4.4 Renormalize after everything

Any operation that filters a normal map — blur, resize, mipmap, warp — produces vectors that are no longer unit length. The lighting math assumes they are.

**Renormalize as the last step of every node that touches a normal.** Make it part of the node contract rather than something users have to remember, and consider a validation pass that samples the output and warns if magnitudes deviate.

### 4.5 Height to normal

The standard conversion is a Sobel gradient of the height field:

```wgsl
@compute @workgroup_size(8, 8)
fn height_to_normal(@builtin(global_invocation_id) id: vec3u) {
    let uv = vec2i(id.xy);
    let texel = 1.0 / vec2f(params.size);

    // Sobel, with WRAPPING sampling — see §5.1
    let tl = sampleWrapped(uv + vec2i(-1, -1));
    let t  = sampleWrapped(uv + vec2i( 0, -1));
    let tr = sampleWrapped(uv + vec2i( 1, -1));
    let l  = sampleWrapped(uv + vec2i(-1,  0));
    let r  = sampleWrapped(uv + vec2i( 1,  0));
    let bl = sampleWrapped(uv + vec2i(-1,  1));
    let b  = sampleWrapped(uv + vec2i( 0,  1));
    let br = sampleWrapped(uv + vec2i( 1,  1));

    let dx = (tr + 2.0 * r + br) - (tl + 2.0 * l + bl);
    let dy = (bl + 2.0 * b + br) - (tl + 2.0 * t + tr);

    // strength is in height units per texel — resolution-dependent!
    let n = normalize(vec3f(-dx * params.strength,
                            -dy * params.strength,
                            1.0));
    textureStore(output, id.xy, vec4f(n * 0.5 + 0.5, 1.0));
}
```

**The strength parameter is resolution-dependent and that's a trap.** A Sobel gradient measures change per texel, so the same height map at 1K and 4K produces different-looking normals for the same strength value — the 4K version looks flatter because each texel represents a smaller step.

Normalize for it: scale strength by resolution so a graph authored at 512 preview exports correctly at 4K. Otherwise the preview lies.

```ts
const effectiveStrength = userStrength * (resolution / REFERENCE_RESOLUTION);
```

---

## 5. The tiling invariant ★

### 5.1 One node can break it for the whole graph

A procedural texture should tile seamlessly. That's a property of the *entire* graph, and any node that reads outside its own texel breaks it if it clamps rather than wraps.

```wgsl
// WRONG — clamping at the edge creates a visible seam when tiled
let s = textureLoad(tex, clamp(uv, vec2i(0), size - 1), 0);

// RIGHT — wrapping preserves the tiling invariant
fn sampleWrapped(uv: vec2i) -> vec4f {
    let w = (uv % size + size) % size;   // handles negatives correctly
    return textureLoad(tex, w, 0);
}
```

Note the double modulo. WGSL's `%` on negative integers returns a negative result, so `(-1) % 512` is `-1`, not `511`. `(x % n + n) % n` is the correction, and forgetting it produces a one-pixel seam that's maddening to track down.

**Every kernel operation must wrap**: blur, Sobel, warp, dilate, distance transform, convolution, downsample. Make it a lint rule and a code-review item — a single `clamp` in a sampling helper is the bug.

Enforce it structurally where you can: provide `sampleWrapped` as the only sampling helper in your WGSL prelude, and don't expose a clamping variant.

### 5.2 Noise must be periodic ★

Standard Perlin and simplex noise are not periodic. They tile only by accident, which is to say not at all. The seam appears as soon as you tile the result.

Periodic variants take a period parameter and wrap the lattice:

```wgsl
fn hash_periodic(p: vec2i, period: vec2i) -> vec2f {
    let wrapped = (p % period + period) % period;
    // ... hash from wrapped coordinates
}

fn perlin_periodic(p: vec2f, period: vec2f) -> f32 {
    let i = floor(p);
    let f = fract(p);
    let pi = vec2i(i);
    let pp = vec2i(period);
    // gradients fetched from wrapped lattice points → tiles exactly
    // ... standard Perlin with hash_periodic at each corner
}
```

For FBM, **every octave must use a period scaled to match its frequency**, or the higher octaves break tiling even when the base doesn't:

```wgsl
fn fbm_periodic(p: vec2f, period: vec2f, octaves: i32) -> f32 {
    var sum = 0.0;
    var amp = 0.5;
    var freq = 1.0;
    for (var i = 0; i < octaves; i++) {
        sum += amp * perlin_periodic(p * freq, period * freq);  // period scales too
        amp *= 0.5;
        freq *= 2.0;
    }
    return sum;
}
```

The `period * freq` is the detail people miss. Pass the base period to every octave and octave 4 tiles at a quarter the rate of octave 1, producing seams that only appear in the fine detail.

Same applies to Worley/cellular noise — the feature points must wrap.

### 5.3 Test it mechanically

Tiling is verifiable, so verify it:

```ts
function tilingSeamError(tex: Float32Array, w: number, h: number): number {
  let maxErr = 0;
  // Compare the left edge against the right edge and top against bottom.
  // In a seamless texture the gradient across the wrap should match the
  // gradient within the interior.
  for (let y = 0; y < h; y++) {
    const gradAcross = Math.abs(px(tex, 0, y) - px(tex, w - 1, y));
    const gradInside = Math.abs(px(tex, 1, y) - px(tex, 0, y));
    maxErr = Math.max(maxErr, Math.abs(gradAcross - gradInside));
  }
  // ... same for vertical
  return maxErr;
}
```

Run it on every node's output in CI with a synthetic input. A node that breaks tiling fails the build rather than shipping a seam.

Also show a 2×2 tiled preview in the UI by default. Seams are obvious when tiled and invisible when not, and defaulting to the tiled view means users find problems immediately.

---

## 6. The memory arithmetic ★

### 6.1 The numbers

| Resolution | RGBA8 | RGBA16F | RGBA32F |
|---|---|---|---|
| 512² | 1 MiB | 2 MiB | 4 MiB |
| 1024² | 4 MiB | 8 MiB | 16 MiB |
| 2048² | 16 MiB | 32 MiB | 64 MiB |
| **4096²** | 64 MiB | **128 MiB** | 256 MiB |
| 8192² | 256 MiB | 512 MiB | 1 GiB |

Now a realistic graph of thirty nodes, caching every intermediate:

| Preview resolution | Total cache |
|---|---|
| 512² RGBA16F | **60 MiB** |
| 1024² RGBA16F | 240 MiB |
| 2048² RGBA16F | 960 MiB |
| **4096² RGBA16F** | **3.75 GiB** |

Browsers won't give you 3.75 GiB of GPU memory, and WebGPU's default device limits are well below it. **This is the constraint that determines the architecture**, not a performance optimization.

### 6.2 Two resolutions, one evaluator

```
Live preview     512² or 1024², full graph cached, evaluated on every change
Export           4096², tiled, minimal cache, evaluated once
```

The same evaluator and the same shaders serve both. Resolution is a parameter, never a code path (§7.4).

Preview resolution should be user-selectable with an honest warning: at 512 you cannot see what 4K detail will look like, and a graph tuned at 512 may look different at 4K (see §4.5 on resolution-dependent strength).

### 6.3 Cache eviction

Even at preview resolution a large graph exceeds a sensible budget. LRU with a size cap:

```ts
class TextureCache {
  private entries = new Map<string, CacheEntry>();
  private bytes = 0;

  constructor(private maxBytes = 512 * 1024 * 1024) {}

  set(key: string, tex: GPUTexture, bytes: number): void {
    while (this.bytes + bytes > this.maxBytes && this.entries.size > 0) {
      const oldest = this.entries.keys().next().value!;
      this.evict(oldest);
    }
    this.entries.set(key, { tex, bytes, at: performance.now() });
    this.bytes += bytes;
  }

  get(key: string): GPUTexture | null {
    const e = this.entries.get(key);
    if (!e) return null;
    this.entries.delete(key);   // Map preserves insertion order — re-insert = MRU
    this.entries.set(key, e);
    return e.tex;
  }
}
```

**Pin the nodes currently visible in the preview** so they never evict — otherwise the node you're actively editing gets thrown out and recomputed on every parameter tweak.

Evicted entries are recomputable from their inputs, so eviction is a cost, not a correctness issue. But an eviction storm during interactive editing feels like the app has frozen, so tune the budget generously.

---

## 7. Graph evaluation

### 7.1 Content hashing

The cache key must capture everything that affects a node's output:

```ts
function nodeHash(node: Node, inputHashes: string[]): string {
  return sha256(JSON.stringify({
    type: node.type,
    version: NODE_VERSIONS[node.type],   // bump to invalidate on a shader fix
    params: canonicalize(node.params),
    inputs: inputHashes,                  // recursive — the whole upstream subgraph
    resolution: node.resolution,
    format: node.format,
  }));
}
```

Because input hashes are included, the hash transitively covers the entire upstream subgraph. Two structurally identical subgraphs in different parts of the document share a cache entry for free — which is common, since people duplicate branches constantly.

`NODE_VERSIONS` is the escape hatch: fix a bug in a node's shader and bump its version, and every cached result derived from it invalidates automatically.

### 7.2 Dirty propagation

Changing one parameter shouldn't re-evaluate the graph.

```ts
function dirtySet(graph: Graph, changed: NodeId): Set<NodeId> {
  // Only nodes downstream of the change need recomputing.
  const dirty = new Set<NodeId>([changed]);
  const queue = [changed];
  while (queue.length) {
    const id = queue.pop()!;
    for (const consumer of graph.consumersOf(id)) {
      if (!dirty.has(consumer)) {
        dirty.add(consumer);
        queue.push(consumer);
      }
    }
  }
  return dirty;
}
```

Combined with content hashing this is close to optimal: a change propagates forward, but any node whose recomputed hash matches its cached one stops the propagation there. Change a parameter and change it back, and nothing recomputes.

### 7.3 Topological evaluation

```ts
async function evaluate(graph: Graph, target: NodeId, ctx: EvalContext): Promise<Texture> {
  const order = topoSort(graph, target);       // detect cycles here, loudly
  for (const id of order) {
    const node = graph.node(id);
    const inputs = node.inputs.map(i => ctx.cache.get(hashOf(i))!);
    const hash = nodeHash(node, inputs.map(t => t.hash));

    const cached = ctx.cache.get(hash);
    if (cached) continue;

    const out = await NODE_IMPLS[node.type].run(inputs, node.params, ctx);
    ctx.cache.set(hash, out, byteSize(out));
  }
  return ctx.cache.get(hashOf(target))!;
}
```

Cycle detection belongs in the UI too — refuse the connection that would create one, with an explanation, rather than failing at evaluation time.

### 7.4 One evaluator, two resolutions ★

The same failure as in any preview/export system: two code paths diverge, and the exported result differs from what the user approved.

**Resolution is a parameter in `EvalContext`, never a branch.** No node implementation should contain `if (isPreview)`. If a node needs to behave differently at different resolutions — like §4.5's Sobel strength — that's a *resolution-dependent parameter computed from the context*, applied identically in both cases, not a separate path.

Test it: evaluate a graph at 512, evaluate at 1024, downsample the 1024 result to 512, and compare. They won't be bit-identical (filtering differs), but they should be perceptually very close. A large divergence means a node is resolution-dependent in a way it shouldn't be.

---

## 8. Derived maps

### 8.1 What gets generated versus authored

| Map | Typical source |
|---|---|
| Albedo | Authored in the graph |
| Height | Authored in the graph |
| **Normal** | Derived from height (§4.5), optionally blended with authored detail |
| **AO** | Derived from height (§8.3) |
| **Curvature** | Derived from height (§8.4) |
| Roughness | Authored, often masked by curvature or AO |
| Metallic | Authored, usually near-binary |

Height is the backbone. Most of the derived maps come from it, which means the height node's quality and bit depth determine the quality of everything derived (§3.4).

### 8.2 Roughness is a scalar, and near-binary metallic

Two notes worth encoding in the node defaults:

**Metallic should be close to 0 or 1.** Physically, a surface is a metal or it isn't. Intermediate values exist only to blend at boundaries — a uniform 0.5 metallic is almost always an authoring error. Consider a warning when a metallic output has substantial mid-range mass.

**Roughness rarely wants to hit 0.** Perfectly smooth is a mirror and looks synthetic under most lighting. A minimum of ~0.04–0.08 is a common floor.

### 8.3 AO from a height field is an approximation ★

Be clear about what this AO is. Real ambient occlusion is a geometric quantity computed from mesh geometry — how much of the hemisphere above a point is blocked. What you can compute from a tiling height field is *local* occlusion: how much the nearby height variation shadows this point.

That's useful and it's what texture AO means in practice, but it is not a substitute for a baked AO map from a high-poly mesh, and users coming from Substance Painter will expect the difference to be stated.

The horizon-based approach:

```wgsl
@compute @workgroup_size(8, 8)
fn ao_from_height(@builtin(global_invocation_id) id: vec3u) {
    let uv = vec2i(id.xy);
    let h0 = sampleHeightWrapped(uv);
    var occlusion = 0.0;

    let DIRS = 16;
    let STEPS = 12;
    for (var d = 0; d < DIRS; d++) {
        let angle = f32(d) / f32(DIRS) * 6.28318530718;
        let dir = vec2f(cos(angle), sin(angle));

        // Track the maximum elevation angle along this direction.
        var maxTan = 0.0;
        for (var s = 1; s <= STEPS; s++) {
            let dist = f32(s) * params.radiusTexels / f32(STEPS);
            let sample = sampleHeightWrapped(uv + vec2i(dir * dist));
            let dh = (sample - h0) * params.heightScale;
            maxTan = max(maxTan, dh / dist);
        }
        occlusion += maxTan / sqrt(maxTan * maxTan + 1.0);   // tan → sin
    }

    let ao = 1.0 - occlusion / f32(DIRS);
    textureStore(output, id.xy, vec4f(ao, ao, ao, 1.0));
}
```

Two parameters that matter and interact: `radiusTexels` (how far to look) and `heightScale` (how tall the height field is in the same units). **Both are resolution-dependent**, with the same problem as §4.5 — the radius in texels means a different physical distance at 512 and 4096. Express the radius in UV space and convert.

Cost: 16 directions × 12 steps = 192 samples per pixel. At 4096² that's 3.2 billion samples. Fine on a GPU in compute, but it's the most expensive node in a typical graph and worth a quality/speed setting.

### 8.4 Curvature

Curvature — the second derivative of height — drives edge-wear masks, which is one of the most-used techniques in PBR texturing. Convex edges get scratched and worn; concave crevices accumulate dirt.

```wgsl
// Laplacian: positive = convex (edges), negative = concave (crevices)
let curvature = (l + r + t + b - 4.0 * c) * params.scale;
```

Output it signed and remapped to [0,1] with 0.5 as flat, so downstream nodes can threshold either direction easily.

---

## 9. The GPU backend

### 9.1 WebGPU compute, not WebGL2 ★

WebGL2 has **no compute pipeline**. Doing image processing on it means encoding data as float textures, treating draw calls as dispatches, and running a full render pass with rasterization overhead for every operation — with no shared memory between invocations, no atomics, and no programmer control over the texture cache. The WebGL2 Compute Shader proposal was deprecated in favor of WebGPU.

WebGPU's compute pipeline is described in the platform documentation as being for exactly this: physics, ML inference, and **image processing pipelines**.

And the availability argument has resolved. WebGPU reached Baseline in January 2026 — Chrome and Edge 113+, Firefox 141+ on Windows and 145+ on macOS Tahoe ARM64, Safari 26+ across macOS, iOS, iPadOS and visionOS. The W3C spec reached Candidate Recommendation Draft in May 2026.

**This is the opposite conclusion from a shader playground**, and for a specific reason: a playground needs GLSL compatibility because its content is user-pasted shaders from a GLSL ecosystem. Here the nodes are yours. There's no corpus to stay compatible with, and the workload is compute.

Caveats worth knowing: Firefox on Linux and Intel Macs is still in progress, and WebGPU compute has platform quirks — Unity documents that `RWBuffer` is unavailable in web compute (structured buffers only), and that render-texture channel counts must match shader outputs or you get uninitialized channels.

**No WebGL2 fallback in v1.** Maintaining two backends doubles the shader surface and halves the pace. Detect and message clearly instead.

### 9.2 Formats

| Use | Format |
|---|---|
| Working intermediates | `rgba16float` |
| Height, accumulation, high dynamic range | `r32float` / `rgba32float` |
| Masks and single-channel data | `r16float` |
| Final 8-bit outputs | `rgba8unorm` |
| Albedo at export | `rgba8unorm-srgb` (hardware encode) |

`rgba16float` is the workhorse: half the memory of 32-bit, and ~3 decimal digits of precision, which is ample for values in [0,1] through a reasonable chain of operations.

Use `rgba32float` deliberately where precision accumulates — a long chain of blends, or a height field with a large range — not by default.

Note `rgba8unorm-srgb`: WebGPU can do the sRGB encode in hardware on write, which is both faster and exactly correct. Use it for albedo rather than encoding in a shader.

### 9.3 Node implementation shape

```wgsl
@group(0) @binding(0) var<uniform> params: NodeParams;
@group(0) @binding(1) var inputA: texture_2d<f32>;
@group(0) @binding(2) var inputB: texture_2d<f32>;
@group(0) @binding(3) var output: texture_storage_2d<rgba16float, write>;

@compute @workgroup_size(8, 8)
fn main(@builtin(global_invocation_id) id: vec3u) {
    if (id.x >= params.size.x || id.y >= params.size.y) { return; }
    // ...
    textureStore(output, id.xy, result);
}
```

The bounds check is required — dispatch sizes round up to workgroup multiples, so the last workgroup overruns.

`8×8` is a reasonable default workgroup size for 2D image work. Worth benchmarking `16×16` on your target hardware; the optimum varies.

### 9.4 Device loss

WebGPU devices can be lost — driver update, GPU reset, or the browser reclaiming resources.

```ts
device.lost.then((info) => {
  console.error("GPU device lost:", info.reason, info.message);
  if (info.reason !== "destroyed") {
    reinitializeDevice();      // recreate device, pipelines, textures
    rebuildFromGraph();        // the graph is the source of truth
  }
});
```

The graph description being the source of truth is what makes recovery tractable — everything on the GPU is derived and rebuildable. Keep it that way; don't let GPU state become authoritative for anything.

---

## 10. Preview

### 10.1 Show the material, not the maps

A grid of four grayscale images tells you very little about whether a material works. The preview should be a lit 3D surface with the maps applied.

- A sphere for general assessment
- A plane with tiling visible for seam checking
- A cube or cylinder for edge behavior
- **Default to a 2×2 or 3×3 tiled plane** so seams are visible without asking

### 10.2 Lighting

The preview's job is to make problems visible:

- **An HDRI environment** for realistic IBL — a material that only looks right under a single point light isn't validated
- **A rotatable light** — normal map problems appear at grazing angles and vanish at others
- **Multiple environment presets** — studio, outdoor, dark interior. Roughness reads completely differently under each.

### 10.3 Declare the convention

The preview must state which normal convention it's rendering (§4.2). A small label saying "Normal: OpenGL (+Y)" costs nothing and prevents the most confusing category of bug report — where the tool is right and the engine is right and they disagree.

### 10.4 Channel inspection

Alongside the 3D view, let users inspect individual maps: single-channel isolation, histogram, value readout under the cursor, and a range indicator. A roughness map with all its values between 0.4 and 0.45 will look flat in the render and the histogram shows why instantly.

---

## 11. Export

### 11.1 The pipeline

```
graph → evaluate at full resolution (tiled) → per-output:
          apply convention (normal flip)
          encode (sRGB for albedo, linear otherwise)
          quantize (8 or 16 bit)
          pack channels (ORM etc.)
          encode file (PNG / EXR / TGA)
```

Each step is per-output and driven by the export preset (§11.5), never by the user remembering.

### 11.2 Tiled evaluation

4096² exceeds a sane cache budget (§6.1), so evaluate in tiles:

```ts
async function exportTiled(graph: Graph, outputs: OutputSpec[],
                           size: number, tileSize = 512): Promise<Map<string, Float32Array>> {
  const apron = computeApron(graph);        // §11.3 — NOT a constant
  const tiles = Math.ceil(size / tileSize);

  for (let ty = 0; ty < tiles; ty++) {
    for (let tx = 0; tx < tiles; tx++) {
      const region = {
        x: tx * tileSize - apron,
        y: ty * tileSize - apron,
        w: tileSize + apron * 2,
        h: tileSize + apron * 2,
      };
      // Because the texture tiles, out-of-bounds apron samples wrap
      // from the opposite edge — no special-casing at the image border.
      const result = await evaluateRegion(graph, region, size);
      writeTile(outputs, result, tx, ty, tileSize, apron);
    }
  }
}
```

The tiling invariant (§5) and tiled export compose nicely: because the texture wraps, the apron at the image border reads from the opposite edge, which is exactly correct. A non-tiling texture would need edge handling here.

### 11.3 The apron is a graph property ★

The apron must be at least the **accumulated kernel radius through the longest path** in the graph. A single 16-texel blur needs 16; three chained 16-texel blurs need 48.

```ts
function computeApron(graph: Graph): number {
  const radius = new Map<NodeId, number>();

  for (const id of topoSort(graph)) {
    const node = graph.node(id);
    const own = NODE_IMPLS[node.type].kernelRadius(node.params);   // per node
    const upstream = Math.max(0, ...node.inputs.map(i => radius.get(i) ?? 0));
    radius.set(id, own + upstream);
  }

  return Math.max(...graph.outputs.map(o => radius.get(o) ?? 0));
}
```

Every node implementation must declare its `kernelRadius` as a function of its parameters. Get it wrong and tiled export shows a faint grid of discontinuities at the tile boundaries — a bug that doesn't appear in preview (which isn't tiled) and only shows up at export.

Add a test: export the same graph tiled and untiled at a resolution where both fit, and assert they match within tolerance.

Watch for unbounded nodes. A distance transform or a large-radius flood fill has effectively infinite support, and those nodes cannot be tiled. Mark them, and fall back to whole-image evaluation at a reduced resolution when one is present — with a warning explaining why.

### 11.4 The browser can't write 16-bit PNG ★

`canvas.toBlob()` produces 8-bit PNG only. There is no browser API for 16-bit output, and §3.4 says normal and height maps need it.

So you write the encoder. 16-bit PNG isn't hard: bit depth 16, big-endian samples, and `CompressionStream("deflate")` handles the zlib layer natively.

```ts
async function encodePng16(data: Uint16Array, w: number, h: number,
                           channels: 1 | 3 | 4): Promise<Blob> {
  const bytesPerPixel = channels * 2;
  const stride = w * bytesPerPixel;
  const raw = new Uint8Array((stride + 1) * h);   // +1 for the filter byte

  for (let y = 0; y < h; y++) {
    raw[y * (stride + 1)] = 0;                    // filter type 0 (None)
    const row = new DataView(raw.buffer, y * (stride + 1) + 1, stride);
    for (let i = 0; i < w * channels; i++) {
      row.setUint16(i * 2, data[y * w * channels + i], false);  // PNG is big-endian
    }
  }
  const compressed = await deflate(raw);
  return new Blob([pngSignature(), ihdr(w, h, 16, colorType(channels)),
                   idat(compressed), iend()], { type: "image/png" });
}
```

Filter type 0 keeps it simple at the cost of file size. Implementing the Paeth filter later is a meaningful compression win if it matters.

**For EXR** (32-bit float, the right format for height and displacement in a film or high-end game pipeline), a minimal uncompressed writer is also achievable, or use a WASM build of a library.

### 11.5 Export presets ★

Conventions differ per engine, and this must not be the user's problem.

```ts
const PRESETS: Record<string, ExportPreset> = {
  "unity-urp": {
    normalConvention: "opengl",
    maps: {
      albedo:    { file: "_BaseMap",   encoding: "srgb",   bits: 8 },
      normal:    { file: "_BumpMap",   encoding: "linear", bits: 16 },
      // Unity's Standard shader uses SMOOTHNESS, not roughness.
      smoothness:{ file: "_MaskMap",   encoding: "linear", bits: 8,
                   pack: { r: "metallic", g: "ao", b: "detail", a: "invert(roughness)" } },
    },
  },

  "unreal": {
    normalConvention: "directx",        // ← the flip
    maps: {
      albedo: { file: "_BaseColor", encoding: "srgb",   bits: 8 },
      normal: { file: "_Normal",    encoding: "linear", bits: 16 },
      orm:    { file: "_ORM",       encoding: "linear", bits: 8,
                pack: { r: "ao", g: "roughness", b: "metallic" } },
    },
  },

  "gltf": {
    normalConvention: "opengl",
    maps: {
      albedo: { file: "_baseColor",         encoding: "srgb",   bits: 8 },
      normal: { file: "_normal",            encoding: "linear", bits: 8 },
      orm:    { file: "_occlusionRoughnessMetallic", encoding: "linear", bits: 8,
                pack: { r: "occlusion", g: "roughness", b: "metallic" } },
    },
  },
};
```

Three things these encode that people get wrong by hand:

- **The normal convention flip** for Unreal
- **Unity wants smoothness, not roughness** — the inverse. Exporting a roughness map into a smoothness slot makes rough things shiny and smooth things matte, which is a memorable bug.
- **ORM channel order** — glTF specifies occlusion in R, roughness in G, metallic in B. Other engines differ.

Show a preview of the packed result in the export dialog so the packing is visible before it ships.

### 11.6 Naming and metadata

Consistent suffixes matter for engine auto-import. And write the graph itself into the export — a sidecar JSON, or a PNG `tEXt` chunk — so a texture can be traced back to the graph that produced it. That's cheap and occasionally saves a lot of archaeology.

---

## 12. Tech stack and setup

| Layer | Choice | Why |
|---|---|---|
| **Language** | TypeScript | |
| **GPU** | **WebGPU + WGSL compute** | §9.1 |
| **Node graph UI** | React Flow, or hand-rolled canvas | React Flow is quick; hand-rolled scales better past a few hundred nodes |
| **3D preview** | Three.js with `three/webgpu` | Ships a WebGPU renderer with WebGL2 fallback |
| **State** | Plain objects + immutable updates | The graph is the source of truth (§9.4); keep it serializable |
| **PNG 16-bit** | Own encoder (§11.4) | No browser API exists |
| **EXR** | Own minimal writer, or WASM | |
| **Compression** | `CompressionStream("deflate")` | Native, no dependency |
| **Testing** | Vitest + headless WebGPU (Dawn/`webgpu` node bindings) | |

Two notes:

**Three.js's TSL** is a JS-native node-graph shader language that compiles to both GLSL and WGSL. It's worth looking at before building your own node-to-shader compiler — not necessarily to adopt, but because it's solving an adjacent problem and the design is informative.

**Get headless WebGPU running in CI early.** Node bindings for Dawn exist and make the §16 tests possible. Tests that require a browser with a GPU are tests that don't run.

---

## 13. Repository layout

```
Procedural-Texture-Baker/
├── README.md
├── docs/
│   ├── design.md                ← this document
│   ├── color-space.md           ← ★ the §3 rules, for contributors
│   ├── conventions.md           ← ★ normal conventions, packing, per engine
│   └── node-authoring.md        ← how to add a node correctly
├── src/
│   ├── graph/
│   │   ├── model.ts
│   │   ├── hash.ts              ← content hashing (§7.1)
│   │   ├── dirty.ts
│   │   ├── evaluate.ts          ← ★ ONE evaluator, resolution as a parameter
│   │   ├── apron.ts             ← ★ graph-derived kernel support (§11.3)
│   │   └── cache.ts             ← LRU with pinning
│   ├── gpu/
│   │   ├── device.ts            ← init, loss, rebuild
│   │   ├── formats.ts
│   │   └── prelude.wgsl         ← ★ sampleWrapped ONLY; no clamping helper
│   ├── nodes/
│   │   ├── generators/          ← periodic noise, shapes, gradients
│   │   ├── filters/             ← blur, warp, transform — ALL wrapping
│   │   ├── color/
│   │   ├── derive/              ← normal, AO, curvature
│   │   └── registry.ts          ← type, params, kernelRadius, version
│   ├── color/
│   │   ├── encoding.ts          ← ★ typed, throws on double-encode
│   │   └── spaces.ts
│   ├── export/
│   │   ├── tiled.ts
│   │   ├── png16.ts             ← ★ the browser can't do this
│   │   ├── exr.ts
│   │   ├── pack.ts
│   │   └── presets.ts           ← ★ per-engine conventions
│   ├── preview/
│   └── ui/
└── tests/
    ├── tiling/                  ← ★ every node, seam error under tolerance
    ├── apron/                   ← tiled vs untiled export match
    ├── encoding/                ← round-trip, double-encode detection
    └── golden/                  ← reference images, tolerance-based
```

`gpu/prelude.wgsl` exposing only `sampleWrapped` is a small structural decision that enforces §5.1 better than any amount of documentation.

---

## 14. Milestone ladder

### M0 — Conventions specification ★ **before any node code**
**Est. 3–4 days**

Write `docs/color-space.md` and `docs/conventions.md`: which maps are linear, which are sRGB, bit depths per map, normal convention per target engine, channel packing per engine.

These decisions propagate into every node and every export path. Getting them wrong later means auditing everything.

**Done when:** you can state, for each of the four maps, its encoding, bit depth, and what each target engine expects.

---

### M1 — Graph core
**Est. 1.5 weeks**

Model, topological sort with cycle detection, content hashing, dirty propagation, LRU cache with pinning. No GPU yet — a CPU reference implementation of three trivial nodes is enough to validate the evaluator.

**Done when:** changing a parameter recomputes only downstream nodes, and changing it back recomputes nothing.

---

### M2 — WebGPU infrastructure
**Est. 1.5 weeks**

Device init, pipeline management, texture allocation, the WGSL prelude, device-loss rebuild.

**Done when:** a device loss followed by rebuild restores the preview from the graph alone.

---

### M3 — The tiling invariant ★
**Est. 1.5 weeks**

Periodic Perlin and Worley with correct per-octave periods, wrapping sample helpers, the seam test harness in CI.

**Build this before the node library, not after.** Retrofitting tiling means auditing every node you've already written.

**Done when:** every implemented node passes the seam test, and the test runs in CI.

---

### M4 — Node library
**Est. 3 weeks**

Generators, filters, color operations, blends, masks. Each node declares its `kernelRadius` and version.

Breadth matters here more than depth — a tool with twenty solid nodes beats one with five beautiful ones.

---

### M5 — Derived maps
**Est. 1.5 weeks**

Height-to-normal with resolution-normalized strength, horizon-based AO, curvature, RNM normal blending, renormalization contracts.

**Done when:** the same graph at 512 and 4096 produces perceptually matching normals (§4.5).

---

### M6 — Preview
**Est. 2 weeks**

Three.js WebGPU renderer, PBR material, HDRI environments, tiled plane by default, convention label, channel inspection with histograms.

---

### M7 — Export ★
**Est. 2 weeks**

Tiled evaluation with graph-derived apron, 16-bit PNG encoder, EXR writer, channel packing, per-engine presets.

**Done when:** tiled and untiled export match within tolerance, and a normal map imported into both Unity and Unreal lights correctly with the respective preset.

That last check needs the actual engines. It's worth the afternoon.

---

### M8 — Graph UI
**Est. 3 weeks**

Node canvas, connections with type checking, parameter panels, undo/redo, save/load, shareable URLs.

---

### M9 — Performance and determinism
**Est. 1 week**

Profiling, workgroup size tuning, cache budget tuning, golden-image regression with cross-vendor tolerance (§16.3).

---

## 15. Reference implementations

### 15.1 The WGSL prelude

```wgsl
// gpu/prelude.wgsl — included in EVERY node shader.
// Deliberately provides no clamping sampler. See docs/design.md §5.1.

fn wrapCoord(uv: vec2i, size: vec2i) -> vec2i {
    // WGSL % on negatives returns negative — the double modulo corrects it.
    return (uv % size + size) % size;
}

fn sampleWrapped(tex: texture_2d<f32>, uv: vec2i, size: vec2i) -> vec4f {
    return textureLoad(tex, wrapCoord(uv, size), 0);
}

fn sampleWrappedBilinear(tex: texture_2d<f32>, uv: vec2f, size: vec2i) -> vec4f {
    let p = uv * vec2f(size) - 0.5;
    let i = vec2i(floor(p));
    let f = fract(p);
    let a = sampleWrapped(tex, i + vec2i(0, 0), size);
    let b = sampleWrapped(tex, i + vec2i(1, 0), size);
    let c = sampleWrapped(tex, i + vec2i(0, 1), size);
    let d = sampleWrapped(tex, i + vec2i(1, 1), size);
    return mix(mix(a, b, f.x), mix(c, d, f.x), f.y);
}

fn srgbToLinear(c: vec3f) -> vec3f {
    let cutoff = c <= vec3f(0.04045);
    let lo = c / 12.92;
    let hi = pow((c + 0.055) / 1.055, vec3f(2.4));
    return select(hi, lo, cutoff);
}

fn linearToSrgb(c: vec3f) -> vec3f {
    let cutoff = c <= vec3f(0.0031308);
    let lo = c * 12.92;
    let hi = 1.055 * pow(c, vec3f(1.0 / 2.4)) - 0.055;
    return select(hi, lo, cutoff);
}
```

Note the sRGB transfer functions use the piecewise definition, not a `pow(x, 2.2)` approximation. The linear segment near black matters — the approximation is visibly wrong in shadows, and shadows are where albedo errors show.

### 15.2 Node registration

```ts
interface NodeImpl<P> {
  type: string;
  version: number;                       // bump to invalidate caches (§7.1)
  inputs: PortSpec[];
  outputs: PortSpec[];
  defaults: P;
  /** Support radius in texels. REQUIRED — drives apron computation (§11.3). */
  kernelRadius(params: P): number;
  run(inputs: Texture[], params: P, ctx: EvalContext): Promise<Texture>;
}

registerNode<BlurParams>({
  type: "blur",
  version: 2,
  inputs: [{ name: "input", encoding: "linear" }],
  outputs: [{ name: "output", encoding: "linear" }],
  defaults: { radius: 4, quality: "medium" },
  kernelRadius: (p) => Math.ceil(p.radius * 3),   // gaussian support ≈ 3σ
  run: async (inputs, params, ctx) => dispatchCompute("blur.wgsl", inputs, params, ctx),
});
```

Making `kernelRadius` a required field of the interface is what prevents someone adding a filter node and silently breaking tiled export.

---

## 16. Testing

### 16.1 Tiling, for every node ★

```ts
describe.each(ALL_NODE_TYPES)("%s tiles seamlessly", (type) => {
  it("has no seam", async () => {
    const graph = singleNodeGraph(type, tilingTestInput());
    const out = await evaluate(graph, 512);
    expect(tilingSeamError(out, 512, 512)).toBeLessThan(0.01);
  });
});
```

Parameterized over the registry, so a new node is tested the moment it's registered. A node that can't tile must opt out explicitly with a documented reason, not by omission.

### 16.2 Apron correctness

```ts
it("tiled export matches untiled", async () => {
  const graph = loadFixture("chained-blurs.json");   // three 16px blurs
  const untiled = await exportWhole(graph, 1024);
  const tiled   = await exportTiled(graph, 1024, 256);
  expect(maxAbsDiff(untiled, tiled)).toBeLessThan(1e-4);
});
```

Use a fixture with deep kernel chains specifically — that's where an under-computed apron shows up.

### 16.3 GPU results aren't bit-identical across vendors ★

Different GPUs produce different results for the same shader: differing transcendental implementations, FMA contraction, and rounding. The differences are usually imperceptible but they are real.

**So golden-image tests must use tolerance, not hashing.**

```ts
function compareGolden(actual: Float32Array, expected: Float32Array): Comparison {
  let maxDiff = 0, sumSq = 0;
  for (let i = 0; i < actual.length; i++) {
    const d = Math.abs(actual[i] - expected[i]);
    maxDiff = Math.max(maxDiff, d);
    sumSq += d * d;
  }
  return { maxDiff, rmse: Math.sqrt(sumSq / actual.length) };
}

// Tolerances chosen to pass across vendors while still catching real regressions.
expect(cmp.maxDiff).toBeLessThan(0.02);
expect(cmp.rmse).toBeLessThan(0.002);
```

Record which GPU produced the golden images, and re-run on multiple vendors when you can. A test that passes only on the author's machine is a test that will block someone else's pull request for no reason.

### 16.4 Encoding round-trips

```ts
it("srgb round-trips within quantization error", () => {
  for (let i = 0; i < 256; i++) {
    const v = i / 255;
    expect(linearToSrgb(srgbToLinear(v))).toBeCloseTo(v, 5);
  }
});

it("refuses a double encode", () => {
  const t = { encoding: "srgb" } as Texture;
  expect(() => toSrgb(t)).toThrow(/double encode/);
});
```

### 16.5 Resolution consistency

Evaluate at 512 and at 1024, downsample the latter, compare. They won't match bit-exactly but should be perceptually close. A large divergence means a node has an unintended resolution dependency (§7.4).

---

## 17. Stretch goals

| Feature | Effort | Value |
|---|---|---|
| **Material presets / library** | Medium | Starting points matter enormously for adoption |
| **Node groups / subgraphs** | Medium | Reusable components; essential past ~50 nodes |
| **Expression nodes** | Small | A small math DSL covers a long tail of one-off needs |
| **Displacement / parallax preview** | Medium | See what the height map actually does |
| **KTX2 / Basis export** | Medium | GPU-compressed output for web and game delivery |
| **Mesh-aware baking** | Large | Import a mesh, bake curvature and AO from geometry — a different pipeline (§8.3) |
| **Variations from a seed** | Small | Reuses the hashing work; generate a family of related materials |
| **Substance `.sbsar` import** | Large | Format is documented but large |
| **Collaborative editing** | Large | CRDT over the graph |
| **CLI / headless batch** | Medium | Same evaluator under Node with Dawn; enables CI texture generation |

That last one is more useful than it sounds: a headless renderer means textures can be generated in a build pipeline from a checked-in graph, which turns materials into version-controlled source rather than binary assets.

---

## 18. References

### PBR

- **Burley**, "Physically-Based Shading at Disney" (SIGGRAPH 2012) — the roughness parameterization everything uses
- **Karis**, "Real Shading in Unreal Engine 4" — the practical PBR reference
- **glTF 2.0 specification**, material section — the ORM packing in §11.5, and the clearest statement of which maps are sRGB
- Unity and Unreal material documentation — smoothness vs roughness, and the normal convention difference

### Textures and graphics

| Source | For |
|---|---|
| **Barré-Brisebois & Hill**, "Blending in Detail" | The RNM blend in §4.3 |
| **Bavoil et al.**, "Image-Space Horizon-Based Ambient Occlusion" | The AO approach in §8.3 |
| Perlin, "Improving Noise" (2002) | Gradient noise, and the periodic variants |
| Worley, "A Cellular Texture Basis Function" | Cellular noise |
| **Ebert et al.**, *Texturing & Modeling: A Procedural Approach* | The canonical reference for this whole field |
| PNG specification (ISO/IEC 15948) | The 16-bit encoder in §11.4 |
| OpenEXR specification | Float export |

### Platform

- **WebGPU specification** (W3C) and WGSL specification
- WebGPU browser availability — Baseline January 2026
- Unity's WebGPU limitations documentation — the `RWBuffer` and render-texture channel gotchas
- Three.js TSL — a node-graph shader language compiling to both GLSL and WGSL; informative adjacent work

### Prior art

- **Substance 3D Designer** — the reference implementation of this category
- **Material Maker** — open source, Godot-based; worth reading
- **Blender shader nodes** — a different model (render-time) but overlapping node vocabulary

---

## Appendix A — Decision record

| Decision | Rationale |
|---|---|
| **WebGPU compute, no WebGL2 fallback** | WebGL2 has no compute pipeline; image processing there means abusing draw calls with no shared memory or atomics. WebGPU is Baseline as of Jan 2026. |
| Opposite call from a shader playground | A playground needs GLSL compatibility for user-pasted shaders; here the nodes are ours and the workload is compute |
| **All graph math in linear light, RGBA16F** | Blending and filtering are only correct in linear; sRGB averaging is too dark |
| Encoding applied once at export, per output | Albedo is sRGB; normal, roughness, AO are linear. The map determines it, not the graph. |
| Encoding tracked in the type, double-encode throws | Both the double encode and the missed encode are silent, and symmetric |
| Port types carry expected encoding | A PNG on a Color input needs linearizing; the same file on Height does not |
| **16-bit for normal and height** | 8-bit linear quantizes visibly — banded spheres and terraced heights |
| `rgba8unorm-srgb` for albedo export | Hardware sRGB encode is faster and exactly correct |
| **Normal convention as an export preset, never a user decision** | Unity expects OpenGL (+Y), Unreal expects DirectX (−Y); wrong choice makes bumps look like dents |
| Preview always labels its convention | Prevents the bug where the tool is right, the engine is right, and they disagree |
| **RNM for normal blending; linear lerp not offered** | Lerping normals flattens detail; intensity adjustment is a different operation |
| Renormalization is part of every normal node's contract | Any filtering breaks unit length, and the lighting math assumes it |
| Sobel strength normalized by resolution | Gradients are per-texel, so the same value looks flatter at 4K — the preview would lie |
| **Tiling is a graph-wide invariant; the prelude exposes only `sampleWrapped`** | One clamped kernel op breaks seamlessness for everything downstream |
| `(x % n + n) % n`, not `x % n` | WGSL's modulo returns negatives, producing a one-pixel seam |
| Periodic noise, with per-octave period scaling | Standard Perlin doesn't tile; passing the base period to every octave breaks the fine detail |
| Seam test parameterized over the whole node registry | New nodes are tested the moment they're registered; opting out must be explicit |
| **Preview at 512–1024, export at full resolution** | 30 nodes at 4K RGBA16F is ~3.75 GiB; at 512² it's 60 MiB. Architecture, not optimization. |
| One evaluator, resolution as a parameter | No `if (isPreview)` anywhere, or export diverges from what the user approved |
| Content hashing includes input hashes transitively | Duplicate subgraphs share cache entries for free, and a node version bump invalidates everything derived |
| LRU cache with visible nodes pinned | Otherwise the node being edited is evicted and recomputed on every tweak |
| **Apron computed from the graph's accumulated kernel radius** | Three chained 16px blurs need 48; a constant apron shows a grid of seams at export only |
| `kernelRadius` required on every node implementation | Prevents silently breaking tiled export by adding a filter |
| Unbounded-support nodes marked and excluded from tiling | Distance transforms can't be tiled; fail loudly rather than subtly |
| **Own 16-bit PNG encoder** | `canvas.toBlob()` is 8-bit only and there is no browser API for more |
| Piecewise sRGB transfer, not `pow(x, 2.2)` | The linear segment near black matters, and shadows are where albedo errors show |
| Export presets carry convention, encoding, bit depth, packing | Unity wants smoothness (1 − roughness); glTF packs ORM as occlusion/roughness/metallic; Unreal flips green |
| Tiled plane as the default preview | Seams are obvious tiled and invisible flat |
| **Golden tests use tolerance, not hashes** | GPU results differ across vendors from FMA and transcendental differences |
| The graph is the source of truth; GPU state is derived | Makes device-loss recovery a rebuild rather than a rewrite |

---

## Appendix B — Quick reference card

```
COLOR SPACE — the #1 PBR correctness failure
  albedo     sRGB-encoded,  8-bit
  normal     LINEAR,       16-bit   ← 8-bit bands visibly
  roughness  LINEAR,        8-bit ok
  metallic   LINEAR,        8-bit ok
  AO         LINEAR,        8-bit ok
  height     LINEAR,       16-bit   ← 8-bit terraces
  ALL graph math is linear. Encode ONCE, at export, per output.
  use the piecewise sRGB transfer, not pow(x, 2.2)

NORMAL CONVENTION
  OpenGL  +Y  → Unity, Blender, Godot, most DCC
  DirectX −Y  → Unreal, 3ds Max viewport
  wrong one = bumps look like DENTS
  flip is just G = 1 − G
  blending: RNM (correct) or whiteout (cheap) — NEVER lerp
  renormalize after every filter

TILING — a graph-wide invariant
  every kernel op WRAPS, never clamps
  wrapCoord = (uv % size + size) % size   ← WGSL % goes negative
  noise must be PERIODIC; FBM needs period × freq per octave
  prelude exposes sampleWrapped ONLY — no clamping helper
  test every node's seam error in CI

MEMORY (this drives the architecture)
  4096² RGBA16F = 128 MiB
  30 nodes @ 4K = 3.75 GiB   ✗
  30 nodes @ 512² = 60 MiB   ✓
  → preview 512/1024, export tiled at full res
  → one evaluator, resolution is a PARAMETER not a branch

TILED EXPORT
  apron = accumulated kernel radius along the longest path
  three chained 16px blurs → apron 48, not 16
  every node declares kernelRadius(params) — required field
  tiling invariant makes border aprons wrap correctly, free
  unbounded nodes (distance transform) can't tile — mark them

EXPORT PRESETS
  Unity   OpenGL normal · SMOOTHNESS (= 1 − roughness) · MaskMap
  Unreal  DirectX normal · ORM as (AO, rough, metal)
  glTF    OpenGL normal · occlusion R, roughness G, metallic B
  canvas.toBlob() is 8-BIT ONLY → write your own PNG16 encoder

GPU
  rgba16float working · r32float height · rgba8unorm-srgb albedo export
  workgroup 8×8, bounds-check (dispatch rounds up)
  device.lost → rebuild from the graph (graph is source of truth)
  results NOT bit-identical across vendors → tolerance tests, not hashes
```
