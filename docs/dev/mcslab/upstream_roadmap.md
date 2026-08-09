# MCSLAB cmap upstream roadmap

**Status:** Active
**Last updated:** 2026-08-09
**Scope:** small upstream cmap contributions that allow MCSLAB to reduce its colormap layer

## Where this stands

Six pull requests are open against `pyapp-kit/cmap`, none reviewed yet. The queue is
**paused** after the last unambiguous defect; everything remaining needs either a maintainer
decision or a design argument, and stacking more unreviewed work raises the cost of rework if
an early PR is rejected.

| PR | Subject | Notes |
|---|---|---|
| #145 | napari `high_color` assigned to `nan_color` | queue item 1 |
| #146 | pydantic `__fields__` probed before `model_fields` | not in the original queue; found while testing #145 |
| #147 | `ColorStops.reversed()` corrupts its source | queue item 2 |
| #148 | unmasked NaNs lost in a masked array | queue item 3 |
| #149 | exceptional-value documentation | queue item 4; **stacked on #148**, which must merge first |
| #150 | interpolation rewritten on borrowed stops, and dropped by `with_extremes()` | queue item 5, plus the interpolation half of item 6 |

`integration` carries all six as `--no-ff` merges. Two of those merges hit trivial conflicts
where separate branches added tests at the same point in `tests/test_colormap.py`; production
hunks have never conflicted.

## Context

MCSLAB currently compensates for several cmap defects and owns capabilities that would be useful in
cmap itself. The changes must go upstream as independent, low-attention pull requests. This note
holds the complete sequence so individual PR descriptions do not need to explain the downstream
migration program.

The verified baseline is cmap commit
`8040ef777c9c7aee7959e6029ffcd1a97582f41e`. Every PR above is cut from it.

### Verified behavior of the baseline

Measured, not inferred. Worth keeping, because several of these were surprises that changed a
plan mid-flight.

| Input | Routed to | When that color is unset |
|---|---|---|
| below 0, negative infinity | `under_color` | first ramp color |
| above 1, positive infinity | `over_color` | last ramp color |
| NaN, masked | `bad_color` | transparent |

- Infinity is not special-cased anywhere. It falls out of the `xa < 0` and `xa >= N`
  comparisons, which is why the constructor docstring calling `bad` the "(NaN, inf)" color was
  simply wrong.
- Integer input indexes the LUT directly, so an integer 2 with `N=4` is a ramp color, not
  over. A negative integer index routes to `under`, it does not wrap around.
- A catalog entry can supply `under`/`over`/`bad` (`napari:HiLo`, `napari:nan`), so "if not
  provided, the first color is used" is false for named colormaps.
- `_norm_interp(None)` is `"linear"`, which is why any path that omits `interpolation`
  silently asserts linear.
- `Colormap.lut()` caches on `(N, gamma, with_over_under)`, so a warmed cache can hide stops
  corruption from a test. Do not warm the cache before the act in a regression test.
- `ColorStops.__init__` stores a passed ndarray **uncopied** (`_colormap.py:918`). #147 and
  #150 fix the two paths that wrote through that sharing; the general hazard remains and is
  not currently queued.
- matplotlib 3.11 carries the same masked/NaN line as cmap, comment included. #148 is a
  deliberate divergence, justified by cmap's own `docs/faq.md` promising that `bad` covers
  "NaN or masked".
- A numeric object-dtype masked array works today only because it never reaches `np.isnan`,
  which raises on object dtype. Any future widening of the NaN test must keep that guard.

## Current Decision

### Shipped upstream, awaiting review

Queue items 1 to 5, plus one defect found along the way, are the six PRs in the table above.
Nothing further should be submitted until at least one of them draws a response: the maintainer's
reaction to #148 in particular (a deliberate divergence from matplotlib) and to #150 (which
touches copy semantics) tells us how the remaining, more speculative items will land.

### Blocked on a maintainer decision

6. **Preserve the rest of the public state in copy/update paths.** #150 took the interpolation
   half, because `with_extremes` already forwards `name` and `category` and the omission was
   plainly accidental. What remains is genuinely undecided:
   - `with_extremes` drops `identifier`. Should a modified copy keep the original identifier,
     derive a new one, or have none?
   - An omitted `bad`/`under`/`over` currently **clears** any existing value. Preserving it is
     what a user expects, but then `with_extremes(under=None)` can no longer remove a color
     without a sentinel. That is an API design question, not a bug fix, and it should be asked
     as an issue before any code.
7. **Make reversal semantics complete and consistent.** `Colormap("name_r")` and
   `Colormap("name").reversed()` should agree, including swapping under/over while preserving
   bad. Needs the maintainer to confirm that swapping extremes is intended; it is defensible
   either way.
