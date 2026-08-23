# Draft: napari PR body for the NaN fix

**Status:** Active
**Last updated:** 2026-08-22
**Scope:** upstream napari. Branch `fix/nan-color-fast-math` in the fork, cut from
`upstream/main`. For the owner to push and open. An agent never submits.

Open as a **draft PR**. There is a real question in it for the maintainers (whether this
belongs here or in vispy), and a draft invites that answer without implying the diff is
final.

---

## Title

Fix `nan_color` being ignored where the shader's NaN test is optimized away

## Body

NaN renders as the first color of the colormap instead of `nan_color` on Apple hardware.
Reproduce with any colormap whose `nan_color` differs from its first ramp color:

```python
import numpy as np
import napari

data = np.full((64, 64), 0.5, dtype=np.float32)
data[:8] = np.nan
viewer = napari.Viewer()
viewer.add_image(data, colormap='viridis', contrast_limits=(0, 1))
# the NaN band draws dark purple, the bottom of viridis, not transparent
napari.run()
```

The NaN test in vispy's float pipeline is a self-comparison:

```glsl
// vispy/visuals/image.py, _APPLY_CLIM_FLOAT
if (!(data <= 0.0 || 0.0 <= data)) return data;
```

A compiler allowed to assume no NaN exists can prove `data <= 0.0 || 0.0 <= data` for every
value and fold the test to `false`. Metal enables fast math by default, so the branch never
runs, the NaN falls through to `clamp()`, and `clamp(NaN, 0, 1)` gives 0 on this driver.

- The same idiom guards `_APPLY_GAMMA_FLOAT`.
- The colormap's `bad_color` injection uses it too, so passing NaN along to the colormap
  does not help: it is folded there as well.

Measured on an Apple M5, GL 2.1 Metal - 90.5, GLSL 1.20. `v != v` and `!(v == v)` fold the
same way, and so does every form comparing against a single bound, literal or uniform,
because `d <= X || X <= d` is provable for any X.

What survives is two bounds the compiler cannot relate. Refuting this requires knowing
`flt_max >= -flt_max`, which a uniform withholds:

```glsl
uniform float u_flt_max;   // FLT_MAX
bool is_nan = !(data <= u_flt_max) && !(data >= -u_flt_max);
```

### What the diff does

`apply_clim` uses that test and returns a negative sentinel; `apply_gamma` passes the
sentinel through, since `pow()` of a negative base is NaN; a napari-side prologue on the
colormap decodes it ahead of vispy's own check. Decoding is opt-in and enabled only for the
image and tiled image nodes, because the mesh visual feeds the colormap an unclamped
normalized value and a surface vertex below the contrast limits must not be read as NaN.
Tile children build napari's image visual rather than vispy's, so an image too large for one
texture does not keep the bug a smaller one no longer has.

### What I would flag rather than hide

**This is arguably vispy's bug.** The idiom is theirs, in three places, and fixing it there
would help every consumer. I put it here because it needs no vispy release and because the
sentinel has to be decoded on the napari side anyway, where `nan_color` lives. Happy to take
it to vispy instead if you would rather.

**No in-shader NaN test is guaranteed.** A compiler assuming no NaN may fold any of them.
This one works because of what this compiler happens to prove, not because it is correct by
construction. A test asserts the shape of the expression so that rewriting it against a
single bound fails loudly, but that is a tripwire, not a proof. If you want something
stronger, the durable answer is classifying on the CPU and carrying the class in a sidecar
texture, which is a much larger change.

**One behavior change.** NaN now takes `nan_color`, which defaults to transparent. On
platforms where the test already worked, nothing changes. On Apple hardware a NaN that used
to draw as the bottom of the colormap now draws transparent, which is what the documented
behavior always was.

### Testing

`src/napari/_vispy/_tests/test_image_nan_color.py` renders a NaN through the image visual
and through the tiled path and checks the pixel. It fails on `main` on this hardware with
`NaN rendered (0, 0, 0), expected nan_color`.

---

## Notes for the poster, not for the PR

- Push the branch `fix/nan-color-fast-math` from the fork; it is cut from `upstream/main`
  and touches four files.
- The endpoint/matplotlib question is deliberately **not** in here. It is a separate
  documentation issue and the behavior should be left alone; see
  `napari_strict_extremes_issue.md`.
- If a maintainer asks for the infinity colors too, that work exists on the fork's
  `integration` but is a feature, not a fix, and should be its own conversation.
