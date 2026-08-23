# Draft: napari upstream reports

**Status:** Active
**Last updated:** 2026-08-22
**Scope:** upstream napari; drafts for the owner to post. An agent never submits.

Two separate things, and they should not be filed together. The first is a defect worth
fixing and changes nothing that works today. The second is a documentation gap where the
behavior itself should be left alone.

---

## 1. Bug: NaN renders as the bottom of the colormap on Apple Silicon

Post as a **bug report**. A PR can follow; the fix is small and does not change any
behavior that currently works.

### Title

`nan_color` is ignored on Apple Silicon: the shader's NaN test is folded away by fast math

### Body

On macOS with Apple's GL-on-Metal, a NaN pixel renders as the first color of the colormap
instead of `nan_color`. Reproduce by displaying an image containing NaN with any colormap
whose `nan_color` differs from its first ramp color.

The cause is in vispy's float pipeline, which napari uses:

```glsl
// vispy/visuals/image.py, _APPLY_CLIM_FLOAT
if (!(data <= 0.0 || 0.0 <= data)) return data;   // intended: pass NaN through
```

That is a self-comparison. A compiler permitted to assume no NaN exists can prove
`data <= 0.0 || 0.0 <= data` for every value and fold the test to `false`. Metal enables
fast math by default, so the branch never runs, the NaN falls through to `clamp()`, and
`clamp(NaN, 0, 1)` returns 0 on this driver. The same idiom appears in the colormap's own
`bad_color` injection, so both places miss it.

Measured on an Apple M5, GL 2.1 Metal - 90.5, GLSL 1.20. `v != v` and `!(v == v)` fold the
same way. What survives is a test against two bounds the compiler cannot relate, because
refuting it would require knowing `u_flt_max >= -u_flt_max` and a uniform withholds that:

```glsl
uniform float u_flt_max;   // 3.4028235e38
bool is_nan = !(data <= u_flt_max) && !(data >= -u_flt_max);
```

Worth saying plainly: no in-shader NaN test is guaranteed. A compiler assuming no NaN may
fold any of them. This one works because of what this compiler happens to prove, so a fix
should come with a way to notice when it stops working.

We have this working in a fork, along with classification of the infinities before the
contrast limits are applied, which is what makes `neg_inf`/`pos_inf` colors expressible at
all. Happy to open a PR for the NaN part alone, which is the part that is purely a fix.

---

## 2. Documentation: `low_color`/`high_color` are not matplotlib's `set_under`/`set_over`

Post as a **documentation issue**, not a change request. The behavior is right; only the
description is incomplete.

`Colormap.map` applies `high_color` at `values >= 1` and `low_color` at `values <= 0`.
matplotlib applies `set_over` strictly above `vmax` and `set_under` strictly below `vmin`,
leaving both endpoints on the ramp. The three fields arrived together in #7846 with
`nan_color` documented as "Equivalent to matplotlib's `bad_color`", so the set reads as the
matplotlib bad/under/over trio and the difference in the other two is easy to miss. A
downstream library converting its own colormaps has to decide between dropping those two
colors or silently changing their meaning.

**The inclusive rule should stay.** napari's automatic contrast limits are the data's own
minimum and maximum, so the extreme pixels of a freshly loaded image sit exactly on the
limits. Under a strict rule the stock `HiLo` colormap would flag nothing on exactly the
images it exists for, and its table is two entries, black and white, with the blue and red
held only in `low_color`/`high_color`, so there is nowhere to bake them. We implemented the
strict rule, found this, and reverted it.

The ask is one sentence in the docstrings saying these are inclusive at the endpoints and
therefore differ from matplotlib's `set_under`/`set_over`, so nobody else has to discover
it by rendering.

---

## Notes for the poster, not for the issues

- File (1) first and on its own. It is a defect with a reproducer, and it is what makes
  `nan_color` work at all on Apple hardware.
- (2) is low stakes. If it draws no interest, nothing is lost.
- Do not offer the strict endpoint change. We tried it and it breaks `HiLo`.
