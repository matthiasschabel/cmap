# Exceptional-value rendering across cmap, vispy and napari: integrated plan

**Status:** Active
**Last updated:** 2026-10-08
**Scope:** cmap -> napari -> vispy rendering stack; CPU- and GPU-scaled paths; NaN and
+/-inf now, masked data and volumes deferred
**Review:** Codex plan-review pass 1 (gpt-5.6-sol, xhigh) 2026-08-12 on the staged plan;
integrated revision 2026-10-08 reviewed by Codex (gpt-6.1-sol, high), verdict Revise;
dispositions in "Review record - integrated revision" below. Then simplified the same day
on owner decisions (contract items 2-3) and re-reviewed in a convergence pass (Revise:
B4-B7, N4, all accepted)

This note began as the staged napari plan (Stages 0-5, 2026-08). The 2026-10-08 revision
puts one layered plan on top: which layer owns each responsibility, how the open PRs and
fork branches fit or are superseded, the upstream PR sequence, and the decisions that
belong to maintainers. The stage sections below are kept as the prototype record and as
detail; where they disagree with "Current Decision", the latter wins.

## Context

cmap PR #151 defines per-class colors with a fallback hierarchy:

| Input class | Resolution order |
|---|---|
| masked | `bad -> transparent` |
| unmasked NaN | `nan -> bad -> transparent` |
| negative infinity | `neg_inf -> under -> first ramp color` |
| positive infinity | `pos_inf -> over -> last ramp color` |

(#151 was narrowed to the IEEE classes; cmap has no mask-specific color and routes masked
entries to `bad`.)

Where each layer stands on 2026-10-08:

- **cmap** maps all classes correctly on the CPU (#151, open, "inclined to accept"; #155
  stacked for persistence).
- **napari CPU path** (`Colormap.map()`, thumbnails, non-image layers): `nan_color`,
  `low_color`, `high_color`, applied inclusively at `<= 0` / `>= 1`. The inclusive endpoint
  is load-bearing (auto-clims put the extremes exactly on the limits; HiLo depends on it)
  and is not to be changed.
- **napari GPU path** (the canvas): napari uses vispy's GPU-scaled float texture whenever
  `texture_float` is available, and falls back to vispy's CPU-scaled path only when it is
  not (`napari/_vispy/layers/image.py:29-38`). In the shader, +/-inf are clamped to the
  clims before normalization and so render as `low_color`/`high_color` or the ramp ends;
  NaN reaches the colormap's `bad_color` branch, provided the NaN test survives the
  driver's compiler.
- **vispy NaN test**: the stock self-comparison `!(x <= 0.0 || 0.0 <= x)` is folded away on
  Apple's GL-on-Metal (fast math), so NaN renders as the bottom of the ramp there. vispy#2796
  replaces it with `vispy/glsl/misc/is-nan.glsl`,
  `!(x <= gl_DepthRange.far || -gl_DepthRange.far <= x)`, pulled in by `#include`
  (brisvag approved the include plan 2026-10-06).
