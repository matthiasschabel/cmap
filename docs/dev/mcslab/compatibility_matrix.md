# MCSLAB cmap compatibility matrix

**Status:** Active
**Last updated:** 2026-08-25
**Scope:** behavior expected from released cmap and the `matthiasschabel/cmap` integration branch

## Context

MCSLAB must remain correct with the released cmap package while incrementally delegating to public
features under upstream review. This matrix records the capability probe, stock fallback, and
retirement condition for each migration seam.

## Current Decision

Seven fixes have merged upstream. `integration` is now `upstream/main` plus open PRs #151 and
#155, the held napari forwarding branch, and maintainer documentation. #151 is the first fork
capability: it adds signed-infinity and NaN colors. Downstream support therefore probes for the
public fields rather than a version number.

The merges do not move the MCS floor. Users install releases, and the fixes are in
upstream `main` but unreleased; `cmap>=0.7` stays the requirement and every fallback stays in
place until a release contains them.

Two rows below are partly overtaken by those fixes:

- *Distinct NaN and mask colors* narrowed to a distinct `nan` override in #151. cmap PR #148
  makes `bad` apply to unmasked NaNs inside a masked array; a separate mask-specific color was
  removed from #151 and remains downstream policy.
- *Complete state propagation* is partly delivered. #147 (merged) stops `ColorStops.reversed()`
  corrupting its source; #150 (merged) stops construction rewriting borrowed interpolation and
  makes `with_extremes()` carry it. PR #155 covers the remaining copy and persistence channels.

| Capability | Released cmap 0.7.2 | Patched integration target | MCS fallback while absent | Retirement condition |
|---|---|---|---|---|
| Signed infinity colors | Not represented | `neg_inf` and `pos_inf` constructor support, with corresponding properties | Keep `SpecialColors.negative_infinity` and `.positive_infinity`; warn when converting to stock cmap | Minimum cmap release exposes both fields and state propagation |
| Distinct NaN and mask colors | `bad` handles NaN/masked data together | Optional `nan` override; masks continue to use `bad` | Keep `bad`; MCS currently rejects masked arrays in `colorize` | Minimum cmap release exposes `nan`; mask-specific policy remains downstream |
| Exact palette variants | Sized entries exist but family relationships are implicit | `available_sizes` plus exact `variant(size)` | Keep the MCS ColorBrewer/Tol family adapter | Minimum cmap release exposes and populates exact variants |
| Complete state propagation | Copy/reversal/update paths lose or alias state | Source-preserving copy and transform behavior | MCS keeps its own immutable evaluated value | Retire only the redundant propagation code, not the MCS facade |
| Lossless persistence | Customized catalog state may serialize as its name | Tagged full-state mapping for non-canonical objects | Persist MCS-owned schema | Adopt only after the serialized contract is released |

Tests should state capability behavior rather than assume a particular version:

- stock tests prove the fallback remains correct when a capability is absent;
- patched tests prove delegation preserves the richer behavior when it is present;
- boundary tests fail if downstream code bypasses the chosen probe and assumes a fork API.

## Alternatives Considered

- **One compatibility boolean for the entire fork.** Rejected because independent PRs merge and
  release independently.
- **Replicate every patched feature downstream.** Rejected where the fallback would become a second
  implementation rather than preserving existing MCS behavior.
- **Remove a fallback when an upstream PR merges.** Rejected; users receive releases, not upstream
  main, so retirement waits for a released minimum version.

## Deferred Work

- Record the retained integration commit and exact test counts after the first feature merge.
  The 242 passed / 2 skipped / 1 xfailed figure recorded on 2026-08-09 under
  `THIRD=1 uv run --no-dev --group test_thirdparty pytest` is stale: upstream has since added
  the crameri and cmocean data-parity tests, so re-measure before quoting it. There is still no
  capability to pin a commit *for*, so pinning waits.
- Add the concrete probe location once the public spelling of signed infinity fields is accepted.
- Decide whether separate NaN and mask colors belong in MCS; they are not required by its current
  API.

## Next Steps

1. Update this matrix with each integration merge and upstream release.
2. Add the first stock/patched test pair with signed-infinity support.
3. Remove rows once the fallback has been retired and the relevant released cmap is the floor.
