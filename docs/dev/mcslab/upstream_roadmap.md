# MCSLAB cmap upstream roadmap

**Status:** Active
**Last updated:** 2026-08-13
**Scope:** small upstream cmap contributions that allow MCSLAB to reduce its colormap layer

## Where this stands

**Seven of the nine pull requests are merged.** The maintainer took every defect fix without
requesting changes, which answers the question the queue was paused on: small, single-defect PRs
against this project land.

| PR | Subject | State |
|---|---|---|
| #145 | napari `high_color` assigned to `nan_color` | merged |
| #146 | pydantic `__fields__` probed before `model_fields` | merged; found while testing #145 |
| #147 | `ColorStops.reversed()` corrupts its source | merged |
| #148 | unmasked NaNs lost in a masked array | merged |
| #149 | exceptional-value documentation | merged; was stacked on #148 |
| #150 | interpolation rewritten on borrowed stops, and dropped by `with_extremes()` | merged as `f0a4aec` |
| #151 | per-class colors for `neg_inf`, `pos_inf`, `nan`, `masked` (#144) | **open**, ready; maintainer approved 2026-08-13 pending @jni's read of the API |
| #152 | non-native byte order reinterpreted instead of byteswapped | merged as `d1521b1` |
| #155 | state lost by `reversed()`, `with_extremes()`, pickle, and `as_dict()` | **open**, draft; stacked on #151 |

#150 and #152 merging left #151 conflicting, so `feat/exceptional-colors` was rebased onto
`upstream/main` on 2026-08-13. The conflict was the predicted one, two lines apart in `__call__`:
keep the byteswap line and #151's mask block. Pre-rebase SHA `0587c892`, post-rebase `9380611`.
Force-pushed the same day with the owner's authorization; #151 now reports mergeable and its diff
is the one feature commit alone, since #150's two commits are no longer riding along.

#155 is cut from the rebased #151, so it carries the same resolution. Its plan, dispositions from
two codex review passes, and PR body are in `state_preservation_plan.md` and
`state_preservation_pr_body.md`.

`integration` was updated in the same pass: merging `fix/state-preservation` brought
`upstream/main` into the history for the first time since the five-PR landing, and one conflict
in `with_extremes()` where integration still held the old clearing behavior. Resolved to the
preserving side. 264 passed, 1 skipped, 1 xfailed under
`THIRD=1 CI=1 pytest` with `pydantic-compat` added; note that `pydantic_compat` is in no
dependency group, so `tests/test_model_fields.py` silently skips without `--with pydantic-compat`.

