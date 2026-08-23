# Multivariate colormaps in cmap

**Status:** Active
**Last updated:** 2026-08-16
**Scope:** `cmap` upstream, issue [#90](https://github.com/pyapp-kit/cmap/issues/90); survey of prior art and a proposed integration

## Recommendation up front

**Revised 2026-08-16 after a critique pass. Read [§0](#0-the-finding-that-changes-the-verdict)
first; the design in Part 4 stands only if the prerequisite there is met.**

If cmap builds this, it should add two sibling classes, `BivarColormap` and `MultivarColormap`,
next to `Colormap`. Do not generalize `Colormap` itself. Name them exactly as matplotlib does.

The genuinely novel contribution cmap can make is **the catalog, not the math**. The 2-D lookup
math is roughly 150 lines and matplotlib already shipped it. What nobody has shipped is a curated,
licensed, cross-referenced catalog of multivariate colormaps with per-colormap documentation and
perceptual analysis. That is exactly what cmap already is for 1-D.

Two cmap-specific design deviations from matplotlib were originally proposed. After critique, only
the first is worth raising, and neither is worth insisting on:

1. Split matplotlib's `shape` parameter into an orthogonal *domain geometry* and *out-of-domain
   policy*. Lossless in both conversion directions, but near-zero user benefit against a real
   "why is this different from matplotlib" cost. Propose, do not defend.
2. ~~Offer mixing in a perceptual space, not only additive/subtractive sRGB.~~ **Withdrawn.** See
   [§0.4](#04-the-oklab-recommendation-was-wrong).

Per the upstream conventions, this is far beyond an unambiguous bug fix, so it starts as a design
comment on #90 and waits for a signal from the maintainer before any code gets written.

---

## 0. The finding that changes the verdict

The first draft of this note led with "Ræder designed 19 bivariate colormaps and matplotlib
released only 3, so 16 are homeless." **That framing was wrong.**

### 0.1 The 16 are queued for matplotlib, not rejected

[matplotlib #30528](https://github.com/matplotlib/matplotlib/issues/30528), "Better bivariate and
multivariate colormaps", is open, filed 2025-09-08 **by Ræder himself**, offering the full set for
inclusion. It is one milestone of [matplotlib project 9](https://github.com/orgs/matplotlib/projects/9),
whose milestone list contains verbatim: *"Select bivariate and multivariate colormaps to include in
matplotlib."* The stated blocker is not quality. It is: *"we will need to have a discussion on the
naming scheme 😅"*.

So the headline value-add is not "colormaps nobody will ship". It is "colormaps matplotlib will
probably ship eventually, and cmap could ship sooner." That is a much weaker claim, and it has a
shelf life.

The same project board tracks [#30527](https://github.com/matplotlib/matplotlib/issues/30527), the
bivariate colorbar. So the 2-D legend gap noted in §1.1 is a known, tracked milestone, not a
permanent hole. Any cmap argument resting on it also has a shelf life.

**The one durable asymmetry:** matplotlib is blocked on flat-namespace naming. cmap has
namespaces. `colorstamps:cone` and `steiger:bremm` cost cmap nothing and are exactly the problem
#30528 is stuck on. That is a real, structural advantage and it is the strongest remaining
argument for cmap doing this at all.

### 0.2 The interop half of cmap's value is zero here

cmap's other half is being a lingua franca: `to_matplotlib`, `to_vispy`, `to_pygfx`, `to_napari`,
`to_bokeh`, `to_plotly`, `to_altair`, `to_gee`, `to_pyqtgraph`. For a bivariate colormap there is
**exactly one** viable target, matplotlib ≥3.10. Nine converters become one. Half of cmap's reason
to exist does not apply to this feature.

### 0.3 The claimed downstream demand does not exist

The first draft asserted that napari, ndv and pygfx "reimplement the combination themselves
because there is no object that owns it," and inferred demand. Checked: **zero** issues mentioning
bivariate or multivariate colormaps in either `napari/napari` or `pyapp-kit/ndv`. ndv #57
("add back channel modes and multi-channel display") is **closed** — ndv solved multichannel
without needing this object. That is evidence against the claim, not for it.

cmap #90 itself has one non-maintainer author and one maintainer reply, "thanks for the heads up!".
That is notification, not demand.

Note also that cmap's own `cmap:` namespace is already 7 single-hue ramps (`red`, `green`, `blue`,
`cyan`, `magenta`, `yellow`, `white`) — the exact primaries used for microscopy compositing. The
use case is already served by handing users per-channel ramps.

### 0.4 The OKLab recommendation was wrong

Part 4.3 listed four combination rules in one table, including `"oklab"` described as "the
perceptually correct one." That conflates two different operations:

- **Compositing** N channels (family A, microscopy) is *additive*. Physically it is a sum of
  radiances, so the correct operation is addition in linear light. matplotlib's `sRGB_add` sums
  gamma-encoded values, which is not right either, but it is at least monotone and additive.
  **A mean in OKLab is not additive at all**: two bright channels average to one mid color instead
  of a brighter one. It is categorically the wrong operation here.
- **Constructing** a 2-D palette from two ramps (families B and C) is a *blend*, and there a
  perceptual space is defensible.

`"multiply"` belongs only to the second, and `"add"`/`"subtract"` only to the first. One table
covering `MultivarColormap` was a mistake. Since cmap's plausible first increment is family B data
only, OKLab buys nothing and costs a new colorspace in `_color.py`. Dropped.

### 0.5 `MultivarColormap` may not deserve to be a class

What does it do that a user cannot write in one line?

```python
rgba = cm0(x0) + cm1(x1)      # this is sRGB_add
```

Its only real content is a *name* for a set of ramps that combine well. That is
`dict[str, tuple[str, ...]]` in a `record.json`, not a class, unless something downstream wants to
pass one object around — and §0.3 found nothing that does. The first draft called this "the
smallest useful increment"; it is more accurately the smallest increment whose usefulness is
unevidenced.

### 0.6 Maintenance cost, with local evidence

Every class in cmap carries pydantic schema, `__reduce__`, `as_dict()`, `__eq__`, `_repr_png_`,
`_repr_html_`, `__rich_repr__`, catalog integration, docs generation, and converters. Three of the
nine PRs in `upstream_roadmap.md` (#147, #150, #155) exist precisely because state preservation
across `reversed()` / `with_extremes()` / pickle / `as_dict()` was broken on the **one existing
class**. Two more classes triple that invariant surface, and each will grow the same family of
bugs. `BivarColormap` alone has more axis-symmetry state to lose than `Colormap` does
(`transposed`, per-axis `reversed`, `origin`, `domain`).

### 0.7 Revised verdict

| | |
|---|---|
| **Reward** | cmap becomes the reference 2-D colormap catalog in Python, ~1 year before matplotlib, with better naming. Speculative and time-limited. |
| **Risk** | Months of work; a maintainer who has not signaled; one conversion target instead of nine; no evidenced downstream consumer; a tripled invariant surface on a codebase that has already leaked bugs through that surface. |

**Do not build the full design in Part 4 on the current signal.**

The ordered next moves, cheapest first:

1. **Comment on matplotlib #30528**, not cmap #90. It is blocked on exactly the problem
   namespacing solves, and unsticking it is close to zero-cost. If matplotlib then ships the
   colormaps, cmap catalogs them through its normal matplotlib-mirroring practice and the work is
   done. cmap originates almost nothing today (its own namespace is 7 trivial ramps plus 4
   contributed maps); "wait and catalog" *is* cmap's relationship with matplotlib.
2. **Ask on cmap #90** whether the maintainer wants this at all, presenting the three-family
   taxonomy and §0.2's interop limitation honestly. Do not present a class design yet.
3. **Only on a yes**, build the minimum: a thin `BivarColormap` holding an `(N, M, 4)` LUT with
   `__call__`, `__getitem__`, `lut`, `_repr_png_`, plus catalog entries — copying matplotlib's
   `shape` verbatim so conversion is trivial. No `MultivarColormap`, no combination rules, no
   OKLab, no `domain`/`outside` redesign. Roughly 250 lines plus data, and it defers every
   contested decision.

Everything below is the survey and the full design, retained because it is what a "yes" would
build toward. It is not a plan of record.

---

## Part 1: state of the art

### 1.1 matplotlib (the compatibility target)

Trygve Magnus Ræder's GSoC 2024 work landed in `matplotlib.colors` via
[PR #28454](https://github.com/matplotlib/matplotlib/pull/28454) (merged 2024-08-23), closing a
feature request open since 2019 (mpl #14168). Verified against the matplotlib 3.11.1 installed in
this repo's `.venv`:

| Class | Role |
|---|---|
| `MultivarColormap` | N independent 1-D `Colormap`s plus a combination rule |
| `BivarColormap` | base class, 2-D `(N, M, 4)` lookup table |
| `SegmentedBivarColormap` | `BivarColormap` from a small `(k, l, 3)` patch, bilinearly supersampled |
| `BivarColormapFromImage` | `BivarColormap` from a full `(N, M, 3\|4)` array |
| `MultiNorm` | a tuple of `Normalize` objects, one per variate |

`MultivarColormap.__init__(colormaps, combination_mode, name)` accepts only two combination
modes, `'sRGB_add'` (`sum(colors)`) and `'sRGB_sub'` (`1 - sum(1 - colors)`).

`BivarColormap.__init__(N=256, M=256, shape='square', origin=(0, 0), name=...)`. Notable methods:

- `__call__((X0, X1), alpha=None, bytes=False)` returns `X.shape + (4,)`.
- `__getitem__(0 | 1)` returns a 1-D `ListedColormap`: the slice through `origin` along the other
  axis. This is how the per-axis colorbars get built.
- `transposed()`, `reversed(axis_0=True, axis_1=True)`, `resampled(lutshape, transposed=False)`.
- `with_extremes(*, bad, outside, shape, origin)`.
- `shape` is one of `'square' | 'circle' | 'ignore' | 'circleignore'`.

**What actually shipped.** Only three of each:

- bivariate: `BiPeak`, `BiOrangeBlue`, `BiCone`
- multivariate: `2VarAddA`, `2VarSubA`, `3VarAddA`

`BiOrangeBlue` is a 2×2×3 patch, i.e. four corner colors bilinearly interpolated. `BiPeak` and
`BiCone` are 65×65×3 patches. All of `_cm_bivar.py` is 96 KB.

**What did not ship.** The GSoC reference page lists 19 designed bivariate colormaps:
`BiOrangeBlue`, `BiGreenPurple`, `BiPeak`, `BiAbyss`, `BiFlat`, `BiCone`, `BiFunnel`, `BiDisk`,
`BiCut`, `BiBarrel`, `BiYellows`, `BiGreens`, `BiBlues`, `BiReds`, `BiHsv`, `BiFourCorners`,
`BiFourEdges`, and variants. Sixteen exist as designed, evaluated colormaps with no home in a
released library.

**What is still missing in matplotlib.** The data path works (`ax.imshow((a, b), cmap='BiPeak')`
and `ax.pcolormesh((a, b), cmap='BiOrangeBlue')` both succeed, producing a `MultiNorm` +
`SegmentedBivarColormap` pair), but `fig.colorbar(pm)` on a bivariate mappable raises inside norm
handling in 3.11.1, and neither the `fig.colorbars(...)` nor `fig.colorbar_2D(...)` API from the
PR description exists. The 2-D legend problem is unsolved upstream. That matters for cmap because
it means a good `_repr_png_` / `_repr_html_` for a 2-D colormap is not redundant with matplotlib.

### 1.2 The design literature Ræder built on

Two blog posts, [Designing 2D colormaps](https://trygvrad.github.io/designing-2d-colormaps/) and
[Multivariate colormaps for n dimensions](https://trygvrad.github.io/multivariate-colormaps-for-n-dimensions/),
extend the viridis/Kovesi perceptual-uniformity argument from curves to surfaces in colorspace.

For 2-D: the colormap is a slice through the sRGB gamut in CAM02-LCD or CIELAB, with the six
category pairings enumerated (sequential × sequential, × diverging, × cyclic; diverging ×
diverging, × cyclic; cyclic × cyclic). Because red-green CVD collapses the `a*` axis, the
preferred designs slice along `b*` (blue-yellow).

For N-D: rather than an intractable 256^n table, use *n independent 1-D LUTs combined additively
in sRGB*. Five stated figures of merit: each channel perceptually uniform; channels of similar
lightness and saturation; perceptual distance between channels maximized; equal mixing yields
gray; no combination leaves sRGB. The additive-in-sRGB choice is explicitly pragmatic, chosen over
mixing in a uniform space because the conversions are expensive and numerically unstable.

**This is the key structural insight for the API:** separable N-D colormaps and non-separable 2-D
colormaps are different objects with different data requirements, not one generalization. That is
why matplotlib made them siblings, and cmap should too.

### 1.3 colorstamps

[trygvrad/colorstamps](https://github.com/trygvrad/colorstamps) (MIT) is the reference
implementation the matplotlib data was generated from. It defines ~24 named 2-D colormaps
*procedurally* as geometric shapes in CAM02-LCD: `flat`, `disk`, `peak`, `cone`, `abyss`,
`funnel`, `hsv`, `fourEdges`, `fourCorners`, `barrel`, `cut`, `blues`, `reds`, `greens`,
`yellows`, `orangeBlue`, `greenPurple`, `greenTealBlue`, `redPurpleBlue`, plus eight `teuling*`
variants reproducing Teuling et al.

Parametric knobs: `l` (LUT size), `rot` (hue rotation), `J` (lightness range), `sat`,
`limit_sat` (`'individual'` or `'shared'`), `a`/`b` (axis ranges). Radial maps take orientation
postfixes (`'cone tr'`).

Relevant caveat for cmap: the generator needs CAM02-LCD ↔ sRGB conversion, which colorstamps gets
from `colorspacious`. cmap has no dependency beyond numpy, so cmap should ship **precomputed
LUTs**, not the generator.

Sibling repo [trygvrad/multivariate_colormaps](https://github.com/trygvrad/multivariate_colormaps)
(MIT) generates the additive/subtractive N-channel sets: 2 variants at n=2, 4 at n=3, 2 at n=4,
single maps for n=5..8, in both additive (black origin) and subtractive (white origin) families.
The subtractive family is reported as more perceptually uniform.

### 1.4 The information-visualization line

Distinct from the colorspace-geometry line above, and mostly predating it.

- **Trumbo 1981**, "A Theory for Coloring Bivariate Statistical Maps", *The American Statistician*
  35(4):220-226. The founding statement: effective schemes are continuous transformations from a
  color model to the unit square, under restrictions on hue, saturation and brightness. It
  criticizes the U.S. Census Bureau scheme on those grounds, which is the `census.blueyellow` that
  `pals` still ships.
- **Teuling, Stöckli & Seneviratne 2011**, "Bivariate colour maps for visualizing climate data",
  *Int. J. Climatology* 31(9):1408-1412. Source of the eight `teuling*` variants in colorstamps
  and of `ColorMap2DTeuling2` in pycolormap-2d.
- **Bremm et al. 2011**, from assisted descriptor selection.
- **Ziegler et al. 2008**, from visual market-sector analysis.
- **Steiger, Bernard et al. 2015**, "Explorative Analysis of 2D Color Maps" (WSCG 23:151-160), and
  **Bernard, Steiger, Mittelstädt, Thum, Keim, Kohlhammer 2015**, "A survey and task-based quality
  assessment of static 2D colormaps" (Proc. SPIE 9397). The survey re-implements the prominent
  designs from **over 50 related works**, in sRGB, CIELAB and HSV variants, and scores seven
  quality measures against seven analysis tasks. This is the most complete published inventory of
  the field and the natural source for a cmap catalog's `info` and `tags` metadata.

Implementations: [spinthil/pycolormap-2d](https://github.com/spinthil/pycolormap-2d) ships six of
these as `ColorMap2D{Bremm,CubeDiagonal,Schumann,Steiger,Teuling2,Ziegler}` with an API of
`cmap = ColorMap2DBremm(range_x=..., range_y=...); color = cmap(x, y)`.
[fraunhofer-igd-iva/colormap-explorer](https://github.com/fraunhofer-igd-iva/colormap-explorer) is
the Java tool from the survey authors. [dominikjaeckle/Color2D](https://github.com/dominikjaeckle/Color2D)
is a JS port.

### 1.5 The cartographic line

Almost entirely 3×3 discrete grids, not continuous surfaces.

- **Joshua Stevens'** widely-copied method: build two 3-class sequential ramps in contrasting
  hues, stretch each to a square, rotate one 90°, and composite with *multiply* or *darken*, then
  boost saturation in the high-high corner. He states plainly that a bivariate scheme cannot
  simultaneously be colorblind-safe, photocopy-safe and print-safe; at least two must be given up.
- **ArcGIS Pro** ships bivariate colors symbology at 2×2, 3×3 or 4×4, treating the scheme as "the
  product of two discrete color schemes," with a legend rotatable 45° to emphasize an extreme.
  Defined-interval and standard-deviation classification are unavailable for it.
- **R `biscale`** (Prener) is the standard R implementation, carrying Stevens' schemes.
- **R `pals`** (Wright) has the largest single collection: `arc.bluepink`, `brewer.{qualbin,
  divbin, divseq, qualseq, divdiv, seqseq1, seqseq2}`, `census.blueyellow`, `tolochko.redblue`,
  `stevens.{pinkgreen, bluered, pinkblue, greenblue, purplegold}`, plus `vsup.{viridis, redblue}`.
  Mostly 3×3, some 3×2. It also carries per-palette criticism, e.g. `arc.bluepink` puts white in
  the lower-left corner so low values are indistinguishable from missing data.
- **[RaczeQ/bivario](https://github.com/RaczeQ/bivario)** is the modern Python entry and the only
  one mixing in a perceptual space: `AccentsBivariateColourmap`, `CornersBivariateColourmap`,
  `MplCmapBivariateColourmap`, `NamedBivariateColourmap`, all blending in **OKLab**.

### 1.6 Value-Suppressing Uncertainty Palettes

Correll, Moritz & Heer, CHI 2018 ([uwdata/vsup](https://github.com/uwdata/vsup)). A VSUP is a
bivariate value × uncertainty palette whose domain is a *tree*, not a grid: the range of the value
channel narrows as uncertainty rises, collapsing to a single color at maximum uncertainty. Their
study found it makes people weight uncertainty more heavily than an independent bivariate encoding
does.

Structurally important because a VSUP is **not** an interpolated 2-D LUT and not separable. It is
the clearest case for keeping a callable escape hatch in the API. Note that `pals` "strongly
discourages" its two VSUP palettes on brightness-confounding grounds, so shipping them needs the
criticism attached.

### 1.7 The separable / multichannel case

This is the use case closest to cmap's own ecosystem. Fluorescence microscopy composites are
exactly `MultivarColormap` with `sRGB_add`: napari's `blending='additive'` over per-channel
colormaps is the same operation, hand-rolled at the render layer. Anything consuming cmap for
multichannel image display (napari, ndv, pygfx) currently reimplements the combination itself
because there is no object that owns it.

### 1.8 Domain coloring

Complex-function visualization maps magnitude → lightness and phase → hue, which is a sequential ×
cyclic bivariate colormap under a different name.
[endolith/complex_colormap](https://github.com/endolith/complex_colormap) does this in a
perceptually uniform space. This is the pairing `BiBarrel` covers, and it is a real,
independently-motivated demand for the sequential × cyclic quadrant that the cartographic
literature never touches.

---

## Part 2: what the survey actually establishes

Three families, not one feature:

| Family | Structure | Separable? | Examples |
|---|---|---|---|
| **A. Separable N-variate** | N 1-D LUTs + a combination rule | yes | `2VarAddA`, napari additive channels |
| **B. Continuous 2-D surface** | `(N, M, 4)` LUT | no | `BiPeak`, `BiCone`, colorstamps, Bremm/Ziegler/Steiger |
| **C. Discrete 2-D grid** | small `(k, l, 4)` LUT, nearest | no | Stevens 3×3, ArcGIS, ColorBrewer pairs |

A and B need different classes. **C is not a separate class**: it is B with a small LUT and
`interpolation="nearest"`, which cmap already models for 1-D discrete colormaps. That collapse is
worth taking, and neither matplotlib nor `pals` takes it.

The literature also converges on a few substantive points worth encoding as catalog metadata
rather than as code: the six category pairings are the right taxonomy; CVD safety hinges on which
colorspace axis a design varies along; and per-palette criticism (the `arc.bluepink` white corner,
the VSUP brightness confound) is as valuable as the palette data itself.

---

## Part 3: constraints specific to cmap

1. **There is no norm layer.** `Colormap.__call__` takes values already in [0, 1]. matplotlib
   needed `MultiNorm` before any of this was usable; cmap needs nothing. A multivariate colormap in
   cmap is purely `(x0, ..., xn) -> RGBA`. This is a large simplification and it also settles the
   scope question below.
2. **`ColorStops` is a `Sequence[ColorStop]`.** It is 1-D by construction and cannot carry a 2-D
   LUT. This is the concrete reason `Colormap` should not be generalized.
3. **One polymorphic class per concept.** cmap already collapses matplotlib's
   `LinearSegmentedColormap` / `ListedColormap` into a single `Colormap` with a polymorphic
   `value` argument. It should do the same here: one `BivarColormap` where matplotlib has three.
4. **The catalog is `record.json` + a `"module:attr"` data pointer.** Adding namespaces is
   routine. `CatalogItem.category` is currently `Literal["sequential", "diverging", "cyclic",
   "qualitative", "miscellaneous"]` and does not express a pairing.
5. **Data size is a non-issue.** `src/cmap/data/` is already 7.3 MB with a single 840 KB file.
   Nineteen 65×65×3 LUTs is under 1 MB.
6. **No dependencies beyond numpy.** Rules out generating colorstamps maps at import time; ship
   precomputed LUTs. OKLab, by contrast, is a matrix plus a cube root and needs nothing.
7. **`_repr_png_` already accepts an `img` argument** and `_png.py` encodes an arbitrary RGB(A)
   array. The 2-D swatch is nearly free.
8. **Out-of-range handling is cmap's strongest existing feature** and is about to get stronger:
   `under`, `over`, `bad`, `neg_inf`, `pos_inf`, `nan`, `masked` (PRs #151/#155). matplotlib's
   bivariate class has only `bad` and `outside`.
9. **Only matplotlib ≥3.10 is a viable `to_*` target.** vispy, pygfx, napari, bokeh, plotly,
   altair have no bivariate colormap type. Keep the converter surface tiny.

---

## Part 4: proposed design

### 4.1 Classes

```python
class MultivarColormap:      # family A
    def __init__(self, colormaps, *, combination="add", name=None, ...): ...
    def __call__(self, x: tuple[ArrayLike, ...]) -> NDArray: ...
    def __getitem__(self, i: int) -> Colormap: ...
    def __len__(self) -> int: ...

class BivarColormap:         # families B and C
    def __init__(self, value, *, name=None, interpolation=None,
                 domain="square", outside=None, bad=None, origin=(0, 0), ...): ...
    def __call__(self, x: tuple[ArrayLike, ArrayLike]) -> NDArray | Color: ...
    def __getitem__(self, i: Literal[0, 1]) -> Colormap: ...
    def transposed(self) -> BivarColormap: ...
    def reversed(self, axis_0=True, axis_1=True) -> BivarColormap: ...
    @property
    def lut(self) -> NDArray: ...   # (N, M, 4)
```

`Colormap` is untouched. `Colormap("BiPeak")` raises with a message naming `BivarColormap`; a
constructor that silently returns a different type would break `isinstance` and the pydantic
integration.

### 4.2 `BivarColormap` accepts, in cmap's usual polymorphic style

| Input | Meaning |
|---|---|
| `str` | catalog name, `"BiPeak"` or `"colorstamps:cone"` |
| `(N, M, 3\|4)` array | direct LUT (matplotlib's `BivarColormapFromImage`) |
| small `(k, l, 3\|4)` array | patch, supersampled (matplotlib's `SegmentedBivarColormap`) |
| `(ColormapLike, ColormapLike)` | outer product of two 1-D colormaps under `combination=` |
| 4 colors | corner interpolation, i.e. a 2×2 patch (bivario's `CornersBivariateColourmap`) |
| `Callable[[NDArray, NDArray], NDArray]` | escape hatch, covers VSUP and domain coloring |

Distinguishing a full LUT from a patch by size is implicit and slightly magic; the alternative is
an explicit `supersample=` flag. Worth raising with the maintainer rather than deciding here.

### 4.3 Combination rules

matplotlib offers two. Given the survey, cmap should offer at minimum:

| Name | Formula | Provenance |
|---|---|---|
| `"add"` | `Σ cᵢ` | matplotlib `sRGB_add`; microscopy composites |
| `"subtract"` | `1 - Σ(1 - cᵢ)` | matplotlib `sRGB_sub` |
| `"multiply"` | `Π cᵢ` | Joshua Stevens' cartographic method |
| `"oklab"` | mean in OKLab | bivario; the perceptually correct one |

`"oklab"` costs a matrix pair and a cube root, no dependency. Names should be short and
cmap-native; `sRGB_add` reads as an implementation detail leaking into the API.

### 4.4 Domain geometry and out-of-domain policy: split them

matplotlib's `shape='square' | 'circle' | 'ignore' | 'circleignore'` is a cross product of two
independent concerns flattened into one string, which is why it has four values and needs
`circleignore`. Split it:

```python
domain: Literal["square", "disk"] = "square"   # geometry of the valid region
outside: ColorLike | None = None               # None => clip to the domain boundary
```

That is four states from two orthogonal arguments, it matches the way cmap already spells
out-of-range behavior for 1-D (`under_color=None` means "use the first color"), and it extends to
a third geometry without a combinatorial explosion of string values. Adding `outside` alongside
the existing `bad`/`nan`/`masked` machinery reuses `_with_exceptional_colors` rather than
duplicating it.

Note that `under`/`over` do **not** generalize: in 2-D there is no single "below." A single
`outside` color, plus `bad`/`nan`/`masked` inherited unchanged, is the right vocabulary.

### 4.5 Catalog schema

Add a `n_variates: int` field (default 1) to the record schema, and let `category` become a pair
for bivariate entries:

```json
"BiCone": {
  "n_variates": 2,
  "category": ["sequential", "cyclic"],
  "domain": "disk",
  "origin": [0.5, 0.5],
  "data": "cmap.data.colorstamps:BiCone",
  "interpolation": "linear"
}
```

`Catalog` stays one flat mapping. `Catalog.unique_keys()` gains an `n_variates` filter alongside
its existing `categories` and `interpolation` filters. Docs generation gets a bivariate template:
the 2-D swatch, the two `__getitem__` axis slices as 1-D swatches, CVD simulation of the 2-D
patch, and a lightness surface.

### 4.6 Catalog contents to target

| Namespace | Contents | License | Notes |
|---|---|---|---|
| `matplotlib` | the 3 shipped bivar + 3 multivar | matplotlib (BSD-style) | compatibility baseline |
| `colorstamps` | the other 16 designed maps | MIT | the actual value-add |
| `stevens` / `pals` | 3×3 cartographic grids | check per-palette | discrete, `interpolation="nearest"` |
| `steiger` | Bremm, Ziegler, Schumann, Teuling, CubeDiagonal | check | the survey's reference set |

Licensing needs checking before anything is copied. colorstamps and multivariate_colormaps are
both MIT and clean. matplotlib's license is BSD-style and cmap already carries PSF-2.0 in
`LICENSES/`. The cartographic palettes are the uncertain ones: `pals` itself is GPL-3, so the
palette values need sourcing from Stevens' and ArcGIS' own publications rather than from `pals`.

---

## Part 5: explicitly out of scope

- **Normalization and classification.** No `MultiNorm`, no natural-breaks, no quantile binning.
  cmap does not normalize 1-D data and should not start at 2-D. This is where `biscale`, `bivario`
  and `mapclassify` live, and where matplotlib needed the most machinery.
- **Legends and colorbars.** `_repr_png_` / `_repr_html_` yes, plotting-library legend widgets no.
- **Plotting integration.** cmap produces colors; it does not draw.
- **N-D non-separable LUTs.** Ræder's argument that 256³ is intractable stands. Family A covers
  N>2; family B stops at 2.
- **Generating colorstamps maps at runtime.** Precomputed LUTs only, to preserve the numpy-only
  dependency.

---

## Part 6: staging for upstream

The whole thing is far too large for one PR. In dependency order:

1. **Design comment on #90.** Present the three families, the two-class proposal, and the two
   deviations from matplotlib (`domain`/`outside` split; perceptual mixing). Ask for a signal
   before writing code. This is the only step that should happen without further discussion.
2. **`MultivarColormap` alone.** Composes existing `Colormap` objects, needs no new catalog data,
   no new LUT machinery, and serves the ecosystem's real driver (multichannel image compositing).
   Smallest useful increment and independently reviewable.
3. **`BivarColormap` core** plus the 3 matplotlib-compatible entries and `to_matplotlib()`.
4. **The colorstamps catalog**, the 16 unshipped designs. Data-only on top of step 3.
5. **Docs generation** for bivariate pages.
6. **Cartographic 3×3 grids**, if the maintainer wants them, once licensing is resolved.

Steps 2 and 3 are each still large by this project's standards. Expect the maintainer to want them
split further.

## Open questions

- Class names: match matplotlib (`BivarColormap` / `MultivarColormap`) or use cmap-native names
  (`Colormap2D` / `MultiColormap`)? Matching lowers friction across the ecosystem and for
  `to_matplotlib()`; deviating avoids implying matplotlib's exact semantics, which the
  `domain`/`outside` split does not preserve. Leaning toward matching.
- Full-LUT vs patch disambiguation: implicit by array size, or an explicit flag?
- Should `category` for a bivariate entry be a 2-tuple, or a new flat vocabulary
  (`"bisequential"`, `"seq_cyclic"`, ...)? A tuple composes with the existing 1-D vocabulary and
  needs no new terms.
- Does the maintainer want the cartographic 3×3 family at all, given that it is unusable without
  the classification step cmap declines to own?

## Next steps

Nothing is written until #90 gets a maintainer signal. Draft the design comment; do not post it.
