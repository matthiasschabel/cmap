Closes #144.

Adds four optional colors so exceptional float values can be told apart: `neg_inf`,
`pos_inf`, `nan`, and `masked`. Today `-inf` is indistinguishable from any other under-range
value, and NaN is indistinguishable from a masked entry. Log transformed signal data and
saturated logistic regression both produce infinities worth marking rather than blending
into the ends of the scale.

Each new color falls back to the one its class uses now, so nothing changes for an existing
colormap:

| Class | Color | Falls back to |
|---|---|---|
| negative infinity | `neg_inf` | `under`, then the first ramp color |
| positive infinity | `pos_inf` | `over`, then the last ramp color |
| NaN | `nan` | `bad`, then transparent |
| masked | `masked` | `bad`, then transparent |

`bad` stays the joint fallback for `nan` and `masked` rather than being replaced by them, so
code that sets it is unaffected and either child can be set alone. A masked entry takes the
masked color whatever value it hides, as it does now.

Two things the diff does not show:

- The infinity masks are taken before `xa *= N`. That multiply overflows large finite values
  to infinity (`float16` 65504 does at N=256), and those are out of range, not infinite.
  There is a test for it, because classifying after the multiply looks right and is not.
- The new colors do not round trip through pickle or `as_dict()`. Neither do
  `under`/`over`/`bad`, so this is an existing limitation rather than a new one. Happy to fix
  it separately if you want.

`to_napari` now prefers `nan_color` over `bad_color` for napari's `nan_color`, since it is
the one converter target that represents the class. matplotlib's `bad` covers NaN and masked
together, so `bad` is still what goes there.

Depends on #150, which this branches from. Only the last commit is mine; the first two are
#150's.