Upstream also shipped its own `crameri` v8 correction (#143) on top of our orientation fix
(#141).

### Reconciled with upstream, 2026-08-13

#150 and #152 were verified landed by reading the changed lines in `upstream/main` rather than
by ancestry, which squash-merging makes useless: the byteswap line, `_with_interpolation`, both
of its call sites, and all three regression tests are present verbatim, with no maintainer
edits during merge. Their branches and worktrees are gone locally, and the two branches were
deleted from the fork with the owner's authorization. `origin/main` was fast-forwarded to
`f0a4aec` so future branches cut from the fork's default start from the right base.

What remains is exactly the two open PRs. `feat/exceptional-colors` and `fix/state-preservation`
are the only local branches besides `main` and `integration`, each with a worktree, each tracking
its fork branch, and each matching its PR head SHA (`9380611`, `4302481`; both report mergeable).

`integration` is `upstream/main` plus those two branches plus this `docs/dev/mcslab/` tree and
the `mkdocs.yml` exclusion. Checked, not assumed: `upstream/main` and both open branches are
ancestors of it, and diffing `integration` against `fix/state-preservation` over `src`, `tests`,
`docs`, and `mkdocs.yml` yields the eleven mcslab notes and the four `mkdocs.yml` lines, nothing
else. No code has drifted onto `integration` that is not in a PR.

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

### Shipped upstream

Queue items 1 to 5, plus one defect found along the way, are the six PRs in the table above;
five are merged. #148 merging is the informative one: it was a deliberate divergence from
matplotlib's identical masked/NaN handling, justified from cmap's own `docs/faq.md`, and it
was accepted as-is. The maintainer's posture on copy semantics is still unknown, because #150
carries that question and has not been reviewed.

### Unblocked by the maintainer's #151 comment (2026-08-12)

The maintainer's comment on #151 (5266155726) asked, unprompted, whether the extreme colors
survive `reversed`, `with_extremes`, pickle, and `as_dict`/pydantic, and said "now is as good
a time as any" to address what was already broken. That is the opening items 6 to 8 were
waiting for. Verified on `main` 2026-08-12: none of the four channels preserve even the old
`under`/`over`/`bad` (`reversed` also drops interpolation; pickle also drops name and
category; a catalog colormap with extremes pydantic-serializes to its bare qualified name).
matplotlib 3.11.1 preserves in all four, and swaps under/over on `reversed()`.

Owner-settled policy, 2026-08-12 (reply draft in `exceptional_colors_maintainer_reply.md`,
not yet posted):

- `reversed()` swaps the directional pairs (`under`/`over`, `neg_inf`/`pos_inf`), preserves
  `bad`/`nan`/`masked` and interpolation. Cite matplotlib.
- `with_extremes()` preserves anything not passed, citing matplotlib; clearing one color
  means constructing a fresh Colormap, the same limitation matplotlib has.
- `__reduce__` carries full constructor state.
- `as_dict()` gains optional keys (interpolation plus the seven extreme colors) emitted only
  when set, so existing payloads are unchanged; the pydantic serializer falls back to dict
  form when a catalog colormap carries extremes. `_validate` already accepts the keys.
- `bad` stays: it is the umbrella tier parallel to under/over above neg_inf/pos_inf, and
  `masked` is the only class that can mark in-range values (predicate masking as a cheap
  contour or polarity overlay).

The maintainer answered on 2026-08-13: "I suppose we should go ahead and split that fix out into
a new PR. And it needn't hold this one up either." Singular PR, and no dependency in the other
direction. Items 6, 7, and 8 are therefore **one** PR, #155, and there is no fifth PR extending
the fix to #151's four colors: #155 is stacked on #151 and covers all seven at once.

The `identifier` question is settled in #155 rather than asked: identifier travels with the
name, so `reversed()` and `shifted()` let it re-derive and `with_extremes()` preserves it. The
double-reversal consequence is stated in the PR body for the maintainer to overrule.

6. **Preserve the rest of the public state in copy/update paths.** #155.
7. **Make reversal semantics complete and consistent.** #155, including the `__init__` swap that
   makes `Colormap("x_r")` agree with `Colormap("x").reversed()`.
8. **Make persistence lossless.** #155, with two documented exclusions: `info` is not restored
   through the dict form, and the converters still cannot express the finer classes.

### Features

The hold on proposing these upstream is lifted: five defect fixes merged without requested
changes, which answers the question the hold existed for. Items 11 and 12 stay in the MCSLAB
quarantine for now.

9 and 10 were merged into one change and **implemented** on 2026-08-10, on
`feat/exceptional-colors`, stacked on #150. `neg_inf`, `pos_inf`, `nan`, and `masked`, each
falling back to the color its class uses today. No shared `inf` parent: the owner decided
against it, since `pos_inf=c, neg_inf=c` already expresses the union. Design, dispositions,
and the two Codex review passes are in `exceptional_colors_plan.md`; the PR body is in
`exceptional_colors_pr_body.md`. Pushed to the fork and opened as draft #151 on 2026-08-10
with the owner's authorization. Because a cross-fork PR cannot target a fork branch, #151 is
based on `main` and shows #150's two commits alongside ours; its diff shrinks to ours alone
once #150 merges.
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

- The byte-order defect became #152 on 2026-08-10. `git log -L` pinned it to the numpy 2
  migration in #60 (`de1fbe2`), which dropped the `.byteswap()` when `ndarray.newbyteorder()`
  was removed. Codex accepted it with no blocking findings. Merging it into `integration`
  conflicted with #151 exactly where predicted, two lines apart in `__call__`; keep the
  byteswap line and #151's mask block. With both merged, exceptional colors now resolve
  correctly for non-native input, which is the outcome the #151 plan deferred to this fix.
- `cmap(scalar, bytes=True)` raises `ValueError`, because `parse_rgba` accepts a 3-element
  integer array but not a 4-element one. Filed as issue #153 on 2026-08-10, not a PR: the
  scalar overload promises a `Color` whatever `bytes` says, while the `bytes` documentation
  promises `uint8`, and only the maintainer can say which contract wins. The two live
  resolutions are a `(4,)` `uint8` return with the overloads split on `bytes`, or
  quantize-then-`Color`, which keeps the scalar contract and shows `bytes=True` only in the
  rounding. Draft body kept as `bytes_scalar_issue.md`. We offered to send the PR either way.
- `with_extremes()` and `shifted()` pass every keyword to `type(self)` unconditionally, so a
  subclass overriding `__init__` with a narrower signature breaks. Verified: released cmap
  already does this in `shifted()`, and #150 extends it to `with_extremes()`. The
  exceptional-color PR follows the surrounding pattern rather than diverging from it. If the
  maintainer wants narrow subclasses supported, it belongs on #150.

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
- Each worktree carries its own `.venv` and costs about 1.1 GB. Only `exceptional-colors` and
  `state-preservation` remain; remove each as its PR closes.

## Next Steps

1. Mark #155 ready for review once #151 draws @jni's response, so the two do not arrive together.
2. Build the signed-infinity work (item 9) behind the MCS compatibility layer, where it does not
   wait on review.
3. Raise the MCS cmap floor and delete the corresponding fallbacks once the seven merged fixes
   ship in a release. They are in `main` but unreleased; the current floor is `cmap>=0.7`.
