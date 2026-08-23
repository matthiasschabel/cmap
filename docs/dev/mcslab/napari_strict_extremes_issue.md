# Draft: napari issue on low_color/high_color endpoint semantics

**Status:** Active
**Last updated:** 2026-08-22
**Scope:** upstream napari; draft for the owner to post. An agent never submits.

Post as an **issue** first, not a PR. It is a behavior change to a shipped colormap
(`HiLo`), so the maintainers should agree on the semantics before a diff exists. The fix
is ready locally if they want it.

---

## Title

`low_color` and `high_color` apply at the range endpoints, unlike matplotlib's `set_under`/`set_over`

## Body

`Colormap.map` applies `high_color` at `values >= 1` and `low_color` at `values <= 0`, so a
value sitting exactly on a contrast limit takes the out-of-range color. matplotlib applies
`set_over` above `vmax` and `set_under` below `vmin`, leaving both endpoints on the ramp.

```python
import numpy as np
from napari.utils.colormaps import Colormap

cmap = Colormap(
    [[0, 0, 0, 1], [1, 1, 1, 1]], low_color=[0, 1, 1, 1], high_color=[1, 1, 0, 1]
)
print(np.round(cmap.map(np.array([0.0, 1.0])) * 255))
# [[  0 255 255 255]     <- low_color; matplotlib gives the ramp start
#  [255 255   0 255]]    <- high_color; matplotlib gives the ramp end
```

matplotlib for comparison:

```python
import matplotlib as mpl
from matplotlib.colors import Normalize

cm = mpl.colormaps['gray'].copy()
cm.set_under('cyan')
cm.set_over('yellow')
print(cm(Normalize(0.0, 1.0)(np.array([0.0, 1.0])), bytes=True))
# [[  0   0   0 255]     <- ramp start
#  [255 255 255 255]]    <- ramp end
```

The three fields arrived together in #7846, and `nan_color` is documented there as
"Equivalent to matplotlib's `bad_color`". The set reads as matplotlib's bad/under/over
trio, so the difference in the other two is easy to miss.

I do not think the inclusive comparison was a deliberate choice. The 2D image path clamps
data into the contrast limits before the colormap sees it, which turns every out-of-range
value into exactly 0 or 1, and `>= 1` is the only test that still catches them afterwards.
The cost is that it also catches values that were in range to begin with.

### Why it matters downstream

A library converting its own colormaps into napari cannot map its under/over colors onto
these two without changing their meaning, so it has to drop them and warn. That is the
position we are in, and the colors are lost on the way into the viewer.

### What we did locally, if it is useful

Classifying before the clamp rather than after removes the need for the inclusive test.
Our fork emits a class marker from `apply_clim` for NaN, both infinities, and finite
out-of-range values, decodes it in the colormap function ahead of the LUT lookup, and uses
strict comparisons in `Colormap.map`. The thumbnail stops clipping before mapping for the
same reason. Happy to open a PR if the semantics are agreed.

### One knock-on worth naming

`HiLo` changes. Its table is a plain black-to-white ramp and the blue and red come from
`low_color`/`high_color`, so under the current rule they appear at the range endpoints and
reproduce ImageJ's LUT, where the two colors are baked in. Under a strict rule they mark
only values outside the limits. That is arguably what a saturation indicator should do, and
it is a visible change to a stock colormap, so it is the part most worth a decision rather
than a patch.

---

## Notes for the poster, not for the issue

- The reproducer runs against released napari; it needs no fork.
- Do not lead with the fork's shader work. The question upstream is the semantics; the
  implementation only matters if they say yes.
- If they prefer the current behavior, the fallback downstream is to keep dropping
  under/over on conversion and document why, which is where we were before.
