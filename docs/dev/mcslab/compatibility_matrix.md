# MCSLAB cmap compatibility matrix

**Status:** Active
**Last updated:** 2026-08-09
**Scope:** behavior expected from released cmap and the `matthiasschabel/cmap` integration branch

## Context

MCSLAB must remain correct with the released cmap package while incrementally delegating to public
features under upstream review. This matrix records the capability probe, stock fallback, and
retirement condition for each migration seam.

## Current Decision

`integration` now carries six merged fix branches on top of upstream
`8040ef777c9c7aee7959e6029ffcd1a97582f41e`. **None of them is a fork capability**: all six are
defect fixes or documentation with no new API surface, so there is still nothing for MCSLAB to
probe for and no fallback to write. The first genuine capability will be the signed-infinity
work (roadmap item 9).

Two rows below are partly overtaken by those fixes:

- *Distinct NaN and mask colors* is less pressing than it looked. cmap PR #148 makes the single
  existing `bad` color apply to unmasked NaNs inside a masked array, which was the practical
  failure. A separate `nan`/`masked` split remains optional.
- *Complete state propagation* is partly delivered. #147 stops `ColorStops.reversed()`
  corrupting its source, and #150 stops construction rewriting borrowed interpolation and makes
  `with_extremes()` carry it. What remains is `identifier` and the omitted-versus-explicit-`None`
  question, both of which need a maintainer decision before MCS can rely on anything.

| Capability | Released cmap 0.7.2 | Patched integration target | MCS fallback while absent | Retirement condition |
|---|---|---|---|---|
| Signed infinity colors | Not represented | `neg_inf` and `pos_inf` constructor support, with corresponding properties | Keep `SpecialColors.negative_infinity` and `.positive_infinity`; warn when converting to stock cmap | Minimum cmap release exposes both fields and state propagation |
| Shared infinity color | Not represented | Optional `inf` fallback shared by both signs | Expand to the two MCS signed fields if needed; otherwise leave unset | Minimum cmap release exposes `inf` |
| Distinct NaN and mask colors | `bad` handles NaN/masked data together | Optional `nan` and `masked` overrides | Keep `bad`; MCS currently rejects masked arrays in `colorize` | Only if MCS adopts masked input after the upstream API ships |
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
  As of the six fix merges, the patched suite is 242 passed, 2 skipped, 1 xfailed under
  `THIRD=1 uv run --no-dev --group test_thirdparty pytest`. There is still no capability to pin
  a commit *for*, so pinning waits.
- Add the concrete probe location once the public spelling of signed infinity fields is accepted.
- Decide whether separate NaN and mask colors belong in MCS; they are not required by its current
  API.

## Next Steps

1. Update this matrix with each integration merge and upstream release.
2. Add the first stock/patched test pair with signed-infinity support.
3. Remove rows once the fallback has been retired and the relevant released cmap is the floor.
