# napari support for cmap #151 exceptional-value semantics: staged plan

**Status:** Active
**Last updated:** 2026-08-22
**Scope:** cmap -> napari -> vispy rendering stack; CPU and GPU paths
**Review:** Codex plan-review pass 1 (gpt-5.6-sol, xhigh) 2026-08-12; dispositions recorded
below. All four blocking findings accepted or revised into this version.

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
  `nan|bad`, `over`, `under` into them since cmap #145. Note the endpoint semantics:
  napari applies `high_color` at `values >= 1` and `low_color` at `values <= 0`
  (inclusive), while cmap's over/under apply strictly out of range. This is a
  pre-existing divergence, documented rather than changed here (see N1).
- **GPU path** (the image canvas): raw float32 texture upload, contrast limits applied in
  the vispy fragment shader. `apply_clim` detects NaN via `!(v <= 0.0 || 0.0 <= v)` and
  deliberately passes it through; the colormap lookup then does `clamp(t, 0.0, 1.0)`, which
  is **undefined for NaN per the GLSL spec** (AMD `v_med3` slot dependence, Metal fast-math
  on Apple's GL-on-Metal). Positive and negative infinity are clamped to the clims *before*
  normalization, so they already render deterministically as the first/last ramp colors,
  which coincidentally matches cmap's under/over fallback. `nan_color`/`high_color`/
  `low_color` are never consulted on the GPU path. Masked arrays lose their mask before
  upload.
- **Dtype boundary (load-bearing, from review finding B2):** the vispy layer's
  `_on_data_change` calls `fix_data_dtype()` **before** `node.set_data()`, casting
  float64 (and unsupported dtypes) to float32. Finite float64 values above float32 max
  become infinity at that cast. Any classification or sanitization must therefore hook in
  **before** `fix_data_dtype()`, at the layer/slice boundary, not at the visual.

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

Stages with an explicit dependency DAG (revised per B1). CPU correctness first, a
capability spike before any shader work, GPU classification with two candidate designs,
and honest degradation reporting keyed to the actual render configuration.

```
Stage 0 (cmap)      ──────────────┐
Stage 1 (CPU model) ──────┐       │
Stage 4a (capability spike) ──┐   │
                              ▼   ▼
Stage 2 (GPU 2D images, design chosen from 4a + benchmarks)
                              │
Stage 3 (masked) ─────────────┤
                              ▼
Stage 4b (conformance probe + support record)
                              ▼
Stage 5 (volume modes)
```

Stage 1 and Stage 4a are independent and can proceed in parallel; Stage 2 needs both;
Stage 4b probes the integrated result of Stages 2-3; Stage 5 builds on all of it.

### Stage 0 - cmap side (in flight)

Land #151 and the persistence follow-up (reversed/with_extremes/pickle/as_dict). cmap is
the semantic source of truth; nothing downstream should re-derive the fallback hierarchy.
**Check:** cmap test suite; the resolution table above is pinned by
`test_exceptional_colors_fall_back_to_the_legacy_extremes`.

### Stage 1 - napari colormap model + CPU-correct rendering  [IMPLEMENTED 2026-08-22]

Landed on the fork: `feature/colormap-inf-colors` (napari, commit 54cbfb8c) and
`feat/napari-inf-color-forwarding` (cmap, stacked on `feat/exceptional-colors`), both
merged into their repo's `integration`. Neither is pushed and neither has a PR; the cmap
branch is held until napari accepts the fields.

Two things the plan did not anticipate, both caught by review and confirmed in code:
`_napari_cmap_to_vispy()` passes the whole `model_dump()` to `VispyColormap`, so the new
fields had to be popped there or every image, surface, and colorbar conversion raises
`TypeError`; and `same_colors()` already compares nan/high/low, so the new fields had to
join it or the colormap registry would deduplicate maps that differ only in an infinity
color. A test covers each. The legacy-default test encodes the pre-change outputs over
`[-inf, -0.5, 0, 0.5, 1, 1.5, +inf, nan]` in both interpolation modes, so a behavior
change for colormaps that do not set the new fields fails loudly.

Verification: 246 colormap tests, 907 utils tests, 2103 layer tests, 185 vispy tests
green on the merged fork `integration`; cmap 246 tests green with the forwarding, and the
forwarding test passes both against released napari 0.8.0 (inert) and against the fork
(forwarded).

Unrelated breakage found while running the cmap suite against napari main: cmap's
`tests/test_data.py::test_napari_name_parity` reads
`colormap_utils._VISPY_COLORMAPS_ORIGINAL`, which napari has removed. It fails on the
unmodified base branch too, so it is a pre-existing incompatibility that will bite cmap
when napari 0.9 releases. Worth a separate upstream issue; not part of this work.


Extend napari's `Colormap` model with `neg_inf_color` and `pos_inf_color` (default `None`
-> fall back to `low_color`/`high_color` exactly as cmap falls back to under/over), and
apply the full resolution order in `Colormap.map()`. Extend `cmap.to_napari()` to forward
the new fields when the installed napari accepts them (same feature-detection pattern
`_napari_colormap_param_names()` already uses). `masked_color` is deferred to Stage 3
because napari has no mask in its data model yet.

