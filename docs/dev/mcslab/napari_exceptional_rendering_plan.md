# napari support for cmap #151 exceptional-value semantics: staged plan

**Status:** Active
**Last updated:** 2026-08-12
**Scope:** cmap -> napari -> vispy rendering stack; CPU and GPU paths

## Context

cmap PR #151 defines per-class colors with a fallback hierarchy:

| Input class | Resolution order |
|---|---|
| masked | `masked -> bad -> transparent` |
| unmasked NaN | `nan -> bad -> transparent` |
| negative infinity | `neg_inf -> under -> first ramp color` |
| positive infinity | `pos_inf -> over -> last ramp color` |

These semantics are fully honored wherever cmap itself maps values (CPU). napari has two
color paths that differ completely on exceptional values (verified against napari 0.8.0 and
its vendored vispy on 2026-08-12):

- **CPU path** (`napari.utils.colormaps.Colormap.map()`, thumbnails, non-image layers):
  deterministic numpy. The model already has `nan_color` (default transparent),
  `high_color`, `low_color`, applied via `np.where`. cmap's `to_napari()` forwards
  `nan|bad`, `over`, `under` into them since cmap #145.
- **GPU path** (the image canvas): raw float32 texture upload, contrast limits applied in
  the vispy fragment shader. `apply_clim` detects NaN via `!(v <= 0.0 || 0.0 <= v)` and
  deliberately passes it through; the colormap lookup then does `clamp(t, 0.0, 1.0)`, which
  is **undefined for NaN per the GLSL spec** (AMD `v_med3` slot dependence, Metal fast-math
  on Apple's GL-on-Metal). Positive and negative infinity are clamped to the clims *before*
  normalization, so they already render deterministically as the first/last ramp colors,
  which coincidentally matches cmap's under/over fallback. `nan_color`/`high_color`/
  `low_color` are never consulted on the GPU path. Masked arrays lose their mask before
  upload.

Design constraints taken from the GPU floating-point survey (reviewed 2026-08-12):

1. `isnan()`/comparison-based NaN tests can be folded away under fast-math (Metal default;
   Apple's GL runs on Metal). Bit-pattern tests via `floatBitsToUint` cannot.
2. Never feed a possible NaN to `min`/`max`/`clamp` (undefined per GLSL; slot-dependent on
   AMD `v_med3`).
3. Classify **before** any arithmetic: normalization, gamma, and filtering all launder or
   poison exceptional values.
4. Capability differences across GPUs/drivers are not enumerable from specs; they must be
   probed empirically per context.

## Current Decision

Six stages, each independently shippable and testable. CPU correctness first, GPU
classification second, capability probing and honest degradation reporting as a
first-class deliverable rather than an afterthought.

### Stage 0 - cmap side (in flight)

Land #151 and the persistence follow-up (reversed/with_extremes/pickle/as_dict). cmap is
the semantic source of truth; nothing downstream should re-derive the fallback hierarchy.
**Check:** cmap test suite; the resolution table above is pinned by
`test_exceptional_colors_fall_back_to_the_legacy_extremes`.

### Stage 1 - napari colormap model + CPU-correct rendering

Extend napari's `Colormap` model with `neg_inf_color` and `pos_inf_color` (default `None`
-> fall back to `low_color`/`high_color` exactly as cmap falls back to under/over), and
apply the full resolution order in `Colormap.map()`. Extend `cmap.to_napari()` to forward
the new fields when the installed napari accepts them (same feature-detection pattern
`_napari_colormap_param_names()` already uses). `masked_color` is deferred to Stage 3
because napari has no mask in its data model yet.

Everything CPU-mapped (thumbnails, points/vectors/surface layers, any CPU fallback)
becomes correct for NaN and both infinities with no GPU work at all.

**Check:** golden-array tests comparing `napari.Colormap.map()` against
`cmap.Colormap.__call__` on `[-inf, under-range, in-range, over-range, +inf, nan]` for
every fallback combination.

### Stage 2 - GPU classification for the 2D image path

Insert a classification step in the fragment shader **ahead of** `apply_clim`:

```glsl
// requires floatBitsToUint: GLSL >= 1.30 / ES 3.0 / GL_ARB_shader_bit_encoding
int classify(float v) {
    uint b = floatBitsToUint(v);
    if ((b & 0x7F800000u) == 0x7F800000u) {          // max exponent: inf or nan
        if ((b & 0x007FFFFFu) != 0u) return CLASS_NAN;
        return (b & 0x80000000u) != 0u ? CLASS_NEG_INF : CLASS_POS_INF;
    }
    return CLASS_FINITE;
}
```

- The seven-color hierarchy is resolved on the CPU into at most four concrete RGBA
  uniforms (cmap #151 does the same LUT-row resolution); the shader never sees the
  fallback chain.
- A CPU scan at `set_data` time (one `np.isfinite` reduction, amortized alongside the
  contrast-limits pass, skipped entirely for integer dtypes) sets a
  `u_has_exceptional` uniform. When false, the classification branch is uniform-false and
  costs nothing; when true, it is ~6 ALU ops per fragment. Clean data pays zero.
- Where `floatBitsToUint` is unavailable (GLSL 1.20 contexts), fall back to
  comparison-based tests (`v != v` for NaN, `abs(v) > FLT_MAX` for inf) and let the
  Stage 4 probe decide whether they actually work on that driver.
- Upload hazards handled in the same CPU scan: float64 values above float32 max become
  +/-inf at cast (clamp to +/-FLT_MAX before upload so over-range stays over-range, and
  warn); float16 65504 keeps the guard cmap #151 established.
- Filtering policy: classification is exact under `nearest` interpolation. Under
  linear/cubic, the data texture is filtered before the shader sees a value, so an
  exceptional texel poisons its filter footprint. That is documented behavior, not a bug
  to fix here; NaN-excluding filtering is out of scope.

Mechanism: napari already rewrites vispy shader source in its own visual subclasses
(`napari/_vispy/visuals/volume.py` splits and re-assembles the fragment shader), so the
prototype can live entirely in a napari fork with no vispy release dependency. Whether the
final home is vispy's `ImageVisual` (cleaner: `apply_clim` lives there) or napari's
subclass is a maintainer conversation, not a technical constraint.

**Check:** offscreen render of a test texture through the real pipeline, read back with
`gloo.read_pixels`, compared against the Stage 1 CPU reference; runs in CI under Mesa
llvmpipe.

### Stage 3 - masked data

- Image layer accepts `numpy.ma.MaskedArray`; the mask is split off at the layer boundary
  instead of being silently dropped.
- CPU path: already correct via cmap semantics.
- GPU path: optional R8 mask texture sampled with `nearest`, checked before value
  classification (masked wins over the value it hides, matching cmap). Allocated only when
  a mask exists: one byte per texel, zero cost otherwise.
- A settable mask also delivers the in-range highlighting use case (predicate masking as a
  cheap contour or polarity overlay) that no value-keyed class can express.

**Check:** masked/NaN/inf all present in one array, GPU readback equals CPU reference;
mask update without data re-upload.

### Stage 4 - capability probe and degradation reporting

The mechanism the whole plan is accountable to: never silently render something other than
what the colormap promises.

- **Runtime probe:** at first canvas creation (piggybacking on napari's existing
  `_opengl_context()` / `get_gl_extensions()` pattern, `lru_cache`d), render a 1x8 texture
  of known values (finite endpoints, NaN, +/-inf, masked, denormal) through the actual
  shader pipeline and read back. Compare to the CPU reference. This yields ground truth
  per GPU/driver/compiler, which no spec table can: fast-math folding, `v_med3` slot
  choice, and FTZ all show up in the readback.
- **Support record:** per class, one of `exact` (renders the configured color),
  `fallback` (renders the pre-#151 legacy destination, e.g. inf as clim endpoint color),
  `undefined` (probe readback matched neither). Exposed as
  `viewer.exceptional_rendering_support` and per-layer.
- **Warnings that fire only when they matter:** warn once per layer when (a) the data
  actually contains a class, and (b) the colormap configures a color for it, and (c) the
  probe says support is not `exact`. Clean data or default colormaps never warn.
- **Escape hatch:** an opt-in CPU pre-mapping mode (full RGBA mapped by cmap on the CPU,
  uploaded as a color texture). Correct on every GPU ever made, at 4x upload memory and
  loss of interactive clim changes. This is the guaranteed-correct floor the warning can
  point to.
- **Docs:** a page recording probe results per platform, populated from CI and community
  reports rather than vendor documentation.

**Check:** probe self-test (llvmpipe must report all-`exact` for the bit-pattern path);
deliberately breaking the shader in a test build must flip the record to `undefined`.

### Stage 5 - volume (3D) rendering

Raymarch accumulation makes per-sample special colors ill-defined (the "max" of a NaN is
meaningless in MIP; attenuation integrals cannot absorb a discrete class). Policy:

- Plane/slice modes: same classification as 2D, full color support.
- Accumulating modes (MIP, attenuated MIP, average, iso): exceptional voxels are
  **excluded from accumulation** (treated as transparent), which is the only semantics
  that does not corrupt neighboring rays; documented as such in the support record
  (`excluded`, a fourth state alongside exact/fallback/undefined).
- napari's existing volume-shader patching is the insertion point.

**Check:** MIP over a volume with an interior NaN plane must equal MIP over the same
volume with those voxels removed.

## Sequencing and venue

Stage 1 and the Stage 4 probe have no dependency on each other and can proceed in
parallel; Stage 2 needs both. Prototype stages 1-4 in a napari fork (napari's existing
shader-patching precedent means no vispy release is on the critical path), then bring a
design issue to napari, where the #151 thread has already pinged jni and tlambert03
maintains both vispy and cmap. The venue question (vispy `ImageVisual` vs napari subclass)
is theirs to settle; the prototype works either way. WebGPU/vispy-next is explicitly not
blocked on: gpuweb is still debating NaN-in-clamp (gpuweb#5192), and
classify-before-arithmetic with bit tests is the design that survives that outcome too.

## Alternatives Considered

- **Sentinel substitution on upload** (recode exceptional values as reserved finite
  values): rejected; any finite sentinel collides with legal data, and re-encoding on
  every clim change forfeits the GPU clim advantage.
- **Pre-normalized CPU upload always**: correct everywhere but gives up interactive
  contrast limits and 4x memory; kept only as the Stage 4 escape hatch.
- **Whitelist GPUs from the survey document**: rejected; the survey itself shows behavior
  varies by driver revision and compiler slot allocation. Probing the real pipeline is
  cheaper and true.
- **`isnan()` in shaders**: rejected as primary mechanism (fast-math folding); retained
  only as the probed fallback for GLSL 1.20 contexts.

## Deferred Work

- NaN-excluding texture filtering (linear interpolation that ignores exceptional
  neighbors).
- Labels/segmentation layers (integer-valued; no exceptional classes).
- Plugin-facing API for custom classification (e.g. user-defined sentinel classes).

## Next Steps

1. Codex plan-review of this document per the standing MCSLAB loop.
2. Stage 1 implementation in a napari worktree, red/green.
3. Stage 4 probe prototype (offscreen readback harness) on the owner's Apple Silicon
   machine, since Apple's GL-on-Metal is the highest-risk platform.
4. Design issue draft for napari once 1 and the probe agree locally; owner posts.
