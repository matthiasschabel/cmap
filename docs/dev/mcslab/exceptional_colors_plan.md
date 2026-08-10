# Plan: additive exceptional-value colors for cmap issue #144

**Status:** Approved for implementation (revision 4)
**Last updated:** 2026-08-10
**Scope:** `src/cmap/_colormap.py`, `src/cmap/_external.py`, `tests/test_colormap.py`, `docs/`
**Baseline:** `fix/interpolation-aliasing` (PR #150), which is two commits above `upstream/main`

## Goal

Add four optional special colors to `Colormap` so the six distinguishable float classes can be
colored independently:

| Input class | Resolution order |
|---|---|
| negative infinity | `neg_inf` -> `under` -> first ramp color |
| below 0 (finite) | `under` -> first ramp color (unchanged) |
| in range | ramp (unchanged) |
| above 1 (finite) | `over` -> last ramp color (unchanged) |
| positive infinity | `pos_inf` -> `over` -> last ramp color |
| unmasked NaN | `nan` -> `bad` -> transparent |
| masked | `masked` -> `bad` -> transparent |

Hard constraint: with all four unset, every public output must be bit-identical to today,
including today's exceptions.

## Design

### Routing (`Colormap.__call__`)

Keep the `(N + 3, 4)` LUT contract of `Colormap.lut()` exactly as it is. Inside `__call__`
only, when at least one of the four colors is set, append four resolved rows to a local copy
of the LUT and give the four classes their own indices:

```
lut = self.lut(N=N, gamma=gamma, with_over_under=True)   # (N+3, 4), unchanged, still cached
N = len(lut) - 3
if self._has_exceptional:                 # slotted bool, False for every existing colormap
    lut = np.vstack((lut, self._exceptional_rows(lut)))  # (N+7, 4), local only
if bytes:
    lut = (lut * 255).astype(np.uint8)    # unchanged; extra rows convert with the rest
...
xa[mask_under] = N                        # unchanged
xa[mask_over]  = N + 1                    # unchanged
xa[mask_bad]   = N + 2                    # unchanged
if self._has_exceptional:
    xa[mask_neg_inf] = N + 3
    xa[mask_pos_inf] = N + 4
    xa[mask_nan]     = N + 5              # NaN, masked or not
    xa[mask_masked]  = N + 6              # masked wins over NaN and over infinity
rgba = lut.take(xa, axis=0, mode="clip")  # unchanged
```

The four appended rows resolve their own fallbacks, reading the already-computed under, over
and bad entries out of the LUT:

```
rows[0] = neg_inf_color or lut[N]         # under (itself already `under or lut[0]`)
rows[1] = pos_inf_color or lut[N + 1]     # over
rows[2] = nan_color     or lut[N + 2]     # bad
rows[3] = masked_color  or lut[N + 2]     # bad
```

Two properties fall out of this, and they are the whole argument for the change being safe:

- Each new row is **equal to the row the value would have hit anyway** when its color is
  unset, so re-routing is a no-op rather than something that needs a compatibility flag.
- The guard means an unconfigured colormap does not allocate, does not `vstack`, and does not
  run `isinf`. The existing hot path gains one slot read.

`_has_exceptional` is a fifth new slot, set once in `__init__`. It is not a property, so the
"one slot read" claim holds.

### Masks

The existing three masks keep their current definitions and their current placement, so
today's behavior survives even if a new mask is wrong. The new masks are computed only under
the guard, and only from arrays whose dtype already permits the ufunc. Exact logic:

```
xa = np.array(x, copy=True)                 # unchanged; drops any mask, keeps raw data
<non-native byte-order handling>            # unchanged, see below
is_float = xa.dtype.kind == "f"

if self._has_exceptional and is_float:      # BEFORE the scaling below, see note
    mask_neg_inf = np.isneginf(xa)
    mask_pos_inf = np.isposinf(xa)
else:
    mask_neg_inf = mask_pos_inf = False

if is_float:                                # unchanged
    xa *= N
    xa[xa == N] = N - 1

mask_under = xa < 0                         # unchanged; catches -inf
mask_over  = xa >= N                        # unchanged; catches +inf

if np.ma.is_masked(x):                      # unchanged shape, one variable factored out
    mask_masked = x.mask
    mask_nan    = np.isnan(xa) if is_float else False
    mask_bad    = mask_masked | mask_nan if is_float else mask_masked
else:
    mask_masked = False
    mask_nan    = np.isnan(xa)              # unchanged: still raises TypeError on object dtype
    mask_bad    = mask_nan
```

The infinity masks must be captured **before** `xa *= N`, and this is not a stylistic
choice. The scaling overflows: `float16` 65504 and `float32` 3.4e38 both become `inf` when
multiplied by 256. Verified. Today those finite values land in `mask_over` and take `over`,
which is correct and must not change; classifying after the multiply would hand them
`pos_inf` instead. `mask_under`/`mask_over` stay where they are, after the multiply, because
they need the scaled value.

Notes that the first revision of this plan got wrong or left implicit:

- The current masked branch does **not** already keep a separate NaN mask; splitting
  `mask_bad` into `mask_masked | mask_nan` is a new (behavior-preserving) decomposition.
- `False` rather than an allocated zero array, so `xa[False] = ...` is a cheap no-op and the
  non-masked path allocates nothing new.
- The `is_float` guard on `np.isnan` in the masked branch is what keeps a numeric
  object-dtype masked array working. Verified: with a real mask it succeeds today, and with
  `nomask` it raises `TypeError` from `np.isnan` today. Both must still be true after.
- `np.isneginf`/`np.isposinf` run only on float dtype, so they never see object dtype.
- `x.mask` is read, never written. `mask_masked | mask_nan` allocates a new array rather
  than using `|=` (the #148 invariant).

### Non-native byte order

`__call__` currently reinterprets non-native input with `.view()` instead of byteswapping
(`_colormap.py:418-422`), so a real big-endian array of `[-inf, +inf, NaN, 0.5]` on a
little-endian host currently maps **every** element to the first ramp color: `under`, `over`
and `bad` are all already unreachable for such input. Verified against the baseline.

This plan does not fix that. The new masks are computed **after** the byte-order step, in the
same place as the existing three, so non-native input keeps exactly one behavior for all
seven classes rather than gaining a split where the four new colors work and the three old
ones do not. The consequence, stated plainly: exceptional values in a non-native array are
not detected, before or after this change.

Codex proposed the opposite (capture the masks before the view). Rejected because it makes
the identity argument above conditional on dtype, and because it half-fixes a defect that
deserves its own one-line PR (`xa.byteswap().view(...)`, which is what matplotlib does). That
defect is worth filing separately; the tests here pin the current behavior rather than bless
it.

### Public surface

- Constructor: `neg_inf`, `pos_inf`, `nan`, `masked`, all `ColorLike | None = None`,
  keyword-only like the existing three.
- Properties: `neg_inf_color`, `pos_inf_color`, `nan_color`, `masked_color`; five new
  `__slots__` entries including `_has_exceptional`.
- `with_extremes()`: accepts the four new keywords. Semantics match the existing ones
  (omitted means cleared, which is the current, separately-disputed behavior of that method).
- `shifted()`: forwards the four, as it already forwards `under`/`over`/`bad`.
- `__eq__`: compares the four. Two colormaps that differ only in NaN color must not be equal.
- `_external.to_napari()`: `nan_color=(cm.nan_color or cm.bad_color)` when set. napari has a
  real `nan_color` parameter and cmap already forwards `bad_color` into it, so a colormap
  whose NaN color is set more specifically should not lose it. One line, no behavior change
  when `nan_color` is unset.
- `_repr_html_`: shows a patch for each exceptional color that is set. Without it, setting
  `nan="red"` renders no swatch at all, which is the first thing a reviewer tries.

### Deliberately not in scope

| Left alone | Why |
|---|---|
| `Colormap.lut()` shape and cache key | `(N+3, 4)` is the matplotlib-shaped contract consumers index into |
| Catalog schema (`CatalogItem`, `record.json`) | No catalog entry needs these; adding fields is a separate, data-facing change |
| The non-native byte-order defect | Pre-existing, affects `under`/`over`/`bad` too, deserves its own PR |
| Scalar `bytes=True` raising `ValueError` | Pre-existing (`Color` rejects a uint8 array); an API question, not this PR |
| `__reduce__`, `as_dict()`, pydantic | Already lossy for `under`/`over`/`bad`: a pickled `Colormap(bad="black")` loses it and compares unequal. Verified. The new fields inherit that limitation; making persistence lossless is roadmap item 8 |
| `to_mpl` and the other converters | matplotlib's `bad` means NaN *and* masked, so cmap's `bad` stays the right thing to forward; the finer classes have no representation. Documented as a known loss, no warning |
| `reversed()` | Already drops all extremes. Fixing that is roadmap item 7 |
| A shared `inf` color | Decided against, see Settled below |

The PR description must state the persistence and converter losses explicitly rather than
leaving a reviewer to find them.

## Verification

Two different artifacts, and conflating them is how a feature PR turns into a kitchen sink.

### Local harness, not shipped

A standalone script that reimplements the revised `__call__` and diffs it against the real
one over the full identity matrix, run against the baseline before any production edit. It
covers float, `float16`/`float32` overflow, integer, masked float, masked integer,
object-dtype masked with a real mask, object-dtype with `nomask`, real non-native `>f8`, 0-d,
0-d masked, scalars, both `bytes` modes, with and without `under`/`over`/`bad`, including the
two cases that raise (`nomask` -> `TypeError`, scalar `bytes=True` -> `ValueError`). Every
case must match. This lives in the worktree and in this note, never in the diff.

### Shipped tests

Small enough to review in one sitting:

1. **Routing.** One masked float array carrying every class at once, with all four colors
   set to distinct values, asserting the full expected color table in one comparison. This
   also covers precedence, since the array includes a masked NaN and a masked `+inf`.
2. **Fallback.** All four unset with `under`/`over`/`bad` set: output identical to the same
   colormap called before the feature existed. This is the additivity guard.
3. **Overflow.** `float16` 65504 takes `over`, not `pos_inf`, with the two set to different
   colors. The one case where a plausible implementation is silently wrong.
4. **State.** `__eq__` distinguishes on each new color, and `with_extremes()`/`shifted()`
   round trip them.
5. **`to_napari`** forwards `nan_color` when set and `bad_color` when it is not.

The masked input's `.mask` is asserted intact inside test 1 rather than as its own test, and
`bytes=True` rides along in test 1 rather than doubling the matrix.

## Settled

- **No shared `inf` color.** Repository owner, 2026-08-10. `pos_inf=c, neg_inf=c` already
  expresses the union, so the parent buys one keyword and costs a resolution layer in every
  chain, docstring, and test. Addable later without breaking anything.
- **`pos_inf`/`neg_inf` spelling**, matching `np.isposinf`/`np.isneginf`.
- **All four colors in one PR.** They share one mechanism and about ten lines of routing.
- **`bad` is retained deliberately, not redundantly.** `nan` and `masked` are exactly
  granular: each names one class, unambiguously. `bad` is the backward-compatible joint
  fallback that existing code already sets, and it stays because removing or narrowing it
  would break every current user. Either finer color may be set alone, and each falls back
  to `bad` when unset. The PR body should state this in a sentence so a reviewer does not
  have to reconstruct why three fields cover two classes.
- **Stacked on #150.** #150 is open, mergeable, unreviewed, and inserts
  `interpolation=self.interpolation,` into the same `with_extremes` constructor call this
  change extends. Branch from `fix/interpolation-aliasing` and declare the dependency rather
  than cutting from `upstream/main` and resolving the same insertion twice.

## Process

Build the PR now rather than pre-negotiating in the issue: five of the six prior PRs merged
without requested changes, and a working patch with tests answers the maintainer's "what is
the real world use case" better than another paragraph. Post the PR link as a comment on
#144 so it reaches the thread he already engaged with instead of arriving cold in the queue.
The PR body leads with the use case, not with "strictly additive"; additive is the floor,
not the argument. It states the persistence and converter losses, and the `bad` retention,
explicitly.

Nothing is posted, pushed to the fork, or opened as a PR without the owner saying so for
that specific submission.

Two spin-off defects found while planning, each worth its own filing:

- non-native byte order is reinterpreted rather than byteswapped, so every exceptional value
  in such an array is lost (matplotlib byteswaps).
- `cmap(scalar, bytes=True)` raises `ValueError` because `Color` rejects a uint8 array.