Everything CPU-mapped (thumbnails, points/vectors/surface layers, any CPU fallback)
becomes correct for NaN and both infinities with no GPU work at all.

**Check:** golden-array tests comparing `napari.Colormap.map()` against
`cmap.Colormap.__call__` on
`[-inf, under-range, 0.0, in-range, 1.0, over-range, +inf, nan]` — the exact endpoints
included per N1. At exactly 0.0 and 1.0 the two libraries legitimately differ (napari's
inclusive `<= 0` / `>= 1` versus cmap's strict out-of-range); the test asserts each
library's own documented contract and the divergence is flagged to the napari maintainers
rather than silently changed.

### Stage 4a - capability spike (before any integration work)  [IMPLEMENTED 2026-08-22]

Landed on the napari fork as `dev/gl-exceptional-probe` (commit a28f1c99), merged into
`integration`, in `docs/dev/exceptional_rendering/` (force-added; the fork gitignores
`docs`). Six suites, each in its own subprocess, every verdict compared against a numpy
value computed first; `--self-test` inverts the expectations and must report every idiom
failing. Results are keyed by renderer slug so they accumulate across machines.

**Apple M5, GL 2.1 Metal - 90.5, GLSL 1.20** (`results/apple-m5-2-1-metal-90-5.json`):

- **Option A is unreachable here, not merely unavailable.** GLSL 130 is rejected,
  `GL_ARB_shader_bit_encoding` is absent from all 133 extensions, and setting Qt's
  default surface format to 3.3 core before any context exists still yields a 2.1
  context. Bit classification on Apple Silicon would require changing how napari and
  vispy create contexts.
- **vispy's NaN test does not work on this driver.** `!(data <= 0.0 || 0.0 <= data)`
  from `_APPLY_CLIM_FLOAT` returns false for NaN, as do `v != v` and `!(v == v)`. All
  three are self-comparisons, which is what fast math folds, and Metal enables fast math
  by default. The stock-baseline probe shows the consequence: NaN renders as the bottom
  of the colormap, not as `nan_color`. This is a live napari bug independent of the cmap
  work, and it is the strongest thing to lead the design issue with.
- **A NaN test that survives exists.** `(v * 0.0) != 0.0` is true for NaN and both
  infinities and is not a self-comparison, so removing the two infinity cases isolates
  NaN. It classifies all thirteen probe values correctly, payload NaN included. So a
  GLSL 1.20 shader can tell the four classes apart here without reading any bits.
- **Uploads preserve the classes**; r32f works. Bit-exactness is reported `unavailable`
  rather than `pass`, because without `floatBitsToUint` the payload and subnormal bits
  are not observable. The harness does not claim more than it can see.
- **Linear filtering poisons exactly one texel on each side** (2.0 texels of output for a
  one-texel NaN, against 1.0 under nearest). Classification after filtering misclassifies
  a one-texel border around every exceptional value under magnification.

This moves the Option A/B decision but does not close it, and the platforms that would
close it are ones this project cannot reach. The available hardware is one Apple Silicon
laptop; there is no NVIDIA, AMD, or Intel machine to run the probe on, and GitHub-hosted
runners have no discrete GPU, so their Linux and Windows legs measure Mesa's software
rasterizer rather than a vendor driver.

What CI *can* supply, confirmed by reading napari's workflows and the headless-display
action they use: `macos-15` and `macos-13` runners get no special setup because they run
on the real Apple graphics stack, so they would give a second Apple Silicon generation and
an Intel Mac respectively, both on Apple's GL implementation. Linux and Windows legs give
llvmpipe and Mesa3D, which are still worth having as the GLSL 4.x contrast case but are
not driver evidence. A ready but inactive workflow is committed alongside the harness;
activating it means copying it into `.github/workflows/` on a fork and dispatching it,
which is the owner's call.

So the vendor-driver rows are a **placeholder pending outside help**, tracked in the
coverage table in the harness README. Until one arrives, the honest position is that
Option B is the only design demonstrated to work on hardware anyone here has measured,
not that Option A is ruled out. The harness was hardened for that handoff: it needs only
numpy, vispy, and a Qt binding, it records nothing about the machine beyond GL vendor,
renderer, version, and extension count, and `--self-test` now exercises the bit-readback
decoding against synthetic pixels, since that path cannot execute on a GLSL 1.20 platform
and a bug in it would otherwise surface first on a volunteer's machine.


A standalone offscreen harness, independent of napari's shader pipeline, answering the
questions Stage 2's design choice depends on. It compiles minimal self-contained shaders
(not the napari pipeline, which does not exist in modified form yet — this resolves the
B1 circularity) and reads back rendered pixels:

- Is `floatBitsToUint` available in this context (GLSL version / extension)?
- Do NaN and inf bit patterns survive float32 texture upload and nearest sampling?
- What does the *stock* pipeline's `clamp(NaN, 0, 1)` produce here (baseline
  characterization of today's undefined behavior)?
- Do comparison-based fallbacks (`v != v`, `abs(v) > FLT_MAX`) survive this driver's
  compiler?

Run first on Apple Silicon (GL-on-Metal fast-math is the highest-risk platform), then CI
llvmpipe, then community platforms.

**Check:** the harness itself must be falsifiable — a deliberately broken shader variant
must produce a failing readback.

### Stage 2 - GPU classification for the 2D image path

Two candidate designs, decided by Stage 4a results plus prototype benchmarks; the
comparison was elevated to a first-class decision per review finding N2, and it is one of
the two places human/maintainer judgment is explicitly required.

**Common to both designs:**

- The seven-color hierarchy is resolved on the CPU into at most four concrete RGBA
  uniforms (cmap #151 does the same LUT-row resolution); the shader never sees the
  fallback chain.
- A CPU scan runs at the layer/slice boundary, **before `fix_data_dtype()`** (per B2), in
  the same pass that already touches the data for contrast limits; skipped entirely for
  integer dtypes. It detects whether exceptional values are present, and for float64 it
  distinguishes true infinities from finite values above float32 max (clamping the latter
  to +/-FLT_MAX before the cast so over-range stays over-range, with a warning).
- Clean data pays nothing: with no exceptional values detected, both designs collapse to
  the current pipeline (uniform-false branch or absent class texture).

**Option A - in-shader bit classification.** `classify()` via `floatBitsToUint` ahead of
`apply_clim`:

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

No extra memory or texture fetch; works wherever raw data streams to the GPU. Needs the
GLSL-version fallback story and depends on NaN/inf surviving upload (4a verifies), and
classification is exact only under nearest filtering.

**Option B - sidecar class texture + sanitized data.** The CPU scan emits an R8 class
texture (finite / nan / +inf / -inf / masked) and uploads *sanitized* data (exceptional
values replaced by their nearest finite clamp target). The shader samples the class
texture with nearest filtering and branches; the data texture never contains a NaN or
inf.

Properties worth stating plainly: exceptional classes are data properties, independent of
contrast limits, so CPU precomputation forfeits no interactivity; sanitized data makes
linear/cubic filtering well-defined everywhere (no NaN poisoning); the fast-math and
GLSL-version risks nearly vanish (nothing exotic in the shader), which collapses most of
the Stage 4b probe surface; and Stage 3's mask becomes just a fifth class value in the
same texture. Costs: one byte per texel while exceptional values are present, one extra
texture fetch, and the pre-pass. Open question flagged by review: interaction with
tiled/lazy/multiscale data paths.

Lean: Option B for robustness where the data path is eager; Option A where data streams
raw (tiled/lazy) or memory is tight. Possibly both, selected per path. The prototype
benchmark and the napari maintainers settle it.

**Filtering policy (both options):** under Option A, classification is exact only with
`nearest` interpolation and an exceptional texel poisons its filter footprint under
linear/cubic — documented behavior, reflected in the Stage 4b support record. Under
Option B, filtering of sanitized data is well-defined, and the class texture (nearest)
determines the class of the nearest texel.

Mechanism: napari already rewrites vispy shader source in its own visual subclasses
(`napari/_vispy/visuals/volume.py` splits and re-assembles the fragment shader), so the
prototype can live entirely in a napari fork with no vispy release dependency. Whether the
final home is vispy's `ImageVisual` (cleaner: `apply_clim` lives there) or napari's
subclass is a maintainer conversation, not a technical constraint.

**Check:** offscreen render of a test texture through the real pipeline, read back with
`gloo.read_pixels`, compared against the Stage 1 CPU reference; runs in CI under Mesa
llvmpipe. Test vectors include true float64 infinities *and* finite float64 values just
above FLT_MAX, asserted to render differently (per B2).

### Stage 3 - masked data

Scoped per review finding N3 to **eager in-memory `numpy.ma.MaskedArray` input only** in
v1. Multiscale, lazy/dask, and thick-projection inputs carrying masks are rejected with a
clear validation error naming the limitation, not silently stripped; today's silent
`np.asarray()` stripping (in `_scalar_field/_slice.py` and multiscale materialization) is
replaced by that explicit error for masked input on unsupported paths.

- Image layer accepts eager `numpy.ma.MaskedArray`; the mask is split off at the layer
  boundary (the same pre-`fix_data_dtype` hook as the Stage 2 scan).
- CPU path: already correct via cmap semantics.
- GPU path: under Option B the mask is a fifth class-texture value; under Option A it is
  its own R8 texture sampled nearest, checked before value classification (masked wins
  over the value it hides, matching cmap). Allocated only when a mask exists.
- The settable-mask highlighting API (predicate masking as a cheap contour or polarity
  overlay) is **deferred** to Deferred Work: it is a napari feature request in its own
  right and should not ride along on the correctness work.

**Check:** masked/NaN/inf all present in one eager array, GPU readback equals CPU
reference; mask update without data re-upload; multiscale masked input raises the
documented error.

### Stage 4b - conformance probe and degradation reporting

The mechanism the whole plan is accountable to: never silently render something other than
what the colormap promises. Runs against the *integrated* pipeline from Stages 2-3
(resolving the B1 circularity: 4a needed no integration, 4b requires it).

- **Runtime probe:** at first canvas creation (using napari's existing
  `_opengl_context()` pattern), render a small texture of known values (finite endpoints,
  NaN, +/-inf, masked, denormal) through the actual shader pipeline and read back,
  comparing to the CPU reference.
- **Support record, keyed per review finding B3:** support is a property of the
  combination (GL context, node kind, texture format, interpolation mode, render mode) —
  napari switches between image, tiled-image, and volume nodes by data size and display
  dimensionality, and a single process-global cached verdict (the `lru_cache` pattern of
  `get_gl_extensions()`) is wrong the moment any of those change. v1 claims `exact` only
  for the configuration actually probed — the nearest-filtered 2D image node — and
  reports every other configuration honestly as `fallback`, `undefined`, or `excluded`
  until a probe for that configuration exists. The record is invalidated on context loss
  and re-keyed on node/interpolation/format switches.
- **Per class, per configuration, one of:** `exact` (renders the configured color),
  `fallback` (renders the pre-#151 legacy destination, e.g. inf as clim endpoint color),
  `undefined` (probe readback matched neither), `excluded` (deliberate policy, see
  Stage 5). Exposed as `viewer.exceptional_rendering_support` and per-layer.
- **Warnings that fire only when they matter:** warn once per layer when (a) the data
  actually contains a class, and (b) the colormap configures a color for it, and (c) the
  support record for the layer's current configuration is not `exact`. Clean data or
  default colormaps never warn.
- **Escape hatch:** an opt-in CPU pre-mapping mode (full RGBA mapped by cmap on the CPU,
  uploaded as a color texture). Correct on every GPU ever made, at 4x upload memory and
  loss of interactive clim changes. This is the guaranteed-correct floor the warning can
  point to.
- **Docs:** a page recording probe results per platform, populated from CI and community
  reports rather than vendor documentation.

**Check:** probe self-test (llvmpipe must report all-`exact` for the probed
configuration); deliberately breaking the shader in a test build must flip the record to
`undefined`; switching a layer from image node to tiled-image node must re-key the
record rather than reuse the cached verdict.

### Stage 5 - volume (3D) rendering

napari exposes seven rendering modes; the review (B4) correctly noted the original plan
covered four and overclaimed exclusion as the "only" coherent policy. Full matrix, with
exclusion as the **proposed default** and the final policy an explicit napari-maintainer
decision:

| Mode | Proposed policy for exceptional voxels |
|---|---|
| `translucent` | classified color composited normally (front-to-back alpha) — coherent, since each sample contributes a color directly |
| `additive` | classified color added like any other sample — coherent for the same reason |
| `mip` | excluded from the max (a NaN has no magnitude; an inf would permanently saturate the ray) |
| `minip` | excluded from the min (symmetric argument) |
| `attenuated_mip` | excluded from both accumulation and attenuation |
| `average` | excluded from the mean (matching `nanmean` intuition) |
| `iso` | excluded from surface detection (a NaN/inf threshold crossing is not a surface) |
| plane/slice modes | full 2D semantics (same classification as Stage 2) |

Exclusion-vs-priority for the accumulating modes is judgment, not established fact; the
design issue puts the matrix in front of the napari maintainers with exclusion as the
default and per-mode overrides possible later. Modes shipped without implemented
exceptional handling report `excluded` (or `fallback`) in the Stage 4b support record —
no mode is left with silently unspecified behavior.

**Check:** per-mode tests; e.g. MIP over a volume with an interior NaN plane must equal
MIP over the same volume with those voxels removed; `translucent` with an exceptional
voxel must show the classified color at that sample.

## Sequencing and venue

Stage 1 and Stage 4a first, in parallel. Stage 2's design choice (Option A vs B) is made
after 4a results and a small benchmark, with maintainer input. Prototype stages 1-4 in a
napari fork (napari's existing shader-patching precedent means no vispy release is on the
critical path), then bring a design issue to napari, where the #151 thread has already
pinged jni and tlambert03 maintains both vispy and cmap. The venue question (vispy
`ImageVisual` vs napari subclass) is theirs to settle; the prototype works either way.
WebGPU/vispy-next is explicitly not blocked on: gpuweb is still debating NaN-in-clamp
(gpuweb#5192), and both candidate designs (classify-before-arithmetic; sanitized data +
class texture) survive that outcome.

## Alternatives Considered

- **Sentinel substitution on upload** (recode exceptional values as reserved finite
  values in the *data* texture): rejected; any finite sentinel collides with legal data.
  Note this is distinct from Option B, which sanitizes data but carries class identity in
  a separate texture rather than in-band.
- **Pre-normalized CPU upload always**: correct everywhere but gives up interactive
  contrast limits and 4x memory; kept only as the Stage 4b escape hatch.
- **Whitelist GPUs from the survey document**: rejected; the survey itself shows behavior
  varies by driver revision and compiler slot allocation. Probing the real pipeline is
  cheaper and true.
- **`isnan()` in shaders**: rejected as primary mechanism (fast-math folding); retained
  only as the 4a-probed fallback for GLSL 1.20 contexts under Option A.

## Review record - Codex pass 1 (gpt-5.6-sol, xhigh, 2026-08-12)

All reviewer citations were verified against the installed napari 0.8.0 sources before
disposition.

| ID | Finding | Disposition | Rationale / Change |
|---|---|---|---|
| B1 | Stage 2 <-> Stage 4 dependency cycle; probe referenced masked before Stage 3 | Accepted | Stage 4 split into 4a (pre-integration capability spike on standalone shaders) and 4b (post-integration conformance probe); explicit DAG added |
| B2 | Scan placed after `fix_data_dtype()` casts float64 -> float32, destroying the finite-overflow / true-inf distinction | Accepted | Scan and mask extraction moved to the layer/slice boundary before `fix_data_dtype()`; test vectors now include finite float64 above FLT_MAX distinct from true inf |
| B3 | Support record omitted state (node kind, interpolation, format, mode) that changes correctness; `lru_cache` precedent is process-global | Revised | Record keyed by (context, node, format, interpolation, mode); v1 claims `exact` only for the probed nearest-2D-image configuration; invalidation and re-keying specified |
| B4 | Volume policy covered 4 of 7 modes and overclaimed exclusion as the only coherent semantics | Revised | Full seven-mode matrix with per-mode proposed policy; exclusion demoted from "only" to "proposed default"; explicit maintainer decision point |
| N1 | Golden tests omitted the 0/1 endpoints where napari (inclusive) and cmap (strict) legitimately differ | Accepted | Endpoints added to test vectors; divergence documented as pre-existing and flagged to maintainers, not silently changed |
| N2 | Unified R8 class texture + sanitized data deserves comparison against bit classification | Revised | Elevated to first-class Option B with honest tradeoff statement; decision assigned to 4a results + benchmark + maintainers |
| N3 | Mask support scope undefined for multiscale/lazy/thick-projection; settable-mask API is scope creep | Accepted | Stage 3 scoped to eager arrays with explicit validation errors elsewhere; settable-mask API moved to Deferred Work |

### Open items requiring human/maintainer judgment

| Topic | Position in this plan | Decision owner |
|---|---|---|
| Option A (bit classification) vs Option B (class texture + sanitized data) | After the Apple 4a result: A is unreachable on Apple Silicon, and the GLSL 1.20 comparison classification B needs works. Lean B. **Blocked on hardware nobody here has**: NVIDIA, AMD, and Intel results need volunteers or self-hosted runners, and cannot come from GitHub CI | napari maintainers + owner |
| Volume accumulating-mode policy (exclusion vs exceptional-color priority) | Exclusion as default | napari maintainers |

Recommendation after pass 1 dispositions: proceed to Stage 1 / Stage 4a implementation;
no unresolved blocking findings remain.

## Deferred Work

- Settable-mask highlighting API (predicate masking as contour/polarity overlay) — moved
  here from Stage 3 per N3.
- NaN-excluding texture filtering (linear interpolation that ignores exceptional
  neighbors) — moot under Option B, which is part of why Option B is attractive.
- Mask support for multiscale/lazy/dask data paths.
- Labels/segmentation layers (integer-valued; no exceptional classes).
- Plugin-facing API for custom classification (e.g. user-defined sentinel classes).

## Next Steps

1. ~~Codex plan-review~~ — done 2026-08-12; dispositions above.
2. ~~Stage 1 implementation in a napari worktree, red/green~~ — done 2026-08-22.
3. ~~Stage 4a capability-spike harness on the owner's Apple Silicon machine~~ — done
   2026-08-22; findings in the Stage 4a section.
4. Draft the napari issue for the broken NaN test. It does not wait on anything: it is a
   live bug on hardware we have measured, the fix is one line, and it is independent of
   the cmap semantics. Ask for the missing platform rows in the same issue, since the
   people reading it have the hardware this project lacks and running the probe is three
   commands. Owner posts; an agent never submits.
5. Optionally push the draft CI workflow on the fork and dispatch it, for the two macOS
   rows plus the llvmpipe contrast case. Owner's call: it is a push to a remote.
6. Option A/B decision and benchmark once a vendor-driver result exists. Until then
   Stage 2 designs against Option B and notes the assumption.