- **vispy CPU-scaled path**: data is normalized on the CPU and uploaded to an unsized
  `GL_LUMINANCE` texture, which cannot hold NaN (GL 2.1 leaves the conversion unspecified),
  so NaN renders black or white depending on size and driver (vispy#2798). Maintainers
  want NaN -> `bad_color` on both paths; NaN -> 0 was rejected. Infinities are clipped to
  0/1 on this path, so they are indistinguishable from saturated finite values.
- **Dtype boundary** (finding B2, 2026-08-12): napari's `fix_data_dtype()` casts float64 to
  float32 before `node.set_data()`, so finite float64 values above float32 max become inf.
  Today that is invisible, because inf and over-range finite render identically. It becomes
  a defect as soon as infinities get their own colors.

Constraints from the GPU floating-point survey and the Apple probe (Stage 4a):

1. No in-shader NaN test is guaranteed; a compiler assuming no NaN may fold any of them.
   Tests that survive on Apple compare against two bounds the compiler cannot relate.
   Every shipped test is verified in the real chain, not as an isolated idiom.
2. Never feed a possible NaN to `min`/`max`/`clamp`.
3. Classify before any arithmetic (clim, gamma); filtering before classification spreads an
   exceptional texel over its footprint.
4. `floatBitsToUint` is unreachable in the GL 2.1 context vispy gets on Apple Silicon.

## Current Decision

### Contract

What a user can rely on, at every layer:

1. **NaN renders as the NaN color** (`bad_color` in vispy, `nan_color` in napari) on both
   the CPU- and GPU-scaled paths, under any contrast limits and gamma.
2. **Infinity colors are ramp-relative**, like the under/over colors (`low_color`/
   `high_color`). With increasing contrast limits, +inf takes the positive-infinity color
   and -inf the negative one. With inverted limits they swap ends together with low/high.
   With equal limits they keep their sign. napari's contrast limits are always increasing
   (`_validate_increasing`, `napari/layers/intensity_mixin.py:119`), so for napari this is
   simply "+inf is always its color". This is the same model cmap already uses: it
   classifies the normalized value it is called with, and falls back `neg_inf -> under`,
   `pos_inf -> over`.
3. **Exact under `nearest` only (owner, 2026-10-08).** Any other interpolation mixes
   exceptional values with their neighbors, and what renders near them is unspecified:
   NaN, either infinity color, or ramp colors, possibly differing by GPU. That is a
   property of filtering arithmetic, not a rendering defect, so no code tries to repair
   it. It is documented (vispy `interpolation` docstring, napari interpolation docs), not
   warned at runtime. Volumes inherit this: exceptional voxels are unspecified in every
   volume mode unless a mode is explicitly handled later.
4. **Unset colors change nothing.** A colormap that sets no infinity colors renders +/-inf
   exactly as today, under ordinary, inverted and equal contrast limits.

### Layer ownership

| Responsibility | Owner | Mechanism |
|---|---|---|
| Class semantics and fallback order | cmap | #151 (IEEE classes), #155 (persistence) |
| Fallback resolution | each library at its public boundary, on the CPU | cmap in its LUT; napari in `Colormap.map()`, which also feeds the colors it passes to vispy; vispy has no fallback chain, only "unset means today's behavior" |
| Concrete per-class colors in the viewer model | napari `Colormap` | existing `nan_color`/`low_color`/`high_color`; proposed `neg_inf_color`/`pos_inf_color` (napari#9372) |
| Data-boundary hygiene: finite float64 above FLT_MAX must not become inf at the float32 cast | napari layer/slice boundary | clip to +/-FLT_MAX before `fix_data_dtype()` |
| NaN and infinity tests in GLSL | vispy | `misc/is-nan.glsl` (#2796), plus an infinity test in the same include |
| Per-class colors, applied in normalized space | vispy `BaseColormap` (immutable) | `bad_color`, `low_color`, `high_color` today; `neg_inf_color`/`pos_inf_color` added beside them, in `glsl_map`, `map()` and `__getitem__` |
| Getting NaN and infinities to the colormap intact | vispy clim/gamma templates (GPU path and, after U8c, meshes); `CPUScaledTexture2D` (CPU path) | per-sign pass-through when the color is set; a sign-based alpha class code on the CPU |

napari ends up with **no shader code** for exceptional values: it supplies colors and clean
data, and vispy renders them. The fork's template overrides and sentinel protocol (Stage 2)
are superseded and not proposed upstream.

### GPU-scaled path: pass-through to the colormap

All per-class colors live in the colormap, next to `bad_color`/`low_color`/`high_color`,
and are tested on the normalized value `t`, the same place vispy's consumers already
apply their extreme colors:

- `_APPLY_CLIM_FLOAT` passes an infinity through, as it already passes NaN, **only when the
  colormap sets that infinity's color**. The visual knows its colormap when it builds the
  chain, so the pass-through line is generated per sign, not tested at run time. It
  multiplies by the sign of the clim span rather than dividing by it, so equal clims, which
  vispy rewrites to `(c, inf)` (`_scalable_textures.py:132`), keep the sign instead of
  producing `inf/inf`.
- `_APPLY_GAMMA_FLOAT` passes the same infinities through (`pow(-inf, g)` is NaN).
- `glsl_map` gets a `t = +/-inf` check for each infinity color that is set, after the NaN
  check and before the low/high checks (which would otherwise swallow them; vispy prepends
  these snippets with `re.sub`, so the setter order in `__init__` fixes the order).
- An infinity whose color is unset takes exactly today's path: clamped to the clims, then
  normalized. That keeps procedural colormaps such as `_Fire`, which do arithmetic on `t`
  (`colormap.py:610`), and today's equal-clim result (+inf renders `bad_color`, from
  `inf/inf`) unchanged. Contract item 4 therefore holds without exceptions.
- ComplexImage builds its chain from the same templates (`image_complex.py:105`), so it
  generates the same per-sign pass-through from its colormap. RGB/RGBA uses its own clim
  template and is untouched.

The infinity test sits in `apply_clim` beside `is_nan`, the arrangement where the fork's NaN
tests were folded away. G1 (done, see gates) found that every candidate form survives on the
M5. The plan uses `x > FLT_MAX * gl_DepthRange.far` in the shared include: it matches
`is_nan`, and a compiler allowed to assume no infinities could fold a bare literal bound.

### CPU colormap API

- `map()` gets `np.isposinf`/`np.isneginf` checks in `_map_edge_case_colors` for each
  infinity color that is set.
- `__getitem__`, the documented entry point, clips to [0, 1] before calling `map()`
  (`colormap.py:364`), which destroys +/-inf. It records the infinity masks before
  clipping and applies the set infinity colors after mapping; finite clipping and unset
  behavior are unchanged.

### Mesh path

The mesh path does not get infinity colors for free. Its normalization is an unguarded
`(val - cmin) / (cmax - cmin)` (`mesh.py:283`): a finite value far outside tiny clims can
overflow to an infinity and would then be painted with an infinity color, and equal clims
(allowed, `mesh.py:194`) divide by zero. A separate PR (U8c) makes the mesh normalize as
images do: reuse the image clim/gamma templates (the mesh source already says it could be
de-duplicated with `image.py`), clamp finite values before dividing, and apply the same
equal-clim rewrite. Clamping is a behavior change only for procedural colormaps that today
extrapolate outside [0, 1] on meshes; the PR states it. Until U8c, infinity colors are an
image and `map()` feature, and the mesh docs say so.

### CPU-scaled path: alpha class code

The luminance texture cannot hold NaN or inf, so `CPUScaledTexture2D` uploads
`luminance_alpha` for single-band data (always, integer data included: an unsized texture's
`internalformat` stays `None`, so a conditional switch would not trigger `_build_texture`).
Alpha holds a **class code** (finite, NaN, and later -inf and +inf), computed in a
`_scale_data_on_cpu` override after scaling. An image first given integer data converts
later float data to that integer dtype and loses NaN; that is existing dtype pinning in
`CPUScaledTextureMixin`, left out of U4. The shader reads the code with its own
`texture2D($texture, texcoord).a` lookup, never from the filtered texel: software filters
scale every channel by the kernel weights, so a `[[0]]` kernel would mark all finite data
invalid (found in U4 review).

The codes are **by sign, not by ramp end.** A clim change inside the previous data limits
does not re-upload (`_scalable_textures.py:249`; `ImageVisual.clim` then only updates the
uniform, `image.py:420`), so a ramp-end code written at upload would go stale when the
orientation flips. The shader decodes the sign against the current orientation instead,
read from the normalized clim uniform it already has, so the CPU path matches the GPU path
after any clim change without a re-upload.

The fragment template on the CPU path selects by code ahead of `$color_transform`:

```glsl
// sketch; final names are vispy's call
vec4 texel = $get_data(texcoord);
gl_FragColor = $select(texel);   // finite -> $color_transform(texel), else the class color
```

- The code is read with the texture's own sampling. Under `nearest` it is exact; under
  filtering, blended codes are unspecified per the contract. No snapped second lookup.
- Only ordinary single-band images on the CPU path bind `$select`; everything else binds a
  pass-through, so the `FunctionChain` and its positional `color_transform[1]`/`[2]` clim
  and gamma updates are untouched.
- An infinity code whose color is unset falls through to `$color_transform`. The luminance
  there is already clipped to the data range, which reproduces today's rendering.
- The existing NaN pass-through guards stay (ComplexImage reuses them; equal clims can
  produce inf).

U4 ships only the two-value code (finite, NaN), but documents alpha as a class code so
U8b extends it rather than redefining it.

### Superseded by this decision

- **Option A vs Option B**: neither ships as framed. The GPU path tests values in the
  shader, and the CPU path carries an alpha class code. Bit classification is dropped.
- **A classification seam in `ImageVisual` for the GPU path**: unnecessary; the colormap
  already sees every value once clim and gamma pass infinities through.
- **The napari fork's NaN test** `!(v <= $flt_max) && !(v >= -$flt_max)` and its sentinels:
  replaced by vispy's include and real infinities, subject to G1.
- **Value-based (sign-strict) infinity colors**: dropped for ramp-relative ones (contract
  item 2). Sign-strict colors would have been honored by `ImageVisual` only, since
  `glsl_map`, `map()` and the mesh path see only the normalized value.
- **Stage 4b runtime conformance probe, support record and CPU pre-mapping escape hatch**:
  dropped. Conformance is a documented contract plus vispy rendering tests. A driver that
  defeats the shipped tests is a vispy bug report, not a planned stage.
- **Stage 5's seven-mode volume matrix**: dropped; contract item 3 covers volumes.
- **Fast/exact preference**: unnecessary; the tests cost a few comparisons per fragment.

### Verification gates

- **G1, preflight: done 2026-10-08 on the Apple M5.** The plan's GPU chain (vispy's
  `is_nan` plus an infinity test in `apply_clim`, `apply_gamma` and the colormap) was
  rendered with four infinity-test forms (literal FLT_MAX, uniform, FLT_MAX scaled by
  `gl_DepthRange.far`, abs-literal), colors set and unset, increasing/inverted/equal clims,
  gamma 1.0 and 2.2: all pass. The self-comparison NaN test fails in the same run, so the
  harness can fail. Script and result: vispy handoff directory, `g1_probe.py` /
  `g1_result.json`. It also exposed the equal-clim `inf/inf` case now handled by the
  per-sign pass-through.
- **G2, for U4:** exact-class rendering tests under `nearest` on the CPU path; under
  `linear`, the `[[1]]` custom kernel and a signed software filter, only that pixels
  outside an exceptional value's footprint are unaffected.
- **G3, for U10:** a finite float64 just above FLT_MAX renders differently from a true inf.
- **G4, release-blocking, per component; also before napari retires any workaround.**
  Apple M5 readback through the implemented vispy visuals under `nearest`, with distinct
  NaN, infinity, low/high and ramp colors:
  - **G4-U4:** CPU-path NaN; ComplexImage imaginary mode with NaN only in the real part, and
    RGB/RGBA NaN, unchanged.
  - **G4-U8a:** GPU path and colormap API under increasing, inverted and equal clims, with
    neither infinity color set, each set alone, and both set; a procedural colormap
    (`fire`) with colors unset renders as today; `cm[[-inf, inf]]` and `cm.map(...)` return
    the set colors.
  - **G4-U8b:** CPU path, the same cases, plus an increasing-to-inverted clim change on the
    same visual without replacing its data.
  - **G4-U8c:** meshes, the same cases, plus a finite value far outside tiny clims (must not
    take an infinity color) and equal clims.

### Upstream sequence

Small PRs, each with a test that fails before and passes after. The user writes and posts
all upstream text.

| # | Repo | Change | State 2026-10-08 | Depends on |
|---|---|---|---|---|
| U1 | vispy | fast-math-safe `is_nan` include (#2796) | open, plan approved | - |
| U2 | vispy | `Image.bad_color` property (#2795) | open; maintainers favor its immutable-colormap approach | - |
| U3 | vispy | `bool` return type in `shaders/parsing.py` | parked, independent; not needed | - |
| U4 | vispy | CPU-path NaN via alpha class code; docstring states the contract (#2798) | draft PR vispy#2804 (`e41fdfea`), stacked on #2796; design issue vispy#2803 | U1, G2, G4-U4 |
| U5 | cmap | IEEE exceptional colors (#151), then persistence (#155) | #151 CI rerunning on `002882b` | - |
| U6 | napari | rely on vispy's `is_nan`; close the napari#9561 stopgap once napari's minimum vispy contains U1 | - | U1 release, G4-U4 |
| U7 | napari + cmap | `neg_inf_color`/`pos_inf_color` in the model and `Colormap.map()`; cmap forwarding | fork branches ready, held | M4, U5 |
| U8a | vispy | infinity test in the include; per-sign clim/gamma pass-through; `neg_inf_color`/`pos_inf_color` on `BaseColormap` (`glsl_map`, `map()`, `__getitem__`) | not started; issue first | U1, M3, G4-U8a |
| U8b | vispy | CPU-path infinity codes (sign-based, decoded against the current orientation) | not started | U4, U8a, G4-U8b |
| U8c | vispy | meshes normalize like images (shared templates, clamp before divide, equal-clim rewrite) | not started | U8a, G4-U8c |
| U9 | napari | pass infinity colors to vispy | not started | U7; a vispy release containing U8a and U8b (napari falls back to the CPU path without float textures) |
| U10 | napari | clip finite float64 overflow before the float32 cast | not started; separate small fix | G3 |

U7 is useful without U8: napari's CPU path becomes correct, and the canvas keeps today's
behavior until U9.

How existing work maps onto this:

| Work | Disposition |
|---|---|
| vispy#2796, #2795 | U1, U2 as they stand |
| vispy#2798 proposal v2 + astra amendments (vispy handoff dir) | U4, with the mask reframed as a class code and the snapped custom-kernel lookup dropped |
| napari#9561 (stopgap NaN fix) | closed by U6 |
| napari fork `colormap-inf-colors` (Stage 1) | U7 |
| cmap `feat/napari-inf-color-forwarding` | U7 (cmap half) |
| napari fork `gpu-exceptional-colors` (Stage 2) | prototype; superseded by U8a-c + U9; source of G1's probe and of the tiled-image fixes, which are separate napari defects worth their own PRs |
| napari fork `gl-exceptional-probe` (Stage 4a) | stays a dev tool for G1 and volunteer driver rows |

### Decisions that belong to maintainers

| ID | Question | Position in this plan | Owner |
|---|---|---|---|
| M1 | Under software filters, is nearest-texel validity acceptable, or must the whole footprint be marked? | Owner position: behavior near exceptional values is unspecified under filtering (contract item 3), so no footprint marking. Asked on vispy#2798 2026-10-06, no reply yet | vispy |
| M2 | *(retired 2026-10-08: the CPU path now carries infinity codes too)* | - | - |
| M3 | Add `neg_inf_color`/`pos_inf_color` to `BaseColormap`, ramp-relative like `low_color`/`high_color`, honored in `glsl_map`, `map()` and `__getitem__`; meshes follow in U8c | Propose in the U8a issue | vispy |
| M9 | Meshes adopt the image clim/gamma templates (clamp before divide), changing out-of-range extrapolation for procedural colormaps | Propose in U8c | vispy |
| M4 | Infinity colors in napari's model (napari#9372) | Propose once a maintainer engages | napari |
| M5 | *(retired 2026-10-08: volumes covered by contract item 3)* | - | - |
| M6 | Masked data rendering (Stage 3) | Deferred by the owner | owner, then napari |
| M7 | Exceptional values under filtering | Owner decided 2026-10-08: contract item 3. vispy only needs to accept the docstring wording | vispy (wording) |

### Out of scope until the above lands

RGB/RGBA NaN (needs its own clim-aware design), masked data (Stage 3), per-mode volume
handling, and removing the existing NaN pass-through guards.

## Prototype record and stage detail (2026-08)

The original staged DAG, kept for the record. Stage 0 is U5, Stage 1 is U7, Stage 2 is
superseded by U8/U9, Stage 4b and the Stage 5 matrix are dropped, and Stage 3 is deferred.

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

### Stage 0 - cmap side (in flight; U5)

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
`_napari_colormap_param_names()` already uses). Mask-specific rendering is deferred to Stage 3;
cmap #151 no longer proposes a `masked_color`, and napari has no mask in its data model yet.

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

### Stage 2 - GPU classification for the 2D image path  [IMPLEMENTED 2026-08-22 on the fork; SUPERSEDED 2026-10-08 by U8/U9]

Landed on the fork as `feature/gpu-exceptional-colors`, merged into `integration`. Neither
Option A nor Option B as the plan framed them: the probe found a third route that needs no
bit access and no CPU pass.

**Design.** Classification moves ahead of the clim clamp, because clamping is what destroys
the distinction between an infinity and a saturated finite value. The class then travels to
the colormap as a negative sentinel (-1 NaN, -2 +inf, -3 -inf; normalized data is in [0, 1],
so anything below -0.5 is unambiguous). `apply_gamma` passes sentinels through untouched,
since `pow()` of a negative base is NaN. The colormap function decodes them in a prologue
injected ahead of every check vispy injects, because a sentinel is negative and vispy's
`low` check would otherwise swallow it. Fallbacks resolve on the CPU when the colormap is
built, so the per-fragment path carries no fallback logic and a colormap that sets no
infinity colors renders exactly as before.

Three source files, all additive: `_vispy/visuals/image.py` (override vispy's `clim_float`
and `gamma_float` templates, bind the bound uniform by template-variable name rather than
chain position), `utils/colormaps/colormap_utils.py` (sentinels, fallback resolution, a
`VispyColormap` subclass that injects the prologue), and `utils/colormaps/colormap.py` from
Stage 1. napari already subclasses `VispyColormap` and rewrites `glsl_map` for labels, so
this uses an existing seam rather than a new one.

**The NaN test is the subtle part, and the first two attempts were wrong.** The isolated
idiom probe endorsed `(v * 0.0) != 0.0 && !gt_max && !lt_min`; that same expression folds to
false inside the real chain, because the surrounding infinity tests give the compiler enough
to prove the conjunction impossible under its no-NaN assumption. Replacing the literal with
a uniform does not help either: `v <= X || X <= v` is provable for any single bound X. What
survives is two bounds the compiler cannot relate, `!(v <= $flt_max) && !(v >= -$flt_max)`,
where refuting it would require knowing `$flt_max >= -$flt_max` and a uniform denies it that.
A test asserts the shape of that expression, because writing the bound as a literal would
reintroduce the bug on Apple Silicon while every CPU-side test kept passing.

**No CPU pass is needed on this platform.** The open question in the plan was how much CPU
preprocessing the semantics would cost. On hardware measured so far the answer is none: the
GPU classifies all four classes itself, the cost is three comparisons per fragment, and it is
paid only in the float image path. The CPU fallback is still the right design for a driver
that folds even this test, and the runtime check for that has to compile and run the shipped
shader, not a proxy idiom, which is precisely what the isolated probe got wrong.

**Verified** by rendering through napari's own visual on an Apple M5: all eight classes route
correctly, against neg_inf and pos_inf collapsing onto low_color and high_color and NaN onto
low_color before the change. 487 vispy and colormap tests green.

**Codex implementation review, 2026-08-22** (gpt-5.6-sol, xhigh; verdict: revise). Both
blocking findings were real and both are fixed.

| ID | Finding | Disposition |
|---|---|---|
| B1 | Tiled images bypass the shader: children of `TiledImageNode` built vispy's `Image`, so an image changed appearance once it crossed the texture size limit | Accepted. Found independently before the report arrived and confirmed by rendering; children now build napari's visual, with a tiled-vs-untiled regression test |
| B2 | The sentinel range is not unreachable: vispy's mesh visual chains the colormap after an *unclamped* `(val - cmin) / (cmax - cmin)`, so a surface vertex below the contrast limits arrives as a large negative t and gets painted with an exceptional color | Accepted. **I had checked this path and got it wrong**, finding `texture_lut()` at `mesh.py:336` and stopping before the `glsl_map` chain at `:288`. Decoding is now opt-in, set only by the image layer for non-volume nodes |
| N1 | The NaN test is empirically, not specification, supported | Accepted as stated; it is already the README's own conclusion. Nothing to change in code, and the vendor-driver gap is tracked in the coverage table |
| N2 | Transparent NaN is the last texel, so it reads as canvas background and shrinks the detected image region, shifting every sample | Accepted. Opaque guard texels at both ends, asserted mid-ramp, so a slipped sampling grid fails with a message that says so |
| N3 | A crashed probe suite was summarized as UNAVAILABLE, conflating "cannot" with "did not measure" | Accepted. Crashed suites report ERROR and the full run exits non-zero |

B2 also explains five labels and colormap tests that failed only in a full-suite run: the
unconditional prologue was leaking into paths that never wanted it. The failures were
visible before the report arrived; the report supplied the cause.

**Codex review pass 2, 2026-08-22** (same reviewer; verdict: revise, then clean). B2, N1,
N2, N3 confirmed resolved. B1 was maintained, correctly, and one new finding.

| ID | Finding | Disposition |
|---|---|---|
| B1 (maintained) | The tiled fix covered construction but not replacement. Tiles are rebuilt wholesale when a data update changes the tile count, and pass-through attributes reach only the children that exist at assignment time, so a reshape reset colormap, clim, gamma, and interpolation to vispy's defaults | Accepted. Older than this work (the replacement branch is untouched by it) but it makes the tiled-matches-untiled guarantee false, so fixed: values are read before the rebuild and reapplied after. The test now configures the node and *then* changes the tile count |
| N4 | The gate asked whether the node was *not* a volume; `ImageLayerNode` can be handed an arbitrary custom node, so an unknown node would decode sentinels it never emits | Accepted. Positive test for the image and tiled image nodes; unknown nodes fail closed onto the ordinary colormap |
| cleanup | `_resolve_exceptional_colors` duplicated the fallback order already in `Colormap.map` | Accepted, and already applied independently before the report arrived: it now calls `map([nan, inf, -inf])`, verified identical across both interpolation modes and four colormap configurations. A render test compares GPU against CPU directly |
| cleanup | Two tests behaviorally redundant | Partly accepted. Dropped the one asserting generated GLSL source, which pinned implementation detail. Kept the named NaN regression test: a defect stays fixed partly by having a test a reader can find by the defect's name |

Also checked directly on hardware this pass, no defects found: gamma 0.5 and 2.2 (classes
hold while mid-ramp moves), clim inverted, narrow, negative, and zero-width (classification
is clim-independent by construction), and uint8, uint16, RGB, RGBA, and CPU-scaled texture
paths.

**Codex review pass 3, 2026-08-22** (convergence check; verdict: **accept**). All six prior
findings resolved, no new findings, no open disagreements. The reviewer also accepted the one
position maintained against its advice: keeping `test_nan_uses_nan_color_not_the_bottom_of_the_ramp`,
since a regression test a reader can find by the defect's name earns its place even when a
general test covers the same behavior.

One defect was found in this pass and found twice, independently. Carrying tile state across a
rebuild read the pass-through values off the node. That is right for five of the six names,
which are never stored on the node and resolve through `__getattr__` to the first child, and
wrong for `opacity`, which is *also* a real `VisualNode` property: reading it from the node
returned an untouched 1.0, so a reshape pushed 1.0 onto every new child and silently reset the
opacity. That is the same defect the fix was written to close, moved one attribute to the left.
Reading from the first child fixes all six. The regression test walks every name in
`PASS_THROUGH_ATTRIBUTES` rather than only the ones that change pixels, because the render
comparison could never have caught this one.

The convergence is worth noting: three passes, each found a real defect, and the last one found
nothing. The pattern across all three is the same shape of mistake, confirming one path and
generalizing from it. Tiled construction was fixed without tile *replacement*; the surface path
was cleared by reading one of its two colormap routes; five of six pass-through attributes
behaved one way and the sixth did not.

**Strict endpoints: implemented, then reverted, 2026-08-22.** `low_color`/`high_color` apply
at `values <= 0` and `>= 1` in napari, where matplotlib and cmap apply their equivalents
strictly outside the range. That looked like the same clamp-then-classify artifact as the
infinities, and the same fix applied: classify finite out-of-range values before the clamp,
make `map()` strict, stop the thumbnail clipping. It worked, and it was wrong.

napari's automatic contrast limits are the data's own minimum and maximum, so the extreme
pixels of a freshly loaded image sit *exactly* on the limits. Under a strict rule the stock
`HiLo` colormap flags nothing on precisely the images it exists for, and its table is two
entries, black and white, with the blue and red held only in `low_color`/`high_color` -- so
there is nowhere to bake them either. The inclusive comparison is load-bearing, not an
artifact. Reverted in `4110e05e`; no napari behavior changed.

The consequence downstream is that `mcs.visualization.to_napari` forwards `under`/`over`
with a warning naming the difference rather than dropping them, since dropping loses the
colors exactly where they are wanted and the inclusive reading is arguably the useful one in
a viewer. The lesson worth keeping: the infinity divergence and the endpoint divergence look
identical from the code and are not the same kind of thing. One was an artifact of clamping;
the other is a deliberate accommodation of how contrast limits are chosen.

**Codex review pass 4, 2026-08-22** (fresh lens: CPU model, layer integration, multiscale,
thumbnails; verdict: revise, then clean). One new finding, already fixed before the report
arrived.

| ID | Finding | Disposition |
|---|---|---|
| B3 | The thumbnail clipped to the contrast limits and applied gamma *before* calling `Colormap.map`, so +inf and -inf had already become ramp endpoints and could never select their own colors. The layer-list thumbnail disagreed with the canvas | Accepted; found independently while checking the thumbnail path and fixed in `51881ee4`. Only finite values are normalized now; NaN and the infinities pass through for `map` to color. Note gamma alone would also have broken it: `(-inf) ** 0.5` is NaN |

The pass confirmed no defect in the CPU mapping, serialization and registry behavior, layer
node-switch ordering, runtime property updates, or the multiscale canvas path. Verified
separately by hand: the 2D/3D node swap re-applies the colormap under the correct gate, and
nothing outside `colormap.py`/`colormap_utils.py` consumes the extreme-color fields, so no
other consumer needed updating.

That is four passes, each finding a real defect, and the same shape every time: a path
confirmed and then generalized from. The thumbnail is the clearest instance. The GPU was
taught to classify before clamping, and the CPU path that renders the same data eight lines
away was left clamping first.

One trap worth recording for whoever runs these suites next: napari's pytest config sets
`maxfail=5`, and `src/napari/layers/shapes` has order-dependent failures already present in
`integration`. A run that lists paths in one order reports five failures and stops; the same
tree with the paths in another order reports none. Both were reproduced on `integration`
without any of this work applied. Do not read five failures as a signal without checking the
ordering first.

**Not covered:** the volume path (own shaders, Stage 5), masked arrays (Stage 3; needs a mask
to reach the GPU at all, which napari's data model does not yet carry), and multiscale or
tiled paths beyond what the shared image visual covers.


*Superseded 2026-10-08: see "Superseded by this decision". Kept as the original analysis.*

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

### Stage 3 - masked data  [DEFERRED 2026-08-22, by owner decision]

**A mask-specific distinction is the only desired class still missing.** Deferred deliberately,
not blocked: everything else renders, and masked currently takes `bad`, which is correct-looking
and merely undifferentiated. This is now a downstream rendering policy rather than part of #151.

What is already true, so the next session does not re-derive it:

- **cmap intentionally does not have it.** The mask-specific field was removed from #151 after
  API review. cmap continues to route a masked entry to `bad` whatever value it hides.
- **`mcs.visualization.SpecialColors` does not.** It carries five fields (`under`, `over`,
  `negative_infinity`, `positive_infinity`, `bad`) and no `masked`. Adding it is the first
  step, along with the colormap editor surfacing a sixth swatch.
- **ImageScroller already has the mask.** `valid_mask=`/`invalid_mask=` are established
  public API, and its docstring already anticipates this work: masked voxels "are
  represented as NaN at the source boundary ... The boolean mask is retained separately so
  future renderer and colormap support can provide deterministic masked-voxel coloring
  distinct from genuine NaN, infinities, and under/over values." So the mask exists at the
  boundary and is thrown away only for rendering.
- **napari drops the mask during slicing.** Verified: `Image(np.ma.masked_array(...)).data`
  *is* still a `MaskedArray`, but `layer._slice.image.view` is a plain `ndarray`. So the
  mask survives into the layer and dies before the GPU. That slice boundary is the thing to
  change, and it is the reason this is real work rather than a colormap field.
- **The shader is ready for it.** The sentinel protocol now carries five classes
  (`NAN`, `POS_INF`, `NEG_INF`, `UNDER`, `OVER`) decided before the clamp. Masked cannot be
  a sixth sentinel, because a sentinel is derived from the value and masked is a property of
  the *position*, not the value. It needs a sidecar: an R8 mask texture sampled nearest,
  tested before value classification so masked wins over what it hides.

Order to resume in: `SpecialColors.masked` and the editor swatch (self-contained, testable
without any renderer change), then the napari slice boundary, then the sidecar texture and
the shader test. The first step alone is useful, since it makes the class expressible and
lets `to_napari` warn honestly about dropping it instead of silently conflating it with NaN.


Scoped per review finding N3 to **eager in-memory `numpy.ma.MaskedArray` input only** in
v1. Multiscale, lazy/dask, and thick-projection inputs carrying masks are rejected with a
clear validation error naming the limitation, not silently stripped; today's silent
`np.asarray()` stripping (in `_scalar_field/_slice.py` and multiscale materialization) is
replaced by that explicit error for masked input on unsupported paths.

- Image layer accepts eager `numpy.ma.MaskedArray`; the mask is split off at the layer
  boundary (the same pre-`fix_data_dtype` hook as the Stage 2 scan).
- CPU path: cmap correctly routes masks to `bad`; a distinct mask color would require downstream
  handling before cmap mapping.
- GPU path: under Option B the mask is a fifth class-texture value; under Option A it is
  its own R8 texture sampled nearest, checked before value classification (masked wins
  over the value it hides, matching cmap). Allocated only when a mask exists.
- The settable-mask highlighting API (predicate masking as a cheap contour or polarity
  overlay) is **deferred** to Deferred Work: it is a napari feature request in its own
  right and should not ride along on the correctness work.

**Check:** masked/NaN/inf all present in one eager array, GPU readback equals CPU
reference; mask update without data re-upload; multiscale masked input raises the
documented error.

### Stage 4b - conformance probe and degradation reporting  [DROPPED 2026-10-08]

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

### Stage 5 - volume (3D) rendering  [MATRIX DROPPED 2026-10-08; contract item 3]

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
- **Sentinel classes in normalized-t space** (the Stage 2 fork design): works, but the
  sentinel range is reachable from the mesh path's unclamped `t`, so it needed an opt-in
  gate, and it requires napari to override vispy's clim and gamma templates. Replaced by
  passing real infinities through to the colormap (2026-10-08).
- **A `$classify` seam in `ImageVisual` ahead of clim, with sign-strict infinity colors**
  (first 2026-10-08 draft): would have made infinity colors an `ImageVisual`-only feature,
  inconsistent with `glsl_map`, `map()` and the mesh path. Dropped with sign-strictness.
- **NaN-only validity bit on the CPU path** (first 2026-10-08 draft): rejected once
  filtering became unspecified; class codes in the same alpha cost nothing extra.
- **Separate nearest-sampled class texture, or a snapped second lookup**: unnecessary under
  contract item 3.
- **`a < 1.0` on the filtered validity mask under software filters**: rejected. Signed and
  unnormalized weights break it (Catmull-Rom gives 1.0625 with a NaN contributor; a
  `[[0.5]]` kernel marks finite data invalid).
- **NaN -> 0 on the CPU path**: rejected by vispy maintainers on #2798 (2026-10-02); it
  matched the docstring, not the GPU behavior.
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

### Open items requiring human/maintainer judgment (2026-08; superseded by M1-M6)

| Topic | Position in this plan | Decision owner |
|---|---|---|
| Option A (bit classification) vs Option B (class texture + sanitized data) | After the Apple 4a result: A is unreachable on Apple Silicon, and the GLSL 1.20 comparison classification B needs works. Lean B. **Blocked on hardware nobody here has**: NVIDIA, AMD, and Intel results need volunteers or self-hosted runners, and cannot come from GitHub CI | napari maintainers + owner |
| Volume accumulating-mode policy (exclusion vs exceptional-color priority) | Exclusion as default | napari maintainers |

Recommendation after pass 1 dispositions: proceed to Stage 1 / Stage 4a implementation;
no unresolved blocking findings remain.

## Review record - integrated revision (Codex gpt-6.1-sol, high, 2026-10-08)

Evidence for B1 and N2 checked against vispy (`image_complex.py:103`,
`_scalable_textures.py:132`) before disposition.

| ID | Finding | Disposition | Change |
|---|---|---|---|
| B1 | A raw-red validity guard ahead of the whole transform breaks ComplexImage (NaN real, finite imaginary) and changes RGB/RGBA | Accepted | Guard binds only for ordinary single-band images; pass-through elsewhere; regressions in G4 |
| B2 | G1 runs the old sentinel chain, so it cannot certify the new seam | Accepted | G1 demoted to preflight; G4 added as a release-blocking readback through the implemented visual |
| B3 | Classification sees filtered values; infinity sign flips, mixed classes and overflow are not covered by M1 | Accepted | Owner decided M7: exact under `nearest`, unspecified and documented under filtering |
| N1 | Multi-class CPU sidecar needs a sampling policy, not necessarily a second texture | Accepted | "Decided" text corrected; LA class codes offered under M2 |
| N2 | Default infinity behavior under inverted/equal clims | Revised | Infinity classes emitted only when their color is set, so defaults stay compatible; set colors follow the value under any clims (owner invariant, replaces the draft M8); clim cases in G4 |
| N3 | "Fallbacks resolved once by cmap" overstates it | Accepted | Ownership text: semantics owned by cmap, resolved at each library's boundary; vispy defaults for standalone users |

After the same-day simplification: B1's `$is_valid` seam now exists only on the CPU path
(still single-band only); B2's G4 stands; B3 became contract item 3; N1 was adopted (alpha
class codes); N2's open orientation question was settled as ramp-relative (contract item
2); N3 stands.

### Open Disagreements

None.

### Convergence pass (Codex gpt-6.1-sol, high, 2026-10-08)

B1-B3 and N1-N3 resolved. New findings, each verified against vispy before disposition:

| ID | Finding | Disposition | Change |
|---|---|---|---|
| B4 | Ramp-end CPU codes go stale: a clim change inside the data limits does not re-upload (`_scalable_textures.py:249`, `image.py:420`) | Accepted | Codes are by sign and decoded against the current orientation; G4-U8b covers an in-place orientation flip |
| B5 | Unconditional pass-through changes unset rendering: procedural `_Fire` does arithmetic on `t` (`colormap.py:610`); equal clims gave `bad_color` today | Accepted | Pass-through generated per sign only when that color is set; unset infinities take today's path exactly |
| B6 | `__getitem__` clips to [0, 1] before `map()` (`colormap.py:364`), losing +/-inf | Accepted | Masks recorded before clipping, set colors applied after; G4-U8a tests indexing |
| B7 | Mesh normalization is unguarded: finite overflow manufactures infinities; equal clims divide by zero (`mesh.py:283`, `:194`) | Accepted | Mesh is no longer "free": U8c adopts the image templates; M9 added; until then infinity colors are documented as image and `map()` only |
| N4 | U9 needs U8b too; G4 ambiguous across components | Accepted | U9 depends on U8a and U8b; G4 split into G4-U4/U8a/U8b/U8c |

### Open Disagreements

None.

## Deferred Work

- Masked data (Stage 3): owner-deferred. A mask is positional, so it needs a sidecar on the
  GPU path too; start from `SpecialColors.masked` and the napari slice boundary that
  discards the mask.
- Per-mode volume handling, only if someone needs it.
- RGB/RGBA NaN in vispy.
- Removing vispy's NaN pass-through guards once `ComplexImageVisual` handles NaN after its
  scalar conversion.
- vispy's CPU `map()` applies low/high at exactly `t == 0`/`t == 1` while the shader
  allows `1e-12`; pre-existing inconsistency, noted, not ours to fix here.
- Vendor-driver probe rows (NVIDIA/AMD/Intel): need volunteers; pushing the CI workflow to
  the fork is the owner's call.
- Settable-mask highlighting API; mask support for multiscale/lazy data; plugin-facing
  classification API (unchanged from the 2026-08 list).

## Next Steps

1. ~~`/collaborative-refinement` review~~: done 2026-10-08; first pass and convergence pass
   dispositions above. No open disagreements.
2. ~~Gate G1 on the Apple M5~~: done 2026-10-08, all forms pass; `gl_DepthRange`-scaled form chosen.
3. U4 can be implemented now, stacked on #2796, stating the contract in the PR; whether to
   wait for a reply on vispy#2798 first is the owner's call.
4. Draft the U8a issue (M3) once U4 is up.
5. U7 waits on a napari#9372 reply; U5 waits on #151 CI and review cleanup; U10 is
   independent.