8. **Make persistence lossless.** Tagged state mapping for customized objects in
   pickle/Pydantic/psygnal, with the default `as_dict()` shape unchanged. The largest of the
   three and the one most likely to be rejected on scope.

### Deferred to the MCSLAB quarantine

These are features, not defects, and none of them should be proposed upstream while the six
bug fixes are unreviewed. Build them behind the MCS compatibility layer, where they do not
depend on review timing, and revisit upstreaming once the fix queue has a verdict.

9. **Shared and signed infinity colors.** `inf`, `neg_inf`, `pos_inf` with the fallback
   `sign-specific -> inf -> legacy under/over -> ramp endpoint`. The direct MCS migration
   capability, and the subject of the open discussion in #144. Note that #149, if merged,
   documents the current signed-infinity routing, which is the contract this feature extends
   rather than contradicts.
10. **Optionally split masked data from unmasked NaN.** `nan` and `masked` together, each
    resolving through `bad`. Lower priority than it looked: #148 makes the existing single
    `bad` behave correctly for mixed input, which was the practical complaint.
11. **Exact palette-variant selection.** `available_sizes` plus `variant(size)`, without
    changing the sampling meaning of `lut(N)`.
12. **Annotate verified ColorBrewer and Tol families.** Data-only follow-up to 11.

Each upstream item starts from current `upstream/main`. Merge into `integration` only after its
own test passes. Combine adjacent items only when the maintainer explicitly prefers the larger
review unit, or when the repository owner directs it (as happened for the `with_extremes` half
of item 6 in #150).

### Exceptional-value contract

The compatible proposed resolution is:

| Input class | Resolution order |
|---|---|
| masked | `masked -> bad -> transparent` |
| unmasked NaN | `nan -> bad -> transparent` |
| negative infinity | `neg_inf -> inf -> under -> first ramp color` |
| positive infinity | `pos_inf -> inf -> over -> last ramp color` |

Constructor arguments use cmap's unsuffixed convention; properties use `_color`. Every field uses
the existing `ColorLike` parser, including RGBA tuples.

Making `bad` a fallback for infinity is not additive. A cmap configured only with `bad` currently
routes infinities through under/over or the ramp endpoints; changing those colors requires an
explicit compatibility decision. Do not introduce a hidden activation rule or a
`bad_includes_inf` mode flag.

### Scope retained by MCSLAB

Do not upstream the MCS `ColormapSpec`, `SpecialColors` aggregate, mutable registry, physical-limit
normalization, GUI schema, evaluated-value facade, or editing transforms as one package. Upstream
only the generally useful primitives. MCS consumes released capabilities through feature probes and
keeps application policy downstream.

## Alternatives Considered

- **One comprehensive colormap PR.** Rejected because state fixes, exceptional-value policy, and
  catalog variants have independent risk and review ownership.
- **A public upstream `SpecialColors` object.** Rejected because flat constructor fields already fit
  cmap's API and avoid a second construction model.
- **Make `bad` universally mean all non-finite values.** Coherent but breaking; retain as an explicit
  major-version option if the maintainer rejects legacy infinity fallbacks.
- **Expand the N+3 LUT for every exceptional class.** Rejected because it breaks Matplotlib-shaped
  consumers. Rich dispatch remains in `__call__`.
- **Make `lut(N)` choose a published size variant.** Rejected because sampling and selecting an
  independently designed palette are different operations.

## Deferred Work

- Final public spelling of signed fields remains a maintainer decision. `neg_inf`/`pos_inf` is
  preferred over `minf`/`pinf` because it is unambiguous.
- Converter behavior for targets that cannot represent the richer classes needs a per-target loss
  policy. Warnings should identify the user callsite and follow Python's normal warning filter.
- Separate mask coloring is useful but not required for current MCS, whose `colorize` rejects masked
  arrays.
- `ColorStops.__init__` stores a caller's ndarray without copying. Two write-through paths are
  fixed (#147, #150); nothing has decided whether the constructor should copy in general. Doing
  so would change the memory profile of every catalog construction, so it needs a measurement,
  not an opinion.
- The six feature worktrees under `~/GitHub/cmap-feat/` are retained while the PRs are open. Each
  holds its own `.venv`, so they are not free; remove them once the corresponding PR is merged or
  closed.

## Next Steps

1. **Wait.** No new upstream submissions until at least one of the six PRs draws a maintainer
   response.
2. If #148 is accepted, #149 can be rebased onto `main` and unstacked.
3. Ask the item 6 API question as an issue, not a PR: what should an omitted `bad`/`under`/`over`
   mean in `with_extremes`, and should a modified copy keep the original `identifier`?
4. Meanwhile, build the signed-infinity work (item 9) behind the MCS compatibility layer, where
   it does not wait on review.
5. Re-plan items 7 and 8 only after the maintainer's posture on copy semantics is known.
